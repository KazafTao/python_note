# Python笔记

## 模块

### python标准库

python标准库包含内置模块。内置模块可以用来实现系统功能，如文件IO等。此外，标准库还有日常编程中的许多标准解决方案。

### 第三方模块

如django, flask等后台开发框架

### 自定义模块

自定义的业务代码

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
