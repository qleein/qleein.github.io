---
layout: post
title:  "bash执行shell脚本时报错 save_bash_input: buffer already exists for new fd XXX"
date:   2018-05-28 23:11:00 +0800
---

最近发现在lua中通过os.execute执行系统shell脚本时，偶尔会发生错误退出，最后发现是bash本身的问题。

对于下面的shell脚本

```sh
#!/bin/bash

for fd in `seq 11 254`; do
    eval "exec $fd< data.txt"
done

for fd in `seq 256 274`; do
    eval "exec $fd< data.txt"
done

eval "exec 255>&-"
echo "done"
```

bash执行时会发生段错误

```
$ touch data.txt
$ bash test.sh
test.sh: line 11: save_bash_input: buffer already exists for new fd 275
Segmentation fault (core dumped)
```


bash执行shell脚本时，默认使用255的文件描述符打开/读取当前执行的这个shell脚本。这里将11-254，256-274这些文件描述符都使用了，然后调用`exec 255>&-`操作可以关闭代表当前shell脚本的文件描述符。这时便会发生错误。

看了下bash的实现，原来bash中操作的文件描述符，bash会为其创建一个buffer缓冲区，放在一个数组`buffers`中。在shell脚本中调用`exec`关闭文件描述符时， 如果要关闭的文件描述符恰好指向当前执行的shell脚本时，bash会复制出一个新的文件描述符，将旧的关闭，之后用新的文件描述符读取shell脚本。对于这个新的文件描述符如果过大的话，会超过`buffers`数组的大小，导致内存访问越界。

bash的实现代码如下所示

```c
int
save_bash_input (fd, new_fd)
     int fd, new_fd;
{
  int nfd;

  /* Sync the stream so we can re-read from the new file descriptor.  We
     might be able to avoid this by copying the buffered stream verbatim
     to the new file descriptor. */
  if (buffers[fd])
    sync_buffered_stream (fd);

  /* Now take care of duplicating the file descriptor that bash is
     using for input, so we can reinitialize it later. */
  nfd = (new_fd == -1) ? fcntl (fd, F_DUPFD, 10) : new_fd;
  if (nfd == -1)
    {
      if (fcntl (fd, F_GETFD, 0) == 0)
	sys_error (_("cannot allocate new file descriptor for bash input from fd %d"), fd);
      return -1;
    }

  if (buffers[nfd])
    {
      /* What's this?  A stray buffer without an associated open file
	 descriptor?  Free up the buffer and report the error. */
      internal_error (_("save_bash_input: buffer already exists for new fd %d"), nfd);
      free_buffered_stream (buffers[nfd]);
    }
```

`buffers`是一个`buffer`数组，数组的索引是文件描述符，值为对应的缓冲区。这里的`nfd`是复制产生的新的文件描述符，`buffers[nfd]`这里没有判断buffers数组的长度，一旦`nfd`过大，就会发生数组越界访问。

貌似bash 4.4中已经修复了这个问题，通过比较`nfd`与`buffers`数组的长度避免数组越界。相应代码变成下面这样。

```c
  if (nfd < nbuffers && buffers[nfd])
    {
      /* What's this?  A stray buffer without an associated open file
	 descriptor?  Free up the buffer and report the error. */
      internal_error (_("save_bash_input: buffer already exists for new fd %d"), nfd);
      free_buffered_stream (buffers[nfd]);
    }
```
