# API参考

## 核心类API

### TBGatewayService

网关核心服务类，负责整体协调和管理。

#### 构造函数

```python
def __init__(self, config_file: str = None)
```

**参数**：
- `config_file`: 配置文件路径，默认为 `config/tb_gateway.json`

#### 主要方法

##### send_to_storage

```python
def send_to_storage(
    self,
    connector_name: str,
    connector_id: str,
    data: ConvertedData
) -> None
```

将转换后的数据发送到存储系统。

**参数**：
- `connector_name`: 连接器名称
- `connector_id`: 连接器ID
- `data`: 转换后的数据对象

##### add_device

```python
def add_device(
    self,
    device_name: str,
    content: dict,
    device_type: str = None
) -> None
```

添加新设备到网关。

**参数**：
- `device_name`: 设备名称
- `content`: 设备内容（包含连接器信息等）
- `device_type`: 设备类型，默认为 "default"

##### update_device

```python
def update_device(
    self,
    device_name: str,
    event: str,
    content: dict
) -> None
```

更新设备信息。

**参数**：
- `device_name`: 设备名称
- `event`: 事件类型
- `content`: 更新内容

##### del_device

```python
def del_device(self, device_name: str) -> None
```

删除设备。

**参数**：
- `device_name`: 设备名称

### TBClient

ThingsBoard MQTT客户端类。

#### 构造函数

```python
def __init__(
    self,
    config: dict,
    config_folder_path: str,
    logger: logging.Logger
)
```

**参数**：
- `config`: ThingsBoard连接配置
- `config_folder_path`: 配置文件夹路径
- `logger`: 日志记录器

#### 主要方法

##### connect

```python
def connect(self) -> None
```

连接到ThingsBoard平台。

##### disconnect

```python
def disconnect(self) -> None
```

断开与ThingsBoard的连接。

##### gw_connect_device

```python
def gw_connect_device(
    self,
    device_name: str,
    device_type: str = "default"
) -> None
```

通知ThingsBoard设备已连接。

**参数**：
- `device_name`: 设备名称
- `device_type`: 设备类型

##### gw_disconnect_device

```python
def gw_disconnect_device(self, device_name: str) -> None
```

通知ThingsBoard设备已断开。

**参数**：
- `device_name`: 设备名称

##### gw_send_telemetry

```python
def gw_send_telemetry(
    self,
    device_name: str,
    telemetry: List[TelemetryEntry]
) -> None
```

发送遥测数据到ThingsBoard。

**参数**：
- `device_name`: 设备名称
- `telemetry`: 遥测数据列表

##### gw_send_attributes

```python
def gw_send_attributes(
    self,
    device_name: str,
    attributes: dict
) -> None
```

发送属性数据到ThingsBoard。

**参数**：
- `device_name`: 设备名称
- `attributes`: 属性字典

### Connector

连接器基类，所有连接器必须实现此接口。

#### 抽象方法

##### open

```python
@abstractmethod
def open(self) -> None
```

打开连接器，建立连接。

##### close

```python
@abstractmethod
def close(self) -> None
```

关闭连接器，释放资源。

##### get_id

```python
@abstractmethod
def get_id(self) -> str
```

获取连接器ID。

**返回**：连接器ID字符串

##### get_name

```python
@abstractmethod
def get_name(self) -> str
```

获取连接器名称。

**返回**：连接器名称字符串

##### get_type

```python
@abstractmethod
def get_type(self) -> str
```

获取连接器类型。

**返回**：连接器类型字符串

##### get_config

```python
@abstractmethod
def get_config(self) -> dict
```

获取连接器配置。

**返回**：配置字典

##### is_connected

```python
@abstractmethod
def is_connected(self) -> bool
```

检查连接状态。

**返回**：True表示已连接，False表示未连接

##### is_stopped

```python
@abstractmethod
def is_stopped(self) -> bool
```

检查是否已停止。

**返回**：True表示已停止，False表示运行中

##### on_attributes_update

```python
@abstractmethod
def on_attributes_update(self, content: dict) -> None
```

处理属性更新请求。

**参数**：
- `content`: 属性更新内容
  ```python
  {
      'device': 'DeviceName',
      'data': {
          'attribute1': 'value1',
          'attribute2': 'value2'
      }
  }
  ```

##### server_side_rpc_handler

```python
@abstractmethod
def server_side_rpc_handler(self, content: dict) -> dict
```

处理服务器端RPC请求。

**参数**：
- `content`: RPC请求内容
  ```python
  {
      'device': 'DeviceName',
      'data': {
          'id': 1,
          'method': 'methodName',
          'params': {...}
      }
  }
  ```

**返回**：RPC响应字典

### Converter

转换器基类。

#### 抽象方法

##### convert

```python
@abstractmethod
def convert(
    self,
    config: dict,
    data: Any
) -> Union[dict, ConvertedData]
```

转换数据格式。

**参数**：
- `config`: 转换配置
- `data`: 原始数据

**返回**：转换后的数据（ConvertedData对象或字典）

## 数据实体API

### ConvertedData

转换后的数据容器类。

#### 构造函数

```python
def __init__(self, device_name: str, device_type: str)
```

**参数**：
- `device_name`: 设备名称
- `device_type`: 设备类型

#### 属性

- `device_name`: 设备名称（只读）
- `device_type`: 设备类型（只读）
- `telemetry`: 遥测数据列表（只读）
- `attributes`: 属性数据字典（只读）
- `telemetry_datapoints_count`: 遥测数据点数量
- `attributes_datapoints_count`: 属性数据点数量

#### 方法

##### add_to_telemetry

```python
def add_to_telemetry(
    self,
    telemetry_entry: Union[TelemetryEntry, dict]
) -> None
```

添加遥测数据。

**参数**：
- `telemetry_entry`: TelemetryEntry对象或字典

##### add_to_attributes

```python
def add_to_attributes(
    self,
    key_or_dict: Union[str, dict],
    value: Any = None
) -> None
```

添加属性数据。

**参数**：
- `key_or_dict`: 属性键或属性字典
- `value`: 属性值（当第一个参数为键时）

**示例**：
```python
# 方式1: 添加单个属性
converted_data.add_to_attributes('temperature', 25.5)

# 方式2: 添加多个属性
converted_data.add_to_attributes({
    'temperature': 25.5,
    'humidity': 60
})
```

### TelemetryEntry

遥测数据条目类。

#### 构造函数

```python
def __init__(self, values: dict, ts: int = None)
```

**参数**：
- `values`: 遥测值字典
- `ts`: 时间戳（毫秒），默认为当前时间

#### 属性

- `values`: 遥测值字典
- `ts`: 时间戳（毫秒）

#### 方法

##### to_dict

```python
def to_dict(self) -> dict
```

转换为字典格式。

**返回**：
```python
{
    'ts': 1234567890000,
    'values': {
        'temperature': 25.5,
        'humidity': 60
    }
}
```

### DatapointKey

数据点键类，支持报告策略。

#### 构造函数

```python
def __init__(
    self,
    key: str,
    report_strategy: ReportStrategyConfig = None
)
```

**参数**：
- `key`: 数据点键名
- `report_strategy`: 报告策略配置

## 工具类API

### TBUtility

工具类，提供各种辅助功能。

#### 静态方法

##### install_package

```python
@staticmethod
def install_package(
    package: str,
    version: str = None,
    force_install: bool = False
) -> None
```

安装Python包。

**参数**：
- `package`: 包名
- `version`: 版本号
- `force_install`: 是否强制安装

##### convert_key_to_datapoint_key

```python
@staticmethod
def convert_key_to_datapoint_key(
    key: str,
    report_strategy: ReportStrategyConfig,
    config: dict,
    logger: logging.Logger
) -> Union[str, DatapointKey]
```

转换键为数据点键。

**参数**：
- `key`: 原始键
- `report_strategy`: 报告策略
- `config`: 配置
- `logger`: 日志记录器

**返回**：字符串键或DatapointKey对象

##### convert_data_type

```python
@staticmethod
def convert_data_type(
    value: Any,
    data_type: str,
    use_eval: bool = False
) -> Any
```

转换数据类型。

**参数**：
- `value`: 原始值
- `data_type`: 目标类型（'int', 'double', 'bool', 'string'）
- `use_eval`: 是否使用eval

**返回**：转换后的值

### TBModuleLoader

模块加载器，用于动态加载连接器和转换器。

#### 静态方法

##### import_module

```python
@staticmethod
def import_module(
    connector_type: str,
    module_name: str
) -> type
```

导入模块类。

**参数**：
- `connector_type`: 连接器类型
- `module_name`: 模块名称

**返回**：模块类

**示例**：
```python
converter_class = TBModuleLoader.import_module(
    'mqtt',
    'JsonMqttUplinkConverter'
)
converter = converter_class(config, logger)
```

## 统计服务API

### StatisticsService

统计服务类。

#### 静态方法

##### count_connector_message

```python
@staticmethod
def count_connector_message(
    connector_name: str,
    stat_parameter_name: str = 'messagesReceived'
) -> None
```

统计连接器消息数量。

**参数**：
- `connector_name`: 连接器名称
- `stat_parameter_name`: 统计参数名称

## 装饰器API

### CollectStatistics

统计装饰器，用于自动收集统计信息。

```python
@CollectStatistics(
    start_stat_type='receivedBytesFromDevices',
    end_stat_type='convertedBytesFromDevice'
)
def convert(self, config, data):
    # 自动统计字节数
    pass
```

**参数**：
- `start_stat_type`: 开始统计类型
- `end_stat_type`: 结束统计类型

### CollectAllReceivedBytesStatistics

接收字节统计装饰器。

```python
@CollectAllReceivedBytesStatistics(
    start_stat_type='allReceivedBytesFromTB'
)
def on_attributes_update(self, content):
    # 自动统计从ThingsBoard接收的字节数
    pass
```

### CollectAllSentTBBytesStatistics

发送字节统计装饰器。

```python
@CollectAllSentTBBytesStatistics
def send_telemetry(self, data):
    # 自动统计发送到ThingsBoard的字节数
    pass
```

## 日志API

### init_logger

初始化日志记录器。

```python
def init_logger(
    gateway,
    name: str,
    level: str = 'INFO',
    enable_remote_logging: bool = False,
    is_connector_logger: bool = False,
    is_converter_logger: bool = False,
    attr_name: str = None
) -> logging.Logger
```

**参数**：
- `gateway`: 网关实例
- `name`: 日志记录器名称
- `level`: 日志级别
- `enable_remote_logging`: 是否启用远程日志
- `is_connector_logger`: 是否为连接器日志
- `is_converter_logger`: 是否为转换器日志
- `attr_name`: 属性名称

**返回**：日志记录器实例

## 常量

### 连接器类型

```python
DEFAULT_CONNECTORS = {
    "mqtt": "MqttConnector",
    "modbus": "AsyncModbusConnector",
    "opcua": "OpcUaConnector",
    "ble": "BLEConnector",
    "request": "RequestConnector",
    "can": "CanConnector",
    "bacnet": "AsyncBACnetConnector",
    "odbc": "OdbcConnector",
    "rest": "RESTConnector",
    "snmp": "SNMPConnector",
    "ftp": "FTPConnector",
    "socket": "SocketConnector",
    "xmpp": "XMPPConnector",
    "ocpp": "OcppConnector",
    "knx": "KNXConnector",
}
```

### 统计参数

```python
STATISTIC_MESSAGE_RECEIVED_PARAMETER = 'MessagesReceived'
STATISTIC_MESSAGE_SENT_PARAMETER = 'MessagesSent'
```

## 下一步

- 查看源代码获取更多细节
- 阅读 [开发指南](08-development-guide.md) 学习如何使用这些API
- 参考官方示例代码

