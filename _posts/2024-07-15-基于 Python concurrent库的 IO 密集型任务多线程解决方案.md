---
layout:     post   				    # 使用的布局（不需要改）
title:      基于 Python concurrent库的 IO 密集型任务多线程解决方案 				# 标题 
subtitle:   基于 Python concurrent库的 I/O 密集型任务多线程解决方案       # 副标题
creat date:       2024-07-13 				# 时间
newest date:       2025-1-26
author:     G.h. Qi 					# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - 技术
    - Python
    - 多线程
---

# 前言     
> 预计阅读时长: 2 min       
> 原文链接：[基于 Python `concurrent.futures` 库的 I/O 密集型任务多线程解决方案](https://chyiever.github.io/2024/07/15/%E5%9F%BA%E4%BA%8E-Python-concurrent%E5%BA%93%E7%9A%84-IO-%E5%AF%86%E9%9B%86%E5%9E%8B%E4%BB%BB%E5%8A%A1%E5%A4%9A%E7%BA%BF%E7%A8%8B%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88/)   
> 原作者：Gh Qi ，如需引用转载，请注明来源。

# 1. 写在前面


## 1.1 多线程与多进程
在Python中，多线程和多进程的区别主要体现在以下几个方面：

(1). **全局解释器锁（GIL）**：
   - **多线程**：由于Python的GIL，多线程在CPU密集型任务中无法真正实现并行。GIL限制了同一时间只有一个线程执行Python字节码。
   - **多进程**：每个进程都有自己的Python解释器和GIL，因此可以实现真正的并行，适合CPU密集型任务。

(2). **模块支持**：
   - **多线程**：使用`threading`模块来实现多线程编程。适合I/O密集型任务，如网络请求和文件操作。
   - **多进程**：使用`multiprocessing`模块来实现多进程编程。适合需要并行处理的CPU密集型任务。

(3). **内存使用**：
   - **多线程**：线程共享进程的内存空间，因此内存使用较少。
   - **多进程**：每个进程有独立的内存空间，内存开销较大。

(4). **数据共享**：
   - **多线程**：共享数据简单，但需要使用锁等同步机制来避免资源竞争。
   - **多进程**：数据共享复杂，通常通过队列、管道或共享内存来实现。

(5). **稳定性**：
   - **多线程**：一个线程崩溃可能影响整个进程。
   - **多进程**：一个进程崩溃不会影响其他进程。

## 1.2 Python-concurrent库
在Python中，如果需要并行处理CPU密集型任务，推荐使用多进程。而对于I/O密集型任务，多线程可以更有效地提高性能。

在Python中，`concurrent.futures`库提供了一种高级接口来实现多线程和多进程编程。使用`ThreadPoolExecutor`可以方便地进行多线程编程。以下是一些关键点和示例：

### `concurrent.futures`库简介 
 
`ThreadPoolExecutor`：用于管理线程池，适合I/O密集型任务。
`ProcessPoolExecutor`：用于管理进程池，适合CPU密集型任务。

**本文主要针对I/O密集型任务，介绍基于ThreadPoolExecutor的多线程解决方案。**
  
# 2. 基于ThreadPoolExecutor的多线程解决方案

## 2.1 executor.map()函数简介

`executor.map` 是 `concurrent.futures` 模块中 `ThreadPoolExecutor` 和 `ProcessPoolExecutor` 的一个便捷方法，它用于将一个函数应用于可迭代对象的每个元素上，并收集结果。下面是 executor.map 函数参数的官方定义：
```python
executor.map(func, *iterables)
--func: 要应用于每个输入项的函数
--*iterables: 一个或多个可迭代对象(executor.map 将 func 应用于这些可迭代对象的元素)
```

用人话讲：`executor.map(func,*iterables)`的第一个参数是函数名，第二个参数中的 **iterables** 是一个列表，里面存放的是 **func** 函数的参数。 **func** 函数需要几个参数，列表 **iterables** 中就有几个元组，每个元组包含了所有线程所需的该参数的值。

**例如**：定义一个求两数之和函数：`sum(a,b)`。我想求1和2的和，3和4的和，5和6的和，这件事安排给3个线程来干，那么`func=sum`，`iterables=[(1, 3, 5), (2, 4, 6)]`。

```python
data_to_process = [(1, 2), (3, 4), (5, 6)]
iterables = *zip(*data_to_process)  # 第二个参数

print(iterables)    # 打印第二个参数
```

输出为：

    (1, 3, 5) (2, 4, 6)

```python
executor.map(sum,iterables) # 多线程求和
``` 

`iterables` 里面给定参数组数可以低于max_workers，那么并不是所有的线程都会被分配工作。线程池中的线程会等待直到有任务可做，或者直到所有任务完成。当然也可以高于max_workers，那就只能排队干活了。

那么你可能就会问了，我的函数执行完之后还有个`return`呢，你这么搞，我返回的值去哪找呢？

`executor.map` 返回的是一个可迭代对象，它按顺序提供了 func 应用于每个输入项的结果。在 Python 3.7 之前，它返回一个列表；从 Python 3.7 开始，它返回一个 concurrent.futures._base._as_completed 对象，这个对象是可迭代的，但不是列表。

    <generator object Executor.map.<locals>.result_iterator at 0x000001FB535D72E0>

要打印每个线程执行函数的输出结果，可以迭代 executor.map 返回的对象；也可以使用`list()`将其转化为一个列表。

```python
# 方法1  

with ThreadPoolExecutor(max_workers=5) as executor:
    results = executor.map(process_data, *zip(*data_to_process))
print(results)

#返回一个 concurrent.futures._base._as_completed 对象，这个对象是可迭代的

for result in results:
    print(result)
```
输出：

    <generator object Executor.map.<locals>.result_iterator at 0x000002A7435F3790>
    3
    7
    11

```python
# 方式2  

with ThreadPoolExecutor(max_workers=5) as executor:
    results = executor.map(process_data, *zip(*data_to_process))
    results = list(results)
print(results)
```
输出：


    
    [3, 7, 11]


   
完整代码：
```python
from concurrent.futures import ThreadPoolExecutor

# ---------------定义要执行的函数---------------

def process_data(arg1, arg2): 
    # 这里可以是任何需要两个参数的逻辑

    return arg1 + arg2

# ---------------创建数据列表------------------  

# 我想求1和2的和，3和4的和，5和6的和，参数如下：  

data_to_process = [(1, 2), (3, 4), (5, 6)]

# 使用 zip 将 data_to_process 中的元组解包为两个参数

iterables = *zip(*data_to_process)

# iterables=[ (1, 3, 5), (2, 4, 6)]

# -------创建线程池（最大线程数不超过5个）-------    

with ThreadPoolExecutor(max_workers=5) as executor: 
    results = executor.map(process_data,iterables)  # 多线程求和

    results = list(results) # 获取结果  
    
# --------------打印每个线程的输出结果-----------  

for result in results: 
    print(result)

```
## 2.2 示例

### 问题描述

我有很多地震数据，每个数据文件时长1小时，格式为.txt文件，每个月的数据存到一个文件夹中（例如‘202405’、‘202406’...）。现在我想把这些文件批量转化为.mseed格式，对于每个文件，涉及到的操作主要是：根据txt文件路径，读取txt文件中的数据；利用Obspy库创建Stream对象，定义头文件、数据流；存储该Stream对象为.mseed格式到本地。

### 编程实现

```python
'''
程序功能：

  采用并行的方式，读取多个月的数据（每个月一个文件夹），将数据转换为mseed格式，并保存到指定文件夹中（每月一个文件夹，需提前创建好）。

Author: guoheng qi
Date: 2024/7/20
'''
from obspy import Stream,Trace
import numpy as np
import os
from concurrent.futures import ThreadPoolExecutor

def get_all_files(dir_path):
    """获取目录下所有文件的路径"""  
    file_paths = []
    for root, _, files in os.walk(dir_path):
        for file in files:
            file_paths.append(os.path.join(root, file))
    return file_paths

def read_file(file_path):
    """读取文件并打印文件名"""  
    print(f"Reading file: {file_path}")

    try:
        data = np.loadtxt(file_path)
        st = Stream()
        for i in range(16):
            tr = Trace(data=data[:,i])
            tr.stats.channel='{:02d}'.format(i+1) 
            #!!!!!!!!!!!!!!!!!!!!!  need to modify  !!!!!!!!!!!!!!!!!!!!!

            tr.stats.starttime=file_path[-22:-4]
            tr.stats.sampling_rate=100
            #!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

            st +=tr
        st.write(r'..\mseed'+file_path[1:-4]+'.mseed',format='MSEED')
        
    except:
        print('error')

def main():

    # 定义6个文件夹路径

    dir_paths = [r'.\202402', r'.\202403',r'.\202401' ]

    # 获取所有文件路径 
    all_files = [] 
    for dir_path in dir_paths: 
        all_files.extend(get_all_files(dir_path))

    # 定义线程数量

    num_threads = 15
    
    # 使用 ThreadPoolExecutor 进行并行处理

    with ThreadPoolExecutor(max_workers=num_threads) as executor: 
        # 提交任务

        executor.map(read_file, all_files) 

if __name__ == "__main__":
    main()

```

更多开源干货，请关注我的GITHUB账号：https://github.com/chyiever。

# 小结

使用`concurrent.futures`库可以大大简化多线程编程，尤其适合需要并发处理的I/O密集型任务。
