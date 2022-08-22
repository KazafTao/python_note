# Python笔记

## 模块

包是一种用“点式模块名”构造 Python 模块命名空间的方法。Python 只把含 `__init__.py` 文件的目录当成包。这样可以防止以 `string` 等通用名称命名的目录，无意中屏蔽出现在后方模块搜索路径中的有效模块。 最简情况下，`__init__.py` 只是一个空文件，但该文件也可以执行包的初始化代码，或设置 `__all__` 变量

### python标准库

python标准库包含内置模块。内置模块可以用来实现系统功能，如文件IO等。此外，标准库还有日常编程中的许多标准解决方案。

### 第三方模块

如django, flask等后台开发框架

### 自定义模块

自定义的业务代码

### 模块搜索路径

当一个名为 spam 的模块被导入时，解释器首先搜索具有该名称的内置模块。这些模块的名字被列在 sys.builtin_module_names 中。如果没有找到，它就在变量 sys.path 给出的目录列表中搜索一个名为 spam.py 的文件， sys.path 从这些位置初始化:

- 输入脚本的目录（或未指定文件时的当前目录）。

- PYTHONPATH （目录列表，与 shell 变量 PATH 的语法一样）。

- 依赖于安装的默认值（按照惯例包括一个 site-packages 目录，由 site 模块处理）。

## 并发

### 概述

程序分为CPU密集型和IO密集型，如果程序性能的瓶颈在CPU，那程序就是CPU密集型程序，如压缩解压缩，加密解密，正则表达式搜索等。如果程序性能的瓶颈在IO，大部分时间都在读写存储，CPU占用率低，那程序就是IO密集型程序。如网络爬虫，文件处理程序，读写数据库程序等。

程序执行的粒度如下图所示。其中，CPU和IO是可以并行的，读取网络，磁盘中的数据并不需要CPU的参与。如果是串行执行的程序，那么程序在等待IO执行完的时候是在浪费CPU，延长程序执行时间。

python可以使用threading来进行多线程并发，以及multiprocessing进行多CPU并行。由hadoop/hive/spark技术来实现多机器并行。

![image-20220722183930875](https://s2.loli.net/2022/07/22/LG1mpfrCY5cKjeM.png)

python中的并发编程包括

- 多进程 Process
  - 优点：可以利用多核CPU并行运算
  - 缺点：占用资源最多、可启动数目比线程少
  - 适用于CPU密集型计算
- 多线程 Thread
  - 优点：相比进程，更轻量级，占用资源少
  - 缺点
    - 相比进程：多线程只能并发执行，不能利用多CPU (因为python有GIL，其他语言没有)
    - 相比协程：启动数目有限制，占用内存资源，有线程切换开销
  - 适用于IO密集型计算、同时运行的任务数目要求不多。
- 多协程 Coroutine (asyncio)
  - 优点：内存开销最小、启动协程数量最多
  - 缺点：支持的库有限制（aiohttp vs requests）、代码实现复杂
  - 适用于IO密集型计算、需要超多任务运行，且有现成库支持的场景

其中，一个进程可以启动多个线程，一个线程可以启动多个协程。

### GIL

GIL是python中的全局解释器锁。GIL保证了同一时刻，只有一个线程在执行，简化了共享资源的管理，但代价是多线程无法并行，拉低了多线程效率的上限。

对此，python对于CPU密集型程序，推出了使用多进程的multiprocess模块，对于IO密集型程序，推出了threading模块

### 多线程

多线程爬虫demo

```python
import requests
from threading import Thread

from bs4 import BeautifulSoup

urls = [
    f"https://www.cnblogs.com/#p{page}" for page in range(1, 20 + 1)
]


def craw(url):
    """爬取一页"""
    r = requests.get(url)
    soup = BeautifulSoup(r.text, 'lxml')
    print(r.url, soup.title.text)


threads = []
for url in urls:
    # 创建线程
    t = Thread(target=craw, args=(url,))
    threads.append(t)
    # 启动线程
    t.start()
# 等待所有线程结束
for t in threads:
    t.join()

```

#### 多线程数据通信

线程安全：多个线程并发操作数据不发生冲突。

Queue可以用于多线程之间，线程安全的数据通信。

多线程爬虫通信demo

```python
from queue import Queue
from time import sleep
from random import randint
from threading import Thread, current_thread

from spider import craw, parse, urls


def do_craw(url_queue: Queue, html_queue: Queue):
    while True:
        # 获取url队列中的数据，如果url队列为空，则阻塞
        url = url_queue.get()
        html = craw(url)
        # 将爬取的数据放入html队列中，如果html队列满，则阻塞
        html_queue.put(html)
        sleep(randint(1, 2))
        print(f"{current_thread().name} url queue size : {url_queue.qsize()}")


def do_parse(url_queue: Queue, html_queue: Queue, fout):
    while True:
        html = html_queue.get()
        results = parse(html)
        for result in results:
            # 如果希望爬遍整个网站的话，则将解析到的url放入url队列
            # url_queue.put(result[0])
            # 这里只是简单将数据持久化一下
            fout.write(str(result) + "\n")
        print(f"{current_thread().name} html queue size : {html_queue.qsize()}")


if __name__ == '__main__':
    url_queue = Queue()
    html_queue = Queue()
    # 放入起始url
    for url in urls:
        url_queue.put(url)
    # 创建3个生产者线程
    for idx in range(3):
        t = Thread(target=do_craw, args=(url_queue, html_queue), name=f"craw{idx}")
        t.start()
    f = open("data.dat", "w")
    # 创建4个消费者线程
    for idx in range(3):
        t = Thread(target=do_parse, args=(url_queue, html_queue, f), name=f"parse{idx}")
        t.start()
```

#### 加锁确保数据线程安全

threading提供了Lock来实现锁机制

```python
from threading import current_thread, Thread, Lock
from time import sleep

lock = Lock()


class Account:
    def __init__(self, balance):
        self.balance = balance

    def draw(self, amount):
        # 对于竞争资源要加锁来确保线程安全
        with lock:
            if self.balance >= amount:
                # 直接切换线程，但是由于锁的存在，其他线程会处于阻塞状态直到本线程z
                sleep(0.1)
                self.balance -= amount
                print(f"{current_thread().name} 取钱成功,余额为{self.balance}")
            else:
                print(f"{current_thread().name} 取钱失败，余额不足")


if __name__ == '__main__':
    account = Account(1000)
    threads = []
    threads.append(Thread(target=Account.draw, args=(account, 800), name="ta"))
    threads.append(Thread(target=Account.draw, args=(account, 800), name="tb"))
    for thread in threads:
        thread.start()
    for thread in threads:
        thread.join()
```

#### 线程池

##### 线程池原理

![image-20220724101942961](https://s2.loli.net/2022/07/24/evQVdhscMICowXS.png)

一个线程的生命周期有新建，就绪，运行，阻塞，终止5种状态。当需要大量线程的时候，频繁地新建终止线程需要系统进行分配回收资源的操作。线程池预先新建一批线程，然后不断地复用这批线程而不是终止再新建，节省了系统为线程分配/回收资源的开销。

##### 适用场景

适合处理突发性大量请求或大量线程完成任务、但实际任务处理时间较短的场景。如web服务器处理大量请求。

同时，因为线程池可以减少系统开销，防止系统因为频繁分配/回收资源导致系统变慢。

线程池demo

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from spider import craw, urls, parse

# 爬取
with ThreadPoolExecutor() as pool:
    htmls = pool.map(craw, urls)
    htmls = list(zip(urls, htmls))

with ThreadPoolExecutor() as pool:
    futures = {}
    # 解析
    for url, html in htmls:
        future = pool.submit(parse, html)
        futures[url] = future.result()
```

### 多进程

multiprocessing是python为了解决GIL缺陷引入的一个模块，原理是用多进程在多核CPU上并行执行。

多进程和多线程在语法上的对比如下图所示

![image-20220724152349576](https://s2.loli.net/2022/07/24/Vjzsha3ITxYyCdL.png)

#### 多进程demo

```python
import math
from datetime import datetime
from concurrent.futures import ProcessPoolExecutor

PRIMES = [112272535095293, ] * 50


def is_prime(n):
    """判断一个整数是否为素数"""
    if n < 2:
        return False
    elif n == 2:
        return True
    elif n % 2 == 0:
        return False

    sqrt_n = int(math.floor(math.sqrt(n)))
    for i in range(3, sqrt_n + 1, 2):
        if n % i == 0:
            return False
    return True


def single_process():
    if all([is_prime(prime) for prime in PRIMES]):
        print("素数")
    else:
        print("非素数")


def multi_process():
    with ProcessPoolExecutor() as pool:
        print(list(pool.map(is_prime, PRIMES)))


if __name__ == '__main__':
    start = datetime.now()
    single_process()
    end = datetime.now()
    print(f"单进程花费了{(end - start).seconds}秒")
    start = datetime.now()
    multi_process()
    end = datetime.now()
    print(f"多线程花费了{(end - start).seconds}秒")
    
# 结果
# 素数
# 单线程花费了14秒
# [True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True]
# 多线程花费了2秒
```

#### 多进程注意事项

由于进程之间的资源是隔离的，所以使用多进程的时候，声明线程池的时候，使用线程池下所有的代码必须都已声明。

在web开发中，线程池的声明可以放在所有视图函数之后。

### 协程

#### 协程基本原理

一个线程内允许有多个任务，当其中一个任务需要等待IO时，不进行线程切换而是执行线程内的下一个任务。直到操作系统进行调度，才让出CPU。一个任务为一个协程。

## 装饰器

装饰器是是修改其他函数的功能的函数。

在python中，函数可以是对象，函数中也可以定义并调用被定义的函数，同时，函数中也可以返回函数。

最简单的装饰器，就是将函数作为参数，然后在装饰器中定义并调用传递过来的函数，再返回新定义的函数。demo如下。

```python
from functools import wraps


def log(logfile='out.log'):
    # 定义内部函数
    def inner(func):
        # wraps装饰器保留传递过来的函数的函数名
        @wraps(func)
        # 内部函数内嵌套定义内部函数
        def wrapper(*args, **kwargs):
            with open(logfile, 'a') as f:
                f.write(f"开始调用{func.__name__}")
                func(*args, **kwargs)
                f.write(f"{func.__name__}调用结束")

        # 返回内部函数
        return wrapper

    return inner
```

## 业务代码

### 系统信息采集

```python
print(f"cpu核数 : {psutil.cpu_count()}")
print(f"cpu使用率 : {psutil.cpu_percent()}")
mem = psutil.virtual_memory()
print(f"内存容量 ：{mem.total / 1024 / 1024 / 1024} GB")
print(f"空闲内存 ：{mem.free / 1024 / 1024 / 1024} GB")
res = subprocess.run(["df", '-h'], capture_output=True, encoding='utf-8')
for line in str(res.stdout).split('\n'):
    if line.startswith('/'):
        infos = line.split()
        print(f"分区：{infos[0]} 容量: {infos[1]} 已用：{infos[2]} 可用：{infos[3]}")
```
