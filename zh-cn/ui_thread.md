# UI 线程

很多图形界面应用，包括游戏，都会有一个线程专门负责图形界面的更新和重绘任务，这个线程被称为 UI 线程，通常与界面相关的操作都需要在 UI 线程上进行，这样做是为了避免多线程带来的线程安全问题，因为界面相关的资源在整个进程中只有一个，多个线程操作同一块资源若要避免冲突就需要用锁来实现互斥，但这样做的话同一时刻只有一个线程在操作资源，除了在其它线程里写代码操作界面方便点外，和让一个线程专门操作界面资源的做法比起来没多大优势，反而因为互斥锁的时间消耗而拖低了界面整体效率。

LCUI 的主循环包含了图形界面的更新和重绘操作，主循环所在的线程就是 UI 线程，通常应用程序的界面初始化以及主循环都是直接在主线程上进行的，所以也可以说主线程是 UI 线程。

**在 UI 线程中操作界面**

在 LCUI 中，图形界面的事件响应是在 UI 线程上处理的，你可以在事件处理函数中直接操作界面资源，为了避免产生界面卡顿的问题，建议将所有耗时比较长的操作放在其它线程上进行。比如让应用程序在按钮被点击后从文件中载入图像数据，如果直接在事件响应函数里完成这个任务，那么界面会一直处于假死状态直到图像被载入完为止，通常的做法是新建一个线程作为工作线程，然后为它分配一个任务队列，等需要执行耗时任务时就向该队列添加相关数据并通知工作线程执行任务。

**在其它线程中操作界面**

上面讲到的是在 UI 线程里执行其它任务的问题，在其它线程执行完任务后会需要将任务执行结果反馈到图形界面上，这时可以用 `LCUI_PostTask()` 函数将界面相关的操作放到 UI 线程上执行，示例代码如下：

```c
void TaskForUpdateUI( void *arg1, void *arg2 )
{
        char *text = arg2;
        LCUI_Widget textview = arg1;
        TextView_SetText( textview, text );
}


        // ...

        LCUI_Widget textview;
       // 分配一段内存用来存放字符串
        char *text = strdup( "Task has been completed !" );

        // ...

        LCUI_AppTaskRec task = { 0 };

        task.func = TaskForUpdateUI; // 设置回调函数
        task.arg[0] = textview;      // 设置第一个参数
        task.arg[1] = text;          // 设置第二个参数
        task.destroy_arg[0] = NULL;
        task.destroy_arg[1] = free;  // 设置第二个参数的销毁函数

        LCUI_PostTask( &task );
        // ...
```

任务的回调函数原型需要是 `void xxx( void*, void* )`，任务数据中允许为回调函数设置两个参数，这两个参数需要指针类型。由于该任务是异步执行的，如果要向该函数传递参数，那么需要保证这些参数的生命周期在回调函数被调用时没有结束，否则在回调函数操作这些参数时可能会出现内存访问越界、段错误之类的问题。

为了方便操作界面资源，如果任务参数不需要销毁则可以用 `LCUI_PostSimpleTask()` 函数，这个函数是 `LCUI_PostTask()` 的简化版本，只需要传递三个参数：回调函数、参数1、参数2。

**例外情况**

通常涉及到数据删除和全局影响范围较大的操作都必须在 UI 线程中执行，例如:

* `Widget_Destroy()`
* `Widget_RemoveClass()`
* `Widget_RemoveStatus()`
* `Widget_Unwrap()`
* `LCUI_LoadCSSFile()`
* `LCUIFont_LoadFile()`

对于只涉及简单的数据修改的操作可以直接在其它线程中执行，例如：

* `Widget_Move()`
* `Widget_SetPadding()`
* `Widget_SetMargin()`
* `Widget_SetDisabled()`
* `Widget_Resize()`
* `Widget_Show()`
* `Widget_Hide()`
* `Widget_Update()`

