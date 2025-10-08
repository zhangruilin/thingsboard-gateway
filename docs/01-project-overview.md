# 项目概述

## 什么是 ThingsBoard IoT Gateway

ThingsBoard IoT Gateway 是一个开源的、基于Python的应用程序，它作为协议适配器，使传统设备和第三方设备能够与ThingsBoard IoT平台无缝集成。网关从外部数据源收集数据，并以统一格式发布到ThingsBoard平台。

## 核心价值

### 1. 协议转换
- 支持15+种工业和物联网协议
- 将各种协议的数据转换为ThingsBoard统一格式
- 支持自定义协议扩展

### 2. 边缘计算
- 本地数据缓存和持久化
- 离线数据存储，网络恢复后自动上传
- 减少云端负载

### 3. 设备管理
- 自动设备发现和注册
- 设备重命名和删除同步
- 设备状态监控

### 4. 远程管理
- 远程配置更新
- 远程日志查看
- 远程Shell访问
- RPC命令执行

## 技术栈

### 编程语言
- **Python 3.10+**: 主要开发语言
- 支持跨平台运行（Linux, Windows, macOS）

### 核心依赖
```
- tb-mqtt-client: ThingsBoard MQTT客户端
- pymodbus: Modbus协议支持
- asyncua: OPC-UA协议支持
- paho-mqtt: MQTT协议支持
- grpcio: gRPC服务支持
- PyYAML: 配置文件解析
- simplejson/orjson: JSON处理
```

### 架构特点
- **模块化设计**: 连接器、转换器、存储等模块独立
- **异步处理**: 支持asyncio异步编程
- **多线程**: 每个连接器独立线程运行
- **插件化**: 支持动态加载自定义连接器和转换器

## 项目结构

```
thingsboard-gateway/
├── thingsboard_gateway/          # 主代码目录
│   ├── gateway/                  # 网关核心服务
│   │   ├── tb_gateway_service.py # 网关主服务
│   │   ├── tb_client.py          # ThingsBoard客户端
│   │   ├── entities/             # 数据实体
│   │   ├── grpc_service/         # gRPC服务
│   │   ├── statistics/           # 统计服务
│   │   └── shell/                # 远程Shell
│   ├── connectors/               # 协议连接器
│   │   ├── connector.py          # 连接器基类
│   │   ├── converter.py          # 转换器基类
│   │   ├── mqtt/                 # MQTT连接器
│   │   ├── modbus/               # Modbus连接器
│   │   ├── opcua/                # OPC-UA连接器
│   │   ├── bacnet/               # BACnet连接器
│   │   ├── ble/                  # BLE连接器
│   │   ├── can/                  # CAN连接器
│   │   ├── ftp/                  # FTP连接器
│   │   ├── knx/                  # KNX连接器
│   │   ├── ocpp/                 # OCPP连接器
│   │   ├── odbc/                 # ODBC连接器
│   │   ├── request/              # HTTP Request连接器
│   │   ├── rest/                 # REST API连接器
│   │   ├── snmp/                 # SNMP连接器
│   │   ├── socket/               # Socket连接器
│   │   └── xmpp/                 # XMPP连接器
│   ├── extensions/               # 扩展和自定义转换器
│   ├── storage/                  # 数据存储
│   │   ├── memory/               # 内存存储
│   │   ├── file/                 # 文件存储
│   │   └── sqlite/               # SQLite存储
│   ├── tb_utility/               # 工具类
│   ├── grpc_connectors/          # gRPC连接器
│   ├── config/                   # 默认配置文件
│   └── tb_gateway.py             # 入口文件
├── tests/                        # 测试代码
│   ├── unit/                     # 单元测试
│   ├── integration/              # 集成测试
│   └── blackbox/                 # 黑盒测试
├── docker/                       # Docker配置
├── for_build/                    # 构建配置
├── setup.py                      # 安装配置
├── requirements.txt              # 依赖列表
└── README.md                     # 项目说明
```

## 主要功能模块

### 1. 网关服务 (TBGatewayService)
- 网关核心服务，负责整体协调
- 管理连接器生命周期
- 处理与ThingsBoard平台的通信
- 设备数据路由和分发

### 2. 连接器 (Connectors)
- 实现各种协议的数据采集
- 每个连接器独立运行
- 支持热插拔和动态配置

### 3. 转换器 (Converters)
- 上行转换器：设备数据 → ThingsBoard格式
- 下行转换器：ThingsBoard命令 → 设备格式
- 支持自定义转换逻辑

### 4. 存储系统 (Storage)
- 本地数据缓存
- 离线数据持久化
- 支持内存、文件、SQLite三种存储方式

### 5. 统计服务 (Statistics)
- 消息统计
- 性能监控
- 连接器状态跟踪

### 6. 远程管理
- 远程配置更新
- 远程日志查看
- 远程Shell执行
- RPC方法调用

## 数据流

```
设备/传感器
    ↓
协议连接器 (Connector)
    ↓
上行转换器 (Uplink Converter)
    ↓
网关服务 (Gateway Service)
    ↓
本地存储 (Storage)
    ↓
MQTT客户端 (TB Client)
    ↓
ThingsBoard平台
```

反向数据流（RPC/属性更新）：
```
ThingsBoard平台
    ↓
MQTT客户端 (TB Client)
    ↓
网关服务 (Gateway Service)
    ↓
下行转换器 (Downlink Converter)
    ↓
协议连接器 (Connector)
    ↓
设备/传感器
```

## 部署方式

### 1. Python包安装
```bash
pip install thingsboard-gateway
thingsboard-gateway
```

### 2. Docker部署
```bash
docker run -it -v ~/.tb-gateway/logs:/thingsboard_gateway/logs \
  -v ~/.tb-gateway/extensions:/thingsboard_gateway/extensions \
  -v ~/.tb-gateway/config:/thingsboard_gateway/config \
  --name tb-gateway --restart always thingsboard/tb-gateway
```

### 3. 源码运行
```bash
git clone https://github.com/thingsboard/thingsboard-gateway.git
cd thingsboard-gateway
pip install -r requirements.txt
python -m thingsboard_gateway.tb_gateway
```

### 4. 系统服务
- Debian/Ubuntu: DEB包安装
- 作为systemd服务运行

## 配置文件

主要配置文件位于 `/etc/thingsboard-gateway/config/` 或项目的 `config/` 目录：

- `tb_gateway.json`: 网关主配置
- `logs.json`: 日志配置
- `mqtt.json`: MQTT连接器配置
- `modbus.json`: Modbus连接器配置
- `opcua.json`: OPC-UA连接器配置
- 其他协议配置文件...

## 版本信息

- **当前版本**: 3.7.8
- **Python要求**: >= 3.10
- **许可证**: Apache License 2.0

## 下一步

- 阅读 [架构设计](02-architecture.md) 了解系统架构
- 阅读 [核心组件](03-core-components.md) 了解各个模块
- 阅读 [连接器系统](04-connectors.md) 了解如何使用连接器
- 阅读 [开发指南](08-development-guide.md) 了解如何扩展功能

