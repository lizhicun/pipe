## 管道
### 由来
管道的发名者叫，Malcolm Douglas McIlroy，他也是Unix的创建者，是Unix文化的缔造者之一。下面是管道在1964年10月11日，出现的第一个打印稿。
```
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
```
### 内容概要
* 管道的使用
* 虚拟文件系统
* 内核态管道的实现
* 总结

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
```
#include<iostream>
#include<unistd.h>
#include<stdlib.h>
using namespace std;

// parent process

int main() {
    int fd[2];
	int ret = pipe(fd); // fd[1]->fd[0]
	if (ret == -1) {
		cout << "pipe error";
		exit(-1);
	}
	int pid = fork();
	if (pid < 0) {
		cout << "fork error";
	}
	if (pid == 0) {
		close(fd[1]);
		dup2(fd[0], STDIN_FILENO);
		ret = execl("./child2", "child2", "", NULL);
		if (ret == -1) {
			cout << "execpv error";
			exit(-1);
		}
	} else {
		close(fd[0]);
		dup2(fd[1], STDOUT_FILENO);
		char buf[1024];
		int n = read(STDIN_FILENO, buf, 1024);
		if (n < 0) {
			cout << "read error";
			exit(-1);
		}
		write(STDOUT_FILENO, buf, n);
	}
	return 0;
}

```
```
#include<iostream>
#include<stdlib.h>
#include<unistd.h>
using namespace std;

// child process

#define BUFSIZE 1024

char ToUp(char c) {
	if (c > 'a' && c < 'z') {
		c = c - 32;
	}
	return c;
}

int main() {
	char buf[BUFSIZE];
	int n = read(STDIN_FILENO, buf, BUFSIZE);
	if (n < 0) {
		cout << "read error";
	}
	for (int i = 0; i < n; i++) {
		buf[i] = ToUp(buf[i]);
	}
	cout << buf;
	return 0;
}
```
### 虚拟文件系统
  虚拟文件系统是linux内核四大模块之一，我们知道linux下面everything is file。例如磁盘文件，管道，套接字，设备等等；我们都可以通过read和write函数来读取上述文件的数据；为了支持这一特性，linux引入虚拟文件系统，就是通过一层文件系统虚拟层，屏蔽不同文件系统的差异，实现相同的函数接口操作；linux支持非常多的文件系统，我们可以通过查看
```
cat /proc/filesystems
```
![Image text](https://github.com/lizhicun/pipe/blob/master/src/vfs.png)
在用户态调用read函数读取一个文件描述符时，主要过程如下:
```
asmlinkage ssize_t sys_read(unsigned int fd, char __user * buf, size_t count)
{
    struct file *file;
    ssize_t ret = -EBADF;
    int fput_needed;
    // fget_light函数从当前进程的文件描述符表中，通过文件描述符，获取file结构体
    file = fget_light(fd, &fput_needed);
    if (file) {
    	loff_t pos = file_pos_read(file);//获取读取文件的偏移量
        ret = vfs_read(file, buf, count, &pos);//调用虚拟文件系统调用层
    	file_pos_write(file, pos);//　更新当前文件的偏移量
    	fput_light(file, fput_needed);//　更新文件的引用计数
    }
    return ret;
}
```
vf_read虚拟层调用的过程
```
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
	struct inode *inode = file->f_dentry->d_inode;
	ssize_t ret;
 	if (!(file->f_mode & FMODE_READ))
    		return -EBADF;
  	if (!file->f_op || (!file->f_op->read && !file->f_op->aio_read))
    		return -EINVAL;
  	ret = locks_verify_area(FLOCK_VERIFY_READ, inode, file, *pos, count);
  	if (!ret) {
    		ret = security_file_permission (file, MAY_READ);
    	if (!ret) {
      		if (file->f_op->read)
        		// 进入具体文件系统
        		ret = file->f_op->read(file, buf, count, pos);
      		else
        		ret = do_sync_read(file, buf, count, pos);
        	if (ret > 0)
			 dnotify_parent(file->f_dentry, DN_ACCESS);
	}
  }
  return ret;
}
```
  每种文件系统的file->f_op->read是不一样的，像基于磁盘的文件系统，file->f_op->read函数是先到缓存缓存获取数据，如果缓存没有数据，则到磁盘获取；基于内存的文件系统，file->f_op->read则是直接在内核缓存获取数据，而不会到磁盘获取数据;
  file->f_op主要是从inode->i_fop中获得，因此对于不同的文件系统，inode也结构也是有区别的．当创建一个inode时，针对不同的文件系统需要设置不同的属性，最主要就是各种操作函数指针结构体，例如inode->i_op和inode->i_fop；这样不同的文件系统，就可以在f->f_op->read调用中，实现不同的操作.
