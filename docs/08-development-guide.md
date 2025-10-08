# 开发指南

## 开发环境搭建

### 1. 克隆项目

```bash
git clone https://github.com/thingsboard/thingsboard-gateway.git
cd thingsboard-gateway
```

### 2. 创建虚拟环境

```bash
# 使用venv
python3 -m venv venv
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate     # Windows

# 或使用conda
conda create -n tb-gateway python=3.10
conda activate tb-gateway
```

### 3. 安装依赖

```bash
# 安装基础依赖
pip install -r requirements.txt

# 安装完整依赖（包括所有连接器）
pip install -r requirements-full.txt

# 开发模式安装
pip install -e .
```

### 4. 配置开发环境

```bash
# 设置环境变量
export TB_GW_DEV_MODE=true
export TB_GW_CONFIG_DIR=./thingsboard_gateway/config/

# 启用调试服务器
export TB_GW_DEV_DEBUG_SERVER=5678
```

### 5. 运行网关

```bash
# 直接运行
python -m thingsboard_gateway.tb_gateway

# 或使用热重载
python -m thingsboard_gateway.tb_gateway true
```

## 开发自定义连接器

### 步骤1: 创建连接器类

在 `thingsboard_gateway/connectors/` 目录下创建新的连接器目录：

```bash
mkdir thingsboard_gateway/connectors/myconnector
touch thingsboard_gateway/connectors/myconnector/__init__.py
touch thingsboard_gateway/connectors/myconnector/myconnector_connector.py
```

### 步骤2: 实现连接器

```python
# myconnector_connector.py
from thingsboard_gateway.connectors.connector import Connector
from thingsboard_gateway.gateway.entities.converted_data import ConvertedData
from thingsboard_gateway.tb_utility.tb_logger import init_logger
from threading import Thread
from time import sleep, time

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
        
        # 状态
        self.__connected = False
        self.__stopped = False
        
        # 统计
        self.statistics = {
            'MessagesReceived': 0,
            'MessagesSent': 0
        }
        
        # 设置为守护线程
        self.daemon = True
        
        self.__log.info("MyConnector initialized")
    
    def run(self):
        """主循环"""
        self.__log.info("MyConnector started")
        
        while not self.__stopped:
            try:
                # 1. 连接到数据源
                if not self.__connected:
                    self.__connect()
                
                # 2. 读取数据
                data = self.__read_data()
                
                if data:
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
                        self.__log.debug("Data sent: %s", converted_data)
                
                # 5. 等待下一次轮询
                sleep(self.__config.get('pollPeriod', 5))
                
            except Exception as e:
                self.__log.error("Error in main loop: %s", e)
                self.__connected = False
                sleep(5)
    
    def __connect(self):
        """连接到数据源"""
        try:
            self.__log.info("Connecting to data source...")
            # 实现连接逻辑
            # self.__client = MyClient(...)
            self.__connected = True
            self.__log.info("Connected successfully")
        except Exception as e:
            self.__log.error("Connection failed: %s", e)
            raise
    
    def __read_data(self):
        """读取数据"""
        try:
            # 实现数据读取逻辑
            # data = self.__client.read()
            data = {
                'deviceId': 'device1',
                'temperature': 25.5,
                'humidity': 60
            }
            self.statistics['MessagesReceived'] += 1
            return data
        except Exception as e:
            self.__log.error("Error reading data: %s", e)
            return None
    
    def __convert_data(self, data):
        """转换数据"""
        try:
            device_name = data.get('deviceId', 'Unknown')
            
            converted_data = ConvertedData(
                device_name=device_name,
                device_type='Sensor'
            )
            
            # 添加遥测数据
            converted_data.add_to_telemetry({
                'temperature': data.get('temperature'),
                'humidity': data.get('humidity')
            })
            
            # 添加属性
            converted_data.add_to_attributes({
                'lastUpdate': int(time() * 1000)
            })
            
            return converted_data
            
        except Exception as e:
            self.__log.error("Error converting data: %s", e)
            return None
    
    def open(self):
        """打开连接器"""
        self.__log.info("Opening connector")
        self.start()
    
    def close(self):
        """关闭连接器"""
        self.__log.info("Closing connector")
        self.__stopped = True
        if hasattr(self, '__client'):
            # 关闭客户端连接
            # self.__client.close()
            pass
    
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
        try:
            device_name = content['device']
            attributes = content['data']
            
            self.__log.info("Attributes update for %s: %s", device_name, attributes)
            
            # 实现属性更新逻辑
            # self.__client.write_attributes(device_name, attributes)
            
        except Exception as e:
            self.__log.error("Error handling attributes update: %s", e)
    
    def server_side_rpc_handler(self, content):
        """处理RPC请求"""
        try:
            device_name = content['device']
            method = content['data']['method']
            params = content['data'].get('params', {})
            
            self.__log.info("RPC request for %s: %s(%s)", device_name, method, params)
            
            # 实现RPC处理逻辑
            if method == 'getValue':
                result = {'value': 123}
            elif method == 'setValue':
                # self.__client.set_value(params['value'])
                result = {'success': True}
            else:
                result = {'error': 'Unknown method'}
            
            return result
            
        except Exception as e:
            self.__log.error("Error handling RPC: %s", e)
            return {'error': str(e)}
```

### 步骤3: 注册连接器

在 `thingsboard_gateway/gateway/constants.py` 中注册连接器：

```python
DEFAULT_CONNECTORS = {
    "mqtt": "MqttConnector",
    "modbus": "AsyncModbusConnector",
    "opcua": "OpcUaConnector",
    # ... 其他连接器
    "myconnector": "MyConnector",  # 添加自定义连接器
}
```

### 步骤4: 创建配置文件

创建 `thingsboard_gateway/config/myconnector.json`:

```json
{
  "name": "My Custom Connector",
  "pollPeriod": 5,
  "logLevel": "INFO",
  "enableRemoteLogging": false,
  "devices": [
    {
      "deviceId": "device1",
      "deviceName": "My Device 1",
      "deviceType": "Sensor"
    }
  ]
}
```

### 步骤5: 在主配置中启用

在 `tb_gateway.json` 中添加：

```json
{
  "connectors": [
    {
      "name": "My Custom Connector",
      "type": "myconnector",
      "configuration": "myconnector.json"
    }
  ]
}
```

## 开发自定义转换器

### 上行转换器

```python
# extensions/myconnector/my_uplink_converter.py
from thingsboard_gateway.connectors.converter import Converter
from thingsboard_gateway.gateway.entities.converted_data import ConvertedData

class MyUplinkConverter(Converter):
    def __init__(self, config, logger):
        self.__config = config
        self._log = logger
    
    def convert(self, config, data):
        """
        转换设备数据为ThingsBoard格式
        
        Args:
            config: 转换配置
            data: 原始数据
            
        Returns:
            ConvertedData对象
        """
        try:
            # 提取设备信息
            device_name = self.__extract_device_name(data)
            device_type = self.__extract_device_type(data)
            
            converted_data = ConvertedData(device_name, device_type)
            
            # 转换遥测数据
            for ts_config in self.__config.get('timeseries', []):
                key = ts_config['key']
                value = self.__extract_value(data, ts_config['path'])
                converted_data.add_to_telemetry({key: value})
            
            # 转换属性数据
            for attr_config in self.__config.get('attributes', []):
                key = attr_config['key']
                value = self.__extract_value(data, attr_config['path'])
                converted_data.add_to_attributes(key, value)
            
            return converted_data
            
        except Exception as e:
            self._log.error("Conversion error: %s", e)
            return None
    
    def __extract_device_name(self, data):
        # 实现设备名称提取逻辑
        return data.get('deviceId', 'Unknown')
    
    def __extract_device_type(self, data):
        # 实现设备类型提取逻辑
        return data.get('deviceType', 'default')
    
    def __extract_value(self, data, path):
        # 实现值提取逻辑（支持JSONPath等）
        keys = path.split('.')
        value = data
        for key in keys:
            value = value.get(key)
        return value
```

### 下行转换器

```python
# extensions/myconnector/my_downlink_converter.py
from thingsboard_gateway.connectors.converter import Converter

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
            
        Returns:
            设备命令
        """
        try:
            method = data['data']['method']
            params = data['data'].get('params', {})
            
            # 根据方法转换
            if method == 'setValue':
                return self.__convert_set_value(params)
            elif method == 'getValue':
                return self.__convert_get_value(params)
            else:
                self._log.warning("Unknown method: %s", method)
                return None
                
        except Exception as e:
            self._log.error("Conversion error: %s", e)
            return None
    
    def __convert_set_value(self, params):
        # 转换setValue命令
        return {
            'command': 'SET',
            'parameter': params.get('parameter'),
            'value': params.get('value')
        }
    
    def __convert_get_value(self, params):
        # 转换getValue命令
        return {
            'command': 'GET',
            'parameter': params.get('parameter')
        }
```

## 单元测试

### 测试连接器

```python
# tests/unit/connectors/test_myconnector.py
import unittest
from unittest.mock import Mock, patch
from thingsboard_gateway.connectors.myconnector.myconnector_connector import MyConnector

class TestMyConnector(unittest.TestCase):
    def setUp(self):
        self.gateway = Mock()
        self.config = {
            'name': 'Test Connector',
            'pollPeriod': 1
        }
        self.connector = MyConnector(self.gateway, self.config, 'myconnector')
    
    def test_initialization(self):
        self.assertEqual(self.connector.get_name(), 'Test Connector')
        self.assertEqual(self.connector.get_type(), 'myconnector')
    
    def test_data_conversion(self):
        data = {
            'deviceId': 'device1',
            'temperature': 25.5
        }
        converted = self.connector._MyConnector__convert_data(data)
        
        self.assertEqual(converted.device_name, 'device1')
        self.assertIn('temperature', converted.telemetry[0].values)
    
    def tearDown(self):
        self.connector.close()

if __name__ == '__main__':
    unittest.main()
```

### 测试转换器

```python
# tests/unit/converters/test_my_uplink_converter.py
import unittest
from extensions.myconnector.my_uplink_converter import MyUplinkConverter

class TestMyUplinkConverter(unittest.TestCase):
    def setUp(self):
        self.config = {
            'timeseries': [
                {'key': 'temp', 'path': 'temperature'}
            ]
        }
        self.logger = Mock()
        self.converter = MyUplinkConverter(self.config, self.logger)
    
    def test_convert(self):
        data = {'temperature': 25.5}
        result = self.converter.convert(self.config, data)
        
        self.assertIsNotNone(result)
        self.assertEqual(result.telemetry[0].values['temp'], 25.5)

if __name__ == '__main__':
    unittest.main()
```

## 调试技巧

### 1. 启用详细日志

```json
{
  "logLevel": "DEBUG"
}
```

### 2. 使用调试器

```python
# 在代码中设置断点
import debugpy
debugpy.listen(("0.0.0.0", 5678))
debugpy.wait_for_client()
```

### 3. 打印调试信息

```python
self.__log.debug("Data: %s", data)
self.__log.info("Processing device: %s", device_name)
```

### 4. 使用PyCharm调试

1. 设置断点
2. 配置运行配置
3. 启动调试会话

## 代码规范

### 1. 命名规范

- 类名: `PascalCase`
- 函数名: `snake_case`
- 常量: `UPPER_CASE`
- 私有成员: `__private`

### 2. 文档字符串

```python
def convert(self, config, data):
    """
    转换数据格式
    
    Args:
        config (dict): 转换配置
        data (dict): 原始数据
        
    Returns:
        ConvertedData: 转换后的数据
        
    Raises:
        ValueError: 数据格式错误
    """
    pass
```

### 3. 类型提示

```python
from typing import Dict, List, Optional

def process_data(self, data: Dict) -> Optional[ConvertedData]:
    pass
```

## 贡献代码

### 1. Fork项目

### 2. 创建分支

```bash
git checkout -b feature/my-connector
```

### 3. 提交代码

```bash
git add .
git commit -m "Add MyConnector"
git push origin feature/my-connector
```

### 4. 创建Pull Request

## 下一步

- 阅读 [API参考](09-api-reference.md) 了解API详情
- 查看官方示例连接器代码
- 参与社区讨论

