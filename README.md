# 管道
## 由来
管道的发名者叫，Malcolm Douglas McIlroy，他也是Unix的创建者，是Unix文化的缔造者之一。下面是管道在1964年10月11日，出现的第一个打印稿。
                       - 10 -
            Summary--what's most important.
    To put my strongest concerns into a nutshell:
1. We should have some ways of connecting programs like
garden hose--screw in another segment when it becomes when
it becomes necessary to massage data in another way.
This is the way of IO also.
2. Our loader should be able to do link-loading and
controlled establishment.
3. Our library filing scheme should allow for rather
general indexing, responsibility, generations, data path
switching.
4. It should be possible to get private system components
(all routines are system components) for buggering around with.
                                                M. D. McIlroy
                                                October 11, 1964

## 内容概要
### 1. 管道的使用
### 2. 虚拟文件系统
### 3. 内核态管道的实现
### 4. 总结

## 正文
### 管道的使用
```
cat /tmp/simlog.log | grep 'error'
```
这行命令的具体实现过程是：
* shell进程fork子进程，执行exec将```cat /tmp/simlog.log```载入内存
* cat进程中，用pipe定义出管道
* cat进程再fork出子进程,子进程将执行```grep error```
* cat进程中关闭管道读端，将cat进程的标准输出重定向到管道的写端
* grep进程中将管道的写端关闭，将标准输入重定向到管道的读端，再调用exec将grep进程载入内存

现在对上述过程模拟

