指定内存配置：
```
docker run -m 200M --memory-swap=300M ubuntu
```

```
Namespaces
```



# Docker背后的内核知识——Namespace资源隔离

Docker这么火，喜欢技术的朋友可能也会想，如果要自己实现一个资源隔离的容器，应该从哪些方面下手呢？也许你第一反应可能就是chroot命令，这条命令给用户最直观的感觉就是使用后根目录/的挂载点切换了，即文件系统被隔离了。然后，为了在分布式的环境下进行通信和定位，容器必然需要一个独立的IP、端口、路由等等，自然就想到了网络的隔离。同时，你的容器还需要一个独立的主机名以便在网络中标识自己。想到网络，顺其自然就想到通信，也就想到了进程间通信的隔离。可能你也想到了权限的问题，对用户和用户组的隔离就实现了用户权限的隔离。最后，运行在容器中的应用需要有自己的PID,自然也需要与宿主机中的PID进行隔离。

由此，我们基本上完成了一个容器所需要做的六项隔离，Linux内核中就提供了这六种namespace隔离的系统调用，如下表所示。

| Namespace | 系统调用参数        | 隔离内容          |
| --------- | ------------- | ------------- |
| UTS       | CLONE_NEWUTS  | 主机名与域名        |
| IPC       | CLONE_NEWIPC  | 信号量、消息队列和共享内存 |
| PID       | CLONE_NEWPID  | 进程编号          |
| Network   | CLONE_NEWNET  | 网络设备、网络栈、端口等等 |
| Mount     | CLONE_NEWNS   | 挂载点（文件系统）     |
| User      | CLONE_NEWUSER | 用户和用户组        |

表 namespace六项隔离

实际上，Linux内核实现namespace的主要目的就是为了实现轻量级虚拟化（容器）服务。在同一个namespace下的进程可以感知彼此的变化，而对外界的进程一无所知。这样就可以让容器中的进程产生错觉，仿佛自己置身于一个独立的系统环境中，以此达到独立和隔离的目的。

需要说明的是，本文所讨论的namespace实现针对的均是Linux内核3.8及其以后的版本。接下来，我们将首先介绍使用namespace的API，然后针对这六种namespace进行逐一讲解，并通过程序让你亲身感受一下这些隔离效果（参考自<http://lwn.net/Articles/531114/>）。

## 1. 调用namespace的API

namespace的API包括clone()、setns()以及unshare()，还有/proc下的部分文件。为了确定隔离的到底是哪种namespace，在使用这些API时，通常需要指定以下六个常数的一个或多个，通过|（位或）操作来实现。你可能已经在上面的表格中注意到，这六个参数分别是CLONE_NEWIPC、CLONE_NEWNS、CLONE_NEWNET、CLONE_NEWPID、CLONE_NEWUSER和CLONE_NEWUTS。

### （1）通过clone()创建新进程的同时创建namespace

使用clone()来创建一个独立namespace的进程是最常见做法，它的调用方式如下。

```
int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
```

clone()实际上是传统UNIX系统调用fork()的一种更通用的实现方式，它可以通过flags来控制使用多少功能。一共有二十多种CLONE_*的flag（标志位）参数用来控制clone进程的方方面面（如是否与父进程共享虚拟内存等等），下面外面逐一讲解clone函数传入的参数。

- 参数child_func传入子进程运行的程序主函数。
- 参数child_stack传入子进程使用的栈空间
- 参数flags表示使用哪些CLONE_*标志位
- 参数args则可用于传入用户参数

在后续的内容中将会有使用clone()的实际程序可供大家参考。

### （2）查看/proc/[pid]/ns文件

从3.8版本的内核开始，用户就可以在/proc/[pid]/ns文件下看到指向不同namespace号的文件，效果如下所示，形如[4026531839]者即为namespace号。

```
$ ls -l /proc/$$/ns         <<-- $$ 表示应用的PID
total 0
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 ipc -> ipc:[4026531839]
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 mnt -> mnt:[4026531840]
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 net -> net:[4026531956]
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 pid -> pid:[4026531836]
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 user->user:[4026531837]
lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 uts -> uts:[4026531838]
```

如果两个进程指向的namespace编号相同，就说明他们在同一个namespace下，否则则在不同namespace里面。/proc/[pid]/ns的另外一个作用是，一旦文件被打开，只要打开的文件描述符（fd）存在，那么就算PID所属的所有进程都已经结束，创建的namespace就会一直存在。那如何打开文件描述符呢？把/proc/[pid]/ns目录挂载起来就可以达到这个效果，命令如下。

```
# touch ~/uts
# mount --bind /proc/27514/ns/uts ~/uts
```

如果你看到的内容与本文所描述的不符，那么说明你使用的内核在3.8版本以前。该目录下存在的只有ipc、net和uts，并且以硬链接存在。

### （3）通过setns()加入一个已经存在的namespace

上文刚提到，在进程都结束的情况下，也可以通过挂载的形式把namespace保留下来，保留namespace的目的自然是为以后有进程加入做准备。通过setns()系统调用，你的进程从原先的namespace加入我们准备好的新namespace，使用方法如下。

```
int setns(int fd, int nstype);
```

- 参数fd表示我们要加入的namespace的文件描述符。上文已经提到，它是一个指向/proc/[pid]/ns目录的文件描述符，可以通过直接打开该目录下的链接或者打开一个挂载了该目录下链接的文件得到。
- 参数nstype让调用者可以去检查fd指向的namespace类型是否符合我们实际的要求。如果填0表示不检查。

为了把我们创建的namespace利用起来，我们需要引入execve()系列函数，这个函数可以执行用户命令，最常用的就是调用/bin/bash并接受参数，运行起一个shell，用法如下。

```
fd = open(argv[1], O_RDONLY);   /* 获取namespace文件描述符 */
setns(fd, 0);                   /* 加入新的namespace */
execvp(argv[2], &argv[2]);      /* 执行程序 */
```

假设编译后的程序名称为setns。

```
# ./setns ~/uts /bin/bash   # ~/uts 是绑定的/proc/27514/ns/uts
```

至此，你就可以在新的命名空间中执行shell命令了，在下文中会多次使用这种方式来演示隔离的效果。

### （4）通过unshare()在原先进程上进行namespace隔离

最后要提的系统调用是unshare()，它跟clone()很像，不同的是，unshare()运行在原先的进程上，不需要启动一个新进程，使用方法如下。

```
int unshare(int flags);
```

调用unshare()的主要作用就是不启动一个新进程就可以起到隔离的效果，相当于跳出原先的namespace进行操作。这样，你就可以在原进程进行一些需要隔离的操作。Linux中自带的unshare命令，就是通过unshare()系统调用实现的，有兴趣的读者可以在网上搜索一下这个命令的作用。

### （5）延伸阅读：fork（）系统调用

系统调用函数fork()并不属于namespace的API，所以这部分内容属于延伸阅读，如果读者已经对fork()有足够的了解，那大可跳过。

当程序调用fork（）函数时，系统会创建新的进程，为其分配资源，例如存储数据和代码的空间。然后把原来的进程的所有值都复制到新的进程中，只有少量数值与原来的进程值不同，相当于克隆了一个自己。那么程序的后续代码逻辑要如何区分自己是新进程还是父进程呢？

fork()的神奇之处在于它仅仅被调用一次，却能够返回两次（父进程与子进程各返回一次），通过返回值的不同就可以进行区分父进程与子进程。它可能有三种不同的返回值：

- 在父进程中，fork返回新创建子进程的进程ID
- 在子进程中，fork返回0
- 如果出现错误，fork返回一个负值

下面给出一段实例代码，命名为fork_example.c。

```
#include <unistd.h>
#include <stdio.h>
int main (){
    pid_t fpid; //fpid表示fork函数返回的值
    int count=0;
    fpid=fork();
    if (fpid < 0)printf("error in fork!");
    else if (fpid == 0) {
        printf("I am child. Process id is %d/n",getpid());
    }
    else {
        printf("i am parent. Process id is %d/n",getpid());
    }
    return 0;
}
```

编译并执行，结果如下。

```
root@local:~# gcc -Wall fork_example.c && ./a.out
I am parent. Process id is 28365
I am child. Process id is 28366
```

使用fork()后，父进程有义务监控子进程的运行状态，并在子进程退出后自己才能正常退出，否则子进程就会成为“孤儿”进程。

下面我们将分别对六种namespace进行详细解析。

## 2. UTS（UNIX Time-sharing System）namespace

UTS namespace提供了主机名和域名的隔离，这样每个容器就可以拥有了独立的主机名和域名，在网络上可以被视作一个独立的节点而非宿主机上的一个进程。

下面我们通过代码来感受一下UTS隔离的效果，首先需要一个程序的骨架，如下所示。打开编辑器创建uts.c文件，输入如下代码。

```
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

#define STACK_SIZE (1024 * 1024)

static char child_stack[STACK_SIZE];
char* const child_args[] = {
  "/bin/bash",
  NULL
};

int child_main(void* args) {
  printf("在子进程中!\n");
  execv(child_args[0], child_args);
  return 1;
}

int main() {
  printf("程序开始: \n");
  int child_pid = clone(child_main, child_stack + STACK_SIZE, SIGCHLD, NULL);
  waitpid(child_pid, NULL, 0);
  printf("已退出\n");
  return 0;
}
```

编译并运行上述代码，执行如下命令，效果如下。

```
root@local:~# gcc -Wall uts.c -o uts.o && ./uts.o
程序开始:
在子进程中!
root@local:~# exit
exit
已退出
root@local:~#
```

下面，我们将修改代码，加入UTS隔离。运行代码需要root权限，为了防止普通用户任意修改系统主机名导致set-user-ID相关的应用运行出错。

```
//[...]
int child_main(void* arg) {
  printf("在子进程中!\n");
  sethostname("Changed Namespace", 12);
  execv(child_args[0], child_args);
  return 1;
}

int main() {
//[...]
int child_pid = clone(child_main, child_stack+STACK_SIZE,
    CLONE_NEWUTS | SIGCHLD, NULL);
//[...]
}
```

再次运行可以看到hostname已经变化。

```
root@local:~# gcc -Wall namespace.c -o main.o && ./main.o
程序开始:
在子进程中!
root@NewNamespace:~# exit
exit
已退出
root@local:~#  <- 回到原来的hostname
```

也许有读者试着不加CLONE_NEWUTS参数运行上述代码，发现主机名也变了，输入exit以后主机名也会变回来，似乎没什么区别。实际上不加CLONE_NEWUTS参数进行隔离而使用sethostname已经把宿主机的主机名改掉了。你看到exit退出后还原只是因为bash只在刚登录的时候读取一次UTS，当你重新登陆或者使用uname命令进行查看时，就会发现产生了变化。

Docker中，每个镜像基本都以自己所提供的服务命名了自己的hostname而没有对宿主机产生任何影响，用的就是这个原理。

## 3. IPC（Interprocess Communication）namespace

容器中进程间通信采用的方法包括常见的信号量、消息队列和共享内存。然而与虚拟机不同的是，容器内部进程间通信对宿主机来说，实际上是具有相同PID namespace中的进程间通信，因此需要一个唯一的标识符来进行区别。申请IPC资源就申请了这样一个全局唯一的32位ID，所以IPC namespace中实际上包含了系统IPC标识符以及实现POSIX消息队列的文件系统。在同一个IPC namespace下的进程彼此可见，而与其他的IPC namespace下的进程则互相不可见。

IPC namespace在代码上的变化与UTS namespace相似，只是标识位有所变化，需要加上CLONE_NEWIPC参数。主要改动如下，其他部位不变，程序名称改为ipc.c。（测试方法参考自：<http://crosbymichael.com/creating-containers-part-1.html>）

```
//[...]
int child_pid = clone(child_main, child_stack+STACK_SIZE,
           CLONE_NEWIPC | CLONE_NEWUTS | SIGCHLD, NULL);
//[...]
```

我们首先在shell中使用ipcmk -Q命令创建一个message queue。

```
root@local:~# ipcmk -Q
Message queue id: 32769
```

通过ipcs -q可以查看到已经开启的message queue，序号为32769。

```
root@local:~# ipcs -q
------ Message Queues --------
key        msqid   owner   perms   used-bytes   messages
0x4cf5e29f 32769   root    644     0            0
```

然后我们可以编译运行加入了IPC namespace隔离的ipc.c，在新建的子进程中调用的shell中执行ipcs -q查看message queue。

```
root@local:~# gcc -Wall ipc.c -o ipc.o && ./ipc.o
程序开始:
在子进程中!
root@NewNamespace:~# ipcs -q
------ Message Queues --------
key   msqid   owner   perms   used-bytes   messages
root@NewNamespace:~# exit
exit
已退出
```

上面的结果显示中可以发现，已经找不到原先声明的message queue，实现了IPC的隔离。

目前使用IPC namespace机制的系统不多，其中比较有名的有PostgreSQL。Docker本身通过socket或tcp进行通信。

## 4. PID namespace

PID namespace隔离非常实用，它对进程PID重新标号，即两个不同namespace下的进程可以有同一个PID。每个PID namespace都有自己的计数程序。内核为所有的PID namespace维护了一个树状结构，最顶层的是系统初始时创建的，我们称之为root namespace。他创建的新PID namespace就称之为child namespace（树的子节点），而原先的PID namespace就是新创建的PID namespace的parent namespace（树的父节点）。通过这种方式，不同的PID namespaces会形成一个等级体系。所属的父节点可以看到子节点中的进程，并可以通过信号等方式对子节点中的进程产生影响。反过来，子节点不能看到父节点PID namespace中的任何内容。由此产生如下结论（部分内容引自：[http://blog.dotcloud.com/under-the-hood-linux-kernels-on-dotcloud-part](http://blog.dotcloud.com/under-the-hood-linux-kernels-on-dotcloud-part])）。

- 每个PID namespace中的第一个进程“PID 1“，都会像传统Linux中的init进程一样拥有特权，起特殊作用。
- 一个namespace中的进程，不可能通过kill或ptrace影响父节点或者兄弟节点中的进程，因为其他节点的PID在这个namespace中没有任何意义。
- 如果你在新的PID namespace中重新挂载/proc文件系统，会发现其下只显示同属一个PID namespace中的其他进程。
- 在root namespace中可以看到所有的进程，并且递归包含所有子节点中的进程。

到这里，可能你已经联想到一种在外部监控Docker中运行程序的方法了，就是监控Docker Daemon所在的PID namespace下的所有进程即其子进程，再进行删选即可。

下面我们通过运行代码来感受一下PID namespace的隔离效果。修改上文的代码，加入PID namespace的标识位，并把程序命名为pid.c。

```
//[...]
int child_pid = clone(child_main, child_stack+STACK_SIZE,
           CLONE_NEWPID | CLONE_NEWIPC | CLONE_NEWUTS 
           | SIGCHLD, NULL);
//[...]
```

编译运行可以看到如下结果。

```
root@local:~# gcc -Wall pid.c -o pid.o && ./pid.o
程序开始:
在子进程中!
root@NewNamespace:~# echo $$
1                      <<--注意此处看到shell的PID变成了1
root@NewNamespace:~# exit
exit
已退出
```

打印$$可以看到shell的PID，退出后如果再次执行可以看到效果如下。

```
root@local:~# echo $$
17542
```

已经回到了正常状态。可能有的读者在子进程的shell中执行了ps aux/top之类的命令，发现还是可以看到所有父进程的PID，那是因为我们还没有对文件系统进行隔离，ps/top之类的命令调用的是真实系统下的/proc文件内容，看到的自然是所有的进程。

此外，与其他的namespace不同的是，为了实现一个稳定安全的容器，PID namespace还需要进行一些额外的工作才能确保其中的进程运行顺利。

### （1）PID namespace中的init进程

当我们新建一个PID namespace时，默认启动的进程PID为1。我们知道，在传统的UNIX系统中，PID为1的进程是init，地位非常特殊。他作为所有进程的父进程，维护一张进程表，不断检查进程的状态，一旦有某个子进程因为程序错误成为了“孤儿”进程，init就会负责回收资源并结束这个子进程。所以在你要实现的容器中，启动的第一个进程也需要实现类似init的功能，维护所有后续启动进程的运行状态。

看到这里，可能读者已经明白了内核设计的良苦用心。PID namespace维护这样一个树状结构，非常有利于系统的资源监控与回收。Docker启动时，第一个进程也是这样，实现了进程监控和资源回收，它就是dockerinit。

### （2）信号与init进程

PID namespace中的init进程如此特殊，自然内核也为他赋予了特权——信号屏蔽。如果init中没有写处理某个信号的代码逻辑，那么与init在同一个PID namespace下的进程（即使有超级权限）发送给它的该信号都会被屏蔽。这个功能的主要作用是防止init进程被误杀。

那么其父节点PID namespace中的进程发送同样的信号会被忽略吗？父节点中的进程发送的信号，如果不是SIGKILL（销毁进程）或SIGSTOP（暂停进程）也会被忽略。但如果发送SIGKILL或SIGSTOP，子节点的init会强制执行（无法通过代码捕捉进行特殊处理），也就是说父节点中的进程有权终止子节点中的进程。

一旦init进程被销毁，同一PID namespace中的其他进程也会随之接收到SIGKILL信号而被销毁。理论上，该PID namespace自然也就不复存在了。但是如果/proc/[pid]/ns/pid处于被挂载或者打开状态，namespace就会被保留下来。然而，保留下来的namespace无法通过setns()或者fork()创建进程，所以实际上并没有什么作用。

我们常说，Docker一旦启动就有进程在运行，不存在不包含任何进程的Docker，也就是这个道理。

### （3）挂载proc文件系统

前文中已经提到，如果你在新的PID namespace中使用ps命令查看，看到的还是所有的进程，因为与PID直接相关的/proc文件系统（procfs）没有挂载到与原/proc不同的位置。所以如果你只想看到PID namespace本身应该看到的进程，需要重新挂载/proc，命令如下。

```
root@NewNamespace:~# mount -t proc proc /proc
root@NewNamespace:~# ps a
  PID TTY      STAT   TIME COMMAND
    1 pts/1    S      0:00 /bin/bash
   12 pts/1    R+     0:00 ps a
```

可以看到实际的PID namespace就只有两个进程在运行。

注意：因为此时我们没有进行mount namespace的隔离，所以这一步操作实际上已经影响了 root namespace的文件系统，当你退出新建的PID namespace以后再执行ps a就会发现出错，再次执行mount -t proc proc /proc可以修复错误。

### （4）unshare()和setns()

在开篇我们就讲到了unshare()和setns()这两个API，而这两个API在PID namespace中使用时，也有一些特别之处需要注意。

unshare()允许用户在原有进程中建立namespace进行隔离。但是创建了PID namespace后，原先unshare()调用者进程并不进入新的PID namespace，接下来创建的子进程才会进入新的namespace，这个子进程也就随之成为新namespace中的init进程。

类似的，调用setns()创建新PID namespace时，调用者进程也不进入新的PID namespace，而是随后创建的子进程进入。

为什么创建其他namespace时unshare()和setns()会直接进入新的namespace而唯独PID namespace不是如此呢？因为调用getpid()函数得到的PID是根据调用者所在的PID namespace而决定返回哪个PID，进入新的PID namespace会导致PID产生变化。而对用户态的程序和库函数来说，他们都认为进程的PID是一个常量，PID的变化会引起这些进程奔溃。

换句话说，一旦程序进程创建以后，那么它的PID namespace的关系就确定下来了，进程不会变更他们对应的PID namespace。

## 5. Mount namespaces

Mount namespace通过隔离文件系统挂载点对隔离文件系统提供支持，它是历史上第一个Linux namespace，所以它的标识位比较特殊，就是CLONE_NEWNS。隔离后，不同mount namespace中的文件结构发生变化也互不影响。你可以通过/proc/[pid]/mounts查看到所有挂载在当前namespace中的文件系统，还可以通过/proc/[pid]/mountstats看到mount namespace中文件设备的统计信息，包括挂载文件的名字、文件系统类型、挂载位置等等。

进程在创建mount namespace时，会把当前的文件结构复制给新的namespace。新namespace中的所有mount操作都只影响自身的文件系统，而对外界不会产生任何影响。这样做非常严格地实现了隔离，但是某些情况可能并不适用。比如父节点namespace中的进程挂载了一张CD-ROM，这时子节点namespace拷贝的目录结构就无法自动挂载上这张CD-ROM，因为这种操作会影响到父节点的文件系统。

2006 年引入的挂载传播（mount propagation）解决了这个问题，挂载传播定义了挂载对象（mount object）之间的关系，系统用这些关系决定任何挂载对象中的挂载事件如何传播到其他挂载对象（参考自：[http://www.ibm.com/developerworks/library/l-mount-namespaces/](http://www.ibm.com/developerworks/library/l-mount-namespaces/])）。所谓传播事件，是指由一个挂载对象的状态变化导致的其它挂载对象的挂载与解除挂载动作的事件。

- 共享关系（share relationship）。如果两个挂载对象具有共享关系，那么一个挂载对象中的挂载事件会传播到另一个挂载对象，反之亦然。
- 从属关系（slave relationship）。如果两个挂载对象形成从属关系，那么一个挂载对象中的挂载事件会传播到另一个挂载对象，但是反过来不行；在这种关系中，从属对象是事件的接收者。

一个挂载状态可能为如下的其中一种：

- 共享挂载（shared）
- 从属挂载（slave）
- 共享/从属挂载（shared and slave）
- 私有挂载（private）
- 不可绑定挂载（unbindable）

传播事件的挂载对象称为共享挂载（shared mount）；接收传播事件的挂载对象称为从属挂载（slave mount）。既不传播也不接收传播事件的挂载对象称为私有挂载（private mount）。另一种特殊的挂载对象称为不可绑定的挂载（unbindable mount），它们与私有挂载相似，但是不允许执行绑定挂载，即创建mount namespace时这块文件对象不可被复制。

![img](https://res.infoq.com/articles/docker-kernel-knowledge-namespace-resource-isolation/zh/resources/0310020.png)

图1 mount各类挂载状态示意图

共享挂载的应用场景非常明显，就是为了文件数据的共享所必须存在的一种挂载方式；从属挂载更大的意义在于某些“只读”场景；私有挂载其实就是纯粹的隔离，作为一个独立的个体而存在；不可绑定挂载则有助于防止没有必要的文件拷贝，如某个用户数据目录，当根目录被递归式的复制时，用户目录无论从隐私还是实际用途考虑都需要有一个不可被复制的选项。

默认情况下，所有挂载都是私有的。设置为共享挂载的命令如下。

```
mount --make-shared <mount-object>
```

从共享挂载克隆的挂载对象也是共享的挂载；它们相互传播挂载事件。

设置为从属挂载的命令如下。

```
mount --make-slave <shared-mount-object>
```

从从属挂载克隆的挂载对象也是从属的挂载，它也从属于原来的从属挂载的主挂载对象。

将一个从属挂载对象设置为共享/从属挂载，可以执行如下命令或者将其移动到一个共享挂载对象下。

```
mount --make-shared <slave-mount-object>
```

如果你想把修改过的挂载对象重新标记为私有的，可以执行如下命令。

```
mount --make-private <mount-object>
```

通过执行以下命令，可以将挂载对象标记为不可绑定的。

```
mount --make-unbindable <mount-object>
```

这些设置都可以递归式地应用到所有子目录中，如果读者感兴趣可以搜索到相关的命令。

在代码中实现mount namespace隔离与其他namespace类似，加上CLONE_NEWNS标识位即可。让我们再次修改代码，并且另存为mount.c进行编译运行。

```
//[...]
int child_pid = clone(child_main, child_stack+STACK_SIZE,
           CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWIPC 
           | CLONE_NEWUTS | SIGCHLD, NULL);
//[...]
```

执行的效果就如同PID namespace一节中“挂载proc文件系统”的执行结果，区别就是退出mount namespace以后，root namespace的文件系统不会被破坏，此处就不再演示了。

## 6. Network namespace

通过上节，我们了解了PID namespace，当我们兴致勃勃地在新建的namespace中启动一个“Apache”进程时，却出现了“80端口已被占用”的错误，原来主机上已经运行了一个“Apache”进程。怎么办？这就需要用到network namespace技术进行网络隔离啦。

Network namespace主要提供了关于网络资源的隔离，包括网络设备、IPv4和IPv6协议栈、IP路由表、防火墙、/proc/net目录、/sys/class/net目录、端口（socket）等等。一个物理的网络设备最多存在在一个network namespace中，你可以通过创建veth pair（虚拟网络设备对：有两端，类似管道，如果数据从一端传入另一端也能接收到，反之亦然）在不同的network namespace间创建通道，以此达到通信的目的。

一般情况下，物理网络设备都分配在最初的root namespace（表示系统默认的namespace，在PID namespace中已经提及）中。但是如果你有多块物理网卡，也可以把其中一块或多块分配给新创建的network namespace。需要注意的是，当新创建的network namespace被释放时（所有内部的进程都终止并且namespace文件没有被挂载或打开），在这个namespace中的物理网卡会返回到root namespace而非创建该进程的父进程所在的network namespace。

当我们说到network namespace时，其实我们指的未必是真正的网络隔离，而是把网络独立出来，给外部用户一种透明的感觉，仿佛跟另外一个网络实体在进行通信。为了达到这个目的，容器的经典做法就是创建一个veth pair，一端放置在新的namespace中，通常命名为eth0，一端放在原先的namespace中连接物理网络设备，再通过网桥把别的设备连接进来或者进行路由转发，以此网络实现通信的目的。

也许有读者会好奇，在建立起veth pair之前，新旧namespace该如何通信呢？答案是pipe（管道）。我们以Docker Daemon在启动容器dockerinit的过程为例。Docker Daemon在宿主机上负责创建这个veth pair，通过netlink调用，把一端绑定到docker0网桥上，一端连进新建的network namespace进程中。建立的过程中，Docker Daemon和dockerinit就通过pipe进行通信，当Docker Daemon完成veth-pair的创建之前，dockerinit在管道的另一端循环等待，直到管道另一端传来Docker Daemon关于veth设备的信息，并关闭管道。dockerinit才结束等待的过程，并把它的“eth0”启动起来。整个效果类似下图所示。

![img](https://res.infoq.com/articles/docker-kernel-knowledge-namespace-resource-isolation/zh/resources/0310021.png)

图2 Docker网络示意图

跟其他namespace类似，对network namespace的使用其实就是在创建的时候添加CLONE_NEWNET标识位。也可以通过命令行工具ip创建network namespace。在代码中建立和测试network namespace较为复杂，所以下文主要通过ip命令直观的感受整个network namespace网络建立和配置的过程。

首先我们可以创建一个命名为test_ns的network namespace。

```
# ip netns add test_ns
```

当ip命令工具创建一个network namespace时，会默认创建一个回环设备（loopback interface：lo），并在/var/run/netns目录下绑定一个挂载点，这就保证了就算network namespace中没有进程在运行也不会被释放，也给系统管理员对新创建的network namespace进行配置提供了充足的时间。

通过ip netns exec命令可以在新创建的network namespace下运行网络管理命令。

```
# ip netns exec test_ns ip link list
3: lo: <LOOPBACK> mtu 16436 qdisc noop state DOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

上面的命令为我们展示了新建的namespace下可见的网络链接，可以看到状态是DOWN,需要再通过命令去启动。可以看到，此时执行ping命令是无效的。

```
# ip netns exec test_ns ping 127.0.0.1
connect: Network is unreachable
```

启动命令如下，可以看到启动后再测试就可以ping通。

```
# ip netns exec test_ns ip link set dev lo up
# ip netns exec test_ns ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_req=1 ttl=64 time=0.050 ms
...
```

这样只是启动了本地的回环，要实现与外部namespace进行通信还需要再建一个网络设备对，命令如下。

```
# ip link add veth0 type veth peer name veth1
# ip link set veth1 netns test_ns
# ip netns exec test_ns ifconfig veth1 10.1.1.1/24 up
# ifconfig veth0 10.1.1.2/24 up
```

- 第一条命令创建了一个网络设备对，所有发送到veth0的包veth1也能接收到，反之亦然。
- 第二条命令则是把veth1这一端分配到test_ns这个network namespace。
- 第三、第四条命令分别给test_ns内部和外部的网络设备配置IP，veth1的IP为10.1.1.1，veth0的IP为10.1.1.2。

此时两边就可以互相连通了，效果如下。

```
# ping 10.1.1.1
PING 10.1.1.1 (10.1.1.1) 56(84) bytes of data.
64 bytes from 10.1.1.1: icmp_req=1 ttl=64 time=0.095 ms
...
# ip netns exec test_ns ping 10.1.1.2
PING 10.1.1.2 (10.1.1.2) 56(84) bytes of data.
64 bytes from 10.1.1.2: icmp_req=1 ttl=64 time=0.049 ms
...
```

读者有兴趣可以通过下面的命令查看，新的test_ns有着自己独立的路由和iptables。

```
ip netns exec test_ns route
ip netns exec test_ns iptables -L
```

路由表中只有一条通向10.1.1.2的规则，此时如果要连接外网肯定是不可能的，你可以通过建立网桥或者NAT映射来决定这个问题。如果你对此非常感兴趣，可以阅读Docker网络相关文章进行更深入的讲解。

做完这些实验，你还可以通过下面的命令删除这个network namespace。

```
# ip netns delete netns1
```

这条命令会移除之前的挂载，但是如果namespace本身还有进程运行，namespace还会存在下去，直到进程运行结束。

通过network namespace我们可以了解到，实际上内核创建了network namespace以后，真的是得到了一个被隔离的网络。但是我们实际上需要的不是这种完全的隔离，而是一个对用户来说透明独立的网络实体，我们需要与这个实体通信。所以Docker的网络在起步阶段给人一种非常难用的感觉，因为一切都要自己去实现、去配置。你需要一个网桥或者NAT连接广域网，你需要配置路由规则与宿主机中其他容器进行必要的隔离，你甚至还需要配置防火墙以保证安全等等。所幸这一切已经有了较为成熟的方案，我们会在Docker网络部分进行详细的讲解。

## 7. User namespaces

User namespace主要隔离了安全相关的标识符（identifiers）和属性（attributes），包括用户ID、用户组ID、root目录、[key](http://man7.org/linux/man-pages/man2/keyctl.2.html)（指密钥）以及[特殊权限](http://man7.org/linux/man-pages/man7/capabilities.7.html)。说得通俗一点，一个普通用户的进程通过clone()创建的新进程在新user namespace中可以拥有不同的用户和用户组。这意味着一个进程在容器外属于一个没有特权的普通用户，但是他创建的容器进程却属于拥有所有权限的超级用户，这个技术为容器提供了极大的自由。

User namespace是目前的六个namespace中最后一个支持的，并且直到Linux内核3.8版本的时候还未完全实现（还有部分文件系统不支持）。因为user namespace实际上并不算完全成熟，很多发行版担心安全问题，在编译内核的时候并未开启USER_NS。实际上目前Docker也还不支持user namespace，但是预留了相应接口，相信在不久后就会支持这一特性。所以在进行接下来的代码实验时，请确保你系统的Linux内核版本高于3.8并且内核编译时开启了USER_NS（如果你不会选择，可以使用Ubuntu14.04）。

Linux中，特权用户的user ID就是0，演示的最终我们将看到user ID非0的进程启动user namespace后user ID可以变为0。使用user namespace的方法跟别的namespace相同，即调用clone()或unshare()时加入CLONE_NEWUSER标识位。老样子，修改代码并另存为userns.c，为了看到用户[权限(Capabilities)](http://man7.org/linux/man-pages/man7/capabilities.7.html)，可能你还需要安装一下libcap-dev包。

首先包含以下头文件以调用Capabilities包。

```
#include <sys/capability.h>
```

其次在子进程函数中加入geteuid()和getegid()得到namespace内部的user ID，其次通过cap_get_proc()得到当前进程的用户拥有的权限，并通过cap_to_text（）输出。

```
int child_main(void* args) {
        printf("在子进程中!\n");
        cap_t caps;
        printf("eUID = %ld;  eGID = %ld;  ",
                        (long) geteuid(), (long) getegid());
        caps = cap_get_proc();
        printf("capabilities: %s\n", cap_to_text(caps, NULL));
        execv(child_args[0], child_args);
        return 1;
}
```

在主函数的clone()调用中加入我们熟悉的标识符。

```
//[...]
int child_pid = clone(child_main, child_stack+STACK_SIZE,
            CLONE_NEWUSER | SIGCHLD, NULL);
//[...]
```

至此，第一部分的代码修改就结束了。在编译之前我们先查看一下当前用户的uid和guid，请注意此时我们是普通用户。

```
$ id -u
1000
$ id -g
1000
```

然后我们开始编译运行，并进行新建的user namespace，你会发现shell提示符前的用户名已经变为nobody。

```
sun@ubuntu$ gcc userns.c -Wall -lcap -o userns.o && ./userns.o
程序开始:
在子进程中!
eUID = 65534;  eGID = 65534;  capabilities: = cap_chown,cap_dac_override,[...]37+ep  <<--此处省略部分输出，已拥有全部权限
nobody@ubuntu$ 
```

通过验证我们可以得到以下信息。

- user namespace被创建后，第一个进程被赋予了该namespace中的全部权限，这样这个init进程就可以完成所有必要的初始化工作，而不会因权限不足而出现错误。
- 我们看到namespace内部看到的UID和GID已经与外部不同了，默认显示为65534，表示尚未与外部namespace用户映射。我们需要对user namespace内部的这个初始user和其外部namespace某个用户建立映射，这样可以保证当涉及到一些对外部namespace的操作时，系统可以检验其权限（比如发送一个信号或操作某个文件）。同样用户组也要建立映射。
- 还有一点虽然不能从输出中看出来，但是值得注意。用户在新namespace中有全部权限，但是他在创建他的父namespace中不含任何权限。就算调用和创建他的进程有全部权限也是如此。所以哪怕是root用户调用了clone()在user namespace中创建出的新用户在外部也没有任何权限。
- 最后，user namespace的创建其实是一个层层嵌套的树状结构。最上层的根节点就是root namespace，新创建的每个user namespace都有一个父节点user namespace以及零个或多个子节点user namespace，这一点与PID namespace非常相似。

接下来我们就要进行用户绑定操作，通过在/proc/[pid]/uid_map和/proc/[pid]/gid_map两个文件中写入对应的绑定信息可以实现这一点，格式如下。

```
ID-inside-ns   ID-outside-ns   length
```

写这两个文件需要注意以下几点。

- 这两个文件只允许由拥有该user namespace中CAP_SETUID权限的进程写入一次，不允许修改。
- 写入的进程必须是该user namespace的父namespace或者子namespace。
- 第一个字段ID-inside-ns表示新建的user namespace中对应的user/group ID，第二个字段ID-outside-ns表示namespace外部映射的user/group ID。最后一个字段表示映射范围，通常填1，表示只映射一个，如果填大于1的值，则按顺序建立一一映射。

明白了上述原理，我们再次修改代码，添加设置uid和guid的函数。

```
//[...]
void set_uid_map(pid_t pid, int inside_id, int outside_id, int length) {
    char path[256];
    sprintf(path, "/proc/%d/uid_map", getpid());
    FILE* uid_map = fopen(path, "w");
    fprintf(uid_map, "%d %d %d", inside_id, outside_id, length);
    fclose(uid_map);
}
void set_gid_map(pid_t pid, int inside_id, int outside_id, int length) {
    char path[256];
    sprintf(path, "/proc/%d/gid_map", getpid());
    FILE* gid_map = fopen(path, "w");
    fprintf(gid_map, "%d %d %d", inside_id, outside_id, length);
    fclose(gid_map);
}
int child_main(void* args) {
    cap_t caps;
    printf("在子进程中!\n");
    set_uid_map(getpid(), 0, 1000, 1);
    set_gid_map(getpid(), 0, 1000, 1);
    printf("eUID = %ld;  eGID = %ld;  ",
            (long) geteuid(), (long) getegid());
    caps = cap_get_proc();
    printf("capabilities: %s\n", cap_to_text(caps, NULL));
    execv(child_args[0], child_args);
    return 1;
}
//[...]
```

编译后即可看到user已经变成了root。

```
$ gcc userns.c -Wall -lcap -o usernc.o && ./usernc.o
程序开始:
在子进程中!
eUID = 0;  eGID = 0;  capabilities: = [...],37+ep
root@ubuntu:~#
```

至此，你就已经完成了绑定的工作，可以看到演示全程都是在普通用户下执行的。最终实现了在user namespace中成为了root而对应到外面的是一个uid为1000的普通用户。

如果你要把user namespace与其他namespace混合使用，那么依旧需要root权限。解决方案可以是先以普通用户身份创建user namespace，然后在新建的namespace中作为root再clone()进程加入其他类型的namespace隔离。

讲完了user namespace，我们再来谈谈Docker。虽然Docker目前尚未使用user namespace，但是他用到了我们在user namespace中提及的Capabilities机制。从内核2.2版本开始，Linux把原来和超级用户相关的高级权限划分成为不同的单元，称为Capability。这样管理员就可以独立对特定的Capability进行使能或禁止。Docker虽然没有使用user namespace，但是他可以禁用容器中不需要的Capability，一次在一定程度上加强容器安全性。

当然，说到安全，namespace的六项隔离看似全面，实际上依旧没有完全隔离Linux的资源，比如SELinux、 Cgroups以及/sys、/proc/sys、/dev/sd*等目录下的资源。关于安全的更多讨论和讲解，我们会在后文中接着探讨。

## 8. 总结

本文从namespace使用的API开始，结合Docker逐步对六个namespace进行讲解。相信把讲解过程中所有的代码整合起来，你也能实现一个属于自己的“shell”容器了。虽然namespace技术使用起来非常简单，但是要真正把容器做到安全易用却并非易事。PID namespace中，我们要实现一个完善的init进程来维护好所有进程；network namespace中，我们还有复杂的路由表和iptables规则没有配置；user namespace中还有很多权限上的问题需要考虑等等。其中有些方面Docker已经做的很好，有些方面也才刚刚开始。希望通过本文，能为大家更好的理解Docker背后运行的原理提供帮助。





# Docker背后的内核知识——cgroups资源限制



上一篇中，我们了解了Docker背后使用的资源隔离技术namespace，通过系统调用构建一个相对隔离的shell环境，也可以称之为一个简单的“容器”。本文我们则要开始讲解另一个强大的内核工具——cgroups。他不仅可以限制被namespace隔离起来的资源，还可以为资源设置权重、计算使用量、操控进程启停等等。在介绍完基本概念后，我们将详细讲解Docker中使用到的cgroups内容。希望通过本文，让读者对Docker有更深入的了解。

## 1. cgroups是什么

cgroups（Control Groups）最初叫Process Container，由Google工程师（Paul Menage和Rohit Seth）于2006年提出，后来因为Container有多重含义容易引起误解，就在2007年更名为Control Groups，并被整合进Linux内核。顾名思义就是把进程放到一个组里面统一加以控制。官方的定义如下{![引自：https://www.kernel.org/doc/Documentation/cgroups/cgroups.txt]}。

> 
>
> cgroups是Linux内核提供的一种机制，这种机制可以根据特定的行为，把一系列系统任务及其子任务整合（或分隔）到按资源划分等级的不同组内，从而为系统资源管理提供一个统一的框架。

通俗的来说，cgroups可以限制、记录、隔离进程组所使用的物理资源（包括：CPU、memory、IO等），为容器实现虚拟化提供了基本保证，是构建Docker等一系列虚拟化管理工具的基石。

对开发者来说，cgroups有如下四个有趣的特点：

- cgroups的API以一个伪文件系统的方式实现，即用户可以通过文件操作实现cgroups的组织管理。
- cgroups的组织管理操作单元可以细粒度到线程级别，用户态代码也可以针对系统分配的资源创建和销毁cgroups，从而实现资源再分配和管理。
- 所有资源管理的功能都以“subsystem（子系统）”的方式实现，接口统一。
- 子进程创建之初与其父进程处于同一个cgroups的控制组。

本质上来说，cgroups是内核附加在程序上的一系列钩子（hooks），通过程序运行时对资源的调度触发相应的钩子以达到资源追踪和限制的目的。

## 2. cgroups的作用

实现cgroups的主要目的是为不同用户层面的资源管理，提供一个统一化的接口。从单个进程的资源控制到操作系统层面的虚拟化。Cgroups提供了以下四大功能{![参照自：http://en.wikipedia.org/wiki/Cgroups]}。

- 资源限制（Resource Limitation）：cgroups可以对进程组使用的资源总额进行限制。如设定应用运行时使用内存的上限，一旦超过这个配额就发出OOM（Out of Memory）。
- 优先级分配（Prioritization）：通过分配的CPU时间片数量及硬盘IO带宽大小，实际上就相当于控制了进程运行的优先级。
- 资源统计（Accounting）： cgroups可以统计系统的资源使用量，如CPU使用时长、内存用量等等，这个功能非常适用于计费。
- 进程控制（Control）：cgroups可以对进程组执行挂起、恢复等操作。

过去有一段时间，内核开发者甚至把namespace也作为一个cgroups的subsystem加入进来，也就是说cgroups曾经甚至还包含了资源隔离的能力。但是资源隔离会给cgroups带来许多问题，如PID在循环出现的时候cgroup却出现了命名冲突、cgroup创建后进入新的namespace导致脱离了控制等等{![详见：https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=a77aea92010acf54ad785047234418d5d68772e2]}，所以在2011年就被移除了。

## 3. 术语表

- **task（任务）**：cgroups的术语中，task就表示系统的一个进程。
- **cgroup（控制组）**：cgroups 中的资源控制都以cgroup为单位实现。cgroup表示按某种资源控制标准划分而成的任务组，包含一个或多个子系统。一个任务可以加入某个cgroup，也可以从某个cgroup迁移到另外一个cgroup。
- **subsystem（子系统）**：cgroups中的subsystem就是一个资源调度控制器（Resource Controller）。比如CPU子系统可以控制CPU时间分配，内存子系统可以限制cgroup内存使用量。
- **hierarchy（层级树）**：hierarchy由一系列cgroup以一个树状结构排列而成，每个hierarchy通过绑定对应的subsystem进行资源调度。hierarchy中的cgroup节点可以包含零或多个子节点，子节点继承父节点的属性。整个系统可以有多个hierarchy。

## 4. 组织结构与基本规则

大家在namespace技术的讲解中已经了解到，传统的Unix进程管理，实际上是先启动`init`进程作为根节点，再由`init`节点创建子进程作为子节点，而每个子节点由可以创建新的子节点，如此往复，形成一个树状结构。而cgroups也是类似的树状结构，子节点都从父节点继承属性。

它们最大的不同在于，系统中cgroup构成的hierarchy可以允许存在多个。如果进程模型是由`init`作为根节点构成的一棵树的话，那么cgroups的模型则是由多个hierarchy构成的森林。这样做的目的也很好理解，如果只有一个hierarchy，那么所有的task都要受到绑定其上的subsystem的限制，会给那些不需要这些限制的task造成麻烦。

了解了cgroups的组织结构，我们再来了解cgroup、task、subsystem以及hierarchy四者间的相互关系及其基本规则{![参照自：https://access.redhat.com/documentation/en-US/Red*Hat*Enterprise*Linux/6/html/ResourceManagementGuide/sec-Relationships*Between*Subsystems*Hierarchies*ControlGroupsand*Tasks.html]}。

- **规则1：** 同一个hierarchy可以附加一个或多个subsystem。如下图1，cpu和memory的subsystem附加到了一个hierarchy。

  ![img](https://res.infoq.com/articles/docker-kernel-knowledge-cgroups-resource-isolation/zh/resources/0329010.png)

  **图1 同一个hierarchy可以附加一个或多个subsystem**

- **规则2：** 一个subsystem可以附加到多个hierarchy，当且仅当这些hierarchy只有这唯一一个subsystem。如下图2，小圈中的数字表示subsystem附加的时间顺序，CPU subsystem附加到hierarchy A的同时不能再附加到hierarchy B，因为hierarchy B已经附加了memory subsystem。如果hierarchy B与hierarchy A状态相同，没有附加过memory subsystem，那么CPU subsystem同时附加到两个hierarchy是可以的。

  ![img](https://res.infoq.com/articles/docker-kernel-knowledge-cgroups-resource-isolation/zh/resources/0329011.png)

  **图2 一个已经附加在某个hierarchy上的subsystem不能附加到其他含有别的subsystem的hierarchy上**

- **规则3：** 系统每次新建一个hierarchy时，该系统上的所有task默认构成了这个新建的hierarchy的初始化cgroup，这个cgroup也称为root cgroup。对于你创建的每个hierarchy，task只能存在于其中一个cgroup中，即一个task不能存在于同一个hierarchy的不同cgroup中，但是一个task可以存在在不同hierarchy中的多个cgroup中。如果操作时把一个task添加到同一个hierarchy中的另一个cgroup中，则会从第一个cgroup中移除。在下图3中可以看到，`httpd`进程已经加入到hierarchy A中的`/cg1`而不能加入同一个hierarchy中的`/cg2`，但是可以加入hierarchy B中的`/cg3`。实际上不允许加入同一个hierarchy中的其他cgroup野生为了防止出现矛盾，如CPU subsystem为`/cg1`分配了30%，而为`/cg2`分配了50%，此时如果`httpd`在这两个cgroup中，就会出现矛盾。

  ![img](https://res.infoq.com/articles/docker-kernel-knowledge-cgroups-resource-isolation/zh/resources/0329012.png)

  **图3 一个task不能属于同一个hierarchy的不同cgroup**

- **规则4：** 进程（task）在fork自身时创建的子任务（child task）默认与原task在同一个cgroup中，但是child task允许被移动到不同的cgroup中。即fork完成后，父子进程间是完全独立的。如下图4中，小圈中的数字表示task 出现的时间顺序，当`httpd`刚fork出另一个`httpd`时，在同一个hierarchy中的同一个cgroup中。但是随后如果PID为4840的`httpd`需要移动到其他cgroup也是可以的，因为父子任务间已经独立。总结起来就是：初始化时子任务与父任务在同一个cgroup，但是这种关系随后可以改变。

  ![img](https://res.infoq.com/articles/docker-kernel-knowledge-cgroups-resource-isolation/zh/resources/0329013.png)

  **图4 刚fork出的子进程在初始状态与其父进程处于同一个cgroup**

## 5. subsystem简介

subsystem实际上就是cgroups的资源控制系统，每种subsystem独立地控制一种资源，目前Docker使用如下八种subsystem，还有一种`net_cls` subsystem在内核中已经广泛实现，但是Docker尚未使用。他们的用途分别如下。

- **blkio：** 这个subsystem可以为块设备设定输入/输出限制，比如物理驱动设备（包括磁盘、固态硬盘、USB等）。
- **cpu：** 这个subsystem使用调度程序控制task对CPU的使用。
- **cpuacct：** 这个subsystem自动生成cgroup中task对CPU资源使用情况的报告。
- **cpuset：** 这个subsystem可以为cgroup中的task分配独立的CPU（此处针对多处理器系统）和内存。
- **devices** 这个subsystem可以开启或关闭cgroup中task对设备的访问。
- **freezer** 这个subsystem可以挂起或恢复cgroup中的task。
- **memory** 这个subsystem可以设定cgroup中task对内存使用量的限定，并且自动生成这些task对内存资源使用情况的报告。
- **perfevent*** 这个subsystem使用后使得cgroup中的task可以进行统一的性能测试。{![perf: Linux CPU性能探测器，详见https://perf.wiki.kernel.org/index.php/Main*Page]}
- ***net_cls** 这个subsystem Docker没有直接使用，它通过使用等级识别符(classid)标记网络数据包，从而允许 Linux 流量控制程序（TC：Traffic Controller）识别从具体cgroup中生成的数据包。

## 6. cgroups实现方式及工作原理简介

### （1）cgroups实现结构讲解

cgroups的实现本质上是给系统进程挂上钩子（hooks），当task运行的过程中涉及到某个资源时就会触发钩子上所附带的subsystem进行检测，最终根据资源类别的不同使用对应的技术进行资源限制和优先级分配。那么这些钩子又是怎样附加到进程上的呢？下面我们将对照结构体的图表一步步分析，请放心，描述代码的内容并不多。

(点击放大图像)

[![img](https://res.infoq.com/articles/docker-kernel-knowledge-cgroups-resource-isolation/zh/resources/0329014.png)](https://res.infoq.com/articles/docker-kernel-knowledge-cgroups-resource-isolation/zh/resources/0329014.png)

**图5 cgroups相关结构体一览**

Linux中管理task进程的数据结构为`task_struct`（包含所有进程管理的信息），其中与cgroup相关的字段主要有两个，一个是`css_set *cgroups`，表示指向`css_set`（包含进程相关的cgroups信息）的指针，一个task只对应一个`css_set`结构，但是一个`css_set`可以被多个task使用。另一个字段是`list_head cg_list`，是一个链表的头指针，这个链表包含了所有的链到同一个`css_set`的task进程（在图中使用的回环箭头，均表示可以通过该字段找到所有同类结构，获得信息）。

每个`css_set`结构中都包含了一个指向`cgroup_subsys_state`（包含进程与一个特定子系统相关的信息）的指针数组。`cgroup_subsys_state`则指向了`cgroup`结构（包含一个cgroup的所有信息），通过这种方式间接的把一个进程和cgroup联系了起来，如下图6。

![img](https://res.infoq.com/articles/docker-kernel-knowledge-cgroups-resource-isolation/zh/resources/0329015.png)

**图6 从task结构开始找到cgroup结构**

另一方面，`cgroup`结构体中有一个`list_head css_sets`字段，它是一个头指针，指向由`cg_cgroup_link`（包含cgroup与task之间多对多关系的信息，后文还会再解释）形成的链表。由此获得的每一个`cg_cgroup_link`都包含了一个指向`css_set *cg`字段，指向了每一个task的`css_set`。`css_set`结构中则包含`tasks`头指针，指向所有链到此`css_set`的task进程构成的链表。至此，我们就明白如何查看在同一个cgroup中的task有哪些了，如下图7。

![img](https://res.infoq.com/articles/docker-kernel-knowledge-cgroups-resource-isolation/zh/resources/0329016.png)

**图7 cglink多对多双向查询**

细心的读者可能已经发现，`css_set`中也有指向所有`cg_cgroup_link`构成链表的头指针，通过这种方式也能定位到所有的cgroup，这种方式与图1中所示的方式得到的结果是相同的。

那么为什么要使用`cg_cgroup_link`结构体呢？因为task与cgroup之间是多对多的关系。熟悉数据库的读者很容易理解，在数据库中，如果两张表是多对多的关系，那么如果不加入第三张关系表，就必须为一个字段的不同添加许多行记录，导致大量冗余。通过从主表和副表各拿一个主键新建一张关系表，可以提高数据查询的灵活性和效率。

而一个task可能处于不同的cgroup，只要这些cgroup在不同的hierarchy中，并且每个hierarchy挂载的子系统不同；另一方面，一个cgroup中可以有多个task，这是显而易见的，但是这些task因为可能还存在在别的cgroup中，所以它们对应的`css_set`也不尽相同，所以一个cgroup也可以对应多个·`css_set`。

在系统运行之初，内核的主函数就会对`root cgroups`和`css_set`进行初始化，每次task进行fork/exit时，都会附加（attach）/分离（detach）对应的`css_set`。

综上所述，添加`cg_cgroup_link`主要是出于性能方面的考虑，一是节省了`task_struct`结构体占用的内存，二是提升了进程`fork()/exit()`的速度。

![img](https://res.infoq.com/articles/docker-kernel-knowledge-cgroups-resource-isolation/zh/resources/0329017.png)

**图8 css_set与hashtable关系**

当task从一个cgroup中移动到另一个时，它会得到一个新的`css_set`指针。如果所要加入的cgroup与现有的cgroup子系统相同，那么就重复使用现有的`css_set`，否则就分配一个新`css_set`。所有的`css_set`通过一个哈希表进行存放和查询，如上图8中所示，`hlist_node hlist`就指向了`css_set_table`这个hash表。

同时，为了让cgroups便于用户理解和使用，也为了用精简的内核代码为cgroup提供熟悉的权限和命名空间管理，内核开发者们按照Linux 虚拟文件系统转换器（VFS：Virtual Filesystem Switch）的接口实现了一套名为`cgroup`的文件系统，非常巧妙地用来表示cgroups的hierarchy概念，把各个subsystem的实现都封装到文件系统的各项操作中。有兴趣的读者可以在网上搜索并阅读[VFS](http://en.wikipedia.org/wiki/Virtual_file_system)的相关内容，在此就不赘述了。

定义子系统的结构体是`cgroup_subsys`，在图9中可以看到，`cgroup_subsys`中定义了一组函数的接口，让各个子系统自己去实现，类似的思想还被用在了`cgroup_subsys_state`中，`cgroup_subsys_state`并没有定义控制信息，只是定义了各个子系统都需要用到的公共信息，由各个子系统各自按需去定义自己的控制信息结构体，最终在自定义的结构体中把`cgroup_subsys_state`包含进去，然后内核通过`container_of`（这个宏可以通过一个结构体的成员找到结构体自身）等宏定义来获取对应的结构体。

![img](https://res.infoq.com/articles/docker-kernel-knowledge-cgroups-resource-isolation/zh/resources/0329018.png)

**图9 cgroup子系统结构体**

### （2）基于cgroups实现结构的用户层体现

了解了cgroups实现的代码结构以后，再来看用户层在使用cgroups时的限制，会更加清晰。

在实际的使用过程中，你需要通过挂载（mount）`cgroup`文件系统新建一个层级结构，挂载时指定要绑定的子系统，缺省情况下默认绑定系统所有子系统。把cgroup文件系统挂载（mount）上以后，你就可以像操作文件一样对cgroups的hierarchy层级进行浏览和操作管理（包括权限管理、子文件管理等等）。除了cgroup文件系统以外，内核没有为cgroups的访问和操作添加任何系统调用。

如果新建的层级结构要绑定的子系统与目前已经存在的层级结构完全相同，那么新的挂载会重用原来已经存在的那一套（指向相同的css_set）。否则如果要绑定的子系统已经被别的层级绑定，就会返回挂载失败的错误。如果一切顺利，挂载完成后层级就被激活并与相应子系统关联起来，可以开始使用了。

目前无法将一个新的子系统绑定到激活的层级上，或者从一个激活的层级中解除某个子系统的绑定。

当一个顶层的cgroup文件系统被卸载（umount）时，如果其中创建后代cgroup目录，那么就算上层的cgroup被卸载了，层级也是激活状态，其后代cgoup中的配置依旧有效。只有递归式的卸载层级中的所有cgoup，那个层级才会被真正删除。

层级激活后，`/proc`目录下的每个task PID文件夹下都会新添加一个名为`cgroup`的文件，列出task所在的层级，对其进行控制的子系统及对应cgroup文件系统的路径。

一个cgroup创建完成，不管绑定了何种子系统，其目录下都会生成以下几个文件，用来描述cgroup的相应信息。同样，把相应信息写入这些配置文件就可以生效，内容如下。

- `tasks`：这个文件中罗列了所有在该cgroup中task的PID。该文件并不保证task的PID有序，把一个task的PID写到这个文件中就意味着把这个task加入这个cgroup中。
- `cgroup.procs`：这个文件罗列所有在该cgroup中的线程组ID。该文件并不保证线程组ID有序和无重复。写一个线程组ID到这个文件就意味着把这个组中所有的线程加到这个cgroup中。
- `notify_on_release`：填0或1，表示是否在cgroup中最后一个task退出时通知运行`release agent`，默认情况下是0，表示不运行。
- `release_agent`：指定release agent执行脚本的文件路径（该文件在最顶层cgroup目录中存在），在这个脚本通常用于自动化`umount`无用的cgroup。

除了上述几个通用的文件以外，绑定特定子系统的目录下也会有其他的文件进行子系统的参数配置。

在创建的hierarchy中创建文件夹，就类似于fork中一个后代cgroup，后代cgroup中默认继承原有cgroup中的配置属性，但是你可以根据需求对配置参数进行调整。这样就把一个大的cgroup系统分割成一个个嵌套的、可动态变化的“软分区”。

## 7. cgroups的使用方法简介

### （1）安装cgroups工具库

本节主要针对Ubuntu14.04版本系统进行介绍，其他Linux发行版命令略有不同，原理是一样的。不安装cgroups工具库也可以使用cgroups，安装它只是为了更方便的在用户态对cgroups进行管理，同时也方便初学者理解和使用，本节对cgroups的操作和使用都基于这个工具库。

```
apt-get install cgroup-bin
```

安装的过程会自动创建`/cgroup`目录，如果没有自动创建也不用担心，使用 `mkdir /cgroup` 手动创建即可。在这个目录下你就可以挂载各类子系统。安装完成后，你就可以使用`lssubsys`（罗列所有的subsystem挂载情况）等命令。

说明：也许你在其他文章中看到的cgroups工具库教程，会在/etc目录下生成一些初始化脚本和配置文件，默认的cgroup配置文件为`/etc/cgconfig.conf`，但是因为存在使LXC无法运行的bug，所以在新版本中把这个配置移除了，详见：https://bugs.launchpad.net/ubuntu/+source/libcgroup/+bug/1096771。

### （2）查询cgroup及子系统挂载状态

在挂载子系统之前，可能你要先检查下目前子系统的挂载状态，如果子系统已经挂载，根据第4节中讲的规则2，你就无法把子系统挂载到新的hierarchy，此时就需要先删除相应hierarchy或卸载对应子系统后再挂载。

- 查看所有的cgroup：`lscgroup`
- 查看所有支持的子系统：`lssubsys -a`
- 查看所有子系统挂载的位置： `lssubsys –m`
- 查看单个子系统（如memory）挂载位置：`lssubsys –m memory`

### （3）创建hierarchy层级并挂载子系统

在组织结构与规则一节中我们提到了hierarchy层级和subsystem子系统的关系，我们知道使用cgroup的最佳方式是：为想要管理的每个或每组资源创建单独的cgroup层级结构。而创建hierarchy并不神秘，实际上就是做一个标记，通过挂载一个tmpfs{![基于内存的临时文件系统，详见：http://en.wikipedia.org/wiki/Tmpfs]}文件系统，并给一个好的名字就可以了，系统默认挂载的cgroup就会进行如下操作。

```
mount -t tmpfs cgroups /sys/fs/cgroup
```

其中`-t`即指定挂载的文件系统类型，其后的`cgroups`是会出现在`mount`展示的结果中用于标识，可以选择一个有用的名字命名，最后的目录则表示文件的挂载点位置。

挂载完成`tmpfs`后就可以通过`mkdir`命令创建相应的文件夹。

```
mkdir /sys/fs/cgroup/cg1
```

再把子系统挂载到相应层级上，挂载子系统也使用mount命令，语法如下。

```
mount -t cgroup -o subsystems name /cgroup/name
```

其中 subsystems 是使用`,`（逗号）分开的子系统列表，name 是层级名称。具体我们以挂载cpu和memory的子系统为例，命令如下。

```
mount –t cgroup –o cpu,memory cpu_and_mem /sys/fs/cgroup/cg1
```

从`mount`命令开始，`-t`后面跟的是挂载的文件系统类型，即`cgroup`文件系统。`-o`后面跟要挂载的子系统种类如`cpu`、`memory`，用逗号隔开，其后的`cpu_and_mem`不被cgroup代码的解释，但会出现在/proc/mounts里，可以使用任何有用的标识字符串。最后的参数则表示挂载点的目录位置。

说明：如果挂载时提示`mount: agent already mounted or /cgroup busy`，则表示子系统已经挂载，需要先卸载原先的挂载点，通过第二条中描述的命令可以定位挂载点。

### （4）卸载cgroup

目前`cgroup`文件系统虽然支持重新挂载，但是官方不建议使用，重新挂载虽然可以改变绑定的子系统和`release agent`，但是它要求对应的hierarchy是空的并且release_agent会被传统的`fsnotify`（内核默认的文件系统通知）代替，这就导致重新挂载很难生效，未来重新挂载的功能可能会移除。你可以通过卸载，再挂载的方式处理这样的需求。

卸载cgroup非常简单，你可以通过`cgdelete`命令，也可以通过`rmdir`，以刚挂载的cg1为例，命令如下。

```
rmdir /sys/fs/cgroup/cg1
```

rmdir执行成功的必要条件是cg1下层没有创建其它cgroup，cg1中没有添加任何task，并且它也没有被别的cgroup所引用。

cgdelete cpu,memory:/ 使用`cgdelete`命令可以递归的删除cgroup及其命令下的后代cgroup，并且如果cgroup中有task，那么task会自动移到上一层没有被删除的cgroup中，如果所有的cgroup都被删除了，那task就不被cgroups控制。但是一旦再次创建一个新的cgroup，所有进程都会被放进新的cgroup中。

### （5）设置cgroups参数

设置cgroups参数非常简单，直接对之前创建的cgroup对应文件夹下的文件写入即可，举例如下。

- 设置task允许使用的cpu为0和1. `echo 0-1 > /sys/fs/cgroup/cg1/cpuset.cpus`

使用`cgset`命令也可以进行参数设置，对应上述允许使用0和1cpu的命令为：

```
cgset -r cpuset.cpus=0-1 cpu,memory:/
```

### （6）添加task到cgroup

- 通过文件操作进行添加 `echo [PID] > /path/to/cgroup/tasks` 上述命令就是把进程ID打印到tasks中，如果tasks文件中已经有进程，需要使用`">>"`向后添加。
- 通过`cgclassify`将进程添加到cgroup `cgclassify -g subsystems:path_to_cgroup pidlist` 这个命令中，`subsystems`指的就是子系统（如果使用man命令查看，可能也会使用controllers表示），如果mount了多个，就是用`","`隔开的子系统名字作为名称，类似`cgset`命令。
- 通过`cgexec`直接在cgroup中启动并执行进程 `cgexec -g subsystems:path_to_cgroup command arguments` `command`和`arguments`就表示要在cgroup中执行的命令和参数。`cgexec`常用于执行临时的任务。

### （7）权限管理

与文件的权限管理类似，通过`chown`就可以对cgroup文件系统进行权限管理。

```
chown uid:gid /path/to/cgroup
```

uid和gid分别表示所属的用户和用户组。

## 8. subsystem配置参数用法

### （1）blkio - BLOCK IO资源控制

- **限额类** 限额类是主要有两种策略，一种是基于完全公平队列调度（CFQ：Completely Fair Queuing ）的按权重分配各个cgroup所能占用总体资源的百分比，好处是当资源空闲时可以充分利用，但只能用于最底层节点cgroup的配置；另一种则是设定资源使用上限，这种限额在各个层次的cgroup都可以配置，但这种限制较为生硬，并且容器之间依然会出现资源的竞争。
  - **按比例分配块设备IO资源**
  - **blkio.weight**：填写100-1000的一个整数值，作为相对权重比率，作为通用的设备分配比。
  - **blkio.weight_device**： 针对特定设备的权重比，写入格式为`device_types:node_numbers weight`，空格前的参数段指定设备，`weight`参数与`blkio.weight`相同并覆盖原有的通用分配比。{![查看一个设备的`device_types:node_numbers`可以使用：`ls -l /dev/DEV`，看到的用逗号分隔的两个数字就是。有的文章也称之为`major_number:minor_number`。]}
  - 控制IO读写速度上限
    1. **blkio.throttle.read_bps_device**：按每秒读取块设备的数据量设定上限，格式`device_types:node_numbers bytes_per_second`。
    2. **blkio.throttle.write_bps_device**：按每秒写入块设备的数据量设定上限，格式`device_types:node_numbers bytes_per_second`。
    3. **blkio.throttle.read_iops_device**：按每秒读操作次数设定上限，格式`device_types:node_numbers operations_per_second`。
    4. **blkio.throttle.write_iops_device**：按每秒写操作次数设定上限，格式`device_types:node_numbers operations_per_second`
  - 针对特定操作(read, write, sync, 或async)设定读写速度上限
    1. **blkio.throttle.io_serviced**：针对特定操作按每秒操作次数设定上限，格式`device_types:node_numbers operation operations_per_second`
    2. **blkio.throttle.io_service_bytes**：针对特定操作按每秒数据量设定上限，格式`device_types:node_numbers operation bytes_per_second`
- **统计与监控** 以下内容都是只读的状态报告，通过这些统计项更好地统计、监控进程的 io 情况。
  1. **blkio.reset_stats**：重置统计信息，写入一个int值即可。
  2. **blkio.time**：统计cgroup对设备的访问时间，按格式`device_types:node_numbers milliseconds`读取信息即可，以下类似。
  3. **blkio.io_serviced**：统计cgroup对特定设备的IO操作（包括read、write、sync及async）次数，格式`device_types:node_numbers operation number`
  4. **blkio.sectors**：统计cgroup对设备扇区访问次数，格式 `device_types:node_numbers sector_count`
  5. **blkio.io_service_bytes**：统计cgroup对特定设备IO操作（包括read、write、sync及async）的数据量，格式`device_types:node_numbers operation bytes`
  6. **blkio.io_queued**：统计cgroup的队列中对IO操作（包括read、write、sync及async）的请求次数，格式`number operation`
  7. **blkio.io_service_time**：统计cgroup对特定设备的IO操作（包括read、write、sync及async）时间(单位为ns)，格式`device_types:node_numbers operation time`
  8. **blkio.io_merged**：统计cgroup 将 BIOS 请求合并到IO操作（包括read、write、sync及async）请求的次数，格式`number operation`
  9. **blkio.io_wait_time**：统计cgroup在各设备中各类型IO操作（包括read、write、sync及async）在队列中的等待时间(单位ns)，格式`device_types:node_numbers operation time`
  10. __blkio.__recursive_*：各类型的统计都有一个递归版本，Docker中使用的都是这个版本。获取的数据与非递归版本是一样的，但是包括cgroup所有层级的监控数据。

### （2） cpu - CPU资源控制

CPU资源的控制也有两种策略，一种是完全公平调度 （CFS：Completely Fair Scheduler）策略，提供了限额和按比例分配两种方式进行资源控制；另一种是实时调度（Real-Time Scheduler）策略，针对实时进程按周期分配固定的运行时间。配置时间都以微秒（µs）为单位，文件名中用`us`表示。

- **CFS调度策略下的配置**
  - 设定CPU使用周期使用时间上限
  - **cpu.cfs_period_us**：设定周期时间，必须与`cfs_quota_us`配合使用。
  - **cpu.cfs_quota_us** ：设定周期内最多可使用的时间。这里的配置指task对单个cpu的使用上限，若`cfs_quota_us`是`cfs_period_us`的两倍，就表示在两个核上完全使用。数值范围为1000 - 1000,000（微秒）。
  - **cpu.stat**：统计信息，包含`nr_periods`（表示经历了几个`cfs_period_us`周期）、`nr_throttled`（表示task被限制的次数）及`throttled_time`（表示task被限制的总时长）。
  - **按权重比例设定CPU的分配**
  - **cpu.shares**：设定一个整数（必须大于等于2）表示相对权重，最后除以权重总和算出相对比例，按比例分配CPU时间。（如cgroup A设置100，cgroup B设置300，那么cgroup A中的task运行25%的CPU时间。对于一个4核CPU的系统来说，cgroup A 中的task可以100%占有某一个CPU，这个比例是相对整体的一个值。）
- **RT调度策略下的配置** 实时调度策略与公平调度策略中的按周期分配时间的方法类似，也是在周期内分配一个固定的运行时间。
  1. **cpu.rt_period_us** ：设定周期时间。
  2. **cpu.rt_runtime_us**：设定周期中的运行时间。

### （3） cpuacct - CPU资源报告

这个子系统的配置是`cpu`子系统的补充，提供CPU资源用量的统计，时间单位都是纳秒。

1. **cpuacct.usage**：统计cgroup中所有task的cpu使用时长
2. **cpuacct.stat**：统计cgroup中所有task的用户态和内核态分别使用cpu的时长
3. **cpuacct.usage_percpu**：统计cgroup中所有task使用每个cpu的时长

### （4）cpuset - CPU绑定

为task分配独立CPU资源的子系统，参数较多，这里只选讲两个必须配置的参数，同时Docker中目前也只用到这两个。

1. **cpuset.cpus**：在这个文件中填写cgroup可使用的CPU编号，如`0-2,16`代表 0、1、2和16这4个CPU。
2. **cpuset.mems**：与CPU类似，表示cgroup可使用的`memory node`，格式同上

### （5） device - 限制task对device的使用

- **设备黑/白名单过滤 **
  1. **devices.allow**：允许名单，语法`type device_types:node_numbers access type` ；`type`有三种类型：b（块设备）、c（字符设备）、a（全部设备）；`access`也有三种方式：r（读）、w（写）、m（创建）。
  2. **devices.deny**：禁止名单，语法格式同上。
- 统计报告
  1. **devices.list**：报告为这个 cgroup 中的task设定访问控制的设备

### （6） freezer - 暂停/恢复cgroup中的task

只有一个属性，表示进程的状态，把task放到freezer所在的cgroup，再把state改为FROZEN，就可以暂停进程。不允许在cgroup处于FROZEN状态时加入进程。 * **freezer.state **，包括如下三种状态： - FROZEN 停止 - FREEZING 正在停止，这个是只读状态，不能写入这个值。 - THAWED 恢复

### （7） memory - 内存资源管理

- **限额类**
  1. **memory.limit_bytes**：强制限制最大内存使用量，单位有`k`、`m`、`g`三种，填`-1`则代表无限制。
  2. **memory.soft_limit_bytes**：软限制，只有比强制限制设置的值小时才有意义。填写格式同上。当整体内存紧张的情况下，task获取的内存就被限制在软限制额度之内，以保证不会有太多进程因内存挨饿。可以看到，加入了内存的资源限制并不代表没有资源竞争。
  3. **memory.memsw.limit_bytes**：设定最大内存与swap区内存之和的用量限制。填写格式同上。
- **报警与自动控制**
  1. **memory.oom_control**：改参数填0或1， `0`表示开启，当cgroup中的进程使用资源超过界限时立即杀死进程，`1`表示不启用。默认情况下，包含memory子系统的cgroup都启用。当`oom_control`不启用时，实际使用内存超过界限时进程会被暂停直到有空闲的内存资源。
- **统计与监控类**
  1. **memory.usage_bytes**：报告该 cgroup中进程使用的当前总内存用量（以字节为单位）
  2. **memory.max_usage_bytes**：报告该 cgroup 中进程使用的最大内存用量
  3. **memory.failcnt**：报告内存达到在` memory.limit_in_bytes`设定的限制值的次数
  4. **memory.stat**：包含大量的内存统计数据。
  5. cache：页缓存，包括 tmpfs（shmem），单位为字节。
  6. rss：匿名和 swap 缓存，不包括 tmpfs（shmem），单位为字节。
  7. mapped_file：memory-mapped 映射的文件大小，包括 tmpfs（shmem），单位为字节
  8. pgpgin：存入内存中的页数
  9. pgpgout：从内存中读出的页数
  10. swap：swap 用量，单位为字节
  11. active_anon：在活跃的最近最少使用（least-recently-used，LRU）列表中的匿名和 swap 缓存，包括 tmpfs（shmem），单位为字节
  12. inactive_anon：不活跃的 LRU 列表中的匿名和 swap 缓存，包括 tmpfs（shmem），单位为字节
  13. active_file：活跃 LRU 列表中的 file-backed 内存，以字节为单位
  14. inactive_file：不活跃 LRU 列表中的 file-backed 内存，以字节为单位
  15. unevictable：无法再生的内存，以字节为单位
  16. hierarchical_memory_limit：包含 memory cgroup 的层级的内存限制，单位为字节
  17. hierarchical_memsw_limit：包含 memory cgroup 的层级的内存加 swap 限制，单位为字节

## 8. 总结

本文由浅入深的讲解了cgroups的方方面面，从cgroups是什么，到cgroups该怎么用，最后对大量的cgroup子系统配置参数进行了梳理。可以看到，内核对cgroups的支持已经较为完善，但是依旧有许多工作需要完善。如网络方面目前是通过TC（Traffic Controller）来控制，未来需要统一整合；资源限制并没有解决资源竞争，在各自限制之内的进程依旧存在资源竞争，优先级调度方面依旧有很大的改进空间。希望通过本文帮助大家了解cgroups，让更多人参与到社区的贡献中。



cgroup 相关概念


1.任务（task）。在cgroups中，任务就是系统的一个进程。
2.控制族群（control group）。控制族群就是一组按照某种标准划分的进程。Cgroups中的资源控制都是以控制族群为单位实现。一个进程可以加入到某个控制族群，也从一个进程组迁移到另一个控制族群。一个进程组的进程可以使用cgroups以控制族群为单位分配的资源，同时受到cgroups以控制族群为单位设定的限制。
3.层级（hierarchy）。控制族群可以组织成hierarchical的形式，既一颗控制族群树。控制族群树上的子节点控制族群是父节点控制族群的孩子，继承父控制族群的特定的属性。
4.子系统（subsystem）。一个子系统就是一个资源控制器，比如cpu子系统就是控制cpu时间分配的一个控制器。子系统必须附加（attach）到一个层级上才能起作用，一个子系统附加到某个层级以后，这个层级上的所有控制族群都受到这个子系统的控制。
