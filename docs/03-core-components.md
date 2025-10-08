# 核心组件

## 1. TBGatewayService (网关核心服务)

### 概述
`TBGatewayService` 是整个网关的核心服务类，负责协调所有组件的工作。

### 主要职责

#### 1.1 初始化流程
```python
def __init__(self, config_file=None):
    # 1. 初始化变量
    self.__init_variables()
    
    # 2. 加载配置
    self.__config = self.__load_general_config(config_file)
    
    # 3. 初始化日志
    logging.config.dictConfig(log_config)
    
    # 4. 创建ThingsBoard客户端
    self.tb_client = TBClient(self.__config["thingsboard"], ...)
    
    # 5. 初始化存储
    self.__init_storage()
    
    # 6. 初始化统计服务
    StatisticsService.initialize(...)
    
    # 7. 加载连接器
    self.load_connectors()
    
    # 8. 启动gRPC服务（如果启用）
    self.init_grpc_service(...)
```

#### 1.2 连接器管理

**加载连接器**：
```python
def load_connectors(self):
    # 从配置文件读取连接器列表
    for connector_config in self.__config.get('connectors', []):
        connector_type = connector_config['type']
        connector_class = self.__load_connector_class(connector_type)
        
        # 创建连接器实例
        connector = connector_class(self, connector_config, connector_type)
        
        # 保存连接器引用
        self.__connectors.append(connector)
```

**启动连接器**：
```python
def connect_with_connectors(self):
    for connector in self.__connectors:
        try:
            connector.open()
            connector.start()  # 启动线程
        except Exception as e:
            log.error("Failed to start connector: %s", e)
```

**停止连接器**：
```python
def __close_connectors(self):
    for connector in self.__connectors:
        try:
            connector.close()
            connector.join(timeout=10)
        except Exception as e:
            log.error("Error closing connector: %s", e)
```

#### 1.3 设备管理

**添加设备**：
```python
def add_device(self, device_name, content, device_type=None):
    # 1. 检查设备是否已存在
    if device_name in self.__connected_devices:
        return
    
    # 2. 应用设备过滤器
    if not self.__device_filter.filter_device(device_name, device_type):
        return
    
    # 3. 注册设备到ThingsBoard
    self.tb_client.gw_connect_device(device_name, device_type)
    
    # 4. 保存设备信息
    self.__connected_devices[device_name] = {
        'connector': content.get('connector'),
        'device_type': device_type,
        'ts': time()
    }
    
    # 5. 持久化设备列表
    self.__save_persistent_devices()
```

**删除设备**：
```python
def del_device(self, device_name):
    # 1. 从本地列表移除
    if device_name in self.__connected_devices:
        del self.__connected_devices[device_name]
    
    # 2. 通知ThingsBoard
    self.tb_client.gw_disconnect_device(device_name)
    
    # 3. 更新持久化文件
    self.__save_persistent_devices()
```

#### 1.4 数据处理

**接收连接器数据**：
```python
@CollectStorageEventsStatistics
def send_to_storage(self, connector_name, connector_id, data: ConvertedData):
    # 1. 验证数据
    if not data or not data.device_name:
        return
    
    # 2. 添加设备（如果不存在）
    if data.device_name not in self.__connected_devices:
        self.add_device(data.device_name, {...}, data.device_type)
    
    # 3. 应用报告策略
    if data.report_strategy:
        data = self.__report_strategy_service.apply(data)
    
    # 4. 存储数据
    self.__storage.put(data)
```

**发送数据到ThingsBoard**：
```python
def __read_data_from_storage(self):
    while not self.stopped:
        try:
            # 1. 从存储读取数据
            events = self.__storage.get_event_pack()
            
            # 2. 发送遥测数据
            for event in events:
                if event.telemetry:
                    self.tb_client.gw_send_telemetry(
                        event.device_name, 
                        event.telemetry
                    )
                
                # 3. 发送属性数据
                if event.attributes:
                    self.tb_client.gw_send_attributes(
                        event.device_name,
                        event.attributes
                    )
            
            # 4. 确认发送成功
            self.__storage.event_pack_processing_done()
            
        except Exception as e:
            log.error("Error sending data: %s", e)
```

### 关键属性

```python
class TBGatewayService:
    # 配置
    __config: dict                          # 主配置
    _config_dir: str                        # 配置目录
    
    # 核心组件
    tb_client: TBClient                     # ThingsBoard客户端
    __storage: EventStorage                 # 数据存储
    __connectors: List[Connector]           # 连接器列表
    
    # 设备管理
    __connected_devices: dict               # 已连接设备
    __device_filter: DeviceFilter           # 设备过滤器
    
    # 服务组件
    __statistics_service: StatisticsService # 统计服务
    __report_strategy_service: ReportStrategyService  # 报告策略
    remote_configurator: RemoteConfigurator # 远程配置
    remote_shell: RemoteShell               # 远程Shell
    
    # 状态
    stopped: bool                           # 停止标志
    __lock: RLock                          # 线程锁
```

## 2. TBClient (ThingsBoard客户端)

### 概述
`TBClient` 负责与ThingsBoard平台的MQTT通信。

### 主要功能

#### 2.1 连接管理

**建立连接**：
```python
def connect(self):
    # 1. 配置MQTT客户端
    self.client = TBGatewayMqttClient(self.__host, self.__port)
    
    # 2. 设置回调
    self.client.on_connect = self._on_connect
    self.client.on_disconnect = self._on_disconnect
    self.client.on_message = self._on_message
    
    # 3. 配置TLS（如果需要）
    if self.__ca_cert:
        self.client.tls_set(
            ca_certs=self.__ca_cert,
            certfile=self.__cert,
            keyfile=self.__private_key
        )
    
    # 4. 连接
    self.client.connect()
    self.client.loop_start()
```

**自动重连**：
```python
def _on_disconnect(self, client, userdata, rc):
    self.__is_connected = False
    self.__logger.warning("Disconnected from ThingsBoard")
    
    # 自动重连
    while not self.__stopped and not self.__is_connected:
        try:
            self.client.reconnect()
            sleep(self.__min_reconnect_delay)
        except Exception as e:
            self.__logger.error("Reconnection failed: %s", e)
```

#### 2.2 数据发送

**发送遥测数据**：
```python
def gw_send_telemetry(self, device_name, telemetry):
    payload = {
        device_name: [{
            "ts": entry.ts,
            "values": entry.values
        } for entry in telemetry]
    }
    
    self.client.publish(
        "v1/gateway/telemetry",
        dumps(payload),
        qos=self.__default_quality_of_service
    )
```

**发送属性**：
```python
def gw_send_attributes(self, device_name, attributes):
    payload = {
        device_name: attributes
    }
    
    self.client.publish(
        "v1/gateway/attributes",
        dumps(payload),
        qos=self.__default_quality_of_service
    )
```

#### 2.3 设备管理

**连接设备**：
```python
def gw_connect_device(self, device_name, device_type="default"):
    payload = {
        "device": device_name,
        "type": device_type
    }
    
    self.client.publish(
        "v1/gateway/connect",
        dumps(payload)
    )
```

**断开设备**：
```python
def gw_disconnect_device(self, device_name):
    payload = {
        "device": device_name
    }
    
    self.client.publish(
        "v1/gateway/disconnect",
        dumps(payload)
    )
```

#### 2.4 订阅和回调

**订阅主题**：
```python
def subscribe_to_required_topics(self):
    # RPC请求
    self.client.subscribe("v1/gateway/rpc")
    self.client.gw_set_server_side_rpc_request_handler(
        self._on_rpc_request
    )
    
    # 属性更新
    self.client.subscribe("v1/gateway/attributes")
    self.client.gw_set_attributes_request_handler(
        self._on_attributes_update
    )
```

## 3. Storage (存储系统)

### 存储类型

#### 3.1 MemoryEventStorage (内存存储)
- 快速但不持久
- 适合测试环境
- 网关重启数据丢失

#### 3.2 FileEventStorage (文件存储)
- 数据持久化到文件
- 支持大容量存储
- 读写性能中等

#### 3.3 SQLiteEventStorage (SQLite存储)
- 推荐的生产环境存储
- 数据持久化
- 支持事务
- 性能优秀

### 存储接口

```python
class EventStorage(ABC):
    @abstractmethod
    def put(self, data: ConvertedData):
        """存储数据"""
        pass
    
    @abstractmethod
    def get_event_pack(self) -> List[DeviceEventPack]:
        """获取待发送数据包"""
        pass
    
    @abstractmethod
    def event_pack_processing_done(self):
        """确认数据包已处理"""
        pass
```

## 4. Statistics Service (统计服务)

### 功能
- 消息计数统计
- 字节数统计
- 连接器性能监控
- 定期上报统计数据

### 使用装饰器统计

```python
@CollectAllReceivedBytesStatistics(start_stat_type='allReceivedBytesFromTB')
def on_attributes_update(self, content):
    # 自动统计接收字节数
    pass

@CollectAllSentTBBytesStatistics
def send_telemetry(self, data):
    # 自动统计发送字节数
    pass
```

## 5. Remote Management (远程管理)

### 5.1 RemoteConfigurator (远程配置)
- 从ThingsBoard接收配置更新
- 验证配置有效性
- 应用新配置
- 支持热更新

### 5.2 RemoteShell (远程Shell)
- 执行Shell命令
- 返回执行结果
- 安全限制

### 5.3 RPC Methods (RPC方法)
- 网关服务级别的RPC
- 连接器管理
- 设备管理
- 配置查询

## 下一步

- 阅读 [连接器系统](04-connectors.md) 了解连接器实现
- 阅读 [数据转换器](05-converters.md) 了解数据转换
- 阅读 [存储系统](06-storage.md) 了解存储详情

