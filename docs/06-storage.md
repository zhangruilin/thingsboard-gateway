# 存储系统

## 存储系统概述

ThingsBoard IoT Gateway的存储系统负责本地数据缓存和持久化，确保在网络中断或ThingsBoard平台不可用时，数据不会丢失。

## 存储类型

### 1. MemoryEventStorage (内存存储)

**特点**：
- 数据存储在内存中
- 读写速度最快
- 网关重启后数据丢失
- 适合测试环境

**配置**：
```json
{
  "storage": {
    "type": "memory",
    "read_records_count": 100,
    "max_records_count": 100000
  }
}
```

**使用场景**：
- 开发和测试
- 数据不重要的场景
- 对性能要求极高的场景

### 2. FileEventStorage (文件存储)

**特点**：
- 数据持久化到文件
- 支持大容量存储
- 读写性能中等
- 网关重启后数据保留

**配置**：
```json
{
  "storage": {
    "type": "file",
    "data_folder_path": "./data/",
    "max_file_count": 10,
    "max_records_per_file": 10000,
    "max_read_records_count": 100
  }
}
```

**文件结构**：
```
data/
├── events_0.db
├── events_1.db
├── events_2.db
└── ...
```

**使用场景**：
- 需要持久化但不需要复杂查询
- 存储空间充足
- 对性能要求不是特别高

### 3. SQLiteEventStorage (SQLite存储)

**特点**：
- 使用SQLite数据库
- 数据持久化
- 支持事务
- 性能优秀
- **推荐用于生产环境**

**配置**：
```json
{
  "storage": {
    "type": "sqlite",
    "data_file_path": "./data/storage.db",
    "max_records_count": 100000,
    "read_records_count": 100
  }
}
```

**数据库结构**：
```sql
CREATE TABLE events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    device_name TEXT NOT NULL,
    device_type TEXT,
    telemetry TEXT,
    attributes TEXT,
    timestamp INTEGER,
    created_at INTEGER
);

CREATE INDEX idx_device_name ON events(device_name);
CREATE INDEX idx_timestamp ON events(timestamp);
```

**使用场景**：
- 生产环境（推荐）
- 需要可靠的数据持久化
- 需要事务支持

## 存储接口

### EventStorage基类

```python
from abc import ABC, abstractmethod
from typing import List
from thingsboard_gateway.gateway.entities.device_event_pack import DeviceEventPack

class EventStorage(ABC):
    @abstractmethod
    def put(self, data):
        """
        存储数据
        
        Args:
            data: ConvertedData对象或数据字典
        """
        pass
    
    @abstractmethod
    def get_event_pack(self) -> List[DeviceEventPack]:
        """
        获取待发送的数据包
        
        Returns:
            DeviceEventPack列表
        """
        pass
    
    @abstractmethod
    def event_pack_processing_done(self):
        """
        确认数据包已成功处理
        """
        pass
    
    @abstractmethod
    def get_events_count(self) -> int:
        """
        获取存储中的事件数量
        
        Returns:
            事件数量
        """
        pass
```

## 数据流程

### 写入流程

```
1. Connector发送数据
   ↓
2. Gateway.send_to_storage()
   ↓
3. Storage.put(data)
   ↓
4. 数据序列化
   ↓
5. 写入存储介质
   - Memory: 添加到列表
   - File: 追加到文件
   - SQLite: INSERT到数据库
```

### 读取流程

```
1. Storage定期检查
   ↓
2. Storage.get_event_pack()
   ↓
3. 从存储读取数据
   - Memory: 从列表读取
   - File: 从文件读取
   - SQLite: SELECT查询
   ↓
4. 反序列化数据
   ↓
5. 返回DeviceEventPack列表
   ↓
6. Gateway发送到ThingsBoard
   ↓
7. Storage.event_pack_processing_done()
   ↓
8. 删除已发送的数据
```

## DeviceEventPack 数据结构

```python
class DeviceEventPack:
    def __init__(self, device_name: str, device_type: str = None):
        self.device_name = device_name
        self.device_type = device_type
        self.telemetry = []      # List[TelemetryEntry]
        self.attributes = {}     # Dict[str, Any]
    
    def add_telemetry(self, telemetry_entry):
        self.telemetry.append(telemetry_entry)
    
    def add_attributes(self, attributes):
        self.attributes.update(attributes)
```

## SQLiteEventStorage 详解

### 初始化

```python
class SQLiteEventStorage(EventStorage):
    def __init__(self, config):
        self.__config = config
        self.__db_path = config.get('data_file_path', './data/storage.db')
        self.__max_records = config.get('max_records_count', 100000)
        self.__read_records = config.get('read_records_count', 100)
        
        # 创建数据库连接
        self.__db = DatabaseConnector(self.__db_path)
        
        # 初始化表结构
        self.__init_tables()
        
        # 读取指针
        self.__read_pointer = 0
```

### 存储数据

```python
def put(self, data):
    try:
        # 检查存储容量
        if self.get_events_count() >= self.__max_records:
            self.__log.warning("Storage is full, dropping oldest records")
            self.__cleanup_old_records()
        
        # 序列化数据
        telemetry_json = dumps(data.telemetry) if data.telemetry else None
        attributes_json = dumps(data.attributes) if data.attributes else None
        
        # 插入数据库
        self.__db.execute(
            """
            INSERT INTO events 
            (device_name, device_type, telemetry, attributes, timestamp, created_at)
            VALUES (?, ?, ?, ?, ?, ?)
            """,
            (
                data.device_name,
                data.device_type,
                telemetry_json,
                attributes_json,
                int(time() * 1000),
                int(time() * 1000)
            )
        )
        
        self.__db.commit()
        
    except Exception as e:
        self.__log.error("Error storing data: %s", e)
        self.__db.rollback()
```

### 读取数据

```python
def get_event_pack(self) -> List[DeviceEventPack]:
    try:
        # 查询数据
        rows = self.__db.execute(
            """
            SELECT id, device_name, device_type, telemetry, attributes
            FROM events
            WHERE id > ?
            ORDER BY id
            LIMIT ?
            """,
            (self.__read_pointer, self.__read_records)
        ).fetchall()
        
        if not rows:
            return []
        
        # 转换为DeviceEventPack
        event_packs = {}
        for row in rows:
            event_id, device_name, device_type, telemetry_json, attributes_json = row
            
            # 按设备分组
            if device_name not in event_packs:
                event_packs[device_name] = DeviceEventPack(device_name, device_type)
            
            # 添加遥测数据
            if telemetry_json:
                telemetry = loads(telemetry_json)
                for entry in telemetry:
                    event_packs[device_name].add_telemetry(entry)
            
            # 添加属性数据
            if attributes_json:
                attributes = loads(attributes_json)
                event_packs[device_name].add_attributes(attributes)
            
            # 更新读取指针
            self.__read_pointer = event_id
        
        return list(event_packs.values())
        
    except Exception as e:
        self.__log.error("Error reading data: %s", e)
        return []
```

### 确认处理完成

```python
def event_pack_processing_done(self):
    try:
        # 删除已处理的记录
        self.__db.execute(
            "DELETE FROM events WHERE id <= ?",
            (self.__read_pointer,)
        )
        self.__db.commit()
        
    except Exception as e:
        self.__log.error("Error confirming processing: %s", e)
        self.__db.rollback()
```

## FileEventStorage 详解

### 文件管理

```python
class FileEventStorage(EventStorage):
    def __init__(self, config):
        self.__config = config
        self.__data_folder = config.get('data_folder_path', './data/')
        self.__max_file_count = config.get('max_file_count', 10)
        self.__max_records_per_file = config.get('max_records_per_file', 10000)
        
        # 创建数据目录
        if not path.exists(self.__data_folder):
            makedirs(self.__data_folder)
        
        # 文件管理
        self.__current_file_index = 0
        self.__current_file_records = 0
        self.__writer = None
        self.__reader = None
```

### 写入文件

```python
def put(self, data):
    try:
        # 检查是否需要创建新文件
        if self.__current_file_records >= self.__max_records_per_file:
            self.__rotate_file()
        
        # 序列化数据
        event_data = {
            'device_name': data.device_name,
            'device_type': data.device_type,
            'telemetry': data.telemetry,
            'attributes': data.attributes,
            'timestamp': int(time() * 1000)
        }
        
        # 写入文件
        if self.__writer is None:
            self.__open_writer()
        
        self.__writer.write(dumps(event_data) + '\n')
        self.__writer.flush()
        
        self.__current_file_records += 1
        
    except Exception as e:
        self.__log.error("Error writing to file: %s", e)

def __rotate_file(self):
    # 关闭当前文件
    if self.__writer:
        self.__writer.close()
    
    # 删除最旧的文件（如果超过限制）
    files = self.__get_data_files()
    if len(files) >= self.__max_file_count:
        oldest_file = min(files, key=lambda f: path.getctime(f))
        remove(oldest_file)
    
    # 创建新文件
    self.__current_file_index += 1
    self.__current_file_records = 0
    self.__writer = None
```

## 性能优化

### 1. 批量操作

```python
def put_batch(self, data_list):
    """批量存储数据"""
    try:
        self.__db.begin_transaction()
        
        for data in data_list:
            self.__insert_record(data)
        
        self.__db.commit()
        
    except Exception as e:
        self.__log.error("Batch insert failed: %s", e)
        self.__db.rollback()
```

### 2. 索引优化

```sql
-- 为常用查询字段创建索引
CREATE INDEX idx_device_name ON events(device_name);
CREATE INDEX idx_timestamp ON events(timestamp);
CREATE INDEX idx_id_device ON events(id, device_name);
```

### 3. 定期清理

```python
def __cleanup_old_records(self):
    """清理旧记录"""
    try:
        # 保留最新的记录
        keep_count = int(self.__max_records * 0.8)
        
        self.__db.execute(
            """
            DELETE FROM events
            WHERE id NOT IN (
                SELECT id FROM events
                ORDER BY id DESC
                LIMIT ?
            )
            """,
            (keep_count,)
        )
        
        self.__db.commit()
        
        # 压缩数据库
        self.__db.execute("VACUUM")
        
    except Exception as e:
        self.__log.error("Cleanup failed: %s", e)
```

## 监控和统计

### 存储状态

```python
def get_storage_status(self):
    """获取存储状态"""
    return {
        'type': self.__config.get('type'),
        'events_count': self.get_events_count(),
        'max_records': self.__max_records,
        'usage_percent': (self.get_events_count() / self.__max_records) * 100,
        'read_pointer': self.__read_pointer
    }
```

### 统计信息

```python
def get_statistics(self):
    """获取统计信息"""
    return {
        'total_events': self.get_events_count(),
        'events_by_device': self.__get_events_by_device(),
        'oldest_event_time': self.__get_oldest_event_time(),
        'newest_event_time': self.__get_newest_event_time()
    }
```

## 配置建议

### 开发环境

```json
{
  "storage": {
    "type": "memory",
    "read_records_count": 100,
    "max_records_count": 10000
  }
}
```

### 生产环境

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

### 高吞吐量环境

```json
{
  "storage": {
    "type": "sqlite",
    "data_file_path": "/var/lib/thingsboard-gateway/storage.db",
    "max_records_count": 5000000,
    "read_records_count": 1000,
    "batch_size": 100
  }
}
```

## 故障恢复

### 数据恢复

```python
def recover_from_backup(self, backup_path):
    """从备份恢复数据"""
    try:
        # 复制备份文件
        shutil.copy(backup_path, self.__db_path)
        
        # 重新连接数据库
        self.__db.reconnect()
        
        self.__log.info("Data recovered from backup")
        
    except Exception as e:
        self.__log.error("Recovery failed: %s", e)
```

## 下一步

- 阅读 [配置管理](07-configuration.md) 了解配置详情
- 阅读 [开发指南](08-development-guide.md) 学习扩展开发

