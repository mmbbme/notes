# MFC使用指南

严格来说mfc不能说是落伍吧，微软自家GUI库有很多，只是随着windows的落寞，win开发变得不那么吃香了而已，mfc也不是仅仅只有GUI。而且随着各种跨平台技术的发展，开发效率上比不上这些技术，毕竟这些是技术基本都是一套代码多个平台嘛，但是如果要开发或者学习windows开发用一用也无妨。

### 主线程类

> 它的初始化函数创建主对话框

1. InitInstance

   对话框加载之前执行的函数，初始化控件，加载其他库

2. CmfcDigFirstDlg

   主对话框类，dlg.DoModal，模态对话框，在此时主对话框阻塞需要你自己退出

> CMFCDlgFirstApp::ExitInstance() 中的 MessageBox(_T(“XXX”));为什么不好用？_

虽然 CMFCDlgFirstApp 也属于MFC的类，但不是直接或者间接派生自 CWnd，所以默认调用的是全局的 MessageBox API 函数，也就是应该这样：MessageBox(NULL, char\*, char\*, MB_OK);
而不是调用 CWnd 中的这样： MessageBox(_T("XXXX"));

### 主对话框类

> 负责显示界面

1. 对话框类的构造和析构函数，可以初始化内部的一些成员变量的值

   此时对话框窗口并没被创建出来，这时候M_hWnd为null，无法去创建其他其他控件

2. 对话框的初始化函数:OnInitDialog(),也就是WM_INITDIALOG消息的响应函数。这里面可以进行初始化关于界面方面的一些操作。

   但是要保证

   > - CDIalog::OnInitDialog();必须被先调用，这样才能保证对话框界面及其上面的控件被正常的创建出来，在之后的位置再去添加其他代码；
   > - 告诉你在哪里加代码就在那里加，不要乱加

3. DoDataExchange 

   数据校验与控件的变量绑定（限定输入范围），不需要手动改

4. BEGIN_MESSAGE_MAP

   MFC封装的消息的映射宏，每一条消息对应的消息的响应函数绑定

5. OnPaint

   绘制函数，一般没有相关的需求可以不用动

6. OnQueryDragIcon

   当用户拖动最小化窗口时系统调用此函数取得光标
   显示也不用动

## 控件指针的获取

> GetDIgItem函数
>
> 1. 随时用随时取
> 2. 声明类成员变量指针，初始化函数中的读取（推荐）

mfc对于控件的窗口的句柄进行了封装，每一个控件就对应类，所以在mfc中获取控件的窗口句柄就相当于获取这个对象的指针

### 值类型的成员变量

> 需要UpdateDate函数的支持,参数是一个bool类型，只能在对话框线程使用
>
> TRUE是界面值保存参数，FALSE是参数值返回到界面

## MFC创建模态对话框

1. 创建模板

2. 绑定自定义对话框类

3. 创建的模态对话框，DoModel

   初始化销毁函数如果需要自己处理需要在属性中添加

4. 销毁模态对话框，CDialog::OnOK()、CDialog::OnCancel()、CDialog::OnClose()

## MFC创建非模态对话框

1. 创建模板

2. 绑定自定义对话框类

3. 创建非模态对话框,Create、ShowWindow

4. 销毁非模态对话框,CWnd::DestroyWindow()可以

   如果有确认按钮不仅需要在OnCancel中DestroyWindow()，OnOK也需要

5. 若要销毁自身窗口指针的话，可以重载PostNcDestroy()函数，之后添加delete this;