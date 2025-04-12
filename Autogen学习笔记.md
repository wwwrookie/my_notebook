# 为 message_thread 添加一个简单的存储抽象，TTL 策略可以作为此存储抽象实现的参数。这个怎么实现

以下是基于 Python 的简单存储抽象实现，支持 TTL（Time to Live）策略的代码示例。该实现会为每个存储的键值对设置过期时间，并在访问时自动清理过期数据：

---

### **实现代码**
```python
import time
import threading
from collections import defaultdict

class TTLStorage:
    def __init__(self, default_ttl=3600):
        """
        初始化存储抽象
        :param default_ttl: 默认的 TTL 时间（秒），默认 1 小时
        """
        self.default_ttl = default_ttl
        self._data = {}  # 存储结构：{key: (value, expires_at)}
        self._lock = threading.Lock()  # 确保线程安全

    def set(self, key, value, ttl=None):
        """
        存储键值对，支持自定义 TTL
        :param key: 键
        :param value: 值
        :param ttl: 自定义 TTL（秒），若未指定则使用默认值
        """
        ttl = ttl if ttl is not None else self.default_ttl
        expires_at = time.time() + ttl
        with self._lock:
            self._data[key] = (value, expires_at)

    def get(self, key):
        """
        获取键对应的值，若过期则返回 None 并删除键
        """
        with self._lock:
            entry = self._data.get(key)
            if not entry:
                return None
            value, expires_at = entry
            if time.time() > expires_at:
                del self._data[key]  # 惰性删除过期数据
                return None
            return value

    def delete_expired(self):
        """
        主动清理所有过期键值对
        """
        current_time = time.time()
        with self._lock:
            expired_keys = [
                key for key, (_, expires_at) in self._data.items()
                if current_time > expires_at
            ]
            for key in expired_keys:
                del self._data[key]
        return len(expired_keys)

    def __contains__(self, key):
        """检查键是否存在（未过期）"""
        return self.get(key) is not None

    def size(self):
        """返回当前存储的未过期键数量"""
        with self._lock:
            self.delete_expired()  # 可选：主动清理后统计
            return len(self._data)
```

---

### **使用示例**
```python
# 初始化存储，默认 TTL 为 10 秒
storage = TTLStorage(default_ttl=10)

# 存储数据
storage.set("user:1001", {"name": "Alice"}, ttl=5)  # 自定义 TTL
storage.set("user:1002", {"name": "Bob"})           # 使用默认 TTL

# 获取数据
print(storage.get("user:1001"))  # 5 秒内返回数据，之后返回 None

# 检查是否存在
print("user:1001" in storage)    # 5 秒内返回 True，之后返回 False

# 手动清理过期数据
expired_count = storage.delete_expired()
print(f"已清理 {expired_count} 个过期键")

# 获取当前存储大小
print(f"当前存储数量: {storage.size()}")
```

---

### **关键设计点**
1. **TTL 策略集成**：
   - 每个键值对存储时计算过期时间 `expires_at`。
   - 默认 TTL 在初始化时指定，也可在 `set` 方法中为单个键覆盖。
   - 通过 `get` 方法访问时自动清理过期键（惰性删除）。

2. **线程安全**：
   - 使用 `threading.Lock` 确保多线程环境下的数据一致性。

3. **主动清理**：
   - 提供 `delete_expired()` 方法主动清理过期数据，避免内存泄漏。

4. **扩展性**：
   - 可结合后台线程定期调用 `delete_expired()`（例如每 60 秒一次），实现主动过期扫描。

---

### **扩展：后台自动清理**
若需要自动清理过期数据，可以添加后台线程：
```python
class AutoCleanTTLStorage(TTLStorage):
    def __init__(self, default_ttl=3600, cleanup_interval=60):
        super().__init__(default_ttl)
        self._cleanup_interval = cleanup_interval
        self._start_cleanup_thread()

    def _start_cleanup_thread(self):
        def cleanup_loop():
            while True:
                time.sleep(self._cleanup_interval)
                self.delete_expired()
        # 使用守护线程，避免阻塞主程序退出
        thread = threading.Thread(target=cleanup_loop, daemon=True)
        thread.start()

# 使用示例
storage = AutoCleanTTLStorage(default_ttl=10, cleanup_interval=5)
```

---

### **适用场景**
- **临时会话管理**：存储用户会话，超时自动清理。
- **缓存系统**：缓存 API 响应或计算结果，避免长期占用内存。
- **消息队列**：存储需超时处理的任务（如订单未支付自动关闭）。

通过此设计，你可以轻松将 TTL 策略嵌入到 `message_thread` 的存储逻辑中，确保数据按需自动过期。
