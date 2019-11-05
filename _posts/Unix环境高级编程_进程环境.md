---
layout: post
title: Unix环境高级编程_进程环境
---

### 进程终止

有8种方式使进程终止，其中5种为征程终止，它们是

1. 从main返回
2. 调用exit
3. 调用\_exit或\_Exit
4. 最后一个线程从其启动例程返回。
5. 最后一个线程调用pthread_exit

  

异常终止有3种，它们是

6. 调用abort
7. 接到一个信号并终止。
8. 最后一个线程对取消请求做出响应。

  

#### 1. exit函数

有三个函数用于正常终止一个程序：\_exit和\_Exit立即进入内核。exit则先执行一些清理处理（包括调用执行者各终止处理程序，关闭所有标准IO流等），然后进入内核。  

```c
#include <stdlib.h>

void exit(int status);
void _Exit(int status);

#include <unistd.h>

void _exit(int status);
```

> exit和\_Exit是由ISO C说明的，而\_eixt则是由POSIX.1说明的。

exit函数总是执行一个标准IO库的清理关闭工作：为所有打开流调用fclose函数。这回造成所有缓冲的输出数据被冲洗。  

   

三个exit函数都带有一个整型参数，称之为终止状态。大多数UNIX shell都提供检查进程终止状态的方法。如果(a)若调动这些函数时不带，或(b)main执行了一个无返回值的return语句，或(c)main没有声明返回类型为整型，则该进程的终止状态是未定义的。但是，若main的返回类型为整型，并且main执行到最后一条语句时返回（隐式返回），那么该进程的终止状态是0.  

  

main函数赶回一整型值与用该值调用exit是等价的。于是在main函数中exit(0)等价于return(0)。  

  

#### 2. atexit函数

按照ISO C的规定，一个进程可以登记多达32个函数，这些函数将由exit自动调用。我们称这些函数为终止处理程序，并调用atexit函数来登记这些函数。  

```c
#include <stdlib.h>

int atexit(void (*func)(void));
/* 返回值：若成功，返回0；若出错，返回非0 */
```

其中atexit的参数是一个函数地址，当调用此函数时无需向它床底任何参数，也不期望它返回一个值。exit调用这些函数的顺序与它们登记时候的顺序相反。同一函数如若登记多次，则也会被调用多次。  

  

注意，内核使程序执行的唯一方法是调用一个exec函数。进程自愿终止的唯一方法是显示或隐式地（通过exit）调用\_exit或\_Exit。进程也可非自愿地由一个信号使其终止。  

  <br/>

  <br/>

  <br/>

### 环境表

每个程序都会收到一张环境表。与参数表一样，环境表也是一个字符指针数组，其中每个指针包含一个以null结束的C字符串的地址。全局变量environ为环境指针，指针数组为环境表，其中各指针指向的字符串为环境字符串。   

按照惯例，环境由name=value这样的字符串组成。  

  <br/>

  <br/>

  <br/>

### 存储器分配

ISO C说明了三个用于存储空间动态分配的函数。

1. malloc。分配指定字节数的存储区。此存储区中的初始值不确定。
2. calloc。为指定数量具制定长度的对象分配存储空间。该空间中每一位都初始化为0.
3. realloc。更改以前分配区的长度（增加或减少）。当增长长度时，可能需将以前分配去的内容移到另一个足够大的区域，以便在尾端提供增加的存储区，而新增区域的初始值则不确定。  

  

```c
#include <stdlib.h>

void *malloc(size_t size);
void *calloc(size_t obj, size_t size);
void* realloc(void* ptr, size_t new_size);
/* 三个函数返回值：若成功则返回非空指针，若出错则返回NULL */

void free(void* ptr);
```

这三个分配函数所返回的指针一定是适当对其的，使其可用于任何数据对象。  



函数free释放ptr指向的存储空间。被释放的空间通常被送入可用存储区池。  

  <br/>

  <br/>

  <br/>

### 环境变量

ISO C定义了一个函数getenv，可以用其取环境变量值，但是该标准又称环境的内容是由实现定义的。  

```c
#include <stdlib.h>

char* getenv(const char* name);
/* 返回值：指向与name关联的value的指针，若未找到则返回NULL */
```

注意，此函数返回一个指针，它指向name=value字符串中的value。我们应当使用getenv从环境中取一个指定环境变量的值，而不是直接访问environ。  

  

除了取环境变量，有时也需要设置环境变量。我们可能希望改变现有变量。我们可能虚妄改变现有变量的值，或者增加新的环境变量。不幸的是，并不是所有系统都支持这种能力。  

| 函数     | ISO C | POSIX.1 | Free BSD 5.2.1 | Linux 2.4.22 | Mac OS X 10.3 | Solaris 9 |
| -------- | ----- | ------- | -------------- | ------------ | ------------- | --------- |
| getenv   | *     | *       | *              | *            | *             | *         |
| putenv   |       | XSI     | *              | *            | *             | *         |
| setenv   |       | *       | *              | *            | *             |           |
| unsetenv |       | *       | *              | *            | *             |           |
| clearenv |       |         |                | *            |               |           |

```c
#include <stdlib.h>

int putenv(char* str);
int setenv(cosnt char* name, const char* value, int rewrite);
int unsetenv(const char* name);
/* 三个函数的返回值：若成功则返回0，若出错则返回非0值 */
```

这三个函数的操作是：

- putenv取形式为name=value的字符串，将其放到黄竞标中。如果name已经存在，则先删除其原来的定义。
- setenv将name设置为value。如果在环境中name已经存在，那么(a)若rewrite非0，则首先删除其现有的定义；(b)若rewrite为0，则不删除其现有定义（name不设置为新的value，而且也不出错）。
- unsetenv删除name的定义。即使不存在这种定义也不算出错。  

  <br/>

  <br/>

  <br/>

### setjmp和longjmp函数

在C中，goto语句是不能跨越函数的，而执行这列跳转功能的是函数setjmp和longjmp。这两个函数对于处理发生在深层嵌套函数调用中的出错情况是非常有用的。  

```c
#include <setjmp.h>

int setjmp(jmp_buf env);
/* 返回值：若直接调用则返回0，若从longjmp调用返回则返回非0值 */
void longjmp(jmp_buf env, int val);
```

setjmp参数env的类型是一个特殊类型jmp_buf。这一数据类型是某种形式的数组，其中存放在调用longjmp时能用来恢复栈状态的所有信息。因为需在另一函数中引用env变量，所以规范的处理方式是将env变量定义为全局变量。  

  

longjmp函数第一个参数就是在调用setjmp时所用的env，第二个参数是具有非0值的val，它将成为从setjmp处返回的值。使用第二个参数的原因是对于一个setjmp可以有多个longjmp。   

  

#### 1. 自动、寄存器和易失变量

在调用setjmp并使用longjmp返回后，自动变量和寄存器变量状态如何？这些变量的值是否能恢复到以前调用setjmp时的值，或是保持为longjmp调用时的值？不幸的是，对此的回答是“看情况”。大多数实现并不回滚这些自动变量和寄存器变量的值，而所有标准则说它们的值是不确定的。如果你有一个自动变量（就是函数内在栈上分配的变量），而又不想使其回滚，则可定义为具有valatile属性（此关键字可使编译器不对变量进行优化），声明为全局或静态变量的值在执行longjmp时保持不变。  

  

#### 2. 局部变量的潜在问题

为标准IO设置使用setbuf和setvbuf设置缓冲区时，不应使用栈上的数组，因为当分配栈上变量所在函数返回时栈已经重新被新调用的栈帧使用，而标准IO仍使用其缓冲区的存储空间，这就产生了冲突和混乱。为了校正这一问题，应在全局存储空间静态的（如static或exxtern）或者动态地（使用一种alloc函数）为数组databuf分配空间。  

  <br/>

  <br/>

  <br/>

### getrlimit和setrlimit函数

每个进程都有由自愿限制，其中一些可用getrlimit和setrlimit函数查询和更改。  

```c
#include <sys/resource.h>

int getrlimit(int resource, struct rlimit* rlptr);
int setrlimit(int rseource, const struct rlimit* rlptr);
/* 两个函数返回值：若成功返回0;，若出错返回非0值 */
```

这两个函数在SUS中定义为XSI扩展。进程的资源限制通常是在系统初始化时由进程0建立的，然后由每个后续进程集成。每种实现都可以用自己的方法对各种限制做出调整。  

  

对这两个函数的每一次调用都会制定一个资源以及一个指向下列结构的指针。  

```c
struct rlimit{
    rlim_t rlim_cur;   /* 软限制：当前的限制值 */
    rlim_t rlim_max;   /* 硬限制：rlim_cur的最大值 */
};
```

  

在更改资源限制时，须遵循下列三条规则：

1. 任何一个进程都可将一个软限制值更改为小于或等于其硬限制值。
2. 任何一个进程都可降低其硬限制值，但它必须大于或等于其软限制值。这种降低对普通用户而言是不可逆的。
3. 只有超级用户进程可以提高其硬限制值。

常量RLIM_INFINTY制定了一个无限量的限制。

  

这两个函数的resource参数区下列值之一。下表显示那些资源限制是由SUS定义并被讨论的四中系统实现支持的。  

| 限制           | XSI  | FreeBSD 5.2.1 | Linux 2.4.22 | Mac OS X 10.3 | Solaris 9 |
| -------------- | ---- | ------------- | ------------ | ------------- | --------- |
| RLIMIT_AS      | *    |               | *            |               | *         |
| RLIMIT_CORE    | *    | *             | *            | *             | *         |
| RLIMIT_CPU     | *    | *             | *            | *             | *         |
| RLIMIT_DATA    | *    | *             | *            | *             | *         |
| RLIMIT_FSIZE   | *    | *             | *            | *             | *         |
| RLIMIT_LOCKS   |      |               | *            |               |           |
| RLIMIT_MEMLOCK |      | *             | *            | *             |           |
| RLIMIT_NOFILE  | *    | *             | *            | *             | *         |
| RLIMIT_NPROC   |      | *             | *            | *             |           |
| RLIMIT_RSS     |      | *             | *            | *             |           |
| RLIMIT_SBSIZE  |      | *             |              |               |           |
| RLIMIT_STACK   | *    | *             | *            | *             | *         |
| RLIMIT_VMEM    |      | *             |              |               | *         |

- RLIMIT_AS：进程可用存储区的最大总长度（字节）。这会影响sbrk函数和mmap函数。
- RLIMIT_CORE：core文件的最大字节数，若其值为0则组织创建core文件。
- RLIMIT_CPU：CPU时间的最大值（秒），当超过此软限制时，该向该进程发送SIGXCPU信号。
- RLIMIT_DATA：数据段的最大字节长度。这是初始化数据、非初始化以及堆的总和
- RLIMIT_FSIZE：可以创建的文件的最大字节长度。当唱过此软限制时，该向该进程发送SIGXFSZ信号。
- RLIMIT_LOCKS：一个进程可持有的文件锁的最大数（此数也包括Linux特有的文件租借数）。
- RLIMIT_MEMLOCK：一个进程使用mlock能够锁定在存储器中的最大字节长度。
- RLIMIT_NOFILE：每个进程能打开的最大文件数。更改此限制将影响到sysconfig函数在参数\_SC\_OPEN\_MAX中返回的值。
- RLIMIT_NPROC：每个实际用户ID可拥有的最大子进程数。更改此限制将影响到sysconf函数在参数\_SC\_CHILD\_MAX中返回的值.
- RLIMIT_RSS：最大驻内存集的自己长度。如果物理存储器供不应求，则内核从进程处收回超过RSS的部分。
- RLIMIT_SBSIZE：用户在任一给定时刻都可以占用的套接字缓冲区的最大长度（字节）。
- RLIMIT_STACK：栈的最大字节长度。
- RLIMIT_VMEM：这是RLIMIT_AS的同义词。

  

资源限制影响到调用进程并由其子进程继承。这就意味着为了影响一个用户的所有后续进程，需将资源限制的设置构造在shell之中。确实，Bourne shell、GUN Bourne-again shell和Korn shell具有内置的ulimit命令，C shell具有内置的limit命令。



















