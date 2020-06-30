### 简介
时钟又称为定时器，负责维护时间，防止进程垄断CPU。操作系统需要通过时间来规范其任务的执行，其最小的时间单位是时钟节拍(OS Tick)。
### 时钟节拍
在RT-Thread中，时钟节拍的长度可以根据RT_TICK_PER_SECOND的定义来调整，即通过改变时钟频率来调整时钟节拍。
#### 实现方式
时钟节拍由配置为中断触发模式的硬件定时器产生，当中断到来时，将调用一次：void rt_tick_increase(void),通知操作系统已经过去一个系统时钟；不同硬件定时器中断实现都不同，以STM32定时器作为示例如下：
```
void SysTick_Handler(void)
{
    /* 进入中断 */
    rt_interrupt_enter();
    ……
    rt_tick_increase();
    /* 退出中断 */
    rt_interrupt_leave();
}
```
在中断函数中调用rt_tick_increase()对全局变量rt_tick进行自加，代码请参考[RT-Thread官网](https://www.rt-thread.org/document/site/programming-manual/timer/timer/)

#### 获取时钟节拍
```
rt_tick_t rt_tick_get(void);
```
### 定时器管理
* 硬件定时器：是芯片本身提供的定时功能。一般是由外部晶振提供给芯片输入时钟，芯片向软件模块提供一组配置寄存器，接受控制输入，到达设定时间值后芯片中断控制器产生时钟中断。
* 软件定时器：由操作系统提供的一类系统接口，构建在硬件定时器基础之上，使系统能够提供不受数目限制的定时器服务。
* RT-Thread 的定时器基于系统的节拍，提供了基于节拍整数倍的定时能力。

#### RT-Thread定时器
* 两类定时器机制：1、单次触发定时器；2、周期触发定时器。
* 根据超时函数执行时所处的上下文环境：HARD_TIMER模式和SOFT_TIMER模式，如下图所示
![](05timer_env.png)

* HARD_TIMER模式：该模式的定时器超时函数在中断上下文环境中执行，可以在初始化 / 创建定时器时使用参数 RT_TIMER_FLAG_HARD_TIMER 来指定。
* SOFT_TIMER模式：SOFT_TIMER 模式可配置，通过宏定义 RT_USING_TIMER_SOFT 来决定是否启用该模式。初始化时会创建一个timer线程，在该模式下都会在timer线程的上下文环境中执行。

### 定时器工作机制
* 定时器工作的机制通过两个两个重要的全局变量来维持：当前系统经过的tick时间rt_tick(当硬件定时器中断来临时，它将加1)；定时器链表rt_timer_list(系统新创建并激活的定时器都会按照以超时时间排序的方式插入到rt_timer_list链表中)
* 系统会将定时器按照定时时间的长短从小到大插入到定时器链表中，然后依次执行。

#### 定时器控制块
在 RT-Thread 操作系统中，定时器控制块由结构体 struct rt_timer 定义并形成定时器内核对象，再链接到内核对象容器中进行管理。
```
struct rt_timer
{
    struct rt_object parent;
    rt_list_t row[RT_TIMER_SKIP_LIST_LEVEL];  /* 定时器链表节点 */

    void (*timeout_func)(void *parameter);    /* 定时器超时调用的函数 */
    void      *parameter;                         /* 超时函数的参数 */
    rt_tick_t init_tick;                         /* 定时器初始超时节拍数 */
    rt_tick_t timeout_tick;                     /* 定时器实际超时时的节拍数 */
};
typedef struct rt_timer *rt_timer_t;
```

#### 定时器跳表(Skip List)算法
系统新创建并激活的定时器都会按照以超时时间排序的方式插入到 rt_timer_list 链表中，也就是说 t_timer_list 链表是一个有序链表，RT-Thread 中使用了跳表算法来加快搜索链表元素的速度，查找的时间复杂度均为O(log n)。
对于定时器跳表算法，在后续的博客中，会做详细的介绍，敬请期待。
#### 定时器的管理方式
在系统启动时需要初始化定时器管理系统。可以通过下面的函数接口完成：
```
void rt_system_timer(void);
```
如果需要使用SOFT_TIMER，则系统初始化时，应该调用下面这个函数接口：
```
void rt_system_timer_thread_init(void);
```
定时器控制块中包含定时器相关的重要参数，在定时器各种状态间起到纽带的作用，如下图所示：
![](05timer_ops.png)
如上图所示，定时器控制块主要包括：
* 创建/初始化定时器：rt_timer_create/init()
* 启动定时器:rt_timer_start()
* 停止/控制:rt_timer_stop/control()
* 删除/脱离:rt_timer_delete/detach()