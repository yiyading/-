# 一、什么是共享内存

顾名思义，共享内存就是允许两个不相关的进程访问同一个逻辑内存。共享内存是在两个正在运行的进程之间共享和传递数据的一种非常有效的方式。不同进程之间共享的内存通常安排为同一段物理内存。进程可以将同一段共享内存连接到它们自己的地址空间中，所有进程都可以访问共享内存中的地址，就好像它们是由用C语言函数malloc()分配的内存一样。而如果某个进程向共享内存写入数据，所做的改动将立即影响到可以访问同一段共享内存的任何其他进程。<br>

特别提醒：共享内存并未提供同步机制，也就是说，在第一个进程结束对共享内存的写操作之前，并无自动机制可以阻止第二个进程开始对它进行读取。所以我们通常需要用其他的机制来同步对共享内存的访问，例如信号量。<br>

# 二、共享内存的使用
与信号量一样，在Linux中也提供了一组函数接口用于使用共享内存，而且使用共享内存的接口还与信号量的非常相似，而且比使用信号量的接口来得简单。它们声明在头文件#include<sys/shm.h>中。<br>

## 1.shmget()函数
成功返回共享内存标识符，失败返回-1<br>

该函数用来创建共享内存，它的原型为：<br>
```c
    int shmget(key_t key, size_t size, int shmflg);
```
第一个参数，与信号量的semget函数一样，程序需要提供一个参数key（非0整数），它有效地为共享内存段命名，shmget()函数成功时返回一个与key相关的共享内存标识符（非负整数），用于后续的共享内存函数。调用失败返回-1。<br>

不相关的进程可以通过该函数的返回值访问同一共享内存（key值），它代表程序可能要使用的某个资源，程序对所有共享内存的访问都是间接的，程序先通过调用shmget()函数并提供一个键，再由系统生成一个相应的共享内存标识符（shmget()函数的返回值），只有shmget()函数才直接使用信号量键，所有其他的信号量函数使用由semget函数返回的信号量标识符。

第二个参数，size以字节为单位指定需要共享的内存容量。<br>

第三个参数，shmflg是权限标志，它的作用与open函数的mode参数一样，如果要想在key标识的共享内存不存在时，创建它的话，可以与IPC_CREAT做或操作。共享内存的权限标志与文件的读写权限一样，举例来说，0644,它表示允许一个进程创建的共享内存被内存创建者所拥有的进程向共享内存读取和写入数据，同时其他用户创建的进程只能读取共享内存。<br>

## 2.shmat()函数  --at：attach
把内存段映射到用户进程空间，就是把共享内存连接到当前进程的地址空间；成功返回指向共享内存第一个字节的指针，失败返回一个指向-1的指针。<br>

第一次创建完共享内存时，它还不能被任何进程访问，shmat()函数的作用就是用来启动对该共享内存的访问，并把共享内存连接到当前进程的地址空间。它的原型如下：<br>
```c
    void *shmat(int shm_id, const void *shm_addr, int shmflg);
```
第一个参数，shm_id是由shmget()函数返回的共享内存标识。<br>

第二个参数，shm_addr指定共享内存连接到当前进程中的地址位置，通常为空，表示让系统来选择共享内存的地址。<br>

第三个参数，shm_flg是一组标志位，通常为0。<br>

功时返回一个指向共享内存第一个字节的指针，如果调用失败返回-1。<br>

## 3.shmdt()函数  --dt：detach
成功返回0，失败返回-1<br>

该函数用于将共享内存从当前进程中分离。注意，将共享内存分离并不是删除它，只是使该共享内存对当前进程不再可用。它的原型如下：<br>
```c
int shmdt(const void *shmaddr);
```
参数shmaddr是shmat()函数返回的地址指针，调用成功时返回0，失败时返回-1。<br>

## 4.shmctl()函数  -- ctl：control
成功返回0，失败返回-1.<br>

与信号量的semctl()函数一样，用来控制共享内存，它的原型如下：<br>
```c
int shmctl(int shm_id, int command, struct shmid_ds *buf);
```
第一个参数，shm_id是shmget()函数返回的共享内存标识符。<br>

第二个参数，command是要采取的操作，它可以取下面的三个值：<br>
1.PC_STAT：把shmid_ds结构中的数据设置为共享内存的当前关联值，即用共享内存的当前关联值覆盖shmid_ds的值。<br>
2.IPC_SET：如果进程有足够的权限，就把共享内存的当前关联值设置为shmid_ds结构中给出的值。<br>

第三个参数，buf是一个结构指针，它指向共享内存模式和访问权限的结构。<br>

shmid_ds结构 至少包括以下成员：<br>
```c
struct shmid_ds
{
    uid_t shm_perm.uid;
    uid_t shm_perm.gid;
    mode_t shm_perm.mode;
};
```

# 三、使用共享内存进行进程间通信
以两个不相关的进程来说明进程间如何通过共享内存来进行通信。其中一个文件shmread.c创建共享内存，并读取其中的信息，另一个文件shmwrite.c向共享内存中写入数据。<br>

为了方便操作和数据结构的统一，为这两个文件定义了相同的数据结构，定义在文件shmdata.h中。结构shared_use_st中的written作为一个可读或可写的标志，非0：表示可读，0：表示可写，text则是内存中的文件。

[shmadata.h](https://github.com/yiyading/Embedded-software/blob/master/%E5%85%B1%E4%BA%AB%E5%86%85%E5%AD%98%E9%80%9A%E4%BF%A1/shmdata.h)

[shmread.c](https://github.com/yiyading/Embedded-software/blob/master/%E5%85%B1%E4%BA%AB%E5%86%85%E5%AD%98%E9%80%9A%E4%BF%A1/shmread.c)

[shmwrite.c](https://github.com/yiyading/Embedded-software/blob/master/%E5%85%B1%E4%BA%AB%E5%86%85%E5%AD%98%E9%80%9A%E4%BF%A1/shmwrite.c)


# 四、运行
## 1.可以采用makefile文件进行运行

## 2.使用gcc编译
```c
gcc -o shmread.exe shmread.c -lm
gcc -o shmwrite.exe shmwrite.c -lm
```
server：<br>
```c
./shmread.exe
```
client：<br>
```c
./shmwrite.ext
```

# 五、分析
程序shmread创建共享内存，然后将它连接到自己的地址空间。在共享内存的开始处使用了一个结构struct_use_st。该结构中有个标志written，当共享内存中有其他进程向它写入数据时，共享内存中的written被设置为0，程序等待。当它不为0时，表示没有进程对共享内存写入数据，程序就从共享内存中读取数据并输出，然后重置设置共享内存中的written为0，即让其可被shmwrite进程写入数据。<br>

程序shmwrite取得共享内存并连接到自己的地址空间中。检查共享内存中的written，是否为0，若不是，表示共享内存中的数据还没有被read完，则等待其他进程读取完成，并提示用户等待。若共享内存的written为0，表示没有其他进程对共享内存进行读取，则提示用户输入文本，并再次设置共享内存中的written为1，表示写完成，其他进程可对共享内存进行读操作。<br>

# 六、关于安全性讨论
这个程序是不安全的，当有多个程序同时向共享内存中读写数据时，问题就会出现。可能你会认为，可以改变一下written的使用方式，例如，只有当written为0时进程才可以向共享内存写入数据，而当一个进程只有在written不为0时才能对其进行读取，同时把written进行加1操作，读取完后进行减1操作。这就有点像文件锁中的读写锁的功能。咋看之下，它似乎能行得通。但是这都不是原子操作，所以这种做法是行不能的。试想当written为0时，如果有两个进程同时访问共享内存，它们就会发现written为0，于是两个进程都对其进行写操作，显然不行。当written为1时，有两个进程同时对共享内存进行读操作时也是如些，当这两个进程都读取完是，written就变成了-1。<br>

要想让程序安全地执行，就要有一种进程同步的进制，保证在进入临界区的操作是原子操作。例如，可以使用信号量来进行进程的同步。因为信号量的操作都是原子性的。


# 七、使用共享内存的优缺点
优点：我们可以看到使用共享内存进行进程间的通信真的是非常方便，而且函数的接口也简单，数据的共享还使进程间的数据不用传送，而是直接访问内存，也加快了程序的效率。同时，它也不像匿名管道那样要求通信的进程有一定的父子关系。<br>

缺点：共享内存没有提供同步的机制，这使得我们在使用共享内存进行进程间通信时，往往要借助其他的手段来进行进程间的同步工作。<br>
