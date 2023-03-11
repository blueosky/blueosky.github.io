## 前言

​		看了《Android开发艺术探索》中的“Activity的工作过程”这一节和网上一些相关的优秀文章（如下），还有看了AIDL相关的内容之后；为了加深理解，把两者结合起来，决定把——从“打开一个Activity（即startActivity）的过程”再到“此过程中涉及到的跨进程通信” 记录下来，以供参考。

- https://github.com/singwhatiwanna/android-art-res
- **[四大组件的工作过程](https://www.milovetingting.cn/2020/01/09/Android%E5%BC%80%E5%8F%91%E8%89%BA%E6%9C%AF%E6%8E%A2%E7%B4%A2/%E5%9B%9B%E5%A4%A7%E7%BB%84%E4%BB%B6%E7%9A%84%E5%B7%A5%E4%BD%9C%E8%BF%87%E7%A8%8B/)**

## 准备

        本次探讨过程基于的Android源码版本为：android-7.1.1_r1，因此需要提前准备好源码，下载源码的方式多种多样，这里不做过多解释，比较推荐从清华的源下载，下载之后如果能编译的话更好，更能加深理解。

- https://source.android.google.cn/source/downloading?hl=zh-cn
- https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/
- **源码编译相关**：
  - https://github.com/plter/CompileAndroidSource
  - https://www.bilibili.com/video/BV1BJ411t7Ry/
  - https://notes.sunofbeach.net/pages/bf20b9

## 过程

​		Activity启动流程在milovetingting的文章——**[四大组件的工作过程](https://www.milovetingting.cn/2020/01/09/Android%E5%BC%80%E5%8F%91%E8%89%BA%E6%9C%AF%E6%8E%A2%E7%B4%A2/%E5%9B%9B%E5%A4%A7%E7%BB%84%E4%BB%B6%E7%9A%84%E5%B7%A5%E4%BD%9C%E8%BF%87%E7%A8%8B/)**中已经有了很详细的讲解，本次把打开一个Activity简单概括为三个块，重点介绍其中的两次跨进程。

1. **App端的一些函数调用.....**

   - **App跨进程调用到AMS端（即am.startActivity）**

2. **AMS端的一些函数调用.....**

   - **AMS跨进程调用到App端（即app.thread.scheduleLaunchActivity）**

3. **App端的一些函数调用.....**

   
   
   ​	本次讨论用到的源码文件如下：


```
frameworks\base\core\java\android\app\
    Activity.java
    ActivityManagerNative.java
    ApplicationThreadNative.java
    IActivityManager.java
    Instrumentation.java
   
frameworks\base\services\core\java\com\android\server\am
    ActivityManagerService.java
    
```

### 一、从最外层的startActivity到跨进程为止 App端的调用如下：

1. mActivity.startActivity-->

2. mActivity.startActivityForResult-->

3. mInstrumentation.execStartActivity-->

4. **ActivityManagerNative.getDefault().startActivity**-->

   其他几步都还好，重点是最后一步的跨进程

#### 重点：App跨进程调用到AMS端：

​		类似AIDL自动生成的代码，ActivityManagerNative这个文件在App进程和AMS所在的system_server进程都有，大致结构如下：

```java
public abstract class ActivityManagerNative extends Binder implements IActivityManager
{
    static public IActivityManager getDefault() {
        return gDefault.get();
    }
    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case START_ACTIVITY_TRANSACTION:
        {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();
            //这里通过asInterface把App端传过来的Binder用ApplicationThreadProxy包装了起来
            IApplicationThread app = ApplicationThreadNative.asInterface(b);//
            String callingPackage = data.readString();
            Intent intent = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();
            IBinder resultTo = data.readStrongBinder();
            String resultWho = data.readString();
            int requestCode = data.readInt();
            int startFlags = data.readInt();
            ProfilerInfo profilerInfo = data.readInt() != 0
                    ? ProfilerInfo.CREATOR.createFromParcel(data) : null;
            Bundle options = data.readInt() != 0
                    ? Bundle.CREATOR.createFromParcel(data) : null;
            int result = startActivity(app, callingPackage, intent, resolvedType,
                    resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
            reply.writeNoException();
            reply.writeInt(result);
            return true;
        }
        //......省略了其他各个case
}
class ActivityManagerProxy implements IActivityManager
{
    public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        //注意这一步这里把App端的Binder即ApplicationThread作为参数传了过去
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);//
        data.writeString(callingPackage);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(resultTo);
        data.writeString(resultWho);
        data.writeInt(requestCode);
        data.writeInt(startFlags);
        if (profilerInfo != null) {
            data.writeInt(1);
            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        if (options != null) {
            data.writeInt(1);
            options.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
    //......省略了因为IActivityManager接口要实现的其他方法
}
```

​		可以看到ActivityManagerNativeh和AIDL自动build生成的文件结构很相似，其实用法也是相似的

- 重点看ActivityManagerNative.getDefault()，其中的具体操作如下：

```java
//跨进程调用servicemanager从而获取AMS的IBinder
//用ActivityManagerProxy把AMS的IBinder包装起来，
//再用接口IActivityManager指向ActivityManagerProxy，供下一步调用
IBinder b = ServiceManager.getService("activity");
IActivityManager am = asInterface(b);
```

- ActivityManagerNative.getDefault().startActivity：

```java
//拿到IActivityManager后，am.startActivity（这里的am本质是ActivityManagerProxy），
//跨进程（transact-->onTransact）调用AMS的startActivity
//onTransact里面case START_ACTIVITY_TRANSACTION:
//IApplicationThread app = ApplicationThreadNative.asInterface(b);
```

- 然后就成功通过Binder跨进程调用到了AMS端的ActivityManagerNative的onTransact，进入相应的case START_ACTIVITY_TRANSACTION，调用方法startActivity，又由于AMS是继承自ActivityManagerNative的，startActivity的真正实现在AMS中，到这里就算是成功到了AMS。


### 二、从AMS的startActivity到跨进程到App端的调用如下：

1. ActivityManagerService.startActivity-->
2. ActivityManagerService.startActivityAsUser-->
3. mActivityStarter.startActivityMayWait-->
4. mActivityStarter.startActivityLocked-->
5. mActivityStarter.startActivityUnchecked-->
6. mSupervisor.resumeFocusedStackTopActivityLocked-->
7. ActivityStack.resumeTopActivityUncheckedLocked-->
8. ActivityStack.resumeTopActivityInnerLocked-->
9. mStackSupervisor.startSpecificActivityLocked-->
10. ActivityStackSupervisor.realStartActivityLocked-->
11. **app.thread.scheduleLaunchActivity**-->  //跨进程调用

#### 重点：AMS跨进程调用到App端：

​		app.thread中的具体实现类是ApplicationThreadNative，同样的，这个文件也是跟AIDL生成文件类似：

```java
public abstract class ApplicationThreadNative extends Binder implements IApplicationThread {
     @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION:
        {
            //....
            scheduleLaunchActivity(intent, b, ident, info, curConfig, overrideConfig, compatInfo,
                    referrer, voiceInteractor, procState, state, persistentState, ri, pi,
                    notResumed, isForward, profilerInfo);
            return true;
        }
        //case......省略 
        }
}
class ApplicationThreadProxy implements IApplicationThread {
    
    public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
            ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
            CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
            int procState, Bundle state, PersistableBundle persistentState,
            List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
            boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) throws RemoteException {
        //......省略
    }
    //......其他方法省略
}
```

​		跟上一步同理，通过Binder跨进程调用到ApplicationThreadNative.scheduleLaunchActivity，又因为ApplicationThreadNative是抽象类，其具体实现类在ApplicationThread类中，就调到了ApplicationThread的scheduleLaunchActivity，此时已经回到了App进程中。

### 三、最终回到App端后的调用如下：

1. ApplicationThread.scheduleLaunchActivity

2. ActivityThread.handleLaunchActivity-->

3. sendMessage(H.LAUNCH_ACTIVITY, r);

4. H.handleMessage-->

5. case LAUNCH_ACTIVITY: handleLaunchActivity

6. ActivityThread.handleLaunchActivity-->

7. ActivityThread.performLaunchActivity-->

   

   ​	完成！

## 结语

​		Activity启动过程中的跨进程通信总结如上，源码错综复杂，如有不对的地方望指正。

​		再次感谢秉承开源和分享精神的前辈：

- #### 任玉刚，Android开发艺术探索

  https://github.com/singwhatiwanna
  https://blog.csdn.net/singwhatiwanna

- #### milovetingting

  http://www.milovetingting.cn/
  https://github.com/milovetingting
  https://blog.csdn.net/milovetingting
  http://events.jianshu.io/u/0bcb9876fbc0

  - [四大组件的工作过程](https://www.milovetingting.cn/2020/01/09/Android%E5%BC%80%E5%8F%91%E8%89%BA%E6%9C%AF%E6%8E%A2%E7%B4%A2/%E5%9B%9B%E5%A4%A7%E7%BB%84%E4%BB%B6%E7%9A%84%E5%B7%A5%E4%BD%9C%E8%BF%87%E7%A8%8B/)
    http://events.jianshu.io/p/ca0901603363
    https://blog.csdn.net/milovetingting/article/details/103917121
  
- #### Gityuan

  http://gityuan.com/
  https://blog.csdn.net/gityuan
  https://www.zhihu.com/people/gityuan
  
  - [理解Android ANR的触发原理](http://gityuan.com/2016/07/02/android-anr/)
  
