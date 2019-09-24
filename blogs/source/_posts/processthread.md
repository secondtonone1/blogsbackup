---
title: python学习(十三)进程和线程
date: 2017-11-29 18:22:25
categories: [python]
tags: [python]
---
python多进程
``` python
from multiprocessing import Process
import os

def processFunc(name):
	print("child process is %s, pid is %s"  %(name, os.getpid() ) )
	return
if __name__ == '__main__':
	print("Parent process is %s." %(os.getpid() ))
	p = Process(target = processFunc, args = ('test', ))
	print('Child will start ')
	p.start()
	p.join()
	print("Child stop")
```
<!--more-->
进程池
``` python
from multiprocessing import Pool
import os , time, random

def long_time_task(name):
	print('run task name is %s' %(name))
	start = time.time()
	time.sleep(random.random()*3)
	end = time.time()
	print('Task %s runs %0.2f seconds.' %(name, (end - start )) )


if __name__ == '__main__':
	print('Parent pid is %s' %(os.getpid() ))
	p = Pool(4)
	for i in range(5):
		p.apply_async(long_time_task, args = (str(i) ,) )
	print("Waiting all processes!!!")
	p.close()
	p.join()
	print("All subprocess done")
```

启动进程，并调用命令行
``` python
import subprocess

print('$ nslookup www.python.org')
r = subprocess.call(['nslookup', 'www.python.org'])
print('Exit code:', r)
```

队列Queue可实现两个进程间通信

``` python
from multiprocessing import Process, Queue
import os, time, random

def write(q):
	print('Process to Write pid is %s' %(os.getpid() ) )
	for i in ['A','B','C']:
		q.put(i)
		time.sleep(random.random())

def read(q):
	print('Process to Read pid is %s' %(os.getpid() ) )
	while(True):
		value = q.get(True)
		print('Get %s from queue ' %(value))

if __name__ == '__main__':
	q = Queue()
	pw = Process(target=write, args = (q,))
	pr = Process(target = read , args = (q,) )
	pw.start()
	pr.start()
	pw.join()
	pr.terminate()
```

python多线程

``` python
import threading , time
def loop():
	print('thread %s is running ...' % threading.current_thread().name)
	n = 0
	while n < 5:
		n = n+ 1
		print('thread %s >>> %s' %(threading.current_thread().name, n))
		time.sleep(1)
	print('thread %s ended. ' %(threading.current_thread().name ) )

if __name__ == '__main__':
	print('Thread %s is running...' % threading.current_thread().name)
	t = threading.Thread(target = loop, name = 'LoopThread')
	t.start()
	t.join()
	print('Thread %s ended.' % threading.current_thread().name)
```
多线程访问全局变量，记得加锁
``` python
import time, threading

# 假定这是你的银行存款:
balance = 0
lock = threading.Lock()

def change_it(n):
    # 先存后取，结果应该为0:
    global balance
    balance = balance + n
    balance = balance - n

def run_thread(n):
    for i in range(100000):
    	lock.acquire()
    	try:
    		change_it(n)
    	finally:
    		lock.release()
        

t1 = threading.Thread(target=run_thread, args=(5,))
t2 = threading.Thread(target=run_thread, args=(8,))
t1.start()
t2.start()
t1.join()
t2.join()
print(balance)
```
避免枷锁带来的效率衰退，可使用线程本地变量

``` python
import threading
# 创建全局ThreadLocal对象:
local_school = threading.local()

def process_student():
	# 获取当前线程关联的student:
	std = local_school.student
	print('Hello, %s in thread %s' %(std, threading.current_thread().name ))

def process_thread(name):
	# 绑定ThreadLocal的student:
	local_school.student = name
	process_student()

if __name__ == '__main__':
	t1 = threading.Thread(target = process_thread, args=('Alice',), name = 'Thread-A')
	t2 = threading.Thread(target= process_thread, args=('Bob',), name='Thread-B')
	t1.start()
	t2.start()
	t1.join()
	t2.join()
```

分布式进程，用于不同机器通信，采用BaseManager，在masterprocess.py中实现如下

``` python
import random, time, queue
from multiprocessing.managers import BaseManager

task_queue = queue.Queue()
result_queue = queue.Queue()

def taskqueuefunc():
	global task_queue
	return task_queue

def resultqueuefunc():
	global result_queue
	return result_queue

class QueueManager(BaseManager):
	pass


def ServerStart():
	QueueManager.register('get_task_queue', callable = taskqueuefunc)
	QueueManager.register('get_result_queue', callable = resultqueuefunc)
	manager = QueueManager(address=('127.0.0.1', 5000), authkey=b'abc')
	manager.start()

	task = manager.get_task_queue()

	result = manager.get_result_queue()

	for i in range(10):
		n = random.randint(0,10000)
		print('Put task %d...' %n)
		task.put(n)

	# 从result队列读取结果:
	print('Try get results...')
	for i in range(10):
		r = result.get(timeout=10)
		print('Result: %s' % r)
	# 关闭:
	manager.shutdown()
	print('master exit.')


if __name__ == '__main__':
	ServerStart()
```
在另一个文件workprocess.py中实现另一个进程处理数据
``` python
import time,sys,queue
from multiprocessing.managers import BaseManager

class QueueManager(BaseManager):
    pass

QueueManager.register('get_task_queue')
QueueManager.register('get_result_queue')

server_addr = '127.0.0.1'
print('Connect to server %s...' % server_addr)
m = QueueManager(address=(server_addr,5000),authkey=b'abc')
m.connect()
task = m.get_task_queue()
result = m.get_result_queue()
for i in range(10):
    try:
        n = task.get(timeout=1)
        print('run task %d  %d...' % (n,n))
        r = '%d  %d = %d' % (n,n,n*n)
        time.sleep(1)
        result.put(r)
    except queue.Empty:
        print('task queue is empty.')
print('worker exit.')
```
先启动masterprocess.py，然后启动workprocess.py，可以看到效果
谢谢关注我的公众号
![1.jpg](1.jpg)