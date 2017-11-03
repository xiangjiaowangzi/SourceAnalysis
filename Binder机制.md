# Binder机制 #

[Binder跨进程通信机制](https://juejin.im/entry/594b2b5e0ce46300574262f4)

[Binder机制原理](http://blog.csdn.net/boyupeng/article/details/47011383)

在Android开发中，我们用到的进程间通信机制（IPC）有：socket、pipe，Binder；

Binder是什么？

![](https://i.imgur.com/uUtJ6Cy.png)

## 简单介绍进程 ##

### 进程空间分配 ###

进程空间分为**用户空间**和**内核空间**

进程间用户空间数据是不可共享的，而内核空间的数据是可以共享的

进程内 用户 与 内核 进行交互 称为系统调用

### 进程隔离 ###

为了保证 安全性 & 独立性，一个进程 不能直接操作或者访问另一个进程，即Android的进程是**相互独立、隔离**的

### 跨进程通信（ IPC ） ###

跨进程通信：

先通过 进程间 的内核空间进行 数据交互

再通过 进程内 的用户空间 & 内核空间进行 数据交互，从而实现 进程间的用户空间 的数据交互

**而Binder，就是充当 连接 两个进程（内核空间）的通道。**

![](https://i.imgur.com/7tqIsTM.png)

## Binder模型 ##

Binder基于C/S结构，由Client进程，Server进程，Service Manager 进程（简称SMgr），和Binder驱动构成

### Binder 驱动 ###

类似路由器一样，负责进程之间Binder通信的建立，进程之间的传递，Binder引用计数管理，数据包在进程之间的传递和交互等一系列底层支持。

### ServiceManager ###

和DNS类似，SMgr的作用是将字符形式的Binder名字转化成Client中对该Binder的引用，使得Client能够通过Binder名字获得对Server中Binder实体的引用。

**Server与SMgr**

对于Server，Server先创建了Binder实体，取名”张三“，然后将这个Binder和名字以数据包的形式通过Binder驱动发送给SMgr，通知SMgr注册一个名叫张三的Binder。最后将名字及新建的引用打包传递给SMgr。SMgr收数据包后，从中取出名字和引用填入一张查找表中。

问题来了：SMgr是一个进程，Server是另一个进程，他们的通信也是靠一个Binder机制：   
1. SMgr是Server端，有自己的Binder对象（实体），其它进程都是Client，需要通过这个Binder的引用来实现Binder的注册，查询和获取。                  
2. SMgr提供的Binder比较特殊，它没有名字也不需要注册，当一个进程使用BINDER_SET_CONTEXT_MGR命令将自己注册成SMgr时Binder驱动会自动为它创建Binder实体
3. 这个Binder的引用在所有Client中都固定为0而无须通过其它手段获得。也就是说，一个Server若要向SMgr注册自己Binder就必需通过0这个引用号和SMgr的Binder通信。

**Client与SMgr**

1. Server向SMgr注册了Binder实体及其名字后，Client就可以通过名字获得该Binder的引用了。Client也利用保留的0号引用向SMgr请求访问某个Binder
2. SMgr收到这个连接请求，从请求数据包里获得Binder的名字，在查找表里找到该名字对应的条目，从条目中取出Binder的引用，将该引用作为回复发送给发起请求的Client。

**匿名的Binder**

并不是所有Binder都需要注册给SMgr广而告之的。Server端可以通过已经建立的Binder连接将创建的Binder实体传给Client，当然这条已经建立的Binder连接必须是通过实名Binder实现。由于这个Binder没有向SMgr注册名字，所以是个匿名Binder。Client将会收到这个匿名Binder的引用，通过这个引用向位于Server中的实体发送请求。匿名Binder为通信双方建立一条私密通道，只要Server没有把匿名Binder发给别的进程，别的进程就无法通过穷举或猜测等任何方式获得该Binder的引用，向该Binder发送请求。

### 简单理解 ###

三部分组件之间的关系:

1.Client、Server、ServiceManager均在用户空间中实现，而Binder驱动程序则是在内核空间中实现的；

2.在Binder通信中，Server进程先注册一些Binder到ServiceManager中，ServiceManager负责管理这些Binder并向Client提供相关的接口；

3.Client进程要和某一个具体的Service通信，必须先从ServiceManager中获取该BinderProxy的相关信息，Client根据得到的BinderProxy信息与Service所在的Server进程建立通信，之后Clent就可以与Service进行交互了；

4.Binder驱动程序提供设备文件/dev/binder与用户空间进行交互，Client、Server和ServiceManager通过open和ioctl文件操作函数与Binder驱动程序进行通信；

5.Client、Server、ServiceManager三者之间的交互都是基于Binder通信的，所以通过任意两者这件的关系，都可以解释Binder的机制。

来张图：

![](https://i.imgur.com/JbisnHL.png)

**BBinder与BpBinder的区别**

其实这两者是很好区分，

对于service来说继承了BBinder（BnInterface）因为BBinder有onTransact消息处理函数，

而对于与service通信的client来说需要继承BpBinder(BpInterface)，因为BpBinder有消息传递函数transcat。

## AIDL上分析： ##

我们使用的时候首先是，创建一个binder，add（）方法是我们自己定义的

	Binder binder = new IMyService.Stub() {

        @Override
        public int add(int value1, int value2) throws RemoteException {
            if (callbackList.getRegisteredCallbackCount() > 0) {
                notifyCallBack(text1,text2);
            }
            return value1 + value2;
        }
	...

跟下去，我们主要是构造方法的时候调用Binder.attachInterface方法

		public Stub()
	{
	this.attachInterface(this, DESCRIPTOR);
	}

可以看到attachInterface方法，做了两件事

	public void attachInterface(IInterface owner, String descriptor) {
        mOwner = owner;
        mDescriptor = descriptor;
    }


这时，我们看另外一个进程的调用

	    IMyService iMyService;
			// 通过asInterface(iBinder)方法获得一个我们的 IMyService 对象，
             @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            iMyService = IMyService.Stub.asInterface(iBinder);
         ...
        }

我们看asInterface方法（）

	public static com.example.myapplication.IMyService asInterface(android.os.IBinder obj)
	// 如果传进来的IBinder是个空，直接就返回空
	{
	if ((obj==null)) {
	return null;
	}
	// queryLocalInterface方法就是根据DESCRIPTOR查询我们之前初始化的时候是否一致；
	// 如果是同一个进程的时候，这里就会调用Binder中的queryLocalInterface

	// 但是在不同进程的时候，这里调用的是BinderProxy中的queryLocalInterface，
	// 而这个BinderProxy是返回了空
	// 所以最后会返回 IMyService.Stub.Proxy(obj)
	android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
	if (((iin!=null)&&(iin instanceof com.example.myapplication.IMyService))) {
	return ((com.example.myapplication.IMyService)iin);
	}
	return new com.example.myapplication.IMyService.Stub.Proxy(obj);
	}

IMyService.Stub.Proxy(obj)是什么？

可以看到这里是自己定义的一个静态类，并且实现了IMyService接口

	private static class Proxy implements com.example.myapplication.IMyService
	{
	private android.os.IBinder mRemote;
	Proxy(android.os.IBinder remote)
	{
	mRemote = remote;
	}
	@Override public android.os.IBinder asBinder()
	{
	// 这里是返回了我们传进来的IBinder对象
	return mRemote;
	}
	public java.lang.String getInterfaceDescriptor()
	{
	return DESCRIPTOR;
	}
	@Override public int add(int value1, int value2) throws android.os.RemoteException
	{
	// 数据的读和写 必须用 android.os.Parcel类型
	android.os.Parcel _data = android.os.Parcel.obtain();
	android.os.Parcel _reply = android.os.Parcel.obtain();
	int _result;
	try {
	_data.writeInterfaceToken(DESCRIPTOR);
	_data.writeInt(value1);
	_data.writeInt(value2);
	// 关键点，最终其实是调用了 transact 方法
	mRemote.transact(Stub.TRANSACTION_add, _data, _reply, 0);
	_reply.readException();
	_result = _reply.readInt();
	}
	finally {
	_reply.recycle();
	_data.recycle();
	}
	return _result;
	}
	...

因为我们之前知道获取到的是IBinder 对象 其实是个 BinderProxy 对象，所以查看transact方法：

	 public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
       ...
        try {
			// 最终是调用了原生的transactNative方法
            return transactNative(code, data, reply, flags);
        ...
    }

经过jni调用android_os_BinderProxy_transact方法，然后一层层调用

最终回调了IMyService的onTransact，把Server计算好的数据利用reply.writeInt(_result);写出去

	@Override public boolean onTransact(int code, android.os.Parcel data,   
	           android.os.Parcel reply, int flags) throws android.os.RemoteException
	{
	switch (code)
	{
	case INTERFACE_TRANSACTION:
	{
	reply.writeString(DESCRIPTOR);
	return true;
	}
	case TRANSACTION_add:
	{
	data.enforceInterface(DESCRIPTOR);
	int _arg0;
	_arg0 = data.readInt();
	int _arg1;
	_arg1 = data.readInt();
	int _result = this.add(_arg0, _arg1);
	reply.writeNoException();
	reply.writeInt(_result);
	return true;
	}

	
## 最后 ##

为什么 Android 要采用 Binder 作为 IPC 机制？

[点我](https://www.zhihu.com/question/39440766?sort=created)

（1）从性能的角度
数据拷贝次数：Binder数据拷贝只需要一次，而管道、消息队列、Socket都需要2次，但共享内存方式一次内存拷贝都不需要；从性能角度看，Binder性能仅次于共享内存。

（2）从稳定性的角度

Binder架构优越于共享内存

（3）从安全的角度

Android系统中对外只暴露Client端，Client端将任务发送给Server端，Server端会根据权限控制策略，判断UID/PID是否满足访问权限，目前权限控制很多时候是通过弹出权限询问对话框，让用户选择是否运行。
	













