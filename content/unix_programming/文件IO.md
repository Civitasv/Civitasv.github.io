+++
title = "ch03: 文件 I/O"
date = "2023-05-16T11:01:38+08:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["unix", "i/o"]
keywords = ["unix", "笔记"]
description = "Unix 文件 I/O 相关函数记录"
showFullContent = false
readingTime = true
hideComments = false
color = "" #color from the theme settings
+++

## 文件描述符

对于内核而言，所有打开的文件都通过文件描述符引用。当打开一个现有文件，或者创建一个新文件时，内核向进程返回一个文件描述符。

按照惯例，文件描述符 0 与进程的标准输入关联，文件描述符 1 与进程的标准输出关联，文件描述符 2 与进程的标准错误关联。

From `<unistd.h>`:

```c
#define STDIN_FILENO    0 /* Standard input.  */
#define STDOUT_FILENO   1 /* Standard output.  */
#define STDERR_FILENO   2 /* Standard error output.  */
```

## open & openat

```c
#include <fcbtl.h>

int open(const char *path, int oflag, ...);

int openat(int fd, const char *path, int oflag, ...);
```

如果函数成功，则返回文件描述符，否则，返回 -1。

Example:

```c
#include <unistd.h>
#include <stdio.h>
#include <fcntl.h>

int main()
{
  #if 0
  {
    // An open and read example
    int ret = open("test.txt", O_RDONLY);

    if (ret) {
      // printf("Open Success: %d", ret);
      // operation on fd
      char buf[1024];
      ssize_t count = 0;
      while (count = read(ret, buf, 1024)) {
        if (count == -1) {
          perror("Read fail");
          return -1;
        }
        printf("%s", buf);
      }
    }
  }
  #endif

  {
    int dir = open("test", O_DIRECTORY);
    int fd = openat(dir, "test.txt", O_RDONLY);
    if (fd) {
      // printf("Open Success: %d", ret);
      // operation on fd
      char buf[1024];
      ssize_t count = 0;
      while (count = read(fd, buf, 1024)) {
        if (count == -1) {
          perror("Read fail");
          return -1;
        }
        printf("%s", buf);
      }
    }
  }

  return 0;
}
```

## creat

```c
#include <fcntl.h>

int creat(const char *path, mode_t mode);
```

该函数用于创建文件。

## close

```c
#include <unistd.h>

int close(int fd);
```

进程终止时，内核会自动关闭它所有打开的文件。

## lseek

每个打开的文件都有一个与其相关联的“当前文件偏移量”。它通常是一个非负整数，用以度量从文件开始处计算的字节数，当打开一个文件时，除非显示指定 O_APPEND 选项，否则该偏移量被设置为 0。

lseek 可以用来显式的设置文件的偏移量。

```c
#include <unistd.h>

off_t lseek(int fd, off_t offset, int whence);
```

- 若 whence 是 SEEK_SET，则设置该文件的偏移量为距文件开始处 offset 个字节
- 若 whence 是 SEEK_CUR，则设置该文件的偏移量为其当前值加 offset
- 若 whence 是 SEEK_END，则设置该文件的偏移量为文件长度加 offset

lseek 还可以用于判断所操作的文件是否可以设置偏移量，如果文件描述符指向的是一个管道、FIFO 或网络套接字，那么 lseek 返回 -1，并设置 errno 为 ESPIPE。

```c
if (lseek(STDIN_FILENO, 0, SEEK_CUR) == -1) {
  printf("Cannot seek\n");
} else {
  printf("Seek OK\n");
}
```

该程序用于检测标准输入能否被设置偏移量。

## read

```c
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t nbytes);
```

## write

```c
#include <unistd.h>

ssize_t write(int fd, const void *buf, size_t nbytes);
```

### echo example

```c
#include <fcntl.h>
#include <errno.h>
#include <unistd.h>
#include <stdio.h>

#define BUFFERSIZE 4096

int main()
{
  int n;
  char buf[BUFFERSIZE];

  while ((n = read(STDIN_FILENO, buf, BUFFERSIZE)) > 0) {
    if (write(STDOUT_FILENO, buf, n) != n)
      perror("write error");
  }

  if (n < 0)
    perror("read error");

  return 0;
}
```

**怎么选择 BUFFERSIZE?**

选择缓冲区大小时，一般设置为与磁盘快长度相等时，速度最快。

## 文件系统

Unix 使用了 *进程表->文件表->v 节点和 i 节点* 的三级结构。

文件描述符标志和文件状态标志在作用范围上，前者只作用于一个进程的一个描述符，而后者则作用于指向该给定文件表项的任何进程中的所有描述符。

当两个进程打开同一个文件时，会分别在文件表中创建文件表项，记录文件状态标志、当前文件偏移量以及 v 节点指针，所以不会出现冲突。

但是，当两个进程写同一个文件时，由于文件长度发生变化，会出现冲突，为了避免这种情况，需要使用原子操作。

## 原子操作

多进程操作中，如果两个进程打开了同一文件，可能出现冲突，这时需要使用原子操作。

### pread & pwrite

```c
#include <unistd.h>

ssize_t pread(int fd, void *buf, size_t nbytes, off_t offset);

ssize_t pwrite(int fd, const void *buf, size_t nbytes, off_t offset);
```

一般而言，原子操作指的是由多步组成的一个操作，如果该操作原子地执行，则要么执行完所有步骤，要么一步也不执行，不可能只执行所有步骤的一个子集。

## dup & dup2

```c
#include <unistd.h>

int dup(int fd);
int dup2(int fd, int fd2);
```

返回的新文件描述符与 fd 共享同一个文件表项。

## sync & fsync & fdatasync

延迟写：当我们向文件写入数据时，内核通常先将数据复制到缓冲区中，然后排入队列，晚些时候再写入磁盘。

为了保证磁盘上实际文件系统与缓冲区中内容的一致性，UNIX 系统提供了 sync, fsync 和 fdatasync 三个函数。

```c
#include <unistd.h>

int fsync(int fd);
int fdatasync(int fd);

void sync(void);
```

- `sync` 只是将所有修改过的块缓冲区排入写队列，然后就返回，它并不等待实际写磁盘操作的完成。
- `fsync` 只对由文件描述符 fd 指定的一个文件起作用，并且**等待**写磁盘操作结束才返回。
- `fdatasync` 类似于 `fsync`，但它只影响文件的数据部分，而除数据外，`fsync` 还会同步更新文件的属性。

## fcntl

fcntl 函数用于改变已经打开文件的属性。

```c
#include <fcntl.h>

int fcntl(int fd, int cmd, ... /* int arg */);
```

fcntl 函数有如下 5 种功能：

1. 复制一个已有的描述符(F_DUPFD, F_DUPFD_CLOEXEC)
2. 获取/设置文件描述符标志(F_GETFD, F_SETFD)
3. 获取/设置文件状态标志(F_GETFL, F_SETFL)
4. 获取/设置异步 I/O 所有权(F_GETOWN, F_SETOWN)
5. 获取/设置锁(F_GETLK, F_SETLK, F_SETLKW)

```c
#include <fcntl.h>
#include <stdio.h>

void
set_fl(int fd, int flags)
{
  int val;
  if ((val = fcntl(fd, F_GETFL, 0)) < 0) {
    printf("fcntl F_GETFL error\n");
  }

  val |= flags;

  if (fcntl(fd, F_SETFL, 0) < 0) {
    printf("fcntl F_SETFL error\n");
  }
}
```
