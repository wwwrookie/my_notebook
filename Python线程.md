# 查看程序中运行的线程数量
在 Python 中，可以通过 `threading` 模块查看当前程序中活动的线程数量。以下是几种常用的方法：

---

### **方法 1：使用 `threading.active_count()`**
`threading.active_count()` 会返回当前存活的线程数量（包括主线程和其他子线程）。

```python
import threading

def print_active_threads():
    print(f"当前活动线程数量: {threading.active_count()}")

# 示例：创建并启动多个线程
threads = []
for i in range(3):
    thread = threading.Thread(target=lambda: time.sleep(10))
    thread.start()
    threads.append(thread)

print_active_threads()  # 输出：当前活动线程数量: 4（主线程 + 3个子线程）
```

---

### **方法 2：遍历 `threading.enumerate()`**
`threading.enumerate()` 返回所有存活线程的列表，通过列表长度可直接统计线程数：

```python
import threading

def list_all_threads():
    threads = threading.enumerate()
    print(f"总线程数: {len(threads)}")
    for thread in threads:
        print(f"线程名称: {thread.name}, 状态: {'存活' if thread.is_alive() else '终止'}")

# 输出示例：
# 总线程数: 4
# 线程名称: MainThread, 状态: 存活
# 线程名称: Thread-1, 状态: 存活
# 线程名称: Thread-2, 状态: 存活
# 线程名称: Thread-3, 状态: 存活
```

---

### **方法 3：统计非主线程的数量**
如果需排除主线程，可以手动过滤：

```python
import threading

def count_child_threads():
    main_thread = threading.main_thread()
    child_threads = [t for t in threading.enumerate() if t != main_thread and t.is_alive()]
    print(f"子线程数量: {len(child_threads)}")
```

---

### **注意事项**
1. **主线程默认存在**：所有 Python 程序至少有一个主线程（`MainThread`）。
2. **线程状态**：`is_alive()` 用于检查线程是否存活，但 `threading.enumerate()` 默认会过滤已终止的线程。
3. **线程生命周期**：线程可能在 `start()` 后尚未运行，或已运行结束但未被回收，需结合业务逻辑分析。

---

### **动态监控线程数**
若需实时监控，可以在代码中添加定期检查逻辑：
```python
import threading
import time

def monitor_threads(interval=2):
    while True:
        print(f"当前线程数: {threading.active_count()}")
        time.sleep(interval)

# 启动监控线程
monitor_thread = threading.Thread(target=monitor_threads, daemon=True)
monitor_thread.start()
```

---

### **常见场景**
- **调试多线程泄漏**：若线程数持续增长，可能未正确关闭线程。
- **验证并发控制**：确认线程池（如 `concurrent.futures.ThreadPoolExecutor`）是否按预期工作。

通过上述方法，你可以快速定位线程数量是否符合预期，确保程序的高效与稳定。
