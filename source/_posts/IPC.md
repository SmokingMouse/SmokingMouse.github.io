---
title: IPC

categories: Unix

tags: [Unix,APUE,IPC]

data: 2019-04-08

---

# IPC

### 1. 匿名管道

```c
#include<unistd.h>
int pipe(int fd[2]);
		//返回：若成功则为0，若失败返回-1
```

匿名管道仅能用于存在关系的进程间，并且匿名管道是半双工的，也就是说数据在管道的流动是单向的。当需要两个双向数据流时，必须创建两个管道。并且匿名管道是随进程持续的，也就是说当进程结束时，这种形式的IPC结构便不再存在，这有别于我们后面介绍的其他的IPC结构。

标准I/O函数库提供了<i>popen</i>函数，它创建一个管道并启动另一个进程，该进程要么从该管道读出标准输入，要么往该管道写入标准输入。

```c
#include<stdio.h>
FILE *popen(const char *command,const char *type);
			//返回：若成功则为文件指针，若出错则为NULL
int pclose(FILE *stream);
			//返回：若成功则为shell的终止状态，若出错则为-1
```

其中<i>command</i>时一个<i>shell</i>命令行。由<i>sh</i>程序处理，因此PATH环境变量可用于定位该<i>command</i>。<i>popen</i>在调用进程和所指定的命令之间创建一个管道。由<i>popen</i>返回的值是一个标准I/O FILE的指针，该指针可理解为对调用进程关心的管道一端。比方说当命令是<i>cat /etc/shadow</i>，type是<i>"r"</i>，意味着子进程将执行该命令，并将结果写进管道。对于父进程也就是调用进程来说，它能通过读管道的另一端得到这些结果，而这个所谓的管道另一端就是返回的<i>FILE*</i>.

而<i>pclose</i>则关闭由<i>popen</i>创建的标准I/O流，等待其中命令终止，然后返回shell的最终状态。

后面将会在<i>github</i>上具体实现这两个函数。

### 2. 命名管道(FIFO)

我们可以看到，匿名管道没有名字，因此它们只能用于有一个公共祖先进程的各个进程之间。无法在无亲缘关系的两个进程间创建一个管道。

FIFO指代<b>先进先出</b>，类似于管道，它也是一个单向的数据流。不同于管道的是，每个FIFO有一个路径名相关联，类似于一个文件，从而允许不相关进程访问同一个FIFO。

```c
#include<sys/types.h>
#include<sys/stat.h>

int mkfifo(const char *pathname, mode_t mode);
			//返回：若成功则为0，若出错则为-1
```

pathname是普通的Unix路径名，是该FIFO的名字。mode参数指定文件权限位，类似于open的参数。

<i>mkfifo</i>函数已隐含指定<i>O_CREATE|O_EXCL</i>。也就是说要么创建一个新的FIFO，要么返回一个EEXIST错误。

对于FIFO的打开或者读或者写，不能既读又写，因为它是半双工的。对FIFO的写总是往末尾添加数据，对它们的读总是从开头返回数据。对FIFO调用<i>lseek</i>将返回ESPIPE错误。

管道在所有进程最终都关闭它之后自动消失。FIFO的名字则只有通过调用unlink才从文件系统删除。内核为管道和FIFO维护了引用计数器，它是访问同一个描述符的个数。这种引用计数器的存在有一个性质，假设在还存在进程引用该文件时，调用unlink删除该文件，不会对现在还在使用该文件的进程造成影响，对文件的删除将推迟到引用计数器为0的时刻。这有别于System V对于部分IPC方式的处理。

值得注意的是，默认情况下，对于FIFO的读(写)打开会阻塞到有写(读)的打开。倘若对于一个没有读者的FIFO写，会长生SIGPIPE信号。对于没有写着的FIFO读会检查可用数据量，只返回可用数据。另外，对于管道的写(包括匿名管道和FIFO)在一次写入内容小于PIPE_BUF时会保证原子性。因此，在以非阻塞的方式向管道中写小于PIPE_BUF的数据时，倘若空间不够，为了保证原子性则会以错误的形式告诉进程以后再试，不会存在部分写情况。有别于我们习惯理解的非阻塞写。倘若大于PIPE_BUF，这时原子性本就得不到保证，所以进程会写入能容纳的数据。



### 3. POSIX 消息队列

消息队列可以认为是一个消息链表。有足够写权限的线程可以往队列中放置消息，有足够读权限的线程可以从队列中取走消息。每个消息就是一个<b>记录</b>(有点类似于UDP数据报，有边界)。由发送者赋予优先级。在某个进程往队列中写入消息前，不需要另外某个进程在该队列上等待消息。

消息队列具有<b>内核的持续性</b>，也就是说在不显示删除的情况下，即使当前唯一进程退出后，在后续仍然可以由其他进程读入该队列的消息。

<i>mq_open</i>函数创建一个新的消息队列或打开一个已存在的消息队列。

```c
#include<mqueue.h>
mqd_t mq_open(const char *name, int oflag, /*mode_t mode, struct mq_attr *attr */);
			//返回：若成功则为消息队列描述符，若出错则为-1.
```

已打开的消息队列由<i>mq_close</i>关闭。<b>关闭不等于删除</b>。

```c
#include<mqueue.h>
int mq_close(mqd_t mqdes);
			//返回：若成功则为0，若出错则为-1
```

要从系统中删除用作<i>mq_open</i>第一个参数的某个name，必须调用<i>mq_unlink</i>。

```c
#include<mqueue.h>
int mq_unlink(const char *name);
			//返回，若成功则为0，若出错则为-1
```

正如前面谈到过的引用计数，每个消息队列有一个保存其当前打开着描述符数的引用计数器(有点像文件)，因而本函数能够实现类似于unlink函数删除一个文件的机制：当一个消息队列的引用计数仍大于0时，其name就能删除，但该队列的析构要到最后一个<i>mq_close</i>发生时才进行。

```c
struct mq_attr{
    long mq_flags; /* message queue flag: 0, O_NONBLOCK */
    long mq_maxmsg; /* max number of messages allowed on queue */
    long mq_msgsize; /* max size of a message (in bytes) */
    long mq_curmsgs; /* number of messages currently on queue */
};

#include<mqueue.h>
int mq_getattr(mqd_t mqdes, struct mq_attr *attr);
int mq_setattr(mqd_t mqdes, const struct mq_attr *attr, struct mq_attr *oattr);
			//返回：若成功则为0，若出错则为-1
```

<i>mq_setattr</i>只能设置<i>mq_flags</i>，其他参数需在创建时指定。

下面两个函数用于放置消息和取走消息。每个消息有个优先级，小于<i>MQ_PRIO_MAX</i>的无符号整数。

```c
#include<mqueue.h>
int mq_send(mqd_t mqdts, const char *ptr, size_t len, unsigned int prio);
			//返回：若成功则为0，若出错则为-1
ssize_t mq_receive(mqd_t mqdes, char *ptr, size_t len, unsigned int *priop);
			//返回：若成功则为消息字节数，若出错则为-1
```



### 4. POSIX 信号量

<b>信号量</b>是一种用于提供不同进程间或一个给定进程的不同线程间同步手段的原语。POSIX信号量有两种。

- POSIX有名信号量：使用POSIX IPC名字进行标识，可用于进程或线程间的同步。
- POSIX基于内存的信号量：存放在共享内存区中，可用于进程或线程间的同步。

POSIX的有名信号量是由可能与文件系统中的路径对应的名字来标识的，但是并不要求它们真正存放在文件系统内的某个文件中。



函数<i>sem_open</i>创建一个新的有名信号量或打开一个已存在的有名信号量。有名信号量总是既可用于线程间同步，又可用于进程间的同步。

```c
#include<semaphore.h>
sem_t *sem_open(const char *name, int oflag, /* mode_t mode, unsigned int value */);
			//返回：若成功则为指向信号量的指针，若出错则为SEM_FAILED
```



使用<i>sem_open</i>打开的有名信号量，使用<i>sem_close</i>将其关闭。

```c
#include<semaphore.h>
int sem_close(sem_t *sem);
```

<i>Posix</i>有名信号量是<b>内核持续</b>的，即使当前没有进程打开着某个信号量，它的值仍然保持。



有名信号量使用<i>sem_unlink</i>从系统中删除。

```c
#include<semaphore.h>
int sem_unlink(const char *name);
			//返回：若成功则为0，若出错则为-1
```

类似于消息队列中的操作，不再赘述。



<i>sem_wait</i>函数测试所指定信号量的值，如果该值大于0，那就将它减1并立即返回。如果该值等于0，调用线程就被投入睡眠，直到该值变为大于0.

```c
#include<semaphore.h>
int sem_wait(sem_t *sem);
int sem_trywait(sem_t *sem);
			//返回：若成功则为0，若出错则为-1
```

<i>sem_trywait</i>于<i>sem_wait</i>可以类比为非阻塞read于阻塞read。如果被某个信号中断，<i>sem_wait</i>可能过早返回，返回错误为EINTR。



当线程使用完某个信号量时，调用<i>sem_post</i>。把指定信号量值加1，然后唤醒正在等待该信号量变为正数的任意线程。

```c
#include<semaphore.h>
int sem_post(sem_t *sem);
int sem_getvalue(sem_t *sem, int *valp);
			//返回：若成功则为0，若出错则为-1
```

<i>sem_getvalue</i>在由<i>valp</i>指向的整数中返回所指定信号量的当前值。如果该信号量当前已上锁，那么返回值或为0，或为某个负数，其绝对值为等待该信号量解锁的线程数。



前面提到的是<i>Posix</i>有名信号量。这些信号量由一个name参数标识，它通常指代文件系统中的某个文件。<i>Posix</i>也提供**基于内存**的信号量，它们由应用程序分配信号量的内存空间，然后由系统初始化它们的值。可以这样理解，信号量作为一种控制进程间同步的机制，它需要的是一种全局的环境。这种于所有进程的全局环境可以是文件系统，也可以是进程共享的内存。前者是有名信号量，后者是基于内存信号量。

```c
#include<semaphore.h>
int sem_init(sem_t *sem, int shared, unsigned int value);
			//返回：若出错则为-1
int sem_destroy(sem_t *sem);
			//返回：若成功则为0，若出错则为-1
```

如果<i>shared</i>为0，那么待初始化的信号量是在同一进程的各个线程间共享的，否则该信号量在进程间共享。基于内存的信号量至少具有随进程的持续性，真正的持续性取决于存放信号量的内存区类型。只要含有某个基于内存信号量的内存区有效，信号量就一直存在。

```
Destroying a semaphore that other processes or threads are currently blocked on produces undefined behavior.
Using a semaphore that has been destroyed produces undefined results, until the semaphore has been reinitialized using sem_init(3).
```



<b>补充：</b>生产者与消费者(多个)，其他方式的同步方式，多个缓冲区。



### 5. 共享内存区

共享内存区是可用IPC形式中最快的。一旦这样的内存区映射到共享它的进程的地址空间。这些进程间数据的传递就不再涉及内核。然而不同于管道、消息队列利用阻塞自带的同步机制，在共享内存区间实现同步就得用到我们前面提到的同步方式。

<i>mmap</i>函数把一个文件或一个<i>Posix</i>共享内存区对象映射到调用进程的地址空间。使用该函数有三个目的：

1. 使用普通文件以提供内存映射I/O；
2. 使用特殊文件以提供匿名内存映射；
3. 使用<i>shm_open</i>提供无亲缘关系进程间的<i>Posix</i>共享内存区。

```c
#include<sys/mman.h>
void *mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset);
		//返回：若成功则为被映射区的起始地址，若出错则为MAP_FAILED.
```

<i>len</i>为映射到调用进程地址空间中的字节数。内存映射区的保护由<i>prot</i>指定。

|    prot    |     说明     |
| :--------: | :----------: |
| PROT_READ  |   数据可读   |
| PROT_WRITE |   数据可写   |
| PROT_EXEC  |  数据可执行  |
| PROT_NONE  | 数据不可访问 |

flags必须指定MAP_SHARED或MAP_PRIVATE中的一个，可有选择或上MAP_FIXED。如果指定了MAP_PRIVATE，那么调用进程对被映射数据所作修改支队该进程可见，不修改底层支撑对象。(或是一个文件对象，或是一个<b>共享内存区对象</b>)。如果指定了MAP_SHARED，那么调用进程对被映射数据所作的修改对于共享该对象的所有进程都可见，而且确实改变了底层对象。MAP_FIXED说明<i>addr</i>不为空，由自己指定。



从进程地址空间删除映射关系，调用<i>munmap</i>。

```c
#include<sys/mman.h>
int munmap(void *addr, size_t len);
			//返回：若成功则为0，若出错则为-1.
```

如果映射区使用MAP_PRIVATE标志映射，进程做出的变动都会被丢弃。



有时候我们希望确信硬盘上的文件内容与内存映射区的内容一致，调用<i>msync</i>来执行这种同步。

```c
#include<sys/mman.h>
int msync(void *addr, size_t len, int flags);
			//返回：若成功则为0，若出错则为-1.
```

<i>addr</i>和<i>len</i>可以为映射内存一个子集。对于flags，MS_ASYNC和MS_SYNC必须指定一个。一旦写操作已由内核排入队列，MS_ASYNC即返回，而MS_SYNC则要等到写操作完成后才返回。

BSD提供<b>匿名内存映射</b>，把flags参数指定成MAP_SHARED | MAP_ANON，把<i>fd</i>指定为-1。offset参数被忽略。



为了在无亲缘关系进程间共享内存区，有如下方法。

- 内存映射文件。

- 共享内存区对象。

<i>Posix</i>共享内存区涉及以下两个步骤：

1. 指定一个名字参数调用<i>shm_open</i>，创建一个新的共享内存区对象或打开一个已存在的共享内存区对象。
2. 调用<i>mmap</i>把这个共享内存区映射到调用进程的地址空间。

```c
#include<sys/mman.h>
int shm_open(const char *name, int oflag, mode_t mode);
			//返回：若成功则为非负描述符，若出错则为-1.
int shm_unlink(const char *name);
			//返回：若成功则为0，若出错则为-1.
```

与<i>mq_open</i>和<i>sem_open</i>函数不同，mode参数必须指定。如果没有指定O_CREATE标志则可以为0。可能是因为对于虚拟内存来讲，段的访问权限是必须的吧。

<i>ftruncate</i>可用于设定共享内存区大小。

```c
#include<unistd.h>
int ftruncate(int fd, off_t length);
			//返回：若成功则为0，若出错则为-1.

#include<sys/types.h>
#include<sys/stat.h>
int fstat(int fd, struct stat *buf);
			//返回：若成功则为0，若出错则为-1.
```

