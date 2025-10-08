# 架构设计

## 整体架构

ThingsBoard IoT Gateway 采用模块化、微服务风格的架构设计，主要由以下几个层次组成：

```
┌─────────────────────────────────────────────────────────────┐
│                    ThingsBoard Platform                      │
└─────────────────────────────────────────────────────────────┘
                            ↑ MQTT
                            │
┌─────────────────────────────────────────────────────────────┐
│                    Gateway Core Layer                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  TB Client   │  │   Storage    │  │  Statistics  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         TBGatewayService (Core Service)              │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ↑
                            │
┌─────────────────────────────────────────────────────────────┐
│                  Connector Layer                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  MQTT    │  │  Modbus  │  │  OPC-UA  │  │  BACnet  │   │
│  │Connector │  │Connector │  │Connector │  │Connector │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   REST   │  │  Socket  │  │   SNMP   │  │   ...    │   │
│  │Connector │  │Connector │  │Connector │  │          │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
                            ↑
                            │
┌─────────────────────────────────────────────────────────────┐
│                  Converter Layer                             │
│  ┌──────────────────┐         ┌──────────────────┐         │
│  │ Uplink Converter │         │Downlink Converter│         │
│  │  (Device → TB)   │         │   (TB → Device)  │         │
│  └──────────────────┘         └──────────────────┘         │
└─────────────────────────────────────────────────────────────┘
                            ↑
                            │
┌─────────────────────────────────────────────────────────────┐
│                    Device Layer                              │
│  [Sensors] [PLCs] [Meters] [Controllers] [IoT Devices]     │
└─────────────────────────────────────────────────────────────┘
```

## 核心组件

### 1. TBGatewayService (网关核心服务)

**职责**：
- 网关生命周期管理
- 连接器管理（加载、启动、停止）
- 设备注册和管理
- 数据路由和分发
- 远程配置管理

**关键方法**：
```python
class TBGatewayService:
    def __init__(self, config_file=None)
    def load_connectors()
    def connect_with_connectors()
    def send_to_storage(connector_name, connector_id, data)
    def add_device(device_name, content, device_type=None)
    def update_device(device_name, event, content)
    def del_device(device_name)
```

**工作流程**：
1. 加载主配置文件 `tb_gateway.json`
2. 初始化日志系统
3. 创建ThingsBoard MQTT客户端
4. 初始化存储系统
5. 加载并启动所有配置的连接器
6. 订阅ThingsBoard平台的RPC和属性更新主题
7. 进入主循环，处理数据和事件

### 2. TBClient (ThingsBoard客户端)

**职责**：
- 与ThingsBoard平台建立MQTT连接
- 发送遥测数据和属性
- 接收RPC请求和属性更新
- 处理连接断开和重连

**关键特性**：
- 支持MQTT v3.1, v3.1.1, v5
- 支持TLS/SSL加密
- 支持代理服务器
- 自动重连机制
- QoS配置

### 3. Connector (连接器基类)

**接口定义**：
```python
class Connector(ABC):
    @abstractmethod
    def open(self): pass
    
    @abstractmethod
    def close(self): pass
    
    @abstractmethod
    def get_name(self): pass
    
    @abstractmethod
    def get_type(self): pass
    
    @abstractmethod
    def is_connected(self): pass
    
    @abstractmethod
    def on_attributes_update(self, content): pass
    
    @abstractmethod
    def server_side_rpc_handler(self, content): pass
```

**实现模式**：
- 每个连接器继承自 `Connector` 和 `Thread`
- 独立线程运行，互不干扰
- 通过队列与网关服务通信

### 4. Converter (转换器基类)

**接口定义**：
```python
class Converter(ABC):
    @abstractmethod
    def convert(self, config, data) -> Union[dict, ConvertedData]:
        pass
```

**转换器类型**：
- **Uplink Converter**: 设备数据 → ThingsBoard格式
- **Downlink Converter**: ThingsBoard命令 → 设备格式

### 5. Storage (存储系统)

**存储类型**：
- **MemoryEventStorage**: 内存存储（快速但不持久）
- **FileEventStorage**: 文件存储（持久化）
- **SQLiteEventStorage**: SQLite数据库存储（推荐）

**职责**：
- 缓存待发送的数据
- 离线数据持久化
- 网络恢复后自动重传

## 数据流详解

### 上行数据流（设备 → ThingsBoard）

```
1. 设备产生数据
   ↓
2. Connector接收原始数据
   - MQTT: 订阅主题接收消息
   - Modbus: 轮询寄存器读取数据
   - OPC-UA: 订阅节点变化
   ↓
3. Uplink Converter转换数据
   - 解析原始数据
   - 提取设备名称、类型
   - 转换为ConvertedData对象
   - 包含telemetry和attributes
   ↓
4. Connector调用gateway.send_to_storage()
   - 将ConvertedData发送到网关
   ↓
5. Gateway存储数据
   - 写入Storage（内存/文件/SQLite）
   - 统计消息数量
   ↓
6. Storage定期批量发送
   - 从存储读取数据
   - 通过TBClient发送到ThingsBoard
   ↓
7. TBClient通过MQTT发送
   - 发送到v1/gateway/telemetry主题
   - 发送到v1/gateway/attributes主题
   ↓
8. ThingsBoard平台接收并处理
```

### 下行数据流（ThingsBoard → 设备）

```
1. ThingsBoard发送RPC请求或属性更新
   ↓
2. TBClient接收MQTT消息
   - RPC: v1/gateway/rpc主题
   - 属性: v1/gateway/attributes主题
   ↓
3. Gateway路由到对应Connector
   - 根据设备名称查找Connector
   - 调用on_attributes_update()或server_side_rpc_handler()
   ↓
4. Downlink Converter转换命令
   - 将ThingsBoard格式转换为设备协议格式
   ↓
5. Connector发送到设备
   - MQTT: 发布到设备主题
   - Modbus: 写入寄存器
   - OPC-UA: 写入节点
   ↓
6. 设备执行命令并返回结果
   ↓
7. Connector接收响应
   ↓
8. 通过上行数据流返回结果
```

## 线程模型

```
Main Thread
├── TBGatewayService (主服务)
├── TBClient (MQTT客户端线程)
├── Storage Thread (存储处理线程)
├── Statistics Thread (统计服务线程)
├── Connector Thread 1 (MQTT连接器)
├── Connector Thread 2 (Modbus连接器)
├── Connector Thread 3 (OPC-UA连接器)
├── ...
└── Async Device Actions Thread (异步设备操作)
```

**线程安全**：
- 使用 `RLock` 保护共享资源
- 使用 `Queue` 进行线程间通信
- 使用 `Event` 进行线程同步

## 配置管理

### 配置文件层次

```
/etc/thingsboard-gateway/config/
├── tb_gateway.json          # 主配置
├── logs.json                # 日志配置
├── mqtt.json                # MQTT连接器配置
├── modbus.json              # Modbus连接器配置
├── opcua.json               # OPC-UA连接器配置
└── ...                      # 其他连接器配置
```

### 配置加载流程

1. 加载 `tb_gateway.json` 主配置
2. 根据 `connectors` 列表加载各连接器配置
3. 支持YAML和JSON两种格式
4. 支持环境变量覆盖配置

### 远程配置

- 通过ThingsBoard平台远程更新配置
- 使用 `RemoteConfigurator` 组件
- 支持热更新（无需重启）

## 扩展机制

### 1. 自定义连接器

```python
from thingsboard_gateway.connectors.connector import Connector
from threading import Thread

class CustomConnector(Connector, Thread):
    def __init__(self, gateway, config, connector_type):
        super().__init__()
        self.__gateway = gateway
        # 初始化...
    
    def run(self):
        # 主循环
        pass
    
    def open(self):
        # 打开连接
        pass
    
    def close(self):
        # 关闭连接
        pass
```

### 2. 自定义转换器

```python
from thingsboard_gateway.connectors.converter import Converter
from thingsboard_gateway.gateway.entities.converted_data import ConvertedData

class CustomUplinkConverter(Converter):
    def convert(self, config, data):
        converted_data = ConvertedData(
            device_name="MyDevice",
            device_type="Sensor"
        )
        # 转换逻辑...
        return converted_data
```

### 3. 动态加载

使用 `TBModuleLoader` 动态加载自定义模块：

```python
from thingsboard_gateway.tb_utility.tb_loader import TBModuleLoader

converter = TBModuleLoader.import_module(
    "mqtt", 
    "CustomMqttUplinkConverter"
)(config, logger)
```

## 性能优化

### 1. 异步处理
- 使用 `asyncio` 处理I/O密集型操作
- Modbus、OPC-UA等连接器使用异步实现

### 2. 批量发送
- Storage批量读取和发送数据
- 减少MQTT消息数量

### 3. 数据压缩
- 支持大数据分片发送
- 避免单个消息过大

### 4. 连接池
- Modbus连接器使用连接池
- 复用TCP连接

## 容错机制

### 1. 自动重连
- TBClient自动重连ThingsBoard
- Connector自动重连设备

### 2. 数据持久化
- 离线数据存储到本地
- 网络恢复后自动上传

### 3. 异常处理
- 每个Connector独立异常处理
- 单个Connector异常不影响其他

### 4. 健康检查
- 定期检查连接状态
- 统计服务监控性能指标

## 安全机制

### 1. TLS/SSL
- 支持MQTT over TLS
- 支持证书认证

### 2. 访问控制
- ThingsBoard平台认证
- 设备访问令牌

### 3. 数据加密
- 传输层加密
- 支持代理服务器

## 下一步

- 阅读 [核心组件](03-core-components.md) 了解各组件详细实现
- 阅读 [连接器系统](04-connectors.md) 了解连接器开发
- 阅读 [数据转换器](05-converters.md) 了解数据转换机制

