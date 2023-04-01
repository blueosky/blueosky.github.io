## 参考

- **[Android应用启动全流程分析（源码深度剖析）](https://www.jianshu.com/p/37370c1d17fc)**
- **[一篇文章读懂Android Framework](https://juejin.cn/post/6844903717301387272)**
- **[Android开发十六《AndroidFramework》](https://www.jianshu.com/p/fa0c882a101d)**
- **[深入理解Handler源码框架（你要知道的那点事）](https://www.jianshu.com/p/0ddee6003d6f)**

## 准备

        本次探讨过程基于的Android源码版本为：android-7.1.1_r1，因此需要提前准备好源码，下载源码的方式多种多样，这里不做过多解释，比较推荐从清华的源下载，下载之后如果能编译的话更好，更能加深理解。


- https://source.android.google.cn/source/downloading?hl=zh-cn
- https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/
- **源码编译相关**：
  - https://github.com/plter/CompileAndroidSource
  - https://www.bilibili.com/video/BV1BJ411t7Ry/
  - https://notes.sunofbeach.net/pages/bf20b9

## App启动流程/步骤

1. mActivity.startActivity-->
2. mActivity.startActivityForResult-->
3. mInstrumentation.execStartActivity-->
4. **ActivityManagerNative.getDefault().startActivity**-->  //**跨进程，App调用到AMS**
5. ActivityManagerService.startActivity-->
6. ActivityManagerService.startActivityAsUser-->
7. mActivityStarter.startActivityMayWait-->
8. mActivityStarter.startActivityLocked-->
9. mActivityStarter.startActivityUnchecked-->
10. mSupervisor.resumeFocusedStackTopActivityLocked-->
11. ActivityStack.resumeTopActivityUncheckedLocked-->
12. ActivityStack.resumeTopActivityInnerLocked-->
13. mStackSupervisor.**startSpecificActivityLocked**--> //分岔点，**如果activity所在的进程已存在就直接realStartActivityLocked-->scheduleLaunchActivity，如果不存在才创建相应的进程**
14. ActivityManagerService.startProcessLocked
15. Process.start
16. Process.startViaZygote
17. Process.**zygoteSendArgsAndGetResult**  //**跨进程，AMS通过socket向zygote发送创建新进程的请求**
18. ZygoteConnection.runOnce  //zygote进程启动后一直在runSelectLoop中监听AMS客户端的请求，检测到一次调用runOnce
19. Zygote.forkAndSpecialize
20. ZygoteConnection.**handleChildProc**  //
21. RuntimeInit.zygoteInit
22. RuntimeInit.applicationInit
23. RuntimeInit.invokeStaticMain（有的版本是叫findStaticMain）//**反射加载创建ActivityThread类对象并调用其main方法**
24. ActivityThread.main
25. new ActivityThread().attach
26. ActivityManagerNative.getDefault().attachApplication(mAppThread, startSeq); //**跨进程，将自己（即ApplicationThread）注册到AMS中**
27. ActivityManagerService.attachApplication
28. ActivityManagerService.attachApplicationLocked  //
29. thread.bindApplication  //**跨进程，调用ApplicationThread的bindApplication**
30. sendMessage(H.BIND_APPLICATION, data);
31. handleBindApplication(data); //创建应用的Application等等
32. data.info.makeApplication; // 这里的info是LoadedApk
    - mActivityThread.mInstrumentation.newApplication
      - app.attach(context);
        - attachBaseContext(context); //
33. mInstrumentation.callApplicationOnCreate(app);
34. app.onCreate();   //**最终调用到`Application#onCreate`生命周期函数，即APP应用开发者能控制的第一行代码！**



## 附录

- **系统&&App启动流程图**：

  https://upload-images.jianshu.io/upload_images/5343604-5ad9ce4b8bf2a527.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200

![](https://upload-images.jianshu.io/upload_images/5343604-5ad9ce4b8bf2a527.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)
