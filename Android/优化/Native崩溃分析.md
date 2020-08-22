[参考](https://mp.weixin.qq.com/s/g-WzYF3wWAljok1XjPoo7w?)
## 捕捉Native Crash
Java代码发生异常导致崩溃时，会打印出异常所在的调用栈，可以快速定位问题所在，但是C/C++发生崩溃时，默认情况下没有详细的信息，难以排查问题。<br/>
Native发生异常时，CPU通过异常中断触发异常处理流程，Linux中这些中断处理都使用信号量的方式，不同的异常有不同的信号量，针对各个信号量可以注册不同的信号处理函数。

#### 注册信号处理函数：
```
#include <signal.h> 
int sigaction(int signum,const struct sigaction *act,struct sigaction *oldact));
```
* signum 为异常信号量
* act 指向结构体sigaction的指针
* oldact 保存原本对signum信号量的处理

**重点**在于sigaction，此结构体中的sa_sigaction就是一个函数指针：
```
void (*sa_sigaction)(const int code, siginfo_t *const si, void * const sc) 

siginfo_t {
   int      si_signo;     /* Signal number 信号量 */
   int      si_errno;     /* An errno value */
   int      si_code;      /* Signal code 错误码 */
   }
```
也就是说我们注册的这个函数可以接收到各种错误信息，第三个参数sc是uc_mcontext结构体，是cpu的上下文，包含了当前线程的寄存器信息和崩溃时的pc值，通过pc值就能知道崩溃时执行的指令。

#### 获得共享库名称和相对偏移位置
pc值是可执行程序中的某条指令加载到内存中的绝对位置，而我们要得到的是该引发崩溃的代码在共享库中的相对偏移位置，再通过addr2line就可以知道是哪一行代码异常。dladdr()函数可以得到共享库加载到内存的起始位置，相对偏移位置则可以通过pc减去该起始位置得出。
```
int dladdr(void *addr, Dl_info *info);
```
函数dladdr()确定addr中指定的地址是否位于调用应用程序加载的共享库之一中。如果是，则dladdr()返回有关共享库和与addr重叠的符号的信息。此信息以Dl_info结构返回。这里addr则传入pc。

#### 获取堆栈
前面已经可以获得pc和各个寄存器的信息，SP是父函数即调用者的堆栈首地址，FP是父函数的堆栈结束地址，所以根据SP和FP就可以往上得出全部调用关系。

> 以上叙述了Native异常分析的简要基本原理，实际中已经有一些开源库可以帮助我们快捷处理问题。例如谷歌的Breakpad，可以得到特殊格式的日志，再用它的minidump_stackwalk程序解析日志生成可以看到详细错误的文本，包含错误发生的共享库和相对偏移位置；我们再通过NDK中的addr2line得到哪一行代码发生错误。