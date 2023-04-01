## 前言

​		本文承接上文 ***Activity启动过程及其跨进程通信*** ，在上文的最后走到了ActivityThread的**handleLaunchActivity**这一步，带着问题和目的才能不迷失方向，接下来结合一个问题：“为什么子线程中不能更新UI” 来理解Activity的创建过程。

## 准备

        本次探讨过程基于的Android源码版本为：android-7.1.1_r1，因此需要提前准备好源码，下载源码的方式多种多样，这里不做过多解释，比较推荐从清华的源下载，下载之后如果能编译的话更好，更能加深理解。


- https://source.android.google.cn/source/downloading?hl=zh-cn
- https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/
- **源码编译相关**：
  - https://github.com/plter/CompileAndroidSource
  - https://www.bilibili.com/video/BV1BJ411t7Ry/
  - https://notes.sunofbeach.net/pages/bf20b9

## 一、为什么子线程中不能更新UI ?

​		讨论为什么之前先问一下是不是，验证代码：

```java
    @Override
    protected void onResume() {
        super.onResume();
        //
        new Thread(new Runnable() {
            @Override
            public void run() {
                ((TextView)findViewById(R.id.editText)).setText("---onResume");
            }
        }).start();
    }
```

​		出乎意料的是，跑起来之后没有报错并且文本框的内容成功地被修改成了“---onResume”，这是为什么？，这里先暂且留下一个疑问，然后再试一下另外一种情况：在setText之前先sleep一小段时间（或者通过点击按钮触发子线程setText）。

```java
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                ((TextView)findViewById(R.id.editText)).setText("---onResume");
```

​		这次就成功触发了异常CalledFromWrongThreadException（如下）并导致了应用Crash，但...这又是为什么呢？（PS：从报的异常Log里面就已经包含一些线索）

```java
1970-03-07 06:07:01.615 20091-20116/per.goweii.android.anypermission E/AndroidRuntime: FATAL EXCEPTION: Thread-2
    Process: per.goweii.android.anypermission, PID: 20091
    android.view.ViewRootImpl$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.
        at android.view.ViewRootImpl.checkThread(ViewRootImpl.java:6891)
        at android.view.ViewRootImpl.invalidateChildInParent(ViewRootImpl.java:1083)
        at android.view.ViewGroup.invalidateChild(ViewGroup.java:5205)
        at android.view.View.invalidateInternal(View.java:13656)
        at android.view.View.invalidate(View.java:13592)
        at android.widget.TextView.invalidateRegion(TextView.java:5347)
        at android.widget.TextView.invalidateCursor(TextView.java:5290)
        at android.widget.TextView.spanChange(TextView.java:8284)
        at android.widget.TextView$ChangeWatcher.onSpanAdded(TextView.java:10398)
        at android.text.SpannableStringBuilder.sendSpanAdded(SpannableStringBuilder.java:1228)
        at android.text.SpannableStringBuilder.setSpan(SpannableStringBuilder.java:767)
        at android.text.SpannableStringBuilder.setSpan(SpannableStringBuilder.java:677)
        at android.text.Selection.setSelection(Selection.java:76)
        at android.text.Selection.setSelection(Selection.java:87)
        at android.text.method.ArrowKeyMovementMethod.initialize(ArrowKeyMovementMethod.java:312)
        at android.widget.TextView.setText(TextView.java:4468)
        at android.widget.TextView.setText(TextView.java:4337)
        at android.widget.EditText.setText(EditText.java:89)
        at android.widget.TextView.setText(TextView.java:4312)
        at per.goweii.android.anypermission.MainActivity$3.run(MainActivity.java:91)
        at java.lang.Thread.run(Thread.java:761)
```

​		知其然知其所以然，要弄明白这个问题，可以从两个方向入手：

### 1.搜索源代码

​		在上面的异常Log中有一句比较关键的话：**Only the original thread that created a view hierarchy can touch its views**，可以在Android源码搜索这个字符串：

```java
zz@ubuntu:~/AndroidSource/android-7.1.1_r1$ find ./ -name "*.java" | xargs grep "Only the original thread that created a view hierarchy can touch its views"
./frameworks/base/core/java/android/view/ViewRootImpl.java:                    "Only the original thread that created a view hierarchy can touch its views.");
grep: ./frameworks/data-binding/integration-tests/App: No such file or directory
grep: With: No such file or directory
grep: Spaces/app/src/main/java/android/databinding/appwithspaces/MainActivity.java: No such file or directory
zz@ubuntu:~/AndroidSource/android-7.1.1_r1$
```

​		可以看到搜索结果中只有ViewRootImpl这个文件查到了字符串，打开ViewRootImpl.java可以看到确实存在：

```java
void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
                "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```

​		由此可以推断最底层的原因是执行了checkThread方法，检测当前线程如果不是主线程则抛异常：CalledFromWrongThreadException

### 2.追查setText

​		setText的调用过程如下（其实和上面的异常Log打印的调用是一样的）：

- TextView.setText

- ......省略

- TextView$ChangeWatcher.onSpanAdded

- TextView.spanChange

- TextView.invalidateCursor

- TextView.invalidateRegion

- View.invalidate

- View.invalidateInternal    //找到当前View的父布局ViewParent并调用父布局的invalidateChild

- ViewGroup.invalidateChild   //

- ViewGroup.invalidateChildInParent   //

- ViewRootImpl.invalidateChildInParent

- ViewRootImpl.checkThread

  到这里就算是理清了从setText到抛异常的过程，也就是子线程中不能更新UI的原因。


结束...了吗？然而别忘了还有一个留下的疑问：为什么直接setText没事而sleep了一段时间就会报错？，这里就要结合Activity的创建过程，来了解最关键的ViewRootImpl是什么时候被创建并起作用的。

## 二、Activity的创建和初始化

​		承接另一篇***Activity启动过程及其跨进程通信*** 中的最后一步：ActivityThread的**handleLaunchActivity**，接下来的调用步骤如下：

- ActivityThread.handleLaunchActivity
- ActivityThread.performLaunchActivity
  - activity = mInstrumentation.newActivity   //反射创建Activity对象
  - Application app = r.packageInfo.makeApplication  //创建Application
  - activity.attach   //调用Activity的attach
    - mWindow = new PhoneWindow  //创建表示应用窗口的PhoneWindow对象
    - mWindow.setWindowManager //为PhoneWindow配置WindowManager
    - mInstrumentation.callActivityOnCreate  //**调用Activity生命周期函数onCreate**
- ActivityThread.handleResumeActivity
  - ActivityThread.performResumeActivity
    - r.activity.performResume();  //**调用Activity生命周期函数onResume**
  - WindowManager.addView  //这里的WindowManager真正实现类是WindowManagerImpl
  - mGlobal.addView  //WindowManagerGlobal
  - root = **new ViewRootImpl(view.getContext(), display);**
  - root.setView(view, wparams, panelParentView);

​		从上面的步骤可以看到，Activity创建后，需要在走完onCreate和onResume之后，才会创建ViewRootImpl，再把根布局DecorView设置进ViewRootImpl，此后才会有checkThread检查线程的操作，在此之前在子线程能更新UI且不会抛异常（严格来讲此时setText并不算更新UI，ViewRootImpl还未开启绘制流程，只是改变了TextView中的变量值），综上所述之前的疑问也就得到了解释：因为在子线程sleep了一段时间之后，主线程走完了Activity初始化的流程，ViewRootImpl已经创建并生效，此时再setText会报错也就是理所当然的了。

## 三、如何在子线程中更新UI ？

​		了解以上内容后考虑一个问题：如果非要再子线程中更新UI如何实现？，解答：

- 其中一种是跟上面的操作一样，在ViewRootImpl创建之前“更新”

- 另外一种则是根据checkThread函数的“漏洞”，既然checkThread是判断当前线程和new ViewRootImpl时候的线程（因为checkThread中的mThread是在new的时候初始化的）是否是同一个，那只要在子线程中创建、添加UI，然后再在子线程中更新UI，就能保证是同一个了，下面试验一下：

  ```java
      @Override
      protected void onResume() {
          super.onResume();
          //
          WindowManager wm = (WindowManager) getSystemService(Context.WINDOW_SERVICE);
          new Thread(new Runnable(){
              @Override
              public void run() {
                  try {
                      Thread.sleep(1000);
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }
                  Looper.prepare();
                  TextView asyncText = new TextView(MainActivity.this);
                  asyncText.setText("asyncText");
                  WindowManager.LayoutParams param = new WindowManager.LayoutParams();
                  param.width = WindowManager.LayoutParams.WRAP_CONTENT;
                  param.height = WindowManager.LayoutParams.WRAP_CONTENT;
                  wm.addView(asyncText,param); //这里触发new ViewRootImpl
                  try {
                      Thread.sleep(5000);
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }
                  asyncText.setText("asyncText-----");
                  Looper.loop();
              }
          }).start();
      }
  
  ```

  运行起来，经过几秒后，页面上成功显示出了TextView内容为“asyncText-----”，猜想验证成功。

  

## 参考

- **[Android应用启动全流程分析（源码深度剖析）](https://www.jianshu.com/p/37370c1d17fc)**
- **[深入理解Handler源码框架（你要知道的那点事）](https://www.jianshu.com/p/0ddee6003d6f)**
- **[Android 在子线程中直接更新UI](http://events.jianshu.io/p/a03b2413186d)**
