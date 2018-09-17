---
layout: post
title: Docker Namespace
categories: [docker]
tags: [docker]
---

# Namespace

## Docker用的namespace
容器共用了6种linux namespace来隔离自己和主机的资源：
* UTS：隔离当前进程的`主机名和域名`
* IPC：Inter-Process Communication。linux提供了`信号量，消息队列，共享内存`等方法来支持进程间通信，所以要做到隔离，必须用新的IPC namespace。
* PID：在PID隔离的空间中，我们可以为容器中的进程重新分配ID，这样可以创建init进程，即PID=1的进程，他是其他所有进程的父进程，可以回收僵尸进程等作用。
* NS：mount功能的namespace，因为是最早的namespace所以当时就叫NS了，殊不知后面多了这么多namespace种类。对mount namespace下的文件做任何的mount都不会影响到其他namespace的mount信息。`如果当前ns中的某个文件，在其他ns中被mount了，那么删除当前ns中的此文件时，会报错，因为该文件被其他ns给使用（mount）了，这常常会造成容器删除不掉（报Device or resource busy错误）。`mount namespace往往还需要配合`chroot`命令（或系统调用），改变当前进程的root目录，**模拟容器独占root fs的假象**。
* USER：这个namespace是用来映射主机上的用户/组在当前空间下的用户组ID的，比如将app on host映射成容器中的root。需要配合映射文件`/proc/<pid>/uid_map和/proc/<pid>/gid_map`使用
* NET：隔离进程中的网络资源
## 操作namespace
``` shell
// namespace在/proc/<PID>/ns下面
$ ll /proc/1/ns
总用量 0
lrwxrwxrwx 1 root root 0 7月  27 15:13 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0 7月  27 15:13 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 root root 0 7月  27 15:13 net -> net:[4026531956]
lrwxrwxrwx 1 root root 0 7月   9 14:10 pid -> pid:[4026531836]
lrwxrwxrwx 1 root root 0 7月  27 15:13 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 7月  27 15:13 uts -> uts:[4026531838]
```
后面的数字为namespace的id，数字一样的两个进程在一个对应的namespace中
通过mount操作，可以保证这个namespace文件一直被占用，从而在进程推出后，它还不被删除
``` shell
$ mount --bind /proc/<PID>/ns/net ~/net
```
我们还可以通过setns( )函数调用来进入某个namespace
``` c
fd = open("/proc/<PID>/ns/net", O_RDONLY);  // 获取namespace文件描述符
setns(fd, 0); // 当前进程加入此namespace
```
## unshare
此外还有`unshare`命令，或`unshare( )`函数调用，为当前进程创建新的namespace。
## nsester
nsenter命令可以进入某个进程的指定namespace
``` shell
// 进入为PID进程的pid，mount，net空间
$ nsenter -t <PID> --pid --mount --net
```

## 模拟Container
我们用一段C代码去模拟容器创建的过程，现在它只是简单的架子，clone一个自进程（可以去了解下`fork和clone的区别`，用clone可以指定namespace）。
``` c
// 编译：
// gcc namespace.c -o ns
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/mount.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

/* 定义一个给 clone 用的栈，栈大小1M */
#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];

char* const container_args[] = {
    "/bin/bash",
    NULL
};

int container_main(void* arg)
{
    printf("Container - inside the container!\n");
    /* 直接执行一个shell，以便我们观察这个进程空间里的资源是否被隔离了 */
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}

int main()
{
    printf("Parent - start a container!\n");
    /* 调用clone函数，其中传出一个函数，还有一个栈空间的（为什么传尾指针，因为栈是反着的） */
    int container_pid = clone(container_main, container_stack+STACK_SIZE, SIGCHLD, NULL);
    /* 等待子进程结束 */
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```
## UTS
通过`CLONE_NEWUTS`加入UTS namespace，我们可以单独设置进程的hostname和域名
``` c
int container_main(void* arg)
{
    printf("Container - inside the container!\n");
    /* 直接执行一个shell，以便我们观察这个进程空间里的资源是否被隔离了 */
    sethostname("container",10); /* 设置hostname */
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}

int main()
{
    printf("Parent - start a container!\n");
    /* 调用clone函数，其中传出一个函数，还有一个栈空间的（为什么传尾指针，因为栈是反着的） */
    int container_pid = clone(container_main, container_stack+STACK_SIZE, CLONE_NEWUTS | SIGCHLD, NULL);
    /* 等待子进程结束 */
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```
编译并执行：
``` shell
// 编译
[root@MiWiFi-R1CM-srv c]# gcc namespace.c -o ns
// 执行
[root@MiWiFi-R1CM-srv c]# ./ns
Parent - start a container!
Container - inside the container!
[root@container c]# exit
exit
Parent - container stopped!
```
可以看到新的进程的hostname已经改成container了
## IPC
通过`CLONE_NEWIPC`加入IPC的namespace
没加入之前：
``` shell
// 主机上创建消息队列
[root@MiWiFi-R1CM-srv c]# ipcmk -Q
消息队列 id：0
// 主机上查看消息队列
[root@MiWiFi-R1CM-srv c]# ipcs -q

--------- 消息队列 -----------
键        msqid      拥有者  权限     已用字节数 消息
0xe32adfe6 0          root       644        0            0

// 进入容器
[root@MiWiFi-R1CM-srv c]# ./ns
Parent - start a container!
Container - inside the container!
// 容器中查看消息队列，是可见的
[root@container c]# ipcs -q

--------- 消息队列 -----------
键        msqid      拥有者  权限     已用字节数 消息
0xe32adfe6 0          root       644        0            0
```
将容器加入IPC namespace
``` c
int container_pid = clone(container_main, container_stack+STACK_SIZE, CLONE_NEWUTS | CLONE_NEWIPC | SIGCHLD, NULL);
```
再次进入容器查看message queue，已不可见
``` shell
[root@container c]# ./ns
Parent - start a container!
Container - inside the container!
// message queue已不可见
[root@container c]# ipcs -q

--------- 消息队列 -----------
键        msqid      拥有者  权限     已用字节数 消息
```
## PID
通过`CLONE_NEWPID`加入新的PID namespace
``` c
int container_main(void* arg)
{
    /* 查看子进程的PID，我们可以看到其输出子进程的 pid 为 1 */
    printf("Container [%5d] - inside the container!\n", getpid());
    sethostname("container",10);
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}
 
int main()
{
    printf("Parent [%5d] - start a container!\n", getpid());
    /*启用PID namespace - CLONE_NEWPID*/
    int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | SIGCHLD, NULL); 
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```
编译并执行：
``` shell
[root@container c]# gcc namespace.c -o ns
[root@container c]# ./ns
Container [100237] - inside the container!
Container [    1] - inside the container!
[root@container c]# echo $$
1
```
可以看到容器中的当前进程ID`（$$表示当前进程ID）`已经是1。
但是我们执行top或者ps时，看到的却还是主机的情况。这时，我们需要重新挂载/proc目录：
``` shell
// 注意，在未加入MOUNT namespace之前不要执行，否则会影响到主机的挂载。
// 当然，如果执行并影响到主机了，只需要回到主机的PID namespace，再执行此命令，重新挂载下主机的，就能恢复
$ mount -t proc proc /proc
```
Proc目录是一个伪文件系统，存在于内存当中。是内核动态写到文件系统中的，是用户态和内核态交互的一个途径。（参考[linux 中/proc 详解 - duanxz - 博客园](https://www.cnblogs.com/duanxz/p/4970998.html)）
## NS（mount）
通过`CLONE_NEWNS`加入新的mount namespace（因为是最早的namespace，所以名字就叫NS）。在mount namespace中的任何mount都不会影响到其他mount namespace（如宿主机）。但是，如果一个目录被mount到多个namespace中，那么在删除某个namespace中的此文件是，会报错`Device or resource busy`
``` c
int container_main(void* arg)
{
    /* 查看子进程的PID，我们可以看到其输出子进程的 pid 为 1 */
    printf("Container [%5d] - inside the container!\n", getpid());
    sethostname("container",10);
    // mount / with --make-rprivate flag
    // 这个bug会暴露mount空间中的挂载点，从而影响到主机上的挂载目录
    // 如果不加这个fix，会使mount空间的隔离无效化
    // 查看相关bug：debian bug 739593
    // golang中也是默认加入这个fix：
    //     issue: https://github.com/golang/go/issues/19661
    //     pr:    https://github.com/golang/go/commit/d8ed449d8eae5b39ffe227ef7f56785e978dd5e2
    system("mount --make-rprivate /");  /* fix bug */
    // mount proc to /proc
    system("mount -t proc proc /proc");  
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}
 
int main()
{
    printf("Parent [%5d] - start a container!\n", getpid());
    /*启用PID namespace - CLONE_NEWPID*/
    int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | CLONE_NEWNS | SIGCHLD, NULL); 
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```
这时候通过top或者ps查看进程：
``` shell
[root@MiWiFi-R1CM-srv c]# ./ns
Parent [116919] - start a container!
Container [    1] - inside the container!
// 只会看到当前namespace中进程的状况
[root@container c]# ps -ef
UID         PID   PPID  C STIME TTY          TIME CMD
root          1      0  0 02:34 pts/1    00:00:00 /bin/bash
root         13      1  0 02:34 pts/1    00:00:00 ps -ef
```
::特别注意::
上面在加入mount namespace后还在子进程中多执行了命令`mount --make-rprivate /`，这时因为在新的linux版本中，systemd强制让所有的mount namespace之间都打开share（从根目录开始）。所以mount namespace的隔离型被打破了。故多执行这个命令以在当前的mount namespace中打开隔离性，这样之后才能在里面随便折腾。具体可以查看[debian bug 739593](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=739593)。在golang中也加入了这个[fix](https://github.com/golang/go/commit/d8ed449d8eae5b39ffe227ef7f56785e978dd5e2)。（我在查看此问题时，同时参考了[这个项目的issue](https://github.com/lizrice/containers-from-scratch/issues/3)）
### chroot
Chroot可以将改变当前进程的根目录.
正因为此，我们需要一个自己的rootfs
#### 方法一：从当前主机创建新的root fs，下面还需要拷贝新的root fs里需要的命令:
* /usr/bin和/bin
``` shell
[root@MiWiFi-R1CM-srv rootfs]# mkdir rootfs
[root@MiWiFi-R1CM-srv rootfs]# cd rootfs
// mkdir各种目录
[root@MiWiFi-R1CM-srv rootfs]# mkdir -p bin usr/bin
// cp命令
[root@MiWiFi-R1CM-srv rootfs]# sh
// 执行下面命令最好在sh环境中，在bash中会有alias干扰
sh-4.2# which bash sh netstat rm mkdir top cat cp mv ln ping ps ls tail grep kill chmod pwd less tar | xargs -I {} cp {} ./bin/
sh-4.2# which env id tail vim vi groups top awk | xargs -I {} cp {} ./usr/bin/
```
* 拷贝上面命令在/lib和/lib64下对应的一些so文件，到rootfs下面
``` shell
// 为求简单，上面两类文件直接mount了
$ cd rootfs
$ mkdir lib lib64
$ mount --bind /usr/lib ./lib
$ mount --bind /usr/lib64 ./lib64
```
* 拷贝上面命令在/etc下的一些配置，到rootfs下面。hosts，hostname，resolv.conf文件我们需要删掉，不要重用主机的。
``` shell
$ cp -r /etc ./
$ rm -f ./etc/hosts ./etc/hostname ./etc/resolv.conf
```
我们将这3个文件放在rootfs同级目录，之后mount进去：
``` shell
// 往上走一级
$ [root@MiWiFi-R1CM-srv rootfs]# cd ..
$ [root@MiWiFi-R1CM-srv ~]# mkdir conf
$ [root@MiWiFi-R1CM-srv ~]# cp /etc/hosts /etc/hostname /etc/resolv.conf ./conf
```
* 创建测试目录模拟Docker的-v命令
``` shell
$ mkdir /tmp/kewu
$ touch /tmp/kewu/test
```
其他文件在runtime时mount宿主机上的，所以先做好准备：
``` shell
$ cd rootfs
$ mkdir sys tmp dev dev/pts dev/shm run proc kewu
```

#### 方法二：也可以从某个docker镜像提取出其rootfs：
``` shell
$ mkdir rootfs
$ docker export $(docker create <Docker-image>) | tar -C rootfs -xvf -
```
``` c
// 下面这些mount，除了proc，其他都还没试过
int container_main(void* arg)
{
    printf("Container [%5d] - inside the container!\n", getpid());
 
    //set hostname
    sethostname("container",10);
 
    mount("none", "/", NULL, MS_REC|MS_PRIVATE, NULL);  /* fix bug */
    //remount "/proc" to make sure the "top" and "ps" show container's information
    if (mount("proc", "rootfs/proc", "proc", 0, NULL) !=0 ) { /* 这里我们mount到自己的/proc目录 */
        perror("proc");
    }
    if (mount("sysfs", "rootfs/sys", "sysfs", 0, NULL)!=0) {
        perror("sys");
    }
    if (mount("none", "rootfs/tmp", "tmpfs", 0, NULL)!=0) {
        perror("tmp");
    }
    if (mount("udev", "rootfs/dev", "devtmpfs", 0, NULL)!=0) {
        perror("dev");
    }
    if (mount("devpts", "rootfs/dev/pts", "devpts", 0, NULL)!=0) {
        perror("dev/pts");
    }
    if (mount("shm", "rootfs/dev/shm", "tmpfs", 0, NULL)!=0) {
        perror("dev/shm");
    }
    if (mount("tmpfs", "rootfs/run", "tmpfs", 0, NULL)!=0) {
        perror("run");
    }
    /* 
     * 模仿Docker的从外向容器里mount相关的配置文件 
     * 你可以查看：/var/lib/docker/containers/<container_id>/目录，
     * 你会看到docker的这些文件的。
     */
    if (mount("conf/hosts", "rootfs/etc/hosts", "none", MS_BIND, NULL)!=0 ||
          mount("conf/hostname", "rootfs/etc/hostname", "none", MS_BIND, NULL)!=0 ||
          mount("conf/resolv.conf", "rootfs/etc/resolv.conf", "none", MS_BIND, NULL)!=0 ) {
        perror("conf");
    }
    /* 模仿docker run命令中的 -v, --volume=[] 参数干的事 */
    if (mount("/tmp/kewu", "rootfs/kewu", "none", MS_BIND, NULL)!=0) {
        perror("kewu");
    }
 
    /* chroot 隔离目录 */
    if ( chdir("./rootfs") != 0 || chroot("./") != 0 ){
        perror("chdir/chroot");
    }
 
    execv(container_args[0], container_args);
    perror("exec");
    printf("Something's wrong!\n");
    return 1;
}
```
这样，这个“容器”基本上也就有点模样了
``` shell
// 运行后
$ ls -al /kewu
total 0
drwxr-xr-x  2 root root  17 Jul 30 06:22 .
drwxr-xr-x 12 root root 109 Jul 30 06:22 ..
-rw-r--r--  1 root root   0 Jul 30 06:22 test
```
## USER
通过`CLONE_NEWUSER`加入新的user namespace。
另外，user namespace需要配合`/proc/<pid>/uid_map`和`/proc/<pid>/gid_map`两个映射文件。此文件的格式为：
> ID-inside-ns    ID-outside-ns    length  
其中：
* ID-inside-ns表示容器里显示的UID或GID
* ID-outside-ns表示主机上对应的真实UID或GID
* length表示映射范围，1表示当前值一一对应，“N”表示从ID-inside-ns / ID-outside-ns开始之后的N个ID一一对应
举个例子：
```
// 4294967295是32位无符号int的最大值
// 这表示容器内外的所有UID都是对应相等的
$ cat /proc/16327/uid_map
         0          0 4294967295

// 这表示将容器外的UID=1000映射为容器内的UID=0
$ cat /proc/16328/uid_map
         0       1000          1
```
需要注意的：
* 写这两个文件的进程需要这个namespace中的CAP_SETUID (CAP_SETGID)权限（可参看[capabilities(7) - Linux manual page](http://man7.org/linux/man-pages/man7/capabilities.7.html)）
* 写入的进程必须是此user namespace的父或子的user namespace进程。
* 另外需要满如下条件之一：1）父进程将effective uid/gid映射到子进程的user namespace中，2）父进程如果有CAP_SETUID/CAP_SETGID权限，那么它将可以映射到父进程中的任一uid/gid。
``` c
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/mount.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

/* 定义一个给 clone 用的栈，栈大小1M */
#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];

char* const container_args[] = {
    "/bin/bash",
    NULL
};

int pipefd[2];

void set_map(char* file, int inside_id, int outside_id, int len) {
    FILE* mapfd = fopen(file, "w");
    if (NULL == mapfd) {
        perror("open file error");
        return;
    }
    fprintf(mapfd, "%d %d %d", inside_id, outside_id, len);
    fclose(mapfd);
}

void set_uid_map(pid_t pid, int inside_id, int outside_id, int len) {
    char file[256];
    sprintf(file, "/proc/%d/uid_map", pid);
    set_map(file, inside_id, outside_id, len);
}

void set_gid_map(pid_t pid, int inside_id, int outside_id, int len) {
    char file[256];
    sprintf(file, "/proc/%d/gid_map", pid);
    set_map(file, inside_id, outside_id, len);
}

int container_main(void* arg)
{
    printf("Container [%5d] - inside the container!\n", getpid());

    printf("Container: eUID = %ld;  eGID = %ld, UID=%ld, GID=%ld\n",
            (long) geteuid(), (long) getegid(), (long) getuid(), (long) getgid());

    /* 等待父进程通知后再往下执行（进程间的同步） */
    char ch;
    close(pipefd[1]);
    read(pipefd[0], &ch, 1);

    /* 查看子进程的PID，我们可以看到其输出子进程的 pid 为 1 */
    printf("Container [%5d] - inside the container!\n", getpid());
    sethostname("container",10);

    mount("none", "/", NULL, MS_REC|MS_PRIVATE, NULL);  /* fix bug */
    //remount "/proc" to make sure the "top" and "ps" show container's information
    if (mount("proc", "/proc", "proc", 0, NULL) !=0 ) { /* 这里我们mount到自己的/proc目录 */
        perror("proc");
    }

    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}

int main()
{
    const int gid=getgid(), uid=getuid();

    printf("Parent: eUID = %ld;  eGID = %ld, UID=%ld, GID=%ld\n",
            (long) geteuid(), (long) getegid(), (long) getuid(), (long) getgid());

    pipe(pipefd);

    printf("Parent [%5d] - start a container!\n", getpid());
    /*启用PID namespace - CLONE_NEWPID*/
    int container_pid = clone(container_main, container_stack+STACK_SIZE,
            CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | CLONE_NEWNS | SIGCHLD, NULL);

    if (container_pid < 0) {
        perror("start container_stack");
    }

    printf("Parent [%5d] - Container [%5d]!\n", getpid(), container_pid);

    //To map the uid/gid,
    //   we need edit the /proc/PID/uid_map (or /proc/PID/gid_map) in parent
    //The file format is
    //   ID-inside-ns   ID-outside-ns   length
    //if no mapping,
    //   the uid will be taken from /proc/sys/kernel/overflowuid
    //   the gid will be taken from /proc/sys/kernel/overflowgid
    set_uid_map(container_pid, 0, uid, 1);
    set_gid_map(container_pid, 0, gid, 1);

    printf("Parent [%5d] - user/group mapping done!\n", getpid());

    /* 通知子进程 */
    close(pipefd[1]);

    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```
## NET
通过`CLONE_NEWNET`创建新的network namespace。也可以通过ip命令去单独创建。
``` shell
// 添加一个叫ns1的network namespace
$ ip netns add ns1 
// 这时会在目录/var/run/netns下创建一个同名文件，并mount --bind到/proc/self/ns/net
$ ll /var/run/netns
总用量 0
-rw-r--r-- 1 root root 0 7月  31 07:07 ns1
// 查看mount信息（proc目录为proc类型的内存文件系统）
$ mount | grep ns1
proc on /run/netns/ns1 type proc (rw,relatime)
```
`Docker源码中没有使用ip命令，而是自己实现了ip命令的一些功能（是通过raw socket发一些奇怪的数据）`，这里还是以ip命令来做演示。
``` c
int container_main(void* arg)
{
    /* 查看子进程的PID，我们可以看到其输出子进程的 pid 为 1 */
    printf("Container [%5d] - inside the container!\n", getpid());
    sethostname("container",10);
    // mount / with --make-rprivate flag
    // 这个bug会暴露mount空间中的挂载点，从而影响到主机上的挂载目录
    // 如果不加这个fix，会使mount空间的隔离无效化
    // 查看相关bug：debian bug 739593
    // golang中也是默认加入这个fix：
    //     issue: https://github.com/golang/go/issues/19661
    //     pr:    https://github.com/golang/go/commit/d8ed449d8eae5b39ffe227ef7f56785e978dd5e2
    system("mount --make-rprivate /");  /* fix bug */
    // mount proc to /proc
    system("mount -t proc proc /proc");
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}

int main()
{
    printf("Parent [%5d] - start a container!\n", getpid());
    /*启用PID namespace - CLONE_NEWPID*/
    int container_pid = clone(container_main, container_stack+STACK_SIZE,
            CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET | SIGCHLD, NULL);
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```
执行上面命令并创建出容器
```
$ gcc -o namespace.c ns
$ ./ns
Parent [14176] - start a container!
Container [    1] - inside the container!
```
在主机上找到对应的pid
```
// 以父进程id来搜索
$ ps -ef | grep 14176
root     14176 13713  0 07:33 pts/6    00:00:00 ./ns
root     14178 14176  0 07:33 pts/6    00:00:00 /bin/bash  // 容器进程
root     14222 13662  0 07:34 pts/3    00:00:00 grep --color=auto 14176
```
创建对应的network namespace名称，这样通过ip/nsenter命令才能访问（这里我们换成符号连接的方式）
``` shell
$ mkdir /var/run/netns
$ rm -f /var/run/netns/namespace1
$ ln -s /proc/14178/ns/net /var/run/netns/namespace1
```
然后就可以开始初始化网络了
``` shell
$ brctl addbr mydocker0
$ brctl stp mydocker0 off
$ ifconfig mydocker0 192.168.10.1/24 up //为网桥设置IP地址

// 进入namespace1并激活loopback，即127.0.0.1（使用ip netns exec在某个network namespace下执行命令）
$ ip netns exec namespace1     ip link set dev lo up

// 创建veth pair
$ ip link add veth-namespace1 type veth peer name veth-0.1


// 把veth-namespace1按到容器的namespace中，这样容器就有了新的网卡（之后在容器中都能看到这个网卡了）
$ ip link set veth-namespace1 netns namespace1

// 在容器内把网卡改名为eth0
$ ip netns exec namespace1    ip link set dev veth-namespace1 name eth0
// 为容器中的网卡分配一个ip，并激活它
$ ip netns exec namespace1    ifconfig eth0 192.168.10.11/24 up


// 然后，把veth-0.1添加到网桥上
$ brctl addif mydocker0 veth-0.1
// 激活该网卡
$ ifconfig veth-0.1 up


// 为容器配置gateway，以访问外面的网络
$ ip netns exec namespace1    ip route add default via 192.168.10.1
```
这样，容器基本的bridge网络就算有模有样了。
``` shell
// ping宿主机
$ ping 10.199.234.237
PING 10.199.234.237 (10.199.234.237) 56(84) bytes of data.
64 bytes from 10.199.234.237: icmp_seq=1 ttl=64 time=0.044 ms
64 bytes from 10.199.234.237: icmp_seq=2 ttl=64 time=0.049 ms
^C
--- 10.199.234.237 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.044/0.046/0.049/0.007 ms
```
新的network namespace中已经有独立的路由规则和iptables了
```
$ ip netns exec namespace1    ip route
RTNETLINK answers: Invalid argument
default via 192.168.10.1 dev eth0
192.168.10.0/24 dev eth0  proto kernel  scope link  src 192.168.10.11
$ ip netns exec namespace1    iptables -L
RTNETLINK answers: Invalid argument
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```
清理环境：
``` shell
$ rm -f /var/run/netns/namespace1
$ ifconfig mydocker0 down
$ brctl delbr mydocker0
```
后续如果要访问外网，就还需要配置iptables，添加NAT操作，开启 ip_forward
# Reference

* [Docker基础技术：Linux Namespace（上）](https://coolshell.cn/articles/17010.html)
* [Docker基础技术：Linux Namespace（下）](https://coolshell.cn/articles/17029.html)
* [Docker背后的内核知识：命名空间资源隔离](https://linux.cn/article-5057-1.html)
* [debian bug 739593](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=739593)
* [github.com/containers-from-scratch](https://github.com/lizrice/containers-from-scratch/issues/3)