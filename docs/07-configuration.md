# 配置管理

## 配置文件概述

ThingsBoard IoT Gateway使用JSON或YAML格式的配置文件。主要配置文件位于：
- Linux: `/etc/thingsboard-gateway/config/`
- Windows: `C:\Program Files\ThingsBoard Gateway\config\`
- 开发环境: `项目根目录/config/`

## 主配置文件 (tb_gateway.json)

### 基本结构

```json
{
  "thingsboard": {
    "host": "localhost",
    "port": 1883,
    "remoteShell": false,
    "remoteConfiguration": true,
    "statistics": {
      "enable": true,
      "statsSendPeriodInSeconds": 60
    },
    "deviceFiltering": {
      "enable": false
    },
    "maxPayloadSizeBytes": 1024,
    "minPackSendDelayMS": 200,
    "minPackSizeToSend": 50,
    "checkConnectorsConfigurationInSeconds": 60,
    "handleDeviceRenaming": true,
    "checkingDeviceActivity": {
      "checkDeviceInactivity": false
    },
    "security": {
      "type": "accessToken",
      "accessToken": "YOUR_ACCESS_TOKEN"
    },
    "qos": 1
  },
  "storage": {
    "type": "sqlite",
    "data_file_path": "./data/storage.db",
    "max_records_count": 100000,
    "read_records_count": 100
  },
  "grpc": {
    "enabled": false,
    "serverPort": 9595
  },
  "connectors": [
    {
      "name": "MQTT Broker Connector",
      "type": "mqtt",
      "configuration": "mqtt.json"
    },
    {
      "name": "Modbus Connector",
      "type": "modbus",
      "configuration": "modbus.json"
    }
  ]
}
```

### ThingsBoard连接配置

#### 访问令牌认证

```json
{
  "thingsboard": {
    "host": "thingsboard.cloud",
    "port": 1883,
    "security": {
      "type": "accessToken",
      "accessToken": "YOUR_GATEWAY_ACCESS_TOKEN"
    }
  }
}
```

#### TLS/SSL认证

```json
{
  "thingsboard": {
    "host": "thingsboard.cloud",
    "port": 8883,
    "security": {
      "type": "tls",
      "caCert": "/path/to/ca.pem",
      "cert": "/path/to/client.pem",
      "key": "/path/to/client.key"
    }
  }
}
```

#### 设备自动配置

```json
{
  "thingsboard": {
    "provisioning": {
      "type": "DEVICE_PROFILE_NAME",
      "host": "thingsboard.cloud",
      "port": 1883,
      "deviceName": "Gateway Device Name",
      "provisionDeviceKey": "PROVISION_KEY",
      "provisionDeviceSecret": "PROVISION_SECRET"
    }
  }
}
```

### 存储配置

#### SQLite存储（推荐）

```json
{
  "storage": {
    "type": "sqlite",
    "data_file_path": "/var/lib/thingsboard-gateway/storage.db",
    "max_records_count": 1000000,
    "read_records_count": 500
  }
}
```

#### 文件存储

```json
{
  "storage": {
    "type": "file",
    "data_folder_path": "/var/lib/thingsboard-gateway/data/",
    "max_file_count": 10,
    "max_records_per_file": 10000,
    "max_read_records_count": 100
  }
}
```

#### 内存存储

```json
{
  "storage": {
    "type": "memory",
    "read_records_count": 100,
    "max_records_count": 100000
  }
}
```

### 统计配置

```json
{
  "thingsboard": {
    "statistics": {
      "enable": true,
      "statsSendPeriodInSeconds": 60,
      "customStatsSendPeriodInSeconds": 3600,
      "commands": [
        {
          "attributeOnGateway": "cpuUsage",
          "command": "top -b -n 1 | grep 'Cpu(s)' | awk '{print $2}'"
        }
      ]
    }
  }
}
```

### 设备过滤配置

```json
{
  "thingsboard": {
    "deviceFiltering": {
      "enable": true,
      "filterFile": "device_filter.json"
    }
  }
}
```

### gRPC配置

```json
{
  "grpc": {
    "enabled": true,
    "serverPort": 9595,
    "keepaliveTimeMs": 10000,
    "keepaliveTimeoutMs": 5000,
    "keepalivePermitWithoutCalls": true,
    "maxPingsWithoutData": 0,
    "minTimeBetweenPingsMs": 10000,
    "minPingIntervalWithoutDataMs": 5000
  }
}
```

## 连接器配置

### MQTT连接器配置 (mqtt.json)

```json
{
  "broker": {
    "name": "Default Local Broker",
    "host": "localhost",
    "port": 1883,
    "clientId": "ThingsBoard_gateway",
    "security": {
      "type": "anonymous"
    }
  },
  "mapping": [
    {
      "topicFilter": "sensor/+/temperature",
      "converter": {
        "type": "json",
        "deviceNameJsonExpression": "${serialNumber}",
        "deviceTypeJsonExpression": "Sensor",
        "timeout": 60000,
        "attributes": [
          {
            "key": "model",
            "type": "string",
            "value": "${model}"
          }
        ],
        "timeseries": [
          {
            "key": "temperature",
            "type": "double",
            "value": "${temp}"
          }
        ]
      }
    }
  ],
  "connectRequests": [
    {
      "topicFilter": "sensor/connect",
      "deviceNameJsonExpression": "${serialNumber}"
    }
  ],
  "disconnectRequests": [
    {
      "topicFilter": "sensor/disconnect",
      "deviceNameJsonExpression": "${serialNumber}"
    }
  ],
  "attributeRequests": [
    {
      "retain": false,
      "topicFilter": "sensor/+/request/+",
      "deviceNameJsonExpression": "${serialNumber}",
      "attributeNameJsonExpression": "${attribute}",
      "topicExpression": "sensor/${deviceName}/response/${requestId}",
      "valueExpression": "${value}"
    }
  ],
  "attributeUpdates": [
    {
      "retain": false,
      "deviceNameFilter": ".*",
      "attributeFilter": ".*",
      "topicExpression": "sensor/${deviceName}/attribute/update",
      "valueExpression": "{\"${attributeKey}\":\"${attributeValue}\"}"
    }
  ],
  "serverSideRpc": [
    {
      "deviceNameFilter": ".*",
      "methodFilter": ".*",
      "requestTopicExpression": "sensor/${deviceName}/rpc/request/${methodName}/${requestId}",
      "responseTopicExpression": "sensor/${deviceName}/rpc/response/${methodName}/${requestId}",
      "responseTimeout": 10000,
      "valueExpression": "${params}"
    }
  ]
}
```

### Modbus连接器配置 (modbus.json)

```json
{
  "server": {
    "name": "Modbus Server",
    "type": "tcp",
    "host": "192.168.1.100",
    "port": 502,
    "timeout": 35,
    "method": "socket",
    "byteOrder": "LITTLE",
    "wordOrder": "LITTLE"
  },
  "devices": [
    {
      "unitId": 1,
      "deviceName": "Temperature Sensor",
      "deviceType": "Sensor",
      "pollPeriod": 5000,
      "sendDataOnlyOnChange": false,
      "connectAttemptTimeMs": 5000,
      "connectAttemptCount": 5,
      "waitAfterFailedAttemptsMs": 300000,
      "attributes": [
        {
          "tag": "model",
          "type": "string",
          "functionCode": 3,
          "objectsCount": 8,
          "address": 0
        }
      ],
      "timeseries": [
        {
          "tag": "temperature",
          "type": "16int",
          "functionCode": 4,
          "objectsCount": 1,
          "address": 0,
          "multiplier": 0.1
        }
      ],
      "attributeUpdates": [
        {
          "tag": "setpoint",
          "type": "16int",
          "functionCode": 6,
          "objectsCount": 1,
          "address": 10
        }
      ],
      "rpc": [
        {
          "tag": "reset",
          "type": "bits",
          "functionCode": 5,
          "objectsCount": 1,
          "address": 20
        }
      ]
    }
  ]
}
```

### OPC-UA连接器配置 (opcua.json)

```json
{
  "server": {
    "name": "OPC-UA Server",
    "url": "opc.tcp://localhost:4840",
    "timeoutInMillis": 5000,
    "scanPeriodInMillis": 5000,
    "disableSubscriptions": false,
    "subCheckPeriodInMillis": 100,
    "showMap": false,
    "security": "Basic128Rsa15",
    "identity": {
      "type": "anonymous"
    }
  },
  "mapping": [
    {
      "deviceNodePattern": "Root\\.Objects\\.Device1",
      "deviceNamePattern": "Device ${Root\\.Objects\\.Device1\\.serialNumber}",
      "deviceTypePattern": "Sensor",
      "attributes": [
        {
          "key": "model",
          "path": "${ns=2;i=3}"
        }
      ],
      "timeseries": [
        {
          "key": "temperature",
          "path": "${ns=2;i=4}"
        }
      ],
      "rpc_methods": [
        {
          "method": "reset",
          "arguments": []
        }
      ],
      "attributes_updates": [
        {
          "attributeOnThingsBoard": "setpoint",
          "attributeOnDevice": "${ns=2;i=5}"
        }
      ]
    }
  ]
}
```

## 日志配置 (logs.json)

```json
{
  "version": 1,
  "disable_existing_loggers": false,
  "formatters": {
    "LogFormatter": {
      "format": "%(asctime)s - %(levelname)s - [%(filename)s] - %(module)s - %(funcName)s - %(lineno)d - %(message)s",
      "datefmt": "%Y-%m-%d %H:%M:%S"
    }
  },
  "handlers": {
    "consoleHandler": {
      "class": "logging.StreamHandler",
      "formatter": "LogFormatter",
      "level": "INFO",
      "stream": "ext://sys.stdout"
    },
    "fileHandler": {
      "class": "thingsboard_gateway.tb_utility.tb_rotating_file_handler.TimedRotatingFileHandler",
      "formatter": "LogFormatter",
      "filename": "./logs/connector.log",
      "when": "D",
      "interval": 1,
      "backupCount": 7,
      "encoding": "utf-8"
    }
  },
  "loggers": {
    "service": {
      "level": "INFO",
      "handlers": ["consoleHandler", "fileHandler"],
      "propagate": false
    },
    "connector": {
      "level": "INFO",
      "handlers": ["consoleHandler", "fileHandler"],
      "propagate": false
    },
    "converter": {
      "level": "INFO",
      "handlers": ["consoleHandler", "fileHandler"],
      "propagate": false
    },
    "tb_connection": {
      "level": "INFO",
      "handlers": ["consoleHandler", "fileHandler"],
      "propagate": false
    },
    "storage": {
      "level": "INFO",
      "handlers": ["consoleHandler", "fileHandler"],
      "propagate": false
    }
  },
  "root": {
    "level": "ERROR",
    "handlers": ["consoleHandler"]
  }
}
```

## 环境变量

### 配置目录

```bash
export TB_GW_CONFIG_DIR=/etc/thingsboard-gateway/config/
```

### 开发模式

```bash
export TB_GW_DEV_MODE=true
export TB_GW_DEV_DEBUG_SERVER=5678
```

### ThingsBoard连接

```bash
export TB_GW_HOST=thingsboard.cloud
export TB_GW_PORT=1883
export TB_GW_ACCESS_TOKEN=YOUR_TOKEN
```

## 远程配置

### 启用远程配置

```json
{
  "thingsboard": {
    "remoteConfiguration": true
  }
}
```

### 配置更新流程

1. 在ThingsBoard平台修改网关配置
2. 平台通过MQTT发送配置更新
3. 网关接收并验证配置
4. 应用新配置（热更新或重启）
5. 上报配置更新状态

## 配置验证

### 验证工具

```python
from thingsboard_gateway.tb_utility.tb_utility import TBUtility

# 验证配置文件
config = TBUtility.load_config('tb_gateway.json')
if TBUtility.validate_config(config):
    print("Configuration is valid")
else:
    print("Configuration is invalid")
```

### 常见配置错误

1. **JSON格式错误**: 缺少逗号、括号不匹配
2. **路径错误**: 文件路径不存在
3. **端口冲突**: 端口已被占用
4. **认证错误**: 访问令牌无效
5. **类型错误**: 配置值类型不正确

## 配置最佳实践

1. **使用版本控制**: 将配置文件纳入版本控制
2. **环境分离**: 开发、测试、生产环境使用不同配置
3. **敏感信息**: 使用环境变量存储敏感信息
4. **备份配置**: 定期备份配置文件
5. **文档化**: 为自定义配置添加注释
6. **验证配置**: 部署前验证配置有效性

## 下一步

- 阅读 [开发指南](08-development-guide.md) 学习扩展开发
- 阅读 [API参考](09-api-reference.md) 了解API详情

