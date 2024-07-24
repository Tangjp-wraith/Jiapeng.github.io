# Xv6 and Unix utilities

Lab1主要是在用户层实现一些命令

## sleep
sleep的实现比较简单，使用系统调用sleep，在user/user.h中给出了声明
注意：终端键入的是字符串，调用user/ulib.c中的atoi函数转换成数字，缺少参数要打印错误信息
```cpp
int main(int argc, char *argv[]) {
  if (argc != 2) {
    fprintf(2,"ERROR:sleep less param\n");
    exit(1);
  }
  sleep(atoi(argv[1]));
  exit(0);
}
```

## pingpong
[管道pipe()参考文章](https://www.geeksforgeeks.org/pipe-system-call/)

对于下面代码中的`pd[2]`来说，`pd[0]`是读入端，`pd[1]`是写入端
对于父进程来说，需要先将ping写入管道，供给子进程来读，子进程读取之后，再将pong写入管道，子进程结束后，父进程读取管道中内容

```cpp
#define MSGSIZE 16

int main(int argc, int *argv[]) {
  int pd[2];
  pipe(pd);
  char buf[MSGSIZE];
  int pid = fork();
  if (pid > 0) {
    write(pd[1], "ping", MSGSIZE);
    wait((int *)0);
    read(pd[0], buf, MSGSIZE);
    printf("%d: received %s\n", getpid(), buf);
  } else {
    read(pd[0], buf, MSGSIZE);
    printf("%d: received %s\n", getpid(), buf);
    write(pd[1], "pong", MSGSIZE);
  }
  close(pd[0]);
  close(pd[1]);
  exit(0);
}
```


## primes
利用管道实现素数筛
对于整个过程，通过一个递归来实现，对于父进程，将nums数组写入管道，对于子进程，执行一个prime(int pipe_read,int write)函数，传入参数为管道的读写端
在prime这个函数中，先读取管道的读入端，这是由父进程传入的nums数组，然后取出nums数组中第一个置TRUE的数，打印出来，再将该数的所有倍数置FALSE（埃氏筛思想）
再在prime中创建子进程，重复上述过程，注意prime函数exit(0)的条件：nums数组全为FALSE

```cpp
#define SIZE 36
#define TRUE '1'
#define FALSE '0'

void prime(int pipe_read, int pipe_write) {
  char buf[SIZE];
  int val = 0;
  read(pipe_read, buf, SIZE);
  for (int i = 2; i < SIZE; ++i) {
    if (buf[i] == TRUE) {
      val = i;
      break;
    }
  }
  if (val == 0) {
    exit(0);
  }
  printf("prime %d\n", val);
  for (int i = val; i < SIZE; ++i) {
    if (i % val == 0) {
      buf[i] = FALSE;
    }
  }
  int pid = fork();
  if (pid > 0) {
    write(pipe_write, buf, SIZE);
  } else {
    prime(pipe_read, pipe_write);
  }
}

int main(int agrc, int *agrv[]) {
  int pd[2];
  pipe(pd);
  char nums[SIZE];
  for (int i = 0; i < SIZE; ++i) {
    nums[i] = TRUE;
  }
  nums[0] = FALSE;
  nums[1] = FALSE;
  int pid = fork();
  if (pid > 0) {
    write(pd[1], nums, SIZE);
    wait((int *)0);
  } else {
    prime(pd[0], pd[1]);
  }
  wait((int *)0);
  close(pd[0]);
  close(pd[1]);
  exit(0);
}
```

## find
仿照user/ls.c实现find命令
使用递归允许find下降到子目录中，不要在'.'和'..'目录中递归

在ls.c这个文件中有一个格式化字符串的函数fmtname，该函数将输入的路径类似'./dir'转换成标准目录'dir'
```cpp
char *fmtname(char *path) {
  static char buf[DIRSIZ + 1];
  char *p;

  // Find first character after last slash.
  for (p = path + strlen(path); p >= path && *p != '/'; p--)
    ;
  p++;

  // Return blank-padded name.
  if (strlen(p) >= DIRSIZ) return p;
  memmove(buf, p, strlen(p));
  memset(buf + strlen(p), 0, DIRSIZ - strlen(p));
  return buf;
}
```

我们还需要判断当前这个目录，是不是可以下降的目录：
```cpp
int recursion(char *path) {
  char *buf = fmtname(path);
  if ((buf[0] == '.' && buf[1] == 0) ||
      (buf[0] == '.' && buf[1] == '.' && buf[2] == 0)) {
    return 0;
  }
  return 1;
}
```

有了这两个辅助函数，我们只需要将ls.c中提供的ls函数稍加修改就可以实现我们需要的find函数了，首先，我们需要在每次find的时候，比照当前path下与target是否相同，相同则打印path
```cpp
if (strcmp(fmtname(path), target) == 0) {
  printf("%s\n", path);
}
```
其次，对于当前path如果是目录且满足条件的情况下，要下降到子目录，继续调用find
```cpp
if (recursion(buf)) {
  find(buf, target);
}
```

## xargs

首先讲一下什么是管道符：| 
```bash
cmdA | cmdB
```
对于上面这个命令，cmdA的输出将会通过管道符成为cmdB的输入

而使用xargs命令来说，cmdA的输出将成为cmdB的命令行参数
```bash
cmdA | xagrs cmdB
```

但是linux对xargs有一个优化，会将cmdA输出中的'\n'当成空格无视：
```bash
echo "1\n2" | xargs echo line
# 相当于是
echo line 1 2
```

那么要实现换行需要在xargs后加入参数'-n 1'：
```bash
echo "1\n2" | xargs -n 1 echo line
# 相当于是
echo line 1 && echo line 2
```

但是在这个作业里，我们只需要实现不带这个优化的xargs命令

首先我们要解决的是如何读取cmdA的输出，参考文档1.2中的 { 按照惯例，进程从文件描述符0读取（标准输入），将输出写入文件描述符1（标准输出），并将错误消息写入文件描述符2（标准错误）}，可以用如下代码读取输出到buf中：
```cpp
char buf[SIZE];
read(0, buf, SIZE);
```

其次我们要处理出xargs命令下真正输入，我们需要仿照终端输入参数设置xagrc和xagrv两个参数，遍历终端输入的字符串，注意argv[0]是"xargs"忽略，因此下标从1开始：
```cpp
char *xargv[MAXARG];
int xagrc = 0;
for (int i = 1; i < argc; i++) {
  xargv[xagrc++] = argv[i];
}
```

最后，我们要将buf中的内容作为参数传入xagrv中，但是碰到'\n'我们就需要执行一次命令，
执行命令，我们只需要fork一个子进程，让子进程调用exec执行当前命令，即xargv[0]
父进程只需要将p当前的指针指向buf的下一位即可
```cpp
char *p = buf;
for (int i = 0; i < SIZE; i++) {
  if (buf[i] == '\n') {
    int pid = fork();
    if (pid > 0) {
      p = &buf[i + 1];
      wait((int *)0);
    } else {
      buf[i] = 0;
      xargv[xagrc++] = p;
      exec(xargv[0], xargv);
      exit(0);
    }
  }
}
```