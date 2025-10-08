# 连接器系统

## 连接器概述

连接器（Connector）是ThingsBoard IoT Gateway的核心组件，负责与各种协议的设备进行通信。每个连接器实现特定协议的数据采集和命令下发。

## 连接器基类

### Connector接口

```python
from abc import ABC, abstractmethod

class Connector(ABC):
    @abstractmethod
    def open(self):
        """打开连接"""
        pass
    
    @abstractmethod
    def close(self):
        """关闭连接"""
        pass
    
    @abstractmethod
    def get_id(self):
        """获取连接器ID"""
        pass
    
    @abstractmethod
    def get_name(self):
        """获取连接器名称"""
        pass
    
    @abstractmethod
    def get_type(self):
        """获取连接器类型"""
        pass
    
    @abstractmethod
    def get_config(self):
        """获取连接器配置"""
        pass
    
    @abstractmethod
    def is_connected(self):
        """检查连接状态"""
        pass
    
    @abstractmethod
    def is_stopped(self):
        """检查是否已停止"""
        pass
    
    @abstractmethod
    def on_attributes_update(self, content):
        """处理属性更新"""
        pass
    
    @abstractmethod
    def server_side_rpc_handler(self, content):
        """处理RPC请求"""
        pass
```

## 连接器实现模式

### 标准实现模板

```python
from thingsboard_gateway.connectors.connector import Connector
from threading import Thread
from queue import Queue

class MyConnector(Connector, Thread):
    def __init__(self, gateway, config, connector_type):
        super().__init__()
        self.__gateway = gateway
        self.__config = config
        self._connector_type = connector_type
        self.__id = config.get('id')
        self.name = config.get('name', 'MyConnector')
        
        # 初始化日志
        self.__log = init_logger(
            self.__gateway, 
            self.name,
            config.get('logLevel', 'INFO'),
            enable_remote_logging=config.get('enableRemoteLogging', False),
            is_connector_logger=True
        )
        
        # 状态标志
        self.__connected = False
        self.__stopped = False
        
        # 统计信息
        self.statistics = {
            'MessagesReceived': 0,
            'MessagesSent': 0
        }
        
        # 数据队列
        self.__data_queue = Queue()
        
        # 设置为守护线程
        self.daemon = True
    
    def run(self):
        """主循环"""
        while not self.__stopped:
            try:
                # 1. 连接设备
                self.__connect()
                
                # 2. 读取数据
                data = self.__read_data()
                
                # 3. 转换数据
                converted_data = self.__convert_data(data)
                
                # 4. 发送到网关
                if converted_data:
                    self.__gateway.send_to_storage(
                        self.name,
                        self.get_id(),
                        converted_data
                    )
                    self.statistics['MessagesSent'] += 1
                
            except Exception as e:
                self.__log.error("Error in connector: %s", e)
    
    def open(self):
        """打开连接"""
        self.__log.info("Opening connector")
        self.start()  # 启动线程
    
    def close(self):
        """关闭连接"""
        self.__log.info("Closing connector")
        self.__stopped = True
        self.__disconnect()
    
    def get_id(self):
        return self.__id
    
    def get_name(self):
        return self.name
    
    def get_type(self):
        return self._connector_type
    
    def get_config(self):
        return self.__config
    
    def is_connected(self):
        return self.__connected
    
    def is_stopped(self):
        return self.__stopped
    
    def on_attributes_update(self, content):
        """处理属性更新"""
        device_name = content['device']
        attributes = content['data']
        self.__log.debug("Attributes update for %s: %s", device_name, attributes)
        # 实现属性更新逻辑
    
    def server_side_rpc_handler(self, content):
        """处理RPC请求"""
        device_name = content['device']
        method = content['data']['method']
        params = content['data'].get('params', {})
        self.__log.debug("RPC request for %s: %s", device_name, method)
        # 实现RPC处理逻辑
        return {"success": True}
```

## 内置连接器详解

### 1. MQTT Connector

**功能**：
- 订阅外部MQTT Broker的主题
- 接收MQTT消息并转换为ThingsBoard格式
- 支持发布消息到MQTT Broker

**配置示例**：
```json
{
  "name": "MQTT Broker Connector",
  "type": "mqtt",
  "configuration": "mqtt.json"
}
```

**关键特性**：
- 支持MQTT v3.1, v3.1.1, v5
- 支持TLS/SSL
- 支持QoS 0, 1, 2
- 支持通配符订阅
- 支持动态设备映射

### 2. Modbus Connector

**功能**：
- 支持Modbus TCP和Modbus RTU
- 轮询读取寄存器数据
- 支持写入寄存器（RPC）

**配置示例**：
```json
{
  "name": "Modbus Connector",
  "type": "modbus",
  "configuration": "modbus.json"
}
```

**关键特性**：
- 异步实现（AsyncModbusConnector）
- 支持多个Modbus服务器
- 支持多个从站设备
- 支持功能码：1, 2, 3, 4, 5, 6, 15, 16
- 支持字节序配置
- 连接池管理

**实现要点**：
```python
class AsyncModbusConnector(Connector, Thread):
    def run(self):
        # 创建异步事件循环
        self.__loop = asyncio.new_event_loop()
        asyncio.set_event_loop(self.__loop)
        
        # 运行异步任务
        self.__loop.run_until_complete(self.__start())
    
    async def __start(self):
        # 并发执行多个任务
        await asyncio.gather(
            self.__poll_devices(),
            self.__process_rpc_requests(),
            self.__send_data()
        )
```

### 3. OPC-UA Connector

**功能**：
- 连接OPC-UA服务器
- 订阅节点变化
- 读取/写入节点值

**配置示例**：
```json
{
  "name": "OPC-UA Connector",
  "type": "opcua",
  "configuration": "opcua.json"
}
```

**关键特性**：
- 支持订阅和轮询两种模式
- 支持证书认证
- 支持浏览节点树
- 支持方法调用

### 4. BACnet Connector

**功能**：
- 连接BACnet设备
- 读取BACnet对象属性
- 支持BACnet/IP

**关键特性**：
- 异步实现
- 支持设备发现
- 支持多种对象类型
- 支持EDE文件导入

### 5. REST Connector

**功能**：
- 创建REST API端点
- 接收HTTP请求
- 将请求数据转换为ThingsBoard格式

**关键特性**：
- 支持HTTP/HTTPS
- 支持多种HTTP方法
- 支持SSL证书
- 支持认证

### 6. Request Connector

**功能**：
- 定期发送HTTP请求
- 从REST API拉取数据
- 支持各种HTTP方法

**关键特性**：
- 定时轮询
- 支持认证
- 支持请求模板

### 7. Socket Connector

**功能**：
- TCP/UDP Socket通信
- 支持客户端和服务器模式

**关键特性**：
- 支持TCP和UDP
- 支持多连接
- 支持自定义协议

### 8. SNMP Connector

**功能**：
- SNMP设备监控
- 读取MIB数据
- 支持SNMP v1, v2c, v3

### 9. BLE Connector

**功能**：
- 蓝牙低功耗设备连接
- 扫描BLE设备
- 读取/写入特征值

### 10. CAN Connector

**功能**：
- CAN总线通信
- 支持SocketCAN

### 11. FTP Connector

**功能**：
- 从FTP/SFTP服务器读取文件
- 支持CSV、JSON、TXT文件

### 12. KNX Connector

**功能**：
- KNX建筑自动化协议
- 读取/写入组地址

### 13. OCPP Connector

**功能**：
- 电动汽车充电桩协议
- 支持OCPP 1.6和2.0

### 14. ODBC Connector

**功能**：
- 从数据库读取数据
- 支持SQL查询

### 15. XMPP Connector

**功能**：
- XMPP协议通信
- 接收消息

## 连接器生命周期

```
1. 创建 (Creation)
   ↓
2. 配置 (Configuration)
   - 加载配置文件
   - 初始化转换器
   ↓
3. 打开 (Open)
   - 建立连接
   - 启动线程
   ↓
4. 运行 (Running)
   - 数据采集
   - 数据转换
   - 数据发送
   ↓
5. 关闭 (Close)
   - 断开连接
   - 清理资源
   - 停止线程
```

## 数据采集模式

### 1. 轮询模式 (Polling)
```python
def run(self):
    while not self.__stopped:
        # 读取数据
        data = self.__read_data()
        
        # 处理数据
        self.__process_data(data)
        
        # 等待下一次轮询
        sleep(self.__poll_period)
```

### 2. 订阅模式 (Subscription)
```python
def run(self):
    # 订阅数据变化
    self.__subscribe_to_changes()
    
    # 等待回调
    while not self.__stopped:
        sleep(1)

def __on_data_change(self, data):
    # 数据变化回调
    self.__process_data(data)
```

### 3. 事件驱动模式 (Event-Driven)
```python
def run(self):
    while not self.__stopped:
        # 从队列获取事件
        event = self.__event_queue.get()
        
        # 处理事件
        self.__process_event(event)
```

## 连接器配置

### 通用配置项

```json
{
  "name": "Connector Name",
  "type": "connector_type",
  "id": "unique_id",
  "logLevel": "INFO",
  "enableRemoteLogging": false,
  "configuration": "connector_config.json"
}
```

### 设备配置模式

```json
{
  "devices": [
    {
      "deviceName": "Device1",
      "deviceType": "Sensor",
      "attributes": [...],
      "timeseries": [...],
      "attributeUpdates": [...],
      "serverSideRpc": [...]
    }
  ]
}
```

## 错误处理

### 连接错误
```python
def __connect(self):
    retry_count = 0
    max_retries = 5
    
    while retry_count < max_retries and not self.__stopped:
        try:
            # 尝试连接
            self.__client.connect()
            self.__connected = True
            break
        except Exception as e:
            retry_count += 1
            self.__log.error("Connection failed: %s, retry %d/%d", 
                           e, retry_count, max_retries)
            sleep(5)
```

### 数据处理错误
```python
def __process_data(self, data):
    try:
        converted_data = self.__converter.convert(config, data)
        self.__gateway.send_to_storage(self.name, self.get_id(), converted_data)
    except Exception as e:
        self.__log.error("Error processing data: %s", e)
        # 不中断主循环
```

## 性能优化

### 1. 批量处理
```python
def __send_batch(self):
    batch = []
    while len(batch) < self.__batch_size:
        try:
            data = self.__queue.get_nowait()
            batch.append(data)
        except Empty:
            break
    
    if batch:
        self.__gateway.send_to_storage(self.name, self.get_id(), batch)
```

### 2. 连接池
```python
class ConnectionPool:
    def __init__(self, size):
        self.__pool = Queue(maxsize=size)
        for _ in range(size):
            self.__pool.put(self.__create_connection())
    
    def get_connection(self):
        return self.__pool.get()
    
    def return_connection(self, conn):
        self.__pool.put(conn)
```

## 下一步

- 阅读 [数据转换器](05-converters.md) 了解数据转换
- 阅读 [开发指南](08-development-guide.md) 学习开发自定义连接器

