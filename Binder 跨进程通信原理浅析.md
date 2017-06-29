**关于跨进程：**
为什么要跨进程呢？在Android系统中，每个进程都有分配自己的内存空间，各进程间是不能直接访问其他进程的内存的，那当一个程序要操作与另一个程序的方法怎么办呢？（比如在应用程序中隐藏SystemUI的导航栏）这时就需要跨进程通信了。Binder就是一个帮助进程进通信的虚拟设备，为什么是虚拟设备呢，因为它没有硬件，只只用代码实现的通信架构。

**从哪开始说？**
Android系统首次开机启动时，会启动很多系统服务，然后将这些服务添加到ServiceManager进行统一管理。添加这些服务时就需要跨进程通信，我们就从这里开始吧。对于Binder的各个元素就像TCP/IP网络
o. Binder 驱动 --> 路由器
o. ServiceManager --> DNS
o. Binder Client（BpBinder） --> 客户端
o. Binder Service（BnBinder） --> 服务端
*


----------
下图为Binder通信的简要原理：
![这里写图片描述](http://img.blog.csdn.net/20170614150617785?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzEwMTIwMzM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**关于ServiceManager：**
系统服务众多，为了方便，就需要一个总管，ServiceManger就是这个总管，集中管理系统内的所有服务，它能施加权限控制，并不是任何进程都能注册服务。就像DNS（它的IP地址为0），可以通过字符串去查找所对应的服务。 ServiceManger自身也同样是Binder Service。Android系统开机启动时 ServiceManger会抢先就位。和其他Native服务启动一样，首先启动一个自己的进程，打开Binder驱动，分配内存大小等。不同的是会将自己设置为Binder的大管家，整个Android系统允许有一个ServiceManger存在。后面还有设置为管家的指令会是失败。然后ServiceManger会进入循环，等待客户请求。

**跨进程通信的例子**
系统服务启动的时候会将自己注册到ServiceManger，比如StatusBarManagerService。

```
Slog.i(TAG, "Status Bar");
statusBar = new StatusBarManagerService(context);
ServiceManager.addService(Context.STATUS_BAR_SERVICE, statusBar);

```
frameworks\base\core\java\android\osServiceManager.java -->addService()：

```
    public static void addService(String name, IBinder service) {
        try {
            getIServiceManager().addService(name, service, false);
        } catch (RemoteException e) {
            Log.e(TAG, "error in addService", e);
        }
    }
```
我们先弄清楚getIServiceManager()是啥。看名字就能理解这是获取ServiceManager服务。ServiceManager进程肯定不和当前程序一个进程，此时的跨进程通信已经开始了。（这里不用去DNS找IP了，因为我们已经知道了这个“IP”，就是访问它自己，现在是直接拿IP去Binder"路由器"，看路由器是怎么访问另一端的服务的）

**一，通信开始**

***①首先我们要拿到客户端的java层的代理BpServiceXXX（在JAVA层客户端这边需要它想驱动发送请求）***

frameworks\base\core\java\android\osServiceManager.java -->getIServiceManager() ：

```
private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }

        // Find the service manager
        sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
        return sServiceManager;
    }
```

*这个BinderInternal.getContextObject()又是什么呢？其实他返回的是Native层的代理，原来Java层代理只是对Native层的一个封装*

BinderInternal.getContextObject()是其父类IBinder的一个内部类，用来获取ServiceManager的一个Native代理：
frameworks\base\core\jni\android_util_Binder.cpp

```
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);
}
```
看到这里是不是很多小伙伴更加懵逼了？不要急 ，我们一点点吧谜底揭开。（ProcessState::self（），这里所做的就是上图的映射一块内存地址。它的构造方法做作的大概就是创建一个进程，打开Binder驱动，分配内存地址什么的，具体这里就不讲了。）我们看看ProcessState.cpp里的getContextObject(NULL)。
frameworks\native\libs\binder\ProcessState.cpp

```
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
	//这里传入了0，也就是ServiceManager这个"DNS"的IP地址
    return getStrongProxyForHandle(0);//返回的BpServiceXXX（BpBinder（））
}
... ...
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);

    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
        //第一次获取的话肯定走这里
            if (handle == 0) {
                Parcel data;
                status_t status = IPCThreadState::self()->transact(
                        0, IBinder::PING_TRANSACTION, data, NULL, 0);
                if (status == DEAD_OBJECT)
                   return NULL;
            }
		//这个handle就是前面传进来的那个0
            b = new BpBinder(handle); 
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            //可以看到这里创建了一个handle值为零的BpBinder对象
            result = b;
        } else {.
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }

    return result;
}
```
*返回一个BpBinder，它就是Navtive的代理，而javaObjectForIBinder(env, BpBinder（0）)，为了保证Java层和Navtive的数据类型适配，进而的再一次封装，而java代理对此在进行一次封装，来协议化需要传递的数据*

是不是快忘了我们从哪里出发了？
①sServiceManager .addService();
②sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
相当于ServiceManagerNative.asInterface(BpBinder（0）);
frameworks\base\core\java\android\os\ServiceManagerNative.java-->asInterface():
```
static public IServiceManager asInterface(IBinder obj)
    {  
    ... ...
    //返回ServiceManagerProxy（BpBinder（0）），就是ServiceManager的客户端代理
        return new ServiceManagerProxy(obj);
    }
```
所以ServiceManager.addService(Context.STATUS_BAR_SERVICE, statusBar);就等效于
ServiceManagerProxy（javaObjectForIBinder(env, BpBinder（0））.addService(Context.STATUS_BAR_SERVICE, statusBar);

*ServiceManagerProxy就是BpServiceXXX ，是客户端的一个代理，通过代理向服务端发送请求 。而BpBinder是真正想Binder驱动发送请求的实现类。（BpBinder和BBinder的爸爸都是IBinder）*

***二，现在我们拿到了ServiceManager JAVA层的代理对象，可以开始请求通信了***

*先看看ServiceManager JAVA层的代理的实现*

ServiceManagerProxy类的构造方法。
ServiceManagerProxy是ServiceManagerNative的一个内部类
ServiceManagerProxy.java -->

```
//好像并没有做什么
public ServiceManagerProxy(IBinder remote) {
        mRemote = remote;//把BpBinder交给了自己的成员
    }
```
**开始通信发送请求：ServiceManagerProxy.addService():**
```
public void addService(String name, IBinder service, boolean allowIsolated)
           throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IServiceManager.descriptor);
        data.writeString(name);
        data.writeStrongBinder(service);
        data.writeInt(allowIsolated ? 1 : 0);
        //这里才开始通信，向服务端发送数据和请求
        mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
        reply.recycle();
        data.recycle();
    }
```
讲了这么多了，我们来理一理:
**首先通过IBinder的内部类方法BinderInternal.getContextObject()得到ServiceManager的Native层代理。然后痛过asInterface（）封装，返回一个erviceManagerProxy也就是架构中的JAVA层代理对象，然后通过它发送请求进行通信**

***三，通信原理***
这个神秘transact（）方法到底是怎么怎么想Binder驱动发送请求的呢？我们继续看。mRemote是BpBinder类型，所以这个方法就在BpBinder中。
frameworks\native\libs\binder\BpBinder.cpp -->transact（）:

```
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    if (mAlive) {
	    //mHandle的值为0
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}
```

IPCThreadState，每一个进程都会有一个这样的线程，Binder通信真正的执行者就是。IPCThreadState::self()所做的和ProcessState::self()，开启一个这样的线程，分配缓存区，然后把自己保存起来。有两个容器一个接收来自Binder驱动返回过来的数据（mIn），一个用于向Binder发送数据（mOUt）。

frameworks\native\libs\binder\IPCThreadState.cpp-->transact():

```
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err = data.errorCheck();
	... ...
   if (err == NO_ERROR) {
        LOG_ONEWAY(">>>> SEND from pid %d uid %d %s", getpid(), getuid(),
            (flags & TF_ONE_WAY) == 0 ? "READ REPLY" : "ONE WAY");
            //发送请求
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }
    
	... ...
    if ((flags & TF_ONE_WAY) == 0) {
        #if 0
        if (code == 4) { // relayout
            ALOGI(">>>>>> CALLING transaction 4");
        } else {
            ALOGI(">>>>>> CALLING transaction %d", code);
        }
        #endif
        if (reply) {
        //处理请求
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
        #if 0
        if (code == 4) { // relayout
            ALOGI("<<<<<< RETURNING transaction 4");
        } else {
            ALOGI("<<<<<< RETURNING transaction %d", code);
        }
        #endif
        
        IF_LOG_TRANSACTIONS() {
            TextOutput::Bundle _b(alog);
            alog << "BR_REPLY thr " << (void*)pthread_self() << " / hand "
                << handle << ": ";
            if (reply) alog << indent << *reply << dedent << endl;
            else alog << "(none requested)" << endl;
        }
    } else {
	    
        err = waitForResponse(NULL, NULL);
    }
    
	... ...
}
```
这段代码主要干了两件事：①发送请求err = writeTransactionData(）；②处理返回结果err = waitForResponse(）。我们再来看看这两个函数；
err = writeTransactionData(）只是将请求数据打包写到用于给Binder发送数据的容器mOut中，waitForResponse(）方法中有talkWithDriver()时和驱动打交道（通知Binder驱动有数据请求）的实现，并把返回数据保存到接收放在mIn中。然后一一对应处理。到此为止客户端所做的就这么多了。
![这里写图片描述](http://img.blog.csdn.net/20170515210514393?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzEwMTIwMzM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

***四，服务端收到请求后的反应***

*ServiceManager会在一开始系统启动中就启动，并把自己注册在Binder驱动中，把自己升为“大管家”，进入一个死循环，不断的读取Binder发来的请求，如果有请求就进行处理*

服务端的BBinder和BpBinder都继承与IBinder，都实现transact（）。BpBinder是客户端的代理，而BBinder则是服务端的实行者。

Binder驱动收到请求，找到对应的服务端，并执行BBinder的transact（）方法对请求进行处理：
frameworks\native\libs\binder\Binder.cpp：

```
status_t BBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    data.setDataPosition(0);

    status_t err = NO_ERROR;
    switch (code) {
        case PING_TRANSACTION:
            reply->writeInt32(pingBinder());
            break;
        default:
        //执行了子类的onTransact（）方法
            err = onTransact(code, data, reply, flags);
            break;
    }

    if (reply != NULL) {
        reply->setDataPosition(0);
    }

    return err;

```
服务端会继承BBinder。并实现onTransact（）方法对请求进行处理；

整个Binder架构可以用下图概括：
![这里写图片描述](http://img.blog.csdn.net/20170614140701788?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzEwMTIwMzM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


***ServiceManager的作用图解***

![这里写图片描述](http://img.blog.csdn.net/20170614170535842?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzEwMTIwMzM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

***写在最后***
创建一个Binder服务所需的工作：
1.什么时候去启动；
2.在onTransact方法中处理不同的请求，调用不同接口。对外界提供这些接口。服务端需要去实现这些接口；
3.和Binder交互，随时响应Binder的请求。（ServiceManager通过不同循环读取来交互）
4.给客户端提供代理。（客户端获取到BpBinder时，对其进行一个封装，对不同请求相应的处理）












