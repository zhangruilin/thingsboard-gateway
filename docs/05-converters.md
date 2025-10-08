# 数据转换器

## 转换器概述

转换器（Converter）负责在设备原始数据格式和ThingsBoard平台格式之间进行转换。转换器分为两类：

- **Uplink Converter（上行转换器）**: 设备数据 → ThingsBoard格式
- **Downlink Converter（下行转换器）**: ThingsBoard命令 → 设备格式

## 转换器基类

### Converter接口

```python
from abc import ABC, abstractmethod
from typing import Union
from thingsboard_gateway.gateway.entities.converted_data import ConvertedData

class Converter(ABC):
    @abstractmethod
    def convert(self, config, data) -> Union[dict, ConvertedData]:
        """
        转换数据
        
        Args:
            config: 转换配置
            data: 原始数据
            
        Returns:
            转换后的数据（ConvertedData对象或字典）
        """
        pass
```

## ConvertedData 数据结构

### 核心类

```python
class ConvertedData:
    def __init__(self, device_name: str, device_type: str):
        self.device_name = device_name
        self.device_type = device_type
        self.__telemetry = []      # 遥测数据列表
        self.__attributes = {}     # 属性数据字典
        self.report_strategy = None
    
    def add_to_telemetry(self, telemetry_entry: Union[TelemetryEntry, dict]):
        """添加遥测数据"""
        if isinstance(telemetry_entry, dict):
            telemetry_entry = TelemetryEntry(telemetry_entry)
        self.__telemetry.append(telemetry_entry)
    
    def add_to_attributes(self, key_or_dict, value=None):
        """添加属性数据"""
        if isinstance(key_or_dict, dict):
            self.__attributes.update(key_or_dict)
        else:
            self.__attributes[key_or_dict] = value
    
    @property
    def telemetry(self):
        return self.__telemetry
    
    @property
    def attributes(self):
        return self.__attributes
```

### TelemetryEntry

```python
class TelemetryEntry:
    def __init__(self, values: dict, ts: int = None):
        self.values = values
        self.ts = ts or int(time() * 1000)
    
    def to_dict(self):
        return {
            "ts": self.ts,
            "values": self.values
        }
```

## 上行转换器（Uplink Converter）

### 基本实现

```python
from thingsboard_gateway.connectors.converter import Converter
from thingsboard_gateway.gateway.entities.converted_data import ConvertedData, TelemetryEntry

class MyUplinkConverter(Converter):
    def __init__(self, config, logger):
        self.__config = config
        self._log = logger
    
    def convert(self, config, data):
        # 1. 创建ConvertedData对象
        device_name = self.__extract_device_name(data)
        device_type = self.__extract_device_type(data)
        
        converted_data = ConvertedData(
            device_name=device_name,
            device_type=device_type
        )
        
        # 2. 提取并转换遥测数据
        telemetry = self.__extract_telemetry(data)
        for key, value in telemetry.items():
            converted_data.add_to_telemetry({key: value})
        
        # 3. 提取并转换属性数据
        attributes = self.__extract_attributes(data)
        converted_data.add_to_attributes(attributes)
        
        return converted_data
    
    def __extract_device_name(self, data):
        # 从数据中提取设备名称
        return data.get('deviceName', 'Unknown')
    
    def __extract_device_type(self, data):
        # 从数据中提取设备类型
        return data.get('deviceType', 'default')
    
    def __extract_telemetry(self, data):
        # 提取遥测数据
        return {
            'temperature': data.get('temp'),
            'humidity': data.get('hum')
        }
    
    def __extract_attributes(self, data):
        # 提取属性数据
        return {
            'model': data.get('model'),
            'version': data.get('version')
        }
```

### JSON上行转换器示例

```python
class JsonMqttUplinkConverter(MqttUplinkConverter):
    def __init__(self, config, logger):
        self.__config = config
        self._log = logger
    
    def convert(self, topic, body):
        try:
            # 解析JSON数据
            data = loads(body)
            
            # 提取设备信息
            device_name = self.__parse_device_name(topic, data)
            device_type = self.__parse_device_type(data)
            
            converted_data = ConvertedData(device_name, device_type)
            
            # 处理遥测数据
            for ts_config in self.__config.get('timeseries', []):
                key = self.__parse_key(ts_config, data)
                value = self.__parse_value(ts_config, data)
                timestamp = self.__parse_timestamp(ts_config, data)
                
                converted_data.add_to_telemetry(
                    TelemetryEntry({key: value}, ts=timestamp)
                )
            
            # 处理属性数据
            for attr_config in self.__config.get('attributes', []):
                key = self.__parse_key(attr_config, data)
                value = self.__parse_value(attr_config, data)
                converted_data.add_to_attributes(key, value)
            
            return converted_data
            
        except Exception as e:
            self._log.error("Conversion error: %s", e)
            return None
    
    def __parse_key(self, config, data):
        # 使用JSONPath提取键
        key_expression = config.get('key')
        return self.__evaluate_expression(key_expression, data)
    
    def __parse_value(self, config, data):
        # 使用JSONPath提取值
        value_expression = config.get('value')
        value = self.__evaluate_expression(value_expression, data)
        
        # 类型转换
        value_type = config.get('type', 'string')
        return self.__convert_type(value, value_type)
    
    def __evaluate_expression(self, expression, data):
        # 使用JSONPath或简单表达式
        if expression.startswith('${') and expression.endswith('}'):
            # JSONPath表达式
            path = expression[2:-1]
            return jsonpath_rw.parse(path).find(data)[0].value
        else:
            # 直接键访问
            return data.get(expression)
    
    def __convert_type(self, value, value_type):
        # 类型转换
        if value_type == 'int':
            return int(value)
        elif value_type == 'double':
            return float(value)
        elif value_type == 'bool':
            return bool(value)
        else:
            return str(value)
```

### 字节上行转换器示例

```python
class BytesMqttUplinkConverter(MqttUplinkConverter):
    def __init__(self, config, logger):
        self.__config = config
        self._log = logger
    
    def convert(self, topic, body):
        # 将字节数据转换为十六进制字符串
        hex_data = body.hex()
        
        device_name = self.__config['deviceName']
        device_type = self.__config['deviceType']
        
        converted_data = ConvertedData(device_name, device_type)
        
        # 解析字节数据
        for ts_config in self.__config.get('timeseries', []):
            start = ts_config.get('byteFrom')
            end = ts_config.get('byteTo')
            
            # 提取字节片段
            value_bytes = body[start:end]
            
            # 转换为数值
            if ts_config.get('type') == 'int':
                value = int.from_bytes(value_bytes, byteorder='big')
            elif ts_config.get('type') == 'float':
                value = struct.unpack('>f', value_bytes)[0]
            else:
                value = value_bytes.hex()
            
            key = ts_config['key']
            converted_data.add_to_telemetry({key: value})
        
        return converted_data
```

## 下行转换器（Downlink Converter）

### 基本实现

```python
class MyDownlinkConverter(Converter):
    def __init__(self, config, logger):
        self.__config = config
        self._log = logger
    
    def convert(self, config, data):
        """
        转换ThingsBoard命令为设备格式
        
        Args:
            config: 设备配置
            data: ThingsBoard数据
                {
                    'device': 'DeviceName',
                    'data': {
                        'method': 'setValue',
                        'params': {'value': 100}
                    }
                }
        
        Returns:
            设备可理解的命令格式
        """
        method = data['data']['method']
        params = data['data'].get('params', {})
        
        # 根据方法名转换
        if method == 'setValue':
            return self.__convert_set_value(params)
        elif method == 'getValues':
            return self.__convert_get_values(params)
        else:
            self._log.warning("Unknown method: %s", method)
            return None
    
    def __convert_set_value(self, params):
        # 转换为设备命令格式
        return {
            'cmd': 'SET',
            'value': params.get('value')
        }
    
    def __convert_get_values(self, params):
        return {
            'cmd': 'GET'
        }
```

### JSON下行转换器示例

```python
class JsonRestDownlinkConverter(Converter):
    def __init__(self, config, logger):
        self.__config = config
        self._log = logger
    
    def convert(self, config, data):
        # 属性更新
        if 'id' not in data['data']:
            attribute_key = list(data['data'].keys())[0]
            attribute_value = list(data['data'].values())[0]
            
            # 构建URL
            url = self.__config['requestUrlExpression'] \
                .replace('${attributeKey}', attribute_key) \
                .replace('${attributeValue}', str(attribute_value)) \
                .replace('${deviceName}', data['device'])
            
            # 构建请求体
            body = self.__config['valueExpression'] \
                .replace('${attributeKey}', attribute_key) \
                .replace('${attributeValue}', str(attribute_value))
            
            return {
                'url': url,
                'data': body
            }
        
        # RPC请求
        else:
            method = data['data']['method']
            params = data['data'].get('params', {})
            
            url = self.__config['rpcUrlExpression'] \
                .replace('${methodName}', method) \
                .replace('${deviceName}', data['device'])
            
            return {
                'url': url,
                'data': dumps(params)
            }
```

## 转换器配置

### 上行转换器配置示例

```json
{
  "converter": "JsonMqttUplinkConverter",
  "deviceInfo": {
    "deviceNameExpression": "${serialNumber}",
    "deviceTypeExpression": "${deviceType}"
  },
  "timeseries": [
    {
      "key": "temperature",
      "value": "${temp}",
      "type": "double"
    },
    {
      "key": "humidity",
      "value": "${hum}",
      "type": "double"
    }
  ],
  "attributes": [
    {
      "key": "model",
      "value": "${model}",
      "type": "string"
    }
  ]
}
```

### 下行转换器配置示例

```json
{
  "converter": "JsonRestDownlinkConverter",
  "requestUrlExpression": "http://device/${deviceName}/api/${attributeKey}",
  "valueExpression": "{\"${attributeKey}\": \"${attributeValue}\"}"
}
```

## 高级特性

### 1. 报告策略（Report Strategy）

```python
from thingsboard_gateway.gateway.entities.report_strategy_config import ReportStrategyConfig

# 在转换器中使用报告策略
device_report_strategy = ReportStrategyConfig(config.get('reportStrategy'))

# 创建数据点键
datapoint_key = TBUtility.convert_key_to_datapoint_key(
    key,
    device_report_strategy,
    config,
    logger
)
```

### 2. 时间戳处理

```python
def __parse_timestamp(self, config, data):
    # 从配置获取时间戳表达式
    ts_expression = config.get('ts')
    
    if ts_expression:
        ts_value = self.__evaluate_expression(ts_expression, data)
        
        # 处理不同时间戳格式
        if isinstance(ts_value, str):
            # ISO 8601格式
            dt = datetime.fromisoformat(ts_value)
            return int(dt.timestamp() * 1000)
        elif isinstance(ts_value, int):
            # Unix时间戳（秒或毫秒）
            if ts_value > 10000000000:  # 毫秒
                return ts_value
            else:  # 秒
                return ts_value * 1000
    
    # 默认使用当前时间
    return int(time() * 1000)
```

### 3. 数据验证

```python
def __validate_data(self, data):
    # 验证必需字段
    required_fields = ['deviceName', 'temperature']
    for field in required_fields:
        if field not in data:
            raise ValueError(f"Missing required field: {field}")
    
    # 验证数据范围
    temp = data['temperature']
    if not -50 <= temp <= 150:
        raise ValueError(f"Temperature out of range: {temp}")
    
    return True
```

### 4. 统计装饰器

```python
from thingsboard_gateway.gateway.statistics.decorators import CollectStatistics

class MyConverter(Converter):
    @CollectStatistics(
        start_stat_type='receivedBytesFromDevices',
        end_stat_type='convertedBytesFromDevice'
    )
    def convert(self, config, data):
        # 自动统计转换的字节数
        return converted_data
```

## 自定义转换器开发

### 步骤1: 创建转换器类

```python
# extensions/mqtt/custom_mqtt_uplink_converter.py
from thingsboard_gateway.connectors.mqtt.mqtt_uplink_converter import MqttUplinkConverter
from thingsboard_gateway.gateway.entities.converted_data import ConvertedData

class CustomMqttUplinkConverter(MqttUplinkConverter):
    def __init__(self, config, logger):
        self.__config = config
        self._log = logger
    
    def convert(self, topic, body):
        # 自定义转换逻辑
        pass
```

### 步骤2: 配置使用自定义转换器

```json
{
  "topicFilter": "sensor/+/data",
  "converter": {
    "type": "custom",
    "extension": "CustomMqttUplinkConverter",
    "extension-config": {
      "temperatureMultiplier": 0.1
    }
  }
}
```

## 最佳实践

1. **错误处理**: 始终捕获异常，避免转换失败导致连接器崩溃
2. **日志记录**: 记录转换过程中的关键信息
3. **性能优化**: 避免在转换器中执行耗时操作
4. **数据验证**: 验证输入数据的有效性
5. **类型转换**: 正确处理数据类型转换
6. **时间戳**: 正确处理时间戳格式

## 下一步

- 阅读 [存储系统](06-storage.md) 了解数据存储
- 阅读 [开发指南](08-development-guide.md) 学习开发自定义转换器

