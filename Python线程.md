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


# 线程知识
在Python中使用线程可以通过内置的`threading`模块实现。线程适合处理**I/O密集型任务**（如网络请求、文件读写等），但由于Python的全局解释器锁（GIL），线程不适用于**CPU密集型任务**（如数学计算）。以下是详细用法：

---

### 一、基础使用
#### 1. 创建并启动线程
```python
import threading
import time

def my_task(name):
    print(f"线程 {name} 启动")
    time.sleep(2)  # 模拟耗时操作
    print(f"线程 {name} 结束")

# 创建线程对象
t1 = threading.Thread(target=my_task, args=("线程1",))
t2 = threading.Thread(target=my_task, args=("线程2",))

# 启动线程
t1.start()
t2.start()

# 等待线程结束
t1.join()
t2.join()

print("所有线程执行完毕")
```

---

### 二、线程同步（防止数据竞争）
#### 使用 `Lock` 锁
```python
counter = 0
lock = threading.Lock()

def increment():
    global counter
    for _ in range(100000):
        with lock:  # 自动获取和释放锁
            counter += 1

t1 = threading.Thread(target=increment)
t2 = threading.Thread(target=increment)

t1.start()
t2.start()
t1.join()
t2.join()

print(f"最终计数: {counter}")  # 正确结果为 200000
```

---

### 三、守护线程（Daemon Threads）
- **守护线程**会在主线程结束时自动退出，适合后台服务。
- 通过`daemon=True`设置：
```python
def daemon_task():
    while True:
        print("守护线程运行中...")
        time.sleep(1)

t = threading.Thread(target=daemon_task, daemon=True)
t.start()

time.sleep(3)
print("主线程结束，守护线程自动终止")
```

---

### 四、线程间通信
#### 使用 `Queue` 安全传递数据
```python
import queue

def producer(q):
    for i in range(5):
        q.put(i)
        print(f"生产数据: {i}")
        time.sleep(0.5)

def consumer(q):
    while True:
        item = q.get()
        if item is None:  # 终止信号
            break
        print(f"消费数据: {item}")
        q.task_done()

q = queue.Queue()
t1 = threading.Thread(target=producer, args=(q,))
t2 = threading.Thread(target=consumer, args=(q,))

t1.start()
t2.start()

t1.join()
q.put(None)  # 发送终止信号
t2.join()
```

---

### 五、线程池（推荐方式）
使用`concurrent.futures`模块简化线程管理：
```python
from concurrent.futures import ThreadPoolExecutor

def task(n):
    return n * n

with ThreadPoolExecutor(max_workers=3) as executor:
    # 提交单个任务
    future = executor.submit(task, 5)
    print(future.result())  # 输出 25

    # 批量提交任务
    results = executor.map(task, [1, 2, 3])
    print(list(results))  # 输出 [1, 4, 9]
```

---

### 六、注意事项
1. **GIL限制**：Python线程因全局解释器锁（GIL）无法并行执行CPU密集型任务，此时应改用`multiprocessing`模块。
2. **避免死锁**：确保锁的获取和释放成对出现，推荐使用`with lock`语法。
3. **线程安全**：对共享资源的操作必须加锁。

如果需要处理CPU密集型任务，建议学习`multiprocessing`模块实现多进程并行。
