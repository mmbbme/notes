# win32







## 字符串

略

## 进程

### 什么是进程

进程提供程序所需的资源，如:数据、代码等等

线程来使用这些资源。

### 进程内存空间的地址划分

[Windows NT如何划分进程的地址空间@lwwl12](https://blog.csdn.net/lwwl12/article/details/88859044)

| 范围                    | 大小             | 作用                                                     | 说明                                                         |
| ----------------------- | ---------------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| 0x00000000 - 0x0000FFFF | 64KB             | 用于NULL指针分配，不可访问                               | NULL指针区域，师徒读写这一分区中内存地址将会引起访问冲突     |
| 0x00010000 - 0x7FFEFFFF | 2GB - 64K - 64KB | 可使用的地址空间，用户可以使用的内存，即低2g的虚拟内存   | 进程的私人地址空间，DLL被装入这一分区，内存映射文件映射到该分区。所以<span style="color:red;">每个进程真正使用的虚拟空间其实也就不到俩个G</span> |
| 0x7FFF0000 - 0x7FFFFFFF | 64KB             | 用于环指针分配，不可访问                                 | 该分区禁止使用                                               |
| 0x80000000 - 0xFFFFFFFF | 2GB              | 用于操作系统，不可访问。内核空间，是一个公用的空间即高2g | 装入Windows NT执行体、内核和设备驱动                         |

![应用堆栈](D:%5C%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%5Cnotes%5Cimg%5Cwin32%5C%E5%BA%94%E7%94%A8%E5%A0%86%E6%A0%88.png)





![操作系统](D:%5C%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%5Cnotes%5Cimg%5Cwin32%5C%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F.png)







### 进程的创建过程

> 每个进程都是由别的进程创建出来的，第一个进程是内核创建的。双击应用会由explorer.exe创建进程——调用CreateProcess()

1. 映射EXE文件

   把exe可执行文件放进虚拟内存空间

2. 创建内核对象EPROCESS

   每个进程都有一个对象

3. 映射系统DLL(ntdll.dll)

4. 创建线程内核对象ETHREAD

5. 系统启动线程

   ​	映射DLL(ntdll.LdrlnitializeThunk)

   ​	最后线程开始执行

### 进程的创建



```c++

STARTUPINFO stpi;//程序的启动信息
PROCESS_INFORMATION pi;//进程结果信息
ZeroMemory(&stpi, sizeof(stpi));
ZeroMemory(&pi, sizeof(pi));

stpi.dwFlags = STARTF_USESHOWWINDOW;
stpi.wShowWindow = SW_SHOW;
stpi.cb = sizeof(STARTUPINFO);

TCHAR a[] =_T("notepad");

 CreateProcess(
	NULL,//应用名
	a,//cmd命令行
	NULL,//使用默认进程安全属性
	NULL,//使用默认线程安全属性
	FALSE,//句柄不继承
	NORMAL_PRIORITY_CLASS,//使用正常优先级
	NULL,//使用父进程的环境变量
	NULL,//指定工作目录
	&stpi,//指向一个用于决定新进程的主窗体如何显示的STARTUPINFO结构体。
	&pi//指向一个用来接收新进程的识别信息的PROCESS_INFORMATION结构体。
);
system("pause");
//释放句柄*必须*
CloseHandle(pi.hProcess);
CloseHandle(pi.hThread);
```
### 句柄表

> 句柄相当于内核对象地址

大概是这样子

| 索引 | 内核对象地址 | 引用计数 |
| ---- | ------------ | -------- |
|      |              |          |



> 像进程线程文件互斥体事件等在内核都有一个对应的结构体对象，这些结构体负责管理。这就是内核对象



| 文件 | 进程 | 线程 | ... | 应用层 |
| ---- | ---- | ---- | ---- | ---- |
| FILE_OBJE | EPROCESS | ETHREAD | ... |   内核层   |

每一个进程都有可能有多个内核对象，分别有着不同的操作属性，如果上层想要调用它就不能直接访问地址，这样操作会很危险，所以微软是通过一张表来标识他们的，这就是句柄表。某种程度上说<span style="color:red">句柄就是内核对象地址</span>，但是又隔离了上层对内核层(0环)操作。而这个表只存在于进程内核对象EPROCESS中

> 多个进程共享一个内核对象

举个典型的例子:

OpenProcess就能在一个进程中打开另一个进程，并访问另一个句柄表。注意句柄表是一个私有的，像前面的操作并不会对他们的句柄表产生必然联系，它们的句柄表指向的共用对象的句柄不一定会一致。这个共享的内核对象当访问进程增加的时候它自身里面有一个计数器+1，当这个计数器归零的时候它才会销毁，所以CloseHandle<span style="color:red">不一定</span>会让内核对象销毁，他只是让内核对象的引用计数减一。线程对象要把线程关掉，同时关闭句柄引用计数计0它才会销毁。

**<span style="color:red;">只要有安全描述符他就是内核对象</span>**

> 句柄是否被继承

[安全描述符](https://baike.baidu.com/item/安全描述符)

````c++
typedef struct _SECURITY_ATTRIBUTES {

DWORD nLength; //结构体的大小，可用SIZEOF取得

LPVOID lpSecurityDescriptor; //安全描述符

BOOL bInheritHandle ;//当前内核对象是否被新创建的进程继承  

} SECURITY_ATTRIBUTES，* PSECURITY_ATTRIBUTES;
````

````c++
SECURITY_ATTRIBUTES sa;
ZeroMemory(&sa, sizeof(SECURITY_ATTRIBUTES));
sa.nLength = sizeof(SECURITY_ATTRIBUTES);
sa.bInheritHandle = TRUE;
//例子如果允许继承就构造一个sa并且bInheritHandle设置为TRUE，否则写null
CreateEvent(&sa, 0, 0, 0);
````

```c++
HANDLE CreateEvent(
LPSECURITY_ATTRIBUTES lpEventAttributes,// 安全属性,就是上文的_SECURITY_ATTRIBUTES
BOOL bManualReset,// 复位方式
BOOL bInitialState,// 初始状态
LPCTSTR lpName // 对象名称
);
```



> 是否可以从调用进程处继承所有可继承的句柄

如果可以列如在CreateProcess中把第五个参数写成true，那么子进程就会把父进程的句柄表(进程对象的属性设置为可继承的部分)复制一份，那些没有设置可继承的内核对象填充为0

所以至此就有俩种共享内核的方法，一个是子进程继承，一个OpenProcess共享进程

### 进程相关api

CreateProcess创建成功会返回四个数据，进程编号，进程句柄，线程编号，线程句柄。即PROCESS_INFORMATION所存储的东西，句柄是私有的，编号是全局的，

这俩个编号其实就是全局句柄表中的索引，就是pid！！

所以在进行TerminateProcess,OpenProcess等等设置到pid的操作的时候，进程句柄没有效果，pid是有效的！

TerminateProcess

````c++
HANDLE hProcess;
//hProcess = (HANDLE)0x7D0   无效
hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, 0x2AB8);//0x2AB8注意变化
//终止指定句柄
TerminateProcess(hProcess, 1);
````

CreateProcess

````c++
BOOL CreateProcess(
LPCTSTR lpApplicationName,//应用名
LPTSTR lpCommandLine,//cmd命令行
LPSECURITY_ATTRIBUTES lpProcessAttributes,//进程安全属性  是否继承进程句柄    子进程其实也不需要
LPSECURITY_ATTRIBUTES lpThreadAttributes,//线程安全属性   是否继承线程句柄  子进程其实也不需要
BOOL bInheritHandles,//是否允许继承父进程句柄表
 //指定附加的、用来控制优先类和进程的创建的标志。
 //还用来控制新进程的优先类，
 //优先类用来决定此进程的线程调度的优先级。
DWORD dwCreationFlags,
LPVOID lpEnvironment,//使用父进程的环境变量
LPCTSTR lpCurrentDirectory,//指定工作目录
LPSTARTUPINFO lpStartupInfo,//指向一个用于决定新进程的主窗体如何显示的STARTUPINFO结构体。
LPPROCESS_INFORMATION lpProcessInformation //指向一个用来接收新进程的识别信息的PROCESS_INFORMATION结构体。
);
````

>  dwCreationFlags设置为挂起创建——**CREATE_SUSPENDED**

系统不会去启动线程需要开发者手动启动，这中间就可以做一些东西。(滑稽)

```c++

​```````````````
做你想做的事
​```````````````
ResumeThread(pi.hThread) //唤醒线程
```



## 线程
1. 线程是附属在进程上的执行实体，是代码的执行流程

   进程是一个空间的上的概念，线程是时间的概念，也就是你正在运行的代码

2. 一个进程可以包含多个进程，但是一个进程至少包含一个线程

   每一个线程有一个属于他自己的堆栈

### 创建线程

````c++
HANDLE CreateThread
(
  LPSECURITY_ATTRIBUTES lpsa, //安全描述符
  DWORD cbStack,//需要初始堆栈大小
  LPTHREAD_START_ROUTINE lpStartAddr,//当前线程真正要执行的代码在哪里
  LPVOID lpvThreadParam,//线程的参数
  DWORD fdwCreate,//类似于CreateProcess的 dwCreationFlags指定附加的、用来控制优先类和进程的创建的标志，同样可以设置CREATE_SUSPENDED挂起创建
  LPDWORD lpIDThread //返回的线程id，类型是句柄其实也就是句柄
);
````

在使用参数的时候，需要保证参数的生命周期大于线程生命周期，保证有效。

### 控制线程

#### sleep

让当前线程停止多少秒，单位是毫秒

#### SuspendThread/ResumeThread

挂起某一个线程/唤醒/启动某一个被挂起的线程

````c++
	HANDLE hThread;
	hThread = CreateThread(NULL, 0, ThreadProc, NULL, 0, NULL);
	Sleep(3000);//主线程睡眠三秒
	SuspendThread(hThread);//挂起线程
	Sleep(3000);
	ResumeThread(hThread);//唤醒被挂起的线程
	getchar();
````

#### WaitForSingleObject

等待一个内核对象(线程)执行完毕，等待函数进入等待状态

```c++
    DWORD WaitForSingleObject(
    HANDLE hHandle,
    //参数二:超时时间,如果指定了非零值，在输入的时间内等待线程函数体执行
	//在指定时间内无论执没执行完都会返回，设置为0会立即返回。
	//一直等下去设置为INFINITE,函数执行完毕返回
    DWORD  dwMilliseconds
    );
```



#### WaitForMultipleObjects

等待多个内核对象(线程)执行完毕

```c++
	DWORD WaitForMultipleObjects(
		DWORD        nCount,//参数一,等待几个对象，不能为0
		const HANDLE * lpHandles,//参数二,句柄数组
        //参数三,等待模式  可以指定有几个对象状态变更的时候返回
        //TRUE 全部变更返回，false任意一个对象状态设置已通知信号的时候返回
		BOOL         bWaitAll,
		DWORD        dwMilliseconds//参数四，类似WaitForSingleObject参数二
	);
```

#### GetExitCodeThread

//获取线程执行函数/过程的返回值

```c++
BOOL  GetExitCodeThread (
      HANDLE        hThread,              //线程handle,
      LPDWORD     lpExitCode              //写入线程执行函数/过程的返回值
);
```

#### 线程上下文环境获取


CONTEXT就是线程上下文环境的结构体，他存储的是你获取到程序当前运行使用的寄存器；其中有一个属性可以用来设置你需要获取的段——ContextFlags。

>  使用GetThreadContext获取线程上下文

```c++
CONTEXT context;
context.ContextFlags = CONTEXT_INTEGER;
GetThreadContext(youThreadHandle,&context)
//BOOL GetThreadContext(
//  HANDLE hThread,
//  LPCONTEXT lpContext
//);
```

#### 线程上下文环境设置

> SetThreadContext设置上下文

````c++
BOOL SetThreadContext(
  HANDLE        hThread,
  const CONTEXT *lpContext
);
````

### 临界区

又到经典的线程安全问题，虽然每一个线程都有自己的堆栈但是如果多个线程访问同一个全局变量的时候会有出问题。这个问题不用想太多了，一个最通俗易懂的例子就是老八，不，上厕所问题。如果说小便几个爷们还能一块解决但是开大总不能一起解决吧，叠罗汉吗(草)。解决问题的一个途径就是给厕所上把锁呗。

典型的实例就是售票问题

#### 临界资源

> 多道程序系统中存在许多进程，它们共享各种资源，然而有很多资源一次只能供一个进程使用。一次仅允许一个进程使用的资源称为临界资源。许多物理设备都属于临界资源，如输入机、打印机等。
>
> 而对其访问的代码我们就称其为*临界区*。

#### 线程锁构建临界区

在windows中可以有以下操作

>创建全局变量

```c++
CRITICAL_SECTION g_cs;
```

CRITICAL_SECTION严格来说不是锁，而是当代码执行到EnterCriticalSection后会记录当前占用线程，如果此时并没有到达LeaveCriticalSection其他线程访问EnterCriticalSection将会等待


>初始化全局变量

```c++
=======================================================================
VOID InitializeCriticalSection(LPCRITICAL_SECTION lpCriticalSection );
=======================================================================
InitializeCriticalSection(&g_cs);
````




>定义临界区

```C++
VOID WINAPI EnterCriticalSection(__inout LPCRITICAL_SECTION lpCriticalSection);
VOID WINAPI LeaveCriticalSection( _Inout_LPCRITICAL_SECTION lpCriticalSection);
```

```C++
// 进入临界区
EnterCriticalSection(&g_cs);

/*********************/

你的临界区代码

/*********************/

// 离开临界区
LeaveCriticalSection(&g_cs);
```



这种方法适用于同一个进程的不同线程访问同一个资源



### 互斥体

如果俩个进程访问内核临界资源或者文件，如何避免线程安全问题？

这里就必须介绍一下互斥体，它本身作为一个内核维护的结构，保障了对象对于单个线程的访问权，说人话就是一个内核级别的锁，它可以跨进程使用。对了作为内核维护的对象，当从用户模式转换到内核模式是需要花费更多的CPU资源。但是它可以避免线程死锁的问题

````C++
HANDLE CreateMutex(
LPSECURITY_ATTRIBUTESlpMutexAttributes, // 指向安全属性的指针
BOOL bInitialOwner, //  是否获得互斥对象的初始所有权
LPCTSTR lpName // 指向互斥对象名的指针
);
````

进程A

````c++

using namespace std;
int main(int argc, char* argv[])
{
	HANDLE gHMutex = CreateMutex(NULL, false, _T("XYZ"));
    Sleep(2000);
	WaitForSingleObject(gHMutex, INFINITE);
	for (int i=0;i<100;i++)
	{
		Sleep(1000);
		cout << "A线程的X进程:" << i << endl;
	}
	ReleaseMutex(gHMutex);
	return 0;
}
````



进程M

`````c++

using namespace std;
int main(int argc, char* argv[])
{
   // 创建互斥体
	HANDLE gHMutex = CreateMutex(NULL, TRUE, _T("XYZ"));
    
    //获取令牌
	WaitForSingleObject(gHMutex, INFINITE);
	for (int i = 0; i < 100; i++)
	{
		Sleep(1000);
		cout << "m线程的b进程:" << i << endl;
	}
    //释放令牌
	ReleaseMutex(gHMutex);
	return 0;
}
`````

先明确同一个名字的互斥体创建多次后面的其实获取的就是已经创建的互斥体，并且他会设置一个已存在的错误，可以用获取错误函数来判断，设置反多开代码。

进程A先启动，然后在两秒内启动M进程，可以看到，M进程进入临界区代码，A进程还在等待

进程M先启动，那无论怎么等待都是M先执行完，A进程再执行

当CreateMutex第二参数都为false的时候，先拿到令牌的先执行

## 事件

事件是一个允许一个线程在某种情况发生时，唤醒另外一个线程的同步对象。事件告诉线程何时去执行某一给定的任务，从而使多个线程流平滑

> ```c++
> HANDLE CreateEventA(
> 	LPSECURITY_ATTRIBUTES   lpEventAttributes,   //   SD  一般为空     
> 
> 　BOOL   bManualReset,           //  reset  type  事件是自动复位（false）并且互斥还是人工复位（true）
> 
> 　BOOL   bInitialState,              // initial state  初始状态，有信号（true），无信号（false）     
> 
> 　LPCTSTR   lpName       //object name  事件对象名称   如果不需要在其他的进程使用当前这个内核													//对象可以不用取
> 
> );
> ```
>
> 

如果创建一个对象没有信号且没有使用SetEvent把事件设为"有信号"状态那么WaitForSingleObject会处于阻塞状态

```c++

HANDLE g_event ;

DWORD WINAPI  ThreadProc(LPVOID lpParameter)
{
	WaitForSingleObject(g_event, INFINITE);
	cout << "线程一" << endl;
	return 0;
}
DWORD WINAPI  ThreadProc1(LPVOID lpParameter)
{
	WaitForSingleObject(g_event, INFINITE);
	cout << "线程二" << endl;
	return 0;
}
int main()
{
	HANDLE hThread[2];
    //  无信号
	g_event = CreateEvent(NULL, TRUE, false, NULL);
	hThread[0] = CreateThread(NULL, 0, ThreadProc, NULL, 0, NULL);
	hThread[1] = CreateThread(NULL, 0, ThreadProc1, NULL, 0, NULL);
	
	SetEvent(g_event);//没有这句当前程序不会执行线程函数体然后进入阻塞
	getchar();
	CloseHandle(hThread[0]);
	CloseHandle(hThread[1]);
    
	printf("线程执行结束");
	return 0;
}
```

### 线程同步

同步就是协同步调，按预定的先后次序进行运行。不是同步执行，是俩个互斥的对象对同一个资源有序使用的机制。

> 生产者与消费者

```c++


HANDLE g_hevent_product;

HANDLE g_hevent_cons;
int g_Max = 10;

DWORD WINAPI  ThreadProc(LPVOID lpParameter)
{
	for (int i = 0;i<g_Max;i++)
	{
        //CreateEvent二参数为false，WaitForSingleObject后会改变信号
		WaitForSingleObject(g_hevent_product, INFINITE);
		printf("线程一\n");
        //改变另一个事件为有信号
		SetEvent(g_hevent_cons);
	}
	
	return 0;
}
DWORD WINAPI  ThreadProc1(LPVOID lpParameter)
{
	for (int i = 0; i < g_Max; i++)
	{
		WaitForSingleObject(g_hevent_cons, INFINITE);
		printf("线程二\n");
		SetEvent(g_hevent_product);
	}
	return 0;
}
int main()
{
	HANDLE hThread[2];
	g_hevent_cons = CreateEvent(NULL, FALSE, FALSE, NULL);
	g_hevent_product = CreateEvent(NULL, FALSE, TRUE, NULL);
	hThread[0] = CreateThread(NULL, 0, ThreadProc, NULL, 0, NULL);
	hThread[1] = CreateThread(NULL, 0, ThreadProc1, NULL, 0, NULL);
	
	WaitForMultipleObjects(2, hThread, TRUE, INFINITE);
	getchar();
	CloseHandle(hThread[0]);
	CloseHandle(hThread[1]);
	printf("线程执行结束");
	return 0;
}
```

WaitForSingleObject换成ResetEvent是否能达到同样的效果呢，答案否定的，详情见上文WaitForSingleObject的定义

那么这里使用互斥体会怎样呢？

```c++

HANDLE g_mutx;
int g_Max = 10;
int g_number = 0;
DWORD WINAPI  ThreadProc(LPVOID lpParameter)
{
	for (int i = 0; i < g_Max; i++)
	{
		WaitForSingleObject(g_mutx, INFINITE);
		if (!g_number)
		{
			g_number = 1;
			printf("线程一\n");
		} 
		else
		{
			i--;
		}
		ReleaseMutex(g_mutx);
	}

	return 0;
}
DWORD WINAPI  ThreadProc1(LPVOID lpParameter)
{
	for (int i = 0; i < g_Max; i++)
	{
		WaitForSingleObject(g_mutx, INFINITE);
		if (g_number)
		{
			g_number = 0;
			printf("线程二\n");
		}
		else
		{
			i--;
		}
		ReleaseMutex(g_mutx);
	}
	return 0;
}
int main()
{
	HANDLE hThread[2];
	g_mutx = CreateMutex(NULL, FALSE, NULL);
	hThread[0] = CreateThread(NULL, 0, ThreadProc, NULL, 0, NULL);
	hThread[1] = CreateThread(NULL, 0, ThreadProc1, NULL, 0, NULL);

	WaitForMultipleObjects(2, hThread, TRUE, INFINITE);
	getchar();
	CloseHandle(hThread[0]);
	CloseHandle(hThread[1]);
	printf("线程执行结束");
	CloseHandle(g_mutx);
	return 0;
}
```

可以实现同步，但是会有性能问题，关键就在i--;上面。表面上是执行了规定的次数，但是有它在其实执行的次数是比规定的次数多的

## 窗口的本质

内核中有俩个模块是非常重要的，特别是开发中

| 模块 |   功能    |   dll  |
| ------------ | ---- | ---- |
| win32k.sys   | 图像界面、消息管理 |     user32.dll、gdi32.dll                   |
| ntoskrnl.exe | 进程线程、内存管理等 | kernel32.dll |

 user32.dll、gdi32.dll的区别在于，user32是一些已经绘制好的东西的接口称为gui，而gdi32是你要自己去绘制的东西的接口称为gdi

### 窗口句柄 HWND

>  它是一个全局的句柄，作用于一个且只有一个全局句柄表

### GDI 图形设备接口(Graphics Device Interface)\*\*\* 了解 \*\*\*\*

1. 设备对象(HWND) 

   先获取设备对象，不去获取就是在桌面上画

2. DC(设备上下文,Device Contexts)

   如果想画东西必须向设备对象的内存里面画，这个DC就是那块内存。而你需要画什么需要用到图像对象

3. 图形对象

   这些对象都是句柄

| 图形对象      | 作用                                               |
| ------------- | -------------------------------------------------- |
| 笔刷(Pen)     | 影响线条，包括颜色、粗细、虚实、箭头形状等         |
| 画刷(Brushes) | 影响对形状、区域等操作，如使用的颜色、是否有阴影等 |
| 字体(Fonts)   | 影响文字输出的字体                                 |
| 位图(Bitmaps) | 影响位图创建、位图操作和保存等                     |

	4. 关联图像对象和设备对象
 	5. 开始画
 	6. 释放资源(图像对象，DC)



## 消息队列

> * 鼠标键盘在被按下的同时操作系统会把这些按键信息(内核程序)存储到一个结构体中，这个结构体就是消息
> * 每一个线程只有一个消息队列，消息队列可以视为存储消息的容器(实际上是多个容器)
> * 消息队列是由操作系统来维护的，也就是说首先是操作系统接收到你的动作，此时会进行一些封装处理，再把你的动作(消息)放进给对应线程的消息队列里，再由对应线程进行处理。

界面上那么程序，操作系统是如何把动作发送到指定线程的？

首先程序在内核都有一个或多个线程对象进行管理，拿窗口举例子，创建一个窗口就会有一个窗口对象，它里面会有一个成员会记录当前窗口所属的线程是谁，可以多个窗口对象指向同一个线程对象。鼠标或键盘的信息，首先会根据鼠标位置等条件判断消息属于那个窗口，再由这个窗口对象的成员找到对应的线程对象，找到后把封装好的消息，存进线程对象的消息队列。



## 第一个windows程序

### 从WinMain说起

win32的程序会多道东西才回去调用真正的main函数，这个就是WinMain，它是Windows应用程序的入口点

```c++
int WINAPI WinMain(
  HINSTANCE hInstance, //表示“实例句柄”或“模块句柄”  也就是当前模块在内存的起始位置，
    				   //一般来讲就是当前的堆栈头00400000
  HINSTANCE hPrevInstance,  //父句柄，已经基本不用了
  LPWSTR lpCmdLine, //类似于标准main(int argc,char** argv)中的命令行参数,没有argc来指导参数数量
  int nShowCmd //如何打开程序的窗口
);
```

看下面的代码

```c++

static TCHAR szWindowClass[] = _T("WindwoClass");
static TCHAR szTitle[] = _T("win32API窗口");
UINT IDC_EDIT = 200;
UINT IDC_BUTTON = 201;
static TCHAR testContent = { 0 };

//第五步 窗口函数以及处理(窗口过程)
LRESULT CALLBACK WindowProc(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
	
	PAINTSTRUCT ps;
	HDC hdc;
	TCHAR greeting[] = _T("helloworld");
	
	switch (uMsg)//消息种类，不想处理交给DefWindowProc
	{
	//主窗口创建后，会先返回一个WM_CREATE，这里进行子窗口子控件的创建
	case WM_CREATE:
	{	
		
		HWND HDWND = CreateWindow(WC_EDIT, _T(""),WS_BORDER| WS_CHILD | BS_PUSHBUTTON | WS_VISIBLE, 150, 100, 100, 50, hWnd, (HMENU)IDC_EDIT, NULL, NULL);
		HWND HBWND_1 = CreateWindow(WC_BUTTON, _T("提交"), WS_CHILD | BS_PUSHBUTTON | WS_VISIBLE, 250, 100, 100, 50, hWnd, (HMENU)IDC_BUTTON, NULL, NULL);

	}	
		break;

	case WM_COMMAND:
	{
		//控件id
		UINT cTRLid = LOWORD(wParam);
		//通知码
		UINT cCode = HIWORD(wParam);

		
		
		if (cCode == BN_CLICKED && cTRLid == IDC_BUTTON)
		{
			
			GetWindowText(GetDlgItem(hWnd, IDC_EDIT),&testContent,55);
			MessageBox(NULL,&testContent, _T("2"), MB_OK);
			
		}
	}
		break;
	case WM_PAINT:
		hdc = BeginPaint(hWnd, &ps);
        // 画出欢迎语
		TextOut(hdc,5,5,greeting,_tcslen(greeting));
		EndPaint(hWnd, &ps);
		break;
	case WM_DESTROY:
		PostQuitMessage(0);
		break;
	default:
		return DefWindowProc(hWnd, uMsg, wParam, lParam);
		break;
	}
	
	return 0;
	
}

INT APIENTRY _tWinMain(_In_ HINSTANCE hInstance, _In_opt_ HINSTANCE hPrevInstance, _In_ LPWSTR lpCmdLine, _In_ int nShowCmd) {
	//第一步 注册窗口类，指定窗口类的名称与窗口回调函数
	WNDCLASSEX wcex = { 0 };
	//大小
	wcex.cbSize = sizeof(WNDCLASSEX);
	//当前窗口的处理逻辑 窗口函数以及处理(窗口过程)
	wcex.lpfnWndProc = WindowProc;
	//窗口名字
	wcex.lpszClassName = szWindowClass;
	//注册窗口
	if (!RegisterClassEx(&wcex))
	{
		MessageBox(NULL, _T("注册窗口失败"), _T("tips"), NULL);
		return 1;
	}
	//第二步 创建窗口，指定注册窗口类，窗口标题，窗口的大小
	HWND hWnd = CreateWindow(
		szWindowClass,
		szTitle,
		WS_OVERLAPPEDWINDOW,
		CW_USEDEFAULT,
		CW_USEDEFAULT,
		500,300,
		NULL,
		NULL,
		hInstance,
		NULL
	);
	
	if (!hWnd)
	{
		MessageBox(NULL, _T("注册窗口失败"), _T("tips"), NULL);
		return 1;
	}
	// 第三步：显示窗口
	ShowWindow(hWnd, nShowCmd);
	// 第四步 消息循环
	 msg = {0};
	while (GetMessage(&msg,NULL,0,0))
    {
        //转换消息
		TranslateMessage(&msg);
        //分发消息
		DispatchMessage(&msg);
	}
	return (int)msg.wParam;
}
```

### GetMessage

> ```
> BOOL GetMessage(
>   LPMSG lpMsg, //Msg主要用于封装消息，消息是一些已经定义好的编号,存在其中
>   HWND  hWnd,
>   UINT  wMsgFilterMin,
>   UINT  wMsgFilterMax
> );
> ```
>
> 第一个是指向MSG消息结构的指针，
>
> 第二个是获取那个窗口的的消息，为null即接收所有的消息
>
> 第三个第四个都是过滤条件 

### 分发|转换

由于每一个窗口的处理过程不一样，MSG只能封装到是属于窗口句柄没法找到执行对应的处理即窗口函数，所以需要又经由操作系统转发到指定的窗口函数。这些都是属于内核的操作

如果需要获取按键的值需要进行转换也就是TranslateMessage，消息类型wm_char

## 消息类型

> MSG

```c++
typedef struct tagMSG {     // msg  
   HWND hwnd;//那个窗口
   UINT message;//消息编号
   WPARAM wParam;//
   LPARAM lParam;//描述消息的实际内容
   DWORD time;// 消息什么时候产生的
   POINT pt;//消息产生的坐标是什么
} MSG;
```



## 子窗口

> HWND HDWND = CreateWindow(WC_EDIT, _T(""),WS_BORDER| WS_CHILD | BS_PUSHBUTTON | WS_VISIBLE, 150, 100, 100, 50, hWnd, (HMENU)IDC_EDIT, NULL, NULL);
> HWND HBWND_1 = CreateWindow(WC_BUTTON, _T("提交"), WS_CHILD | BS_PUSHBUTTON | WS_VISIBLE, 250, 100, 100, 50, hWnd, (HMENU)IDC_BUTTON, NULL, NULL);

第一个参数设置系统预定好的窗口控件，他就是子窗口所以也能用CreateWindow函数，主要是参数二和窗口id(IDC_EDIT、IDC_BUTTON)。

### 注意

创建以子窗口创建就是WS_CHILD这个参数是必须的，那为什么样式用|“或”来连接呢，这些宏参数转换成二进制都是整数譬如：2——10，4——100，进行或运算，系统只需要找到1的位置就知道是什么样式了。具体就不拔下来了，CreateWindow函数其实还是比较简单的，别看参数多。



## 虚拟内存与物理内存

一般来讲为了更高效的利用空间，更安全的访问数据，虚拟内存的必不可少的。反过来讲，早期计算机并没有虚拟内存的概念，随着应用体积不断增大，如果直接访问使用物理内存，会出现很多问题，比如

>1. 多个应用占用内存，等到内存不够用的时候，后面的程序只能等待，或者去使用硬盘空间的，一般来讲内存是直通CPU的，频繁的使用硬盘空间效率也不会高到哪里去。
>2. 内存是随机分配的，多个应用占用的话搞不好，数据地址，运行的地址可能是没有关系的，这样操作错误会大大增加
>3. 最后直接操作内存，没有隔离那么每一个进程对于另一个进程那当然是想改就改，包括系统进程，这非常不安全

事实上，当然现在都是64位cpu，那么如果是32位的CPU的电脑平台为什么只能识别4G的cpu呢。首先这里的位宽是在一个时钟周期内可以处理的数据量，取决于cpu最小处理执行单元的位宽，总线啊寄存器啊之类的。所以一般来说32位的寻址能力就是2的32方倍，也就是4G，请注意，这里仅仅是一般情况。具体还是百度来的快，这里就不

### 分页

在最开始的进程中已经展示了window是如何分配内存，那些能用那些不能用，那么当操作系统是如何使用的呢？

如果你使用过OD或许Xdbug64，去打开或附加一个程序你就会发现很多地址，但是你有没有想过这些地址到底是物理地址还是虚拟地址？

答案是显而易见的，是虚拟地址。

由于啊，ram也就是内存，它的操作很大程度上是随机的，所以直接使用其实也不方便，为了方便管理现在的操作系统都是以分页的方式进行管理的。页的单位是4kb。

也就是把虚拟地址和物理地址进行逻辑划分，首先操作系统要记录每个进程页表的相关信息，形成一个目录。再有这个页目录找相应的页表，页表在逻辑上是连续的，最后在映射的物理内存页上面去。

### 虚拟内存(硬盘空间)

如果物理内存不够用怎么办呢，这里就要用到虚拟内存(硬盘空间)，当然这里的虚拟内存跟前面不一样，用来暂时交换数据，同时如果当前程序使用的不再频繁，它的一些数据也会被放进虚拟内存中，原有的物理页调作他用，当再次调用的时候这里会产生一个缺页异常，系统会再分配一个新的物理页。

## 真正的申请内存的方式

1. 通过VirtualAlloc/VirtualAllocEx申请的是私有内存
2. 通过CreateFileMapping映射的是共享内存

C/C++ 的malloc和new其实是一种方式，这里去申请的内存，无论是堆还是栈都是从整个程序已经被系统分配的内存中的内存再利用。而VirtualAlloc这些是向操作系统要实实在在的物理页。

## 私有内存的申请释放

### 私有内存

> 物理页只归当前进程使用

### 申请与释放



> VirtualAlloc/VirtualAllocEx(可以去别的进程申请内存)
>
> 通常是大块内存的分配，如果想在进程A和进程B之间通过共享内存的方式实现通信，可以使用该函数（这也是较常用的情况）。
>
> ```c++
> LPVOID VirtualAlloc(
> 	LPVOID lpAddress, // 要分配的区域的起始地址，注意要设置为没有被占用的空间，通常为空
> 	DWORD dwSize, // 分配的内存大小   虽然这里的单位是bit但是由于操作系统的原因，这里最少也是一个也就是4k
>  	DWORD flAllocationType, // 分配的类型
>  	DWORD flProtect // 该内存的初始保护属性 只读 可写 执行代码
> );
> ```
>

flAllocationType主要使用这俩个参数

| 值                        | 含义                                                         |
| :------------------------ | :----------------------------------------------------------- |
| **MEM_COMMIT**0x00001000  | 为指定的保留内存页分配内存开销（从内存磁盘上的页面文件进行物理存储）。该函数还保证当调用者最初访问存储器时，内容将为零。除非直到实际访问虚拟地址，否则不会分配实际的物理页面。要一步一步保留和提交页面，请使用 调用 **VirtualAlloc**`MEM_COMMIT | MEM_RESERVE`。试图通过指定提交一个特定地址范围**MEM_COMMIT**而不 **MEM_RESERVE**和非**NULL** *lpAddress*除非整个范围已经被预留失败。产生的错误代码是**ERROR_INVALID_ADDRESS**。尝试提交已提交的页面不会导致功能失败。这意味着您无需先确定每个页面的当前承诺状态就可以提交页面。如果*lpAddress*指定一个飞地内的地址，则*flAllocationType*必须为**MEM_COMMIT**。 |
| **MEM_RESERVE**0x00002000 | 保留进程的虚拟地址空间，而不在内存或磁盘上的页面文件中分配任何实际的物理存储。您可以在对**VirtualAlloc**函数的后续调用中提交保留的页面 。为了保存和提交一步到位的网页，打电话**的VirtualAlloc**与 **MEM_COMMIT** \| **MEM_RESERVE**。其他内存分配功能（例如**malloc**和 [LocalAlloc）](https://docs.microsoft.com/en-us/windows/desktop/api/winbase/nf-winbase-localalloc)在释放之前不能使用保留的内存范围。 |

> VirtualFree
>
> ```c++
> BOOL VirtualFree(
> LPVOID lpAddress,//释放的地址
> SIZE_T dwSize,//释放的大小
> DWORD  dwFreeType //操作的类型 
> );
> ```
>

 dwFreeType

| 值                         | 含义                                                         |
| :------------------------- | :----------------------------------------------------------- |
| **MEM_DECOMMIT**0x00004000 | 取消提交页面的指定区域。操作完成后，页面处于保留状态。如果您尝试取消提交未提交的页面，该功能不会失败。这意味着您可以在不首先确定当前提交状态的情况下取消提交一系列页面。所述**MEM_DECOMMIT**当不支持值*lpAddress*参数提供的基址的飞地。 |
| **MEM_RELEASE**0x00008000  | 释放页面的指定区域或占位符（对于占位符，地址空间已释放并可用于其他分配）。完成此操作后，页面将处于空闲状态。如果指定此值，则当保留该区域时，*dwSize*必须为0（零），并且*lpAddress*必须指向[VirtualAlloc](https://docs.microsoft.com/en-us/windows/desktop/api/memoryapi/nf-memoryapi-virtualalloc)函数返回的基地址 。如果不满足这些条件之一，则功能将失败。如果当前已提交该区域中的任何页面，则该函数会先解除授权，然后释放它们。如果您尝试释放处于不同状态，保留状态和提交状态的页面，则该功能不会失败。这意味着您可以释放一系列页面，而无需先确定当前的承诺状态。 |

 使用**MEM_RELEASE时**，此参数可以另外指定以下值之一。

| 值                                      | 含义                                                         |
| :-------------------------------------- | :----------------------------------------------------------- |
| **MEM_COALESCE_PLACEHOLDERS**0x00000001 | 要合并两个相邻的占位符，请指定`MEM_RELEASE | MEM_COALESCE_PLACEHOLDERS`。合并占位符时，*lpAddress*和*dwSize*必须与占位符完全匹配。 |
| **MEM_PRESERVE_PLACEHOLDER**0x00000002  | 将分配释放回占位符（使用[VirtualAlloc2](https://msdn.microsoft.com/en-us/library/Mt832849(v=VS.85).aspx)或[Virtual2AllocFromApp](https://msdn.microsoft.com/en-us/library/Mt832850(v=VS.85).aspx)将占位符替换为私有分配[后](https://msdn.microsoft.com/en-us/library/Mt832850(v=VS.85).aspx)）。要将占位符拆分为两个占位符，请指定`MEM_RELEASE | MEM_PRESERVE_PLACEHOLDER` |



```c++
int main(int argc, char* argv[])
{

	//申请  
    //MEM_COMMIT 虚拟地址保留，需要物理内存
    //MEM_RESERVE 执行完后虚拟地址保留，不需要物理内存
	LPVOID pmem = VirtualAlloc(NULL, 0x2000*2, MEM_COMMIT, PAGE_READWRITE);
	
	//释放
	//MEM_DECOMMIT 虚拟地址保留，物理页清空
	//MEM_RELEASE 虚拟地址,物理页都清空，不过第二个参数得填0
	VirtualFree(pmem, 0x2000 * 2, MEM_DECOMMIT);
	return 0;
}

```

## 共享内存的申请释放

> 一块物理页被多个进程使用，便是内存map共享内存

### 申请与释放

> CreateFileMapping(还可以直接文件直接映射到物理内存)
>
> ```c++
> HANDLE CreateFileMapping(
>   HANDLE hFile,                       //文件句柄
>   LPSECURITY_ATTRIBUTES lpAttributes, //安全描述符，内核对象的标志
>   DWORD flProtect,                    //保护设置 只读 读写 写拷贝
>   DWORD dwMaximumSizeHigh,            //文件大小高32位
>   DWORD dwMaximumSizeLow,             //文件大小低32位
>   LPCTSTR lpName                      //共享内存(内核)名称
> );
> ```

这个函数可以申请物理，同时也可以关联一个具体的文件。如果仅仅像申请内存，那么第一个参数填空，最后一个参数如果不用跨进程共享，可以不用设置。

申请完内存，开始映射虚拟地址

> MapViewOfFile
>
> ```c++
> LPVOID MapViewOfFile(
>   HANDLE hFileMappingObject,//句柄
>   DWORD dwDesiredAccess,//虚拟内存的属性，不能越权申请物理页的权限
>   DWORD dwFileOffsetHigh,
>   DWORD dwFileOffsetLow,
>   DWORD dwNumberOfBytesToMap
> );
> ```
>
> 

```c++
HANDLE g_MapMem;
LPTSTR g_lpaddr;
int main(int argc, char* argv[])
{

	
	//申请物理页
	g_MapMem = CreateFileMapping(INVALID_HANDLE_VALUE, NULL, PAGE_READWRITE, 0, 0x2000, _T("内存map"));
	//映射到具体的虚拟内存
	g_lpaddr = (LPTSTR)MapViewOfFile(g_MapMem, FILE_MAP_ALL_ACCESS, 0, 0, 0x2000);
	g_lpaddr = (TCHAR*)_T("Aaa");
	printf("%ws", g_lpaddr);
	//关闭映射
	UnmapViewOfFile(g_MapMem);
	getchar();
    //关闭句柄
	CloseHandle(g_MapMem);
	return 0;
}

```



第一个写

```c++
HANDLE g_MapMem;
LPTSTR g_lpaddr;
int main(int argc, char* argv[])
{

	
	//申请物理页
	
	g_MapMem = CreateFileMapping(INVALID_HANDLE_VALUE, NULL, PAGE_READWRITE, 0, 0x8000, _T("内存map"));
	g_lpaddr = (LPTSTR)MapViewOfFile(g_MapMem, FILE_MAP_ALL_ACCESS, 0, 0, 0x8000);


	*(PWORD)g_lpaddr = 0x1234;

	printf("%x", *g_lpaddr);
	//释放
	
	getchar();
	UnmapViewOfFile(g_MapMem);
	CloseHandle(g_MapMem);
	
	return 0;
}

```

第二个读

```c++
HANDLE g_MapMem;
LPTSTR g_lpaddr;

int main(int argc, char* argv[])
{

	//同一个物理页存在了创建的时候直接返回存在的那个句柄
	//g_MapMem = CreateFileMapping(INVALID_HANDLE_VALUE, NULL, PAGE_READWRITE, 0, 0x8000, L"AAA");
	//或者使用OpenFileMapping

	g_MapMem = OpenFileMapping(PAGE_READWRITE, TRUE, _T("内存map"));
	g_lpaddr = (LPTSTR)MapViewOfFile(g_MapMem, FILE_MAP_READ, 0, 0, 0x8000);
	
	
	printf("%x", *g_lpaddr);
	//关闭映射
	UnmapViewOfFile(g_MapMem);
	CloseHandle(g_MapMem);
	getchar();
	
	return 0;
}
```



## 文件系统

> 文件系统是操作系统用于管理磁盘上文件的方法和数据结构:简单点说就是在磁盘上如何组织文件的方法

`列如`:

|              | NTFS   | FAT32  |
| ------------ | ------ | ------ |
| 磁盘分区容量 | 2t     | 32g    |
| 单个文件容量 | 4G以上 | 最大4G |
| EFS加密      | 支持   | 不支持 |
| 磁盘配额     | 支持   | 不支持 |

windowsAPI对底层封装了，所以在调用的时候不用管相关内容的差别。

### 卷:这是文件系统的顶层设计

> 一个硬盘被分区后产生的分区被叫做卷

接下来便是目录，然后是文件

### 文件相关的api

1. 创建文件

   CreateFile()

2.  关闭文件

   CloseHandle()

3.  获取文件长度

   GetFileSize()

4.  获取文件的属性和信息

   GetFileAttributes()/GetFileAttributesEx()

5. 读写拷贝删除

   ReadFile()/CopyFile()/DeleteFile()

6. 查找

   FindFirstFile()/FIndNextFile()

### 卷相关API

1. 获取卷

   GetLogicalDrives()

2. 获取一个卷的盘符

   GetLogicalDriveStrings()

3. 获取卷的的类型

   GetDriveType()

4. 获取卷的卷的信息

   GetVolumeInformation()

   ```c++
   BOOL GetVolumeInformation( 
     _In_opt_   LPCTSTR lpRootPathName, // root directory 卷根目录
     _Out_opt_  LPTSTR lpVolumeNameBuffer,// volume name buffer 磁盘驱动器卷标名
     _In_       DWORD nVolumeNameSize,//磁盘驱动器卷标名长度
     _Out_opt_  LPDWORD lpVolumeSerialNumber,//磁盘驱动器卷标序列号与硬盘序列号不一样
     _Out_opt_  LPDWORD lpMaximumComponentLength,//系统允许的最大文件名长度
     _Out_opt_  LPDWORD lpFileSystemFlags,//文件系统标识
     _Out_opt_  LPTSTR lpFileSystemNameBuffer,//文件系统名称
     _In_       DWORD nFileSystemNameSize//文件操作系统名称长度
   );
   ```
   

待续

## 内存映射文件

把硬盘的文件直接映射到物理内存，再由物理内存映射到进程的虚拟内存。这样操作起来会比较简单，如果是一个比较大的文件这样也比直接IO读取好得多，并且可以多进程访问同一份资源。

```c++
HANDLE g_MapMem;
LPTSTR g_lpaddr;
HANDLE hFILE;
int main(int argc, char* argv[])
{
	hFILE = CreateFile(_T("C:\\Users\\userName\\Desktop\\新建文本文档.txt"),
		GENERIC_READ|GENERIC_WRITE,
		0,
		NULL,
		OPEN_EXISTING,
		FILE_ATTRIBUTE_NORMAL,
		NULL);

	//申请物理页 第一个参数不为空就是把那个文件对象映射到内存里面去
	g_MapMem = CreateFileMapping(hFILE, NULL, PAGE_READWRITE, 0, 0, _T("内存map"));
	//将文件映射到具体的虚拟内存  返回的是映射首地址
	g_lpaddr = (LPTSTR)MapViewOfFile(g_MapMem, FILE_MAP_WRITE, 0, 0, 0);

	//读取文件文件

	/*DWORD dwTest1 = *(PDWORD)g_lpaddr;
	DWORD dwEnd = *((PDWORD)g_lpaddr+0x9578);//偏移等于要读的地址与基地址之差除以4
	printf("%x--end%x", dwTest1, dwEnd);*/

	//写入内存(缓存) 注意是小端模式
	*(PDWORD64)g_lpaddr = 0x4F4C4C4548;//HELLO 注意打开的一定要是非空文件
	//更新缓存，将缓存地址里的东西写入文件
	FlushViewOfFile((PDWORD64)g_lpaddr,1);
	
	
	getchar();
	//关闭资源
    //关闭映射
	UnmapViewOfFile(g_MapMem);
	CloseHandle(g_MapMem);
	CloseHandle(hFILE);
	return 0;
}
```

可以使用另一个线程或者进程去读取这块内存CreateFileMapping变成OpenFileMapping(FILE_MAP_ALL_ACCESS,false,你的取得对象名字)再映射到你自己的内存，注意同步问题

### 写拷贝  Copy On Write

这么方便的东西如果另一个进程或者线程进行读操作的时候，如果这块内存依存度对于某个程序较高的话会不会有安全问题，实际上当你尝试修改的时候你会发现并没成功更改相关的内存，这事为什么呢？这时候就需要说到写拷贝了，从Copy On Write这句英语来看就能懂了，就是你读是读的同一块内存，但是你写的时候会进行拷贝生成一个新的物理页让你改，所以你根本没改到原来的内存。

>  这里要实现相关的功能必须让文件的保护设置为Copy On Write，CreateFileMapping物理页映射设置PAGE_WRITECOPY，MapViewOfFile映射虚拟内存设置FILE_MAP_COPY

先说不用写拷贝属性

```c++

using namespace std;
HANDLE g_MapMem;
LPTSTR g_lpaddr;
HANDLE hFILE;
HANDLE threadFile;
HANDLE threadMain;
DWORD WINAPI  ThreadProc(LPVOID lpParameter);
int main(int argc, char* argv[])
{
	threadMain = ::GetCurrentThread();

	hFILE = CreateFile(_T("C:\\Users\\userName\\Desktop\\新建文本文档.txt"),
		GENERIC_READ|GENERIC_WRITE,
		0,
		NULL,
		OPEN_EXISTING,
		FILE_ATTRIBUTE_NORMAL,
		NULL);

	//申请物理页 第一个参数不为空就是把那个文件对象映射到内存里面去
	g_MapMem = CreateFileMapping(hFILE, NULL, PAGE_READWRITE, 0, 0, _T("内存map"));
	//将文件映射到具体的虚拟内存  返回的是映射首地址
	g_lpaddr = (LPTSTR)MapViewOfFile(g_MapMem, FILE_MAP_WRITE, 0, 0, 0);

	//读取文件文件

	DWORD dwFirst1 = *(PDWORD)g_lpaddr;
	//DWORD dwEnd = *((PDWORD)g_lpaddr+0x9578);
	//printf("%x--end%x", dwTest1, dwEnd);
	printf("主线程%x\n", dwFirst1);
	Sleep(3000);
	CreateThread(NULL, 0, ThreadProc, NULL, 0, NULL);
	Sleep(3000);
	
	dwFirst1 = *(PDWORD)g_lpaddr;
	printf("主线程%x\n", dwFirst1);
	getchar();
	//关闭资源
	//关闭映射
	UnmapViewOfFile(g_MapMem);
	CloseHandle(g_MapMem);
	CloseHandle(hFILE);
	CloseHandle(threadMain);
	CloseHandle(threadFile);
	
	return 0;
}
DWORD WINAPI  ThreadProc(LPVOID lpParameter)
{
	threadFile = OpenFileMapping(FILE_MAP_ALL_ACCESS, false, _T("内存map"));
	LPTSTR a = (LPTSTR)MapViewOfFile(threadFile, FILE_MAP_WRITE, 0, 0, 0);

	*(PDWORD)a = 22222;//写入的值
	FlushViewOfFile((PDWORD)a, 4);
	printf("线程二已修改：%x\n", *a);
	return 0;
}
```

> 结果
>
> 主线程802127b
> 线程二已修改：56ce
> 主线程56ce

每次都会不一样。

请注意如果你写入的值没有更改那么运行得到的肯定是一样的，看上去会出现写拷贝一样的效果但是并不意味着没有修改内存

像这样

> 主线程d05
> 线程二已修改：d05
> 主线程d05

`更改ThreadProc`

```c++
DWORD WINAPI  ThreadProc(LPVOID lpParameter)
{
	threadFile = OpenFileMapping(FILE_MAP_ALL_ACCESS, false, _T("内存map"));
	LPTSTR a = (LPTSTR)MapViewOfFile(threadFile, FILE_MAP_COPY, 0, 0, 0);//FILE_MAP_COPY就是写拷贝

	*(PDWORD)a = 333333;
	FlushViewOfFile((PDWORD)a, 4);
	printf("线程二已修改：%x\n", *a);
	return 0;
}
```

> 主线程56ce
> 线程二已修改：1615
> 主线程56ce

这样就解决了

## 静态链接库

windows平台的是\*.lib 动态链接库是\*.dll

linux对应是.a和.so

它是为了应对协同开发，软件功能模块化而产生的运行库文件，在链接时需要它



```c++
#include <iostream>
#include <tchar.h>
#include "Debug/test1.h"
using namespace std;
#pragma comment(lib,"Debug/静态库_1.lib")
int main(int argc, char* argv[])
{
	cout << add(1, 1) << endl;
	system("pause");
	return 0;
}

```

上诉代码需要你把头文件和lib拷进要使用的项目目录，然后如果你没有像我一样放进Debug目录其实不需要目录路径。除此之外，还可以把lib和头文件放进项目目录然后添加头文件后，在vs项目属性链接器》常规或输入那里写上你的lib名字，有路径要加上。

### 缺点

这个东西其实是把需要使用的代码在经过编译后放进你的代码中，如果项目比较大需要使用这个模块的项目也比较多，那么这样看上去会很臃肿，会有很多重复的公共代码。也就是将lib和可执行文件编译到一起

为了解决这些问题，动态链接库由此诞生。

## 动态链接库

1. 在在需要导出的函数声明前面需要使用extern "C" _declspec(dllexport)  (调用约定，不加为项目默认)

2. 使用.def配置文件

   ```c++
   LIBRARY "Dll_1st"
   
   EXPORTS
   	Plus @123
   	Sub @321 //如果只有编号不要名字  在编号后面添加NONAME
   ```

3. 使用lib文件H头文件隐式链接

4. `exe和dll本质上没有区别，只不过一般exe没有导出表，不过不代表一定没有`

### 使用dll，显示链接

**load 动态库dll文件**  

<span style="color:red">注意</span>

>`定义函数指针的时候项目调用约定必须跟dll文件相同`
>
>`如果dll的导出函数没有添加extern "C"那么生成的函数会经过c++的Name Mangling重写了你的函数名这时候需要相关工具知道这个名字才能调用，或者使用编号获取函数地址`

```c++
#include <iostream>
#include <tchar.h>
#include <windows.h>

using namespace std;

//定义函数指针
typedef int (*lpPlus)(int,int);
typedef int (*lpSub)(int,int);

//声明函数指针变量
lpSub sub;

lpPlus _plus;//plus是关键字

int main(int argc, char* argv[])
{
	//加载模块
	HINSTANCE hModule = LoadLibrary(_T("Dll_1st.dll"));
	//获取函数地址
	sub = (lpSub)GetProcAddress(hModule, (char*)0x2);//编号16进制
	_plus = (lpPlus)GetProcAddress(hModule, "Plus");
    //调用
	int a = _plus(1, 1);
	int b = sub(1,1);
	system("pause");
    //释放链接库
	FreeLibrary(hModule);
	return 0;
}


```



### 隐式链接

lib文件和dll都有的情况下可以将lib添加到资源文件或者在代码编码添加，如果是在头文件声明导出还需要手动添加都文件或者添加到头文件，在vc++目录中可以设置包含目录（头文件）和库文件

> #pragma  comment(lib,"Dll_1st.lib")

dll放进运行环境目录

```c++

#include <iostream>
#include <tchar.h>
#include <windows.h>

using namespace std;

#pragma  comment(lib,"Dll_1st.lib")

__declspec(dllimport) int Plus(int x, int y);
__declspec(dllimport) int Sub(int x, int y);

int main(int argc, char* argv[])
{
	int a = Plus(1, 2);
	int b = Sub(3, 2);
	
	return 0;
}




```

相比显示隐式省去的找函数地址的烦恼尤其不知道函数名的情况下，但是一定是在有lib和dll文件并且你对这个dll有所了解情况下使用。

<span style="color:red">注意</span>

> 即便是现在这种编码以及使用lib文件但是它跟静态库还是有本质区别的，就是动态链接库间接调用，跟exe分开编译，具体可以跟进去汇编看

通过导入表可以查看使用的dll模块和具体那个函数。



### dllmain

```c++
#include "pch.h"

BOOL APIENTRY DllMain( HMODULE hModule,
                       DWORD  ul_reason_for_call,
                       LPVOID lpReserved
                     )
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```

> HMODULE hModule

指向加载者的句柄

>  DWORD  ul_reason_for_call

 何时调用

分为四种情况

1. DLL_PROCESS_ATTACH:进程映射

   DLL文件第一次映射到进程的地址空间，具体就是LoadLibrary或者LoadLibraryEx第一次执行

2. DLL_THREAD_ATTACH:线程映射

   再次映射地址空间或者有个新线程加载dll是DLL_THREAD_ATTACH，

3. DLL_THREAD_DETACH:线程卸载

   加载dll的线程结束后

4. DLL_PROCESS_DETACH:进程卸载

   当DLL被从进程的地址空间解除映射，具体就是FreeLibrary。如果进程结束前dll没有解除映射，结束后会自动解除映射。

   注意：当用DLL_PROCESS_ATTACH调用DLL的DllMain函数时，如果返回False，说明没有初始化成功，系统仍会用DLL_PROCESS_DETACH调用DLL的DllMain函数。因此，必须确保清理那些没有成功初始化的东西。



## 远程线程



## 远程线程注入

## 进程间的通信

## 模块隐藏

## 注入代码