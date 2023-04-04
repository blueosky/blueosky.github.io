## 前言

​		本文基于的源码版本——android-7.1.1_r1

## View事件分发

- **Activity --> Window**（真正实现类是PhoneWindow）

```java
//Activity.java
    private Window mWindow;
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
    public Window getWindow() {
        return mWindow;
    }
```

- **PhoneWindow --> DecorView**（即顶层ViewGroup，这里的DecorView是extends FrameLayout）

```java
//PhoneWindow.java
    // This is the top-level view of the window, containing the window decor.
    private DecorView mDecor;
    @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }
```

- 从顶层ViewGroup（DecorView）开始一层一层往下分发

```java
//DecorView.java
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }

```




- **ViewGroup的分发/处理过程**，用伪代码总结如下（仅考虑单指操作的情况）：

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean handled = false;//本层所返回的，表示本层及以下是否已经处理event
    final boolean intercepted;//
    //判断本层是否要拦截
    if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
        if (!disallowIntercept) {
            intercepted = onInterceptTouchEvent(ev);
        } else {
            intercepted = false;
        }
    } else {
        //如果不是DOWN事件，并且一开始的DOWN事件没有分发给下一层View 而是自己处理了，
        //则之后的MOVE、UP事件通通都不用经过onInterceptTouchEvent，而是直接拦截
        intercepted = true;
    }
    boolean alreadyDispatchedToNewTouchTarget = false;
    //如果不拦截的话，尝试把DOWN事件分发给子View处理
    if (!canceled && !intercepted) {
        if ( is MotionEvent.ACTION_DOWN ){
            for(childrenCount){
                //分发给子View，即调用下一层View的dispatchTouchEvent，如果有多层ViewGroup嵌套会形成类似递归的结构
                if(child.dispatchTouchEvent(event)){//下一层View分发完并返回结果，如果返回true表示已经处理了event
                    mFirstTouchTarget = new Target(child);//mFirstTouchTarget用于记录 要分发给的子View
                    alreadyDispatchedToNewTouchTarget = true;
                }
            }
        }
    }
    //
    if (mFirstTouchTarget == null) {//如果没有子View，或者子View没有处理事件（或者本层拦截了）
        // 则本层自己处理事件，调用本层ViewGroup的父类（即View）的dispatchTouchEvent
        handled = super.dispatchTouchEvent();
    }else {//如果mFirstTouchTarget不为空，即有记录要分发给哪个子View
        if (alreadyDispatchedToNewTouchTarget) {
            handled = true;
        } else {
            final boolean cancelChild = xxxx() || intercepted;//
            //如果一开始没有拦截 给子View处理了（DOWN、MOVE等），但是后面如果有MOVE符合拦截条件又会重新拦截，
            //此时重新拦截需要取消给子View事件
            if(cancelChild){
                event.setAction(MotionEvent.ACTION_CANCEL);
            }
            if(child.dispatchTouchEvent(event)){//给子View处理event
                handled = true;
            }
            if (cancelChild) {//如果要取消给子View事件，
                //则清空记录
                mFirstTouchTarget = null;
            }
        }
    } 
}

```

- **View没有子View可以分发所以只有处理过程**，用伪代码总结如下：

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean result = false;
    ListenerInfo li = mListenerInfo;
    //mOnTouchListener的onTouch优先
    if (li != null && li.mOnTouchListener != null 
        && (mViewFlags & ENABLED_MASK) == ENABLED 
        && li.mOnTouchListener.onTouch(this, event)) {
        result = true;
    }
    //onTouchEvent其次
    if (!result && onTouchEvent(event)) {
        result = true;
    }
    return result;
}
/*
  View的onTouchEvent写了默认的点击事件处理流程
*/
public boolean onTouchEvent(MotionEvent event) {
    if (((viewFlags & CLICKABLE) == CLICKABLE || //如果设置了mOnClickListener或者mOnLongClickListener
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                if(xxx){
                    performClick();//调用 --> li.mOnClickListener.onClick(this);
                }
                break;
        }
        //如果可点击就都返回true，表示已处理
        return true;
    }
    return false;
}

```

接下来想到了一个问题：

### 如果ViewGroup和View都没有处理事件的话会怎么样？

可以测试一个场景：在setContentView时设置的main.xml中写一个自定义的ScrollView，ScrollView中包含一些内容使其能滑动；在Activity、PhoneWindow、DecorView、VerticalScrollView的分发事件方法都加上Log打印：

```java
Log.d(TAG, "dispatchTouchEvent:---" + ev.getAction());
或者
Log.d(TAG, "superDispatchTouchEvent:---" + event.getAction());
```

- 测试正常的情况Log如下：可以看到一个DOWN、两个MOVE、最后一个UP事件 在Activity、PhoneWindow、DecorView、VerticalScrollView四个中都有打印。

```java
1970-03-22 11:45:30.341 5529-5529/com.xujun.drag D/Activity: dispatchTouchEvent:---0
1970-03-22 11:45:30.341 5529-5529/com.xujun.drag D/PhoneWindow: superDispatchTouchEvent:---0
1970-03-22 11:45:30.341 5529-5529/com.xujun.drag D/DecorView: superDispatchTouchEvent:---0
1970-03-22 11:45:30.341 5529-5529/com.xujun.drag D/VerticalScrollView: dispatchTouchEvent---0
    
1970-03-22 11:45:30.355 5529-5529/com.xujun.drag D/Activity: dispatchTouchEvent:---2
1970-03-22 11:45:30.355 5529-5529/com.xujun.drag D/PhoneWindow: superDispatchTouchEvent:---2
1970-03-22 11:45:30.355 5529-5529/com.xujun.drag D/DecorView: superDispatchTouchEvent:---2
1970-03-22 11:45:30.355 5529-5529/com.xujun.drag D/VerticalScrollView: dispatchTouchEvent---2
1970-03-22 11:45:30.410 5529-5529/com.xujun.drag D/Activity: dispatchTouchEvent:---2
1970-03-22 11:45:30.410 5529-5529/com.xujun.drag D/PhoneWindow: superDispatchTouchEvent:---2
1970-03-22 11:45:30.410 5529-5529/com.xujun.drag D/DecorView: superDispatchTouchEvent:---2
1970-03-22 11:45:30.410 5529-5529/com.xujun.drag D/VerticalScrollView: dispatchTouchEvent---2
    
1970-03-22 11:45:30.411 5529-5529/com.xujun.drag D/Activity: dispatchTouchEvent:---1
1970-03-22 11:45:30.411 5529-5529/com.xujun.drag D/PhoneWindow: superDispatchTouchEvent:---1
1970-03-22 11:45:30.411 5529-5529/com.xujun.drag D/DecorView: superDispatchTouchEvent:---1
1970-03-22 11:45:30.411 5529-5529/com.xujun.drag D/VerticalScrollView: dispatchTouchEvent---1

```

- 接下来重写VerticalScrollView的dispatchTouchEvent方法直接返回false，模拟页面不处理事件的场景，再次测试Log如下：一个DOWN、三个MOVE、最后一个UP，其中只有DOWN事件能传到VerticalScrollView，其余的都不能；也就是说这种场景下MOVE、UP事件最多只能传到DecorView，DecorView就不会往下传了（具体原因可以看上面的ViewGroup分发过程）

```java
1970-03-22 11:34:44.304 4924-4924/com.xujun.drag D/Activity: dispatchTouchEvent:---0
1970-03-22 11:34:44.304 4924-4924/com.xujun.drag D/PhoneWindow: superDispatchTouchEvent:---0
1970-03-22 11:34:44.304 4924-4924/com.xujun.drag D/DecorView: superDispatchTouchEvent:---0
1970-03-22 11:34:44.304 4924-4924/com.xujun.drag D/VerticalScrollView: dispatchTouchEvent---0
    
1970-03-22 11:34:44.337 4924-4924/com.xujun.drag D/Activity: dispatchTouchEvent:---2
1970-03-22 11:34:44.337 4924-4924/com.xujun.drag D/PhoneWindow: superDispatchTouchEvent:---2
1970-03-22 11:34:44.338 4924-4924/com.xujun.drag D/DecorView: superDispatchTouchEvent:---2
    
1970-03-22 11:34:44.357 4924-4924/com.xujun.drag D/Activity: dispatchTouchEvent:---2
1970-03-22 11:34:44.357 4924-4924/com.xujun.drag D/PhoneWindow: superDispatchTouchEvent:---2
1970-03-22 11:34:44.357 4924-4924/com.xujun.drag D/DecorView: superDispatchTouchEvent:---2
    
1970-03-22 11:34:44.379 4924-4924/com.xujun.drag D/Activity: dispatchTouchEvent:---2
1970-03-22 11:34:44.379 4924-4924/com.xujun.drag D/PhoneWindow: superDispatchTouchEvent:---2
1970-03-22 11:34:44.380 4924-4924/com.xujun.drag D/DecorView: superDispatchTouchEvent:---2
    
1970-03-22 11:34:44.380 4924-4924/com.xujun.drag D/Activity: dispatchTouchEvent:---1
1970-03-22 11:34:44.381 4924-4924/com.xujun.drag D/PhoneWindow: superDispatchTouchEvent:---1
1970-03-22 11:34:44.381 4924-4924/com.xujun.drag D/DecorView: superDispatchTouchEvent:---1
```



## 参考

- **[这次，我把Android事件分发机制翻了个遍](https://juejin.cn/post/6844904150283583502)**

- **[Android 事件分发机制详解](https://www.jianshu.com/p/77e18c200cc0)**

- **[Android 事件分发机制详解](https://www.haomeiwen.com/subject/atfkartx.html)**

- **[学习 View 事件分发，就像外地人上了黑车](https://juejin.cn/post/6844903894103883789)**

- **[完美解决Android中的ScrollView嵌套ScrollView滑动冲突问题](https://www.jianshu.com/p/d205fca2ae0d)**

- **[ViewPager，ScrollView 嵌套ViewPager滑动冲突解决](https://blog.csdn.net/gdutxiaoxu/article/details/52939127)**

- **https://github.com/gdutxiaoxu/TouchDemo**

