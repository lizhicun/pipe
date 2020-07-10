## 管道
### 由来
管道的发名者叫，Malcolm Douglas McIlroy，他也是Unix的创建者，是Unix文化的缔造者之一。下面是管道在1964年10月11日，出现的第一个打印稿。
![Image text](https://github.com/lizhicun/pipe/blob/master/src/pic-3.png)
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
![Image text](https://github.com/lizhicun/pipe/blob/master/src/vfs-2.png)

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
### 内核管道的实现
  linux下的进程的用户态地址空间都是相互独立的，因此两个进程在用户态是没法直接通信的，因为找不到彼此的存在；而内核是进程间共享的，因此进程间想通信只能通过内核作为中间人，来传达信息. 下图显示了两个进程间通过内核缓存进行通信的过程:
  ```
                  写 | 入         +-------+
    +--------------+------------<       |
    |              |            | 进程1 |
+---v----+         |            |       |
|        |         |            +-------+
| 缓 存  |     内  |  用
| (page) |     核  |  户
|        |         |  态
+---v----+         |            +-------+
    |              |            |       |
    |              |            | 进程2 |
    +--------------+------------>       |
                读 | 取         +-------+
                   |
  ```
pipe的实现就是和上述图示一样，在pipefs文件系统的inode中有一个属性
```
struct pipe_inode_info *i_pipe;

//pipe_fs_i.h
struct pipe_inode_info {
    wait_queue_head_t wait;
    char *base;//指向管道缓存首地址
    unsignedint len;//管道缓存使用的长度
    unsignedint start;//读缓存开始的位置
    unsignedint readers;
    unsignedint writers;
    unsignedint waiting_writers;
    unsignedint r_counter;
    unsignedint w_counter;
    struct fasync_struct *fasync_readers;
    struct fasync_struct *fasync_writers;
}
```
  在一个进程中，向管道中写入数据时，其实就是写入这个缓存中；然后在另一个进程读取管道时，其实就是从这个缓存读取.
这个缓存也可以解释为什么管道是单通道的:
  因为只有一个缓存，如果是双通道，那么两个进程同时向这块缓存写数据时，这样会导致数据覆盖，即一个进程的数据被另一个进程的数据覆盖．而向套接字有读写缓存，因此套接字是双通道的

  接下来，从pipe函数开始，看看内核是如何创建管道的．pipe系统调用在内核对应的服务例程为sys_pipe，在sys_pipe函数中，接着调用do_pipe创建两个管道描述符，一个用于写，另一个用于读；接下来看下do_pipe都做了什么．
  #### do_pipe函数
```
error = -ENFILE;
f1 = get_empty_filp();
if (!f1)
	goto no_files;
f2 = get_empty_filp();
if (!f2)
	goto close_f1;

```
接着通过调用get_pipe_inode来实例化一个带有pipe属性的inode
```
structinode* pipe_new(structinode* inode)
{
	unsigned long page;
	// 申请一个内存页，作为pipe的缓存
	page = __get_free_page(GFP_USER);
	if (!page)
		return NULL;
	// 为pipe_inode_info结构体分配内存
	inode->i\_pipe = kmalloc(sizeof(structpipe_inode\_info), GFP_KERNEL);
	if (!inode->i_pipe)
		goto fail_page;	
	// 初始化pipe_inode_info属性
	init_waitqueue_head(PIPE_WAIT(*inode));
	PIPE_BASE(*inode) = (char*) page;
	PIPE_START(*inode) = PIPE_LEN(*inode) = 0;
	PIPE_READERS(*inode) = PIPE_WRITERS(*inode) = 0;
	PIPE_WAITING_WRITERS(*inode) = 0;
	PIPE_RCOUNTER(*inode) = PIPE_WCOUNTER(*inode) = 1;
	*PIPE_FASYNC_READERS(*inode) = *PIPE_FASYNC_WRITERS(*inode) = NULL;
	return inode;
	fail_page:
		free_page(page);
	return NULL;
}
//----------------------------------------------------------------
static struct inode * get_pipe_inode(void)
{
	//　从pipefs超级块中分配一个inode
	struct	inode *inode = new_inode(pipe_mnt->mnt_sb);
	if (!inode)
		goto fail_inode;
	// pipe_new函数主要用来为这个inode初始化pipe属性，就是pipe_inode_info结构体
	if(!pipe_new(inode))
		goto fail_iput;
	PIPE_READERS(*inode) = PIPE_WRITERS(*inode) = 1;
	inode->i_fop = &rdwr_pipe_fops;//设置pipefs的inode操作函数集合，rdwr_pipe_fops
	// 为结构体，包含读写管道所有操作
	inode->i_state = I_DIRTY;
	inode->i_mode = S_IFIFO | S_IRUSR | S_IWUSR;
	inode->i_uid = current->fsuid;
	inode->i_gid = current->fsgid;
	inode->i_atime = inode->i_mtime = inode->i_ctime = CURRENT_TIME;
	inode->i_blksize = PAGE_SIZE
	return inode;
}
```
在当前进程的files_struct结构中获取两个空的文件描述符，分别存储在i和j
```
error = get_unused_fd();
if (error < 0)
	goto close_f12_inode;
i = error;
error = get_unused_fd();
if (error < 0)
	goto close_f12_inode_i;
j = error;
```
下一步就是为这个inode分配dentry目录项，dentry主要用于将file和inode连接起来,以及设置f1和f2的vfsmnt,dentry,mapping属性
```
sprintf(name, "[%lu]", inode->i_ino);
this.name = name;
this.len = strlen(name);
this.hash = inode->i_ino; /* will go */
dentry = d_alloc(pipe_mnt->mnt_sb->s_root, &this);
if (!dentry)
	goto close_f12_inode_i_j;
dentry->d\_op = &pipefs_dentry_operations;
d_add(dentry, inode);
f1->f\_vfsmnt = f2->f\_vfsmnt = mntget(mntget(pipe_mnt));
f1->f\_dentry = f2->f_dentry = dget(dentry);
f1->f\_mapping = f2->f\_mapping = inode->i_mapping;
```
最后，针对读写file实例设置不同的属性，并且将两个fd和两个file实例关联起来
```
/* read file */
f1->f\_pos = f2->f_pos = 0;
f1->f\_flags = O_RDONLY;//f1这个file实例只可读
f1->f\_op = &read_pipe_fops;//这是这个可读file的操作函数集合结构体
f1->f\_mode = FMODE_READ;
f1->f_version = 0;
/* write file */
f2->f_flags = O_WRONLY;//f2这个file实例只可写
f2->f_op = &write_pipe_fops;//这是这个只可写的file操作函数集合结构体
f2->f_mode = FMODE_WRITE;
f2->f_version = 0;
fd_install(i, f1);//将i(fd)和f1(file)关联起来
fd_install(j, f2);// 将j(fd)和f2(file)关联起来
fd[0] = i;
fd[1] = j;
return 0;
```
总结上述过程
*实例化两个空file结构体；
*创建带有pipe属性的inode结构；
*在当前进程文件描述符表中找出两个未使用的文件描述符;
*为这个inode分配dentry结构体，关联file和inode;
*针对可读和可写file结构，分别设置相应属性，主要是操作函数集合属性；
*关联文件描述符和file结构
*将两个文件描述符返回给用户

#### pipe读
  从虚拟文件系统知道，当用户态调用read函数时，对应于内核态sys_read，然后在sys_read函数中调用vfs_read函数，在vfs_read函数中调用file->f_op->read，由上述do_pipe函数可以知道，pipefs的read(file)实例对应的file->f_op为read_pipe_fpos，这个read_pipe_fpos结构体定义如下:
  ```
  struct file_operations read_pipe_fops = {
	.llseek		= no_llseek,
	.read		= pipe_read,
	.readv		= pipe_readv,
	.write		= bad_pipe_w,
	.poll		= pipe_poll,
	.ioctl		= pipe_ioctl,
	.open		= pipe_read_open,
	.release	= pipe_read_release,
	.fasync		= pipe_read_fasync,
}
  ```
  因此，在vfs_read函数中调用的(pipe)file->f_op->read即为pipe_read函数，这个函数定义在fs/pipe.c文件中
  ```
  static ssize_t
pipe_read(structfile *filp, char __user *buf, size_t count, loff_t *ppos)
{
  structiovec iov = &#123; .iov\_base = buf, .iov_len = count &#125;;
  return pipe_readv(filp, &iov, 1, ppos);
}
  ```
  pipe_read函数将用户程序的接收数据缓冲区和大小转换为iovec结构，然后调用pipe_readv函数从缓冲区获取数据;在pipe_readv函数中，最主要部分如下:
  ```
  intsize = PIPE_LEN(*inode);
if (size) {
  // 获取管道缓冲区读首地址
  char *pipebuf = PIPE_BASE(*inode) + PIPE_START(*inode);
  // 缓冲区可读最大值=PIPE\_SIZE - PIPE\_START(inode)
  ssize_t chars = PIPE_MAX_RCHUNK(*inode);
  // 下面两个if语句用于比较缓冲区可读最大值，缓冲区数据长度以及
  // 用户态缓冲区的长度，取最小值
  if (chars > total_len)
  	chars = total_len;
  if (chars > size)
  	chars = size;
  // 调用如下函数把数据拷贝到用户态
  if (pipe_iov_copy_to_user(iov, pipebuf, chars)) {
    if (!ret) ret = -EFAULT;
      break;
  }
  ret += chars;
  // 更新缓冲区读首地址
  PIPE_START(*inode) += chars;
  // 对缓冲区长度取模
  PIPE_START(*inode) &= (PIPE_SIZE - 1);
  // 更新缓冲区数据长度
  PIPE_LEN(*inode) -= chars;
  // 更新用户态缓冲区长度
  total_len -= chars;
  do_wakeup = 1;
  if (!total_len)
    break;	/* 如果用户态缓冲区已满，则读取成功 */
}
  ```
  上述代码是在一个循环中，直到用户态缓冲区已满，或者管道缓冲区全部数据读取完毕；当然这还涉及到如果缓冲区为空，则当前进程阻塞(切换到其他进程)等等；我们来看下pipe_iov_copy_to_user函数
  ```
  static inline int
pipe_iov_copy_to_user(struct iovec *iov, constvoid *from, unsignedlong len)
{
    unsignedlongcopy;
    while (len > 0) {
      while (!iov->iov_len)
      	iov++;
      copy = min_t(unsignedlong, len, iov->iov_len);
      if (copy_to_user(iov->iov_base, from, copy))
          return -EFAULT;
      from += copy;
      len -= copy;
      iov->iov_base += copy;
      iov->iov_len -= copy;
    }
    return0;
}
  ```
  pipe的写过程其实就是和read的过程相反，首先也是通过系统调用嵌入内核write->sys_write->vfs_write,在vfs_write函数中调用file->f_op->write函数，而这个函数对应管道写file实例的pipe_write函数。后面的过程就是将用户态缓冲区的数据拷贝到内核管道缓冲区，这里不做赘述。
