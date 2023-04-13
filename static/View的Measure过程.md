## 参考

​		先阅读以下文章理解DecorView根布局和Measure相关：

- **[Android DecorView 一窥全貌（上）](https://blog.csdn.net/wekajava/article/details/120608380)**
- **[Android DecorView 一窥全貌（下）](https://blog.csdn.net/wekajava/article/details/120608451)**
- **[Android 自定义View之Measure过程](https://www.jianshu.com/p/23822b8f900d)**
- **[Android Window 如何确定大小 onMeasure()多次执行原因](https://www.jianshu.com/p/a7ab49462ebe)**

## 环境

​		本文基于的环境如下：

- 源码版本：android-7.1.1_r1
- 手机型号：Nexus 6P（2560x1440像素）

## View的Measure过程

### 下面以一个最简化的例子分析View的Measure过程

#### **例子的准备过程如下：**

- 设置MainActivity的布局activity_main内容为最最简化的一个View，并且宽高设置为**wrap_content**：

  ```java
  <?xml version="1.0" encoding="utf-8"?>
  <per.goweii.android.anypermission.MyView
      android:layout_width="wrap_content"
      android:layout_height="wrap_content"
      xmlns:android="http://schemas.android.com/apk/res/android"
      />
  ```

- 其中MyView的具体内容如下，仅仅重写了onDraw画了一点简单的图形，未重写onMeasure：

  ```java
  public class MyView extends View {
      private Paint paint;
      public MyView(Context context) {
          super(context);
          init();
      }
      //从xml加载MyView时调用该方法
      public MyView(Context context, @Nullable AttributeSet attrs) {
          super(context, attrs);
          init();
      }
      private void init() {
          paint = new Paint();
          paint.setAntiAlias(true);
      }
      @Override
      protected void onDraw(Canvas canvas) {
          //涂红色
          canvas.drawColor(Color.RED);
          //画笔设置为黄色
          paint.setColor(Color.YELLOW);
          //画实心圆
          canvas.drawCircle(getWidth() / 2, getHeight() / 2, 30, paint);
      }
  }
  ```

  

  运行如上的例子，这个MyView的宽高会怎么样的呢？没错，正如预想的一样，MyView铺满 除了顶部状态栏和底部导航栏的之外的 屏幕内容，那么考虑一个问题：从最顶层的DecorView到MyView有几层布局？又是怎样从DecorView一层一层Measure到MyView？下面使用Layout Inspector再结合源码慢慢分析。

  

#### Layout Inspector查看布局结构

​		使用AS的Layout Inspector查看到页面的布局结构如下：

- **DecorView**---------------------------------------------最顶层布局，继承FrameLayout（高度731dp）

  - **LinearLayout**-------------------------高度 = 731-48 = 683dp（paddingTop：24dp，marginBottom：48dp）
    - action_mode_bar_stub------------------ViewStub（没有使用，高度为0）
    - **content**---------------------------------------FrameLayout（高度 = 683-24 = 659dp）
      - **MyView**--------------------------------最里层的自定义View（高度659dp）
  - navigationBarBackground--------------------底部导航栏（高度48dp）
  - statusBarBackground--------------------------顶部状态栏（高度24dp）

  

  DecorView下的布局结构在**[Android DecorView 一窥全貌（上）](https://blog.csdn.net/wekajava/article/details/120608380)**和**[Android DecorView 一窥全貌（下）](https://blog.csdn.net/wekajava/article/details/120608451)**中有详细的解释，可知上面中的LinearLayout结构源自DecorView所加载的布局文件**R.layout.screen_simple**，打开发现能跟上面的结构对应上，其内容如下，：

  ```java
  <?xml version="1.0" encoding="utf-8"?>
  <!--
  /* //device/apps/common/assets/res/layout/screen_simple.xml
  */
  This is an optimized layout for a screen, with the minimum set of features
  enabled.
  -->
  <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
      android:layout_width="match_parent"
      android:layout_height="match_parent"
      android:fitsSystemWindows="true"
      android:orientation="vertical">
      <ViewStub android:id="@+id/action_mode_bar_stub"
                android:inflatedId="@+id/action_mode_bar"
                android:layout="@layout/action_mode_bar"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:theme="?attr/actionBarTheme" />
      <FrameLayout
           android:id="@android:id/content"
           android:layout_width="match_parent"
           android:layout_height="match_parent"
           android:foregroundInsidePadding="false"
           android:foregroundGravity="fill_horizontal|top"
           android:foreground="?android:attr/windowContentOverlay" />
  </LinearLayout>
  ```

  

#### 从DecorView逐层往下分析Measure

**DecorView的onMeasure**

- **View的每一层onMeasure方法需要传上一层ViewGroup分配给本层的宽高widthMeasureSpec和heightMeasureSpec**，那么作为最顶层的DecorView的onMeasure这两个参数又是怎么来的呢？可参考文章**[Android Window 如何确定大小 onMeasure()多次执行原因](https://www.jianshu.com/p/a7ab49462ebe)**，在ViewRootImpl的**performTraversals**方法中（如下，标记(3)处）：

  ```java
  #ViewRootImpl.java
      final WindowManager.LayoutParams mWindowAttributes = new WindowManager.LayoutParams();//默认的LayoutParams是MATCH_PARENT
      private void performTraversals() {
          ...
          //之前记录的Window LayoutParams
          WindowManager.LayoutParams lp = mWindowAttributes;
          //Window需要的大小
          int desiredWindowWidth;
          int desiredWindowHeight;
          ...
          Rect frame = mWinFrame;
          if (mFirst) {
              ...
              if (shouldUseDisplaySize(lp)) {
                  ...
              } else {
                  //mWinFrame即是之前添加Window时返回的Window最大尺寸
                  desiredWindowWidth = mWinFrame.width();
                  desiredWindowHeight = mWinFrame.height();
              }
              ...
          } else {
              ...
          }
          ...
          if (layoutRequested) {
              ...
              //从方法名看应该是测量ViewTree -----------(1)
              windowSizeMayChange |= measureHierarchy(host, lp, res,
                      desiredWindowWidth, desiredWindowHeight);
          }
          ...
          if (mFirst || windowShouldResize || insetsChanged ||
                  viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
              ...
              try {
                  ...
                  //重新确定Window尺寸 --------(2)
                  relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
                  ...
              } catch (RemoteException e) {
              }
              ...
              if (!mStopped || mReportNextDraw) {
                  ...
                  if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                          || mHeight != host.getMeasuredHeight() || contentInsetsChanged ||
                          updatedConfiguration) {
                      ...
                      //
                      Log.i(mTag, mWidth +"---"+ lp.width);//手动增加的Log
                      Log.i(mTag, mHeight+"---"+ lp.height);//手动增加的Log
                      //
                      int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                      int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
                      //再次测量ViewTree -------- (3)
                      performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                      ...
                  }
              }
          } else {
              ...
          }
          ...
          if (didLayout) {
              //对ViewTree 进行Layout ---------- (4)
              performLayout(lp, mWidth, mHeight);
              ...
          }
          ...
          if (!cancelDraw) {
              ...
              //开始ViewTree Draw过程 ------- (5)
              performDraw();
          } else {
              ...
          }
      }
      private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
          Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
          try {
              mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
          } finally {
              Trace.traceEnd(Trace.TRACE_TAG_VIEW);
          }
      }
  ```

- 为了直观和验证DecorView的onMeasure，在ViewRootImpl和DecorView中增加了Log，DecorView的onMeasure简化如下：

  ```java
      protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
          ...
          final int widthMode = getMode(widthMeasureSpec);
          final int heightMode = getMode(heightMeasureSpec);
          //
          Log.i(mLogTag, widthMode +"---"+ MeasureSpec.getSize(widthMeasureSpec));//手动增加的Log
          Log.i(mLogTag, heightMode +"---"+ MeasureSpec.getSize(heightMeasureSpec));//手动增加的Log
          ...
          if (widthMode == AT_MOST) {
              ...
          }
          if (heightMode == AT_MOST) {
              ...
          }
          super.onMeasure(widthMeasureSpec, heightMeasureSpec);
          if (!fixedWidth && widthMode == AT_MOST) {
              ...
          }
          ...
      }
  ```

- 增加Log后编译源码刷机到Nexus 6P，重写跑起来，打印如下：

  ```java
  1970-03-26 09:05:48.925 11572-11572/per.goweii.android.anypermission I/DecorView[]: 1073741824---1439
  1970-03-26 09:05:48.925 11572-11572/per.goweii.android.anypermission I/DecorView[]: 1073741824---2307
  1970-03-26 09:05:48.943 11572-11572/per.goweii.android.anypermission I/ViewRootImpl[MainActivity]: 1440----1
  1970-03-26 09:05:48.944 11572-11572/per.goweii.android.anypermission I/ViewRootImpl[MainActivity]: 2560----1
  1970-03-26 09:05:48.944 11572-11572/per.goweii.android.anypermission I/DecorView[]: 1073741824---1440
  1970-03-26 09:05:48.944 11572-11572/per.goweii.android.anypermission I/DecorView[]: 1073741824---2560
  ```

- 可以看到ViewRootImpl中的打印：1440  ---  -1 和 2560  ---  -1，结合下面的源码可以得知，**getRootMeasureSpec传入了MATCH_PARENT和手机屏幕的宽、高，因此此时传入DecorView onMeasure的宽高都是EXACTLY  精确的屏幕宽高值**（这一点从上面DecorView的打印也可以看出来，**宽高Mode都 = EXACTLY = 1左移30位 = 1073741824**）

  ```java
  #ViewRootImpl.java
  private static int getRootMeasureSpec(int windowSize, int rootDimension) {
      int measureSpec;
      switch (rootDimension) {
      case ViewGroup.LayoutParams.MATCH_PARENT:
          // Window can't resize. Force root view to be windowSize.
          measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
          break;
      case ViewGroup.LayoutParams.WRAP_CONTENT:
          // Window can resize. Set max size for root view.
          measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
          break;
      default:
          // Window wants to be an exact size. Force root view to be that size.
          measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
          break;
      }
      return measureSpec;
  }
  ```

  ```java
  #ViewGroup.java
  public abstract class ViewGroup extends View implements ViewParent, ViewManager {
      public static class LayoutParams {
          public static final int FILL_PARENT = -1;
          public static final int MATCH_PARENT = -1;
          public static final int WRAP_CONTENT = -2;
      }    
  }
  ```

  ```java
  #View.java
  public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {
      public static class MeasureSpec {
          private static final int MODE_SHIFT = 30;
          private static final int MODE_MASK = 0x3 << MODE_SHIFT;
          public static final int UNSPECIFIED = 0 << MODE_SHIFT;
          public static final int EXACTLY = 1 << MODE_SHIFT;
          public static final int AT_MOST = 2 << MODE_SHIFT;
      }
  }
  ```

- 接下来开始看DecorView的onMeasure，由于DecorView继承了FrameLayout，主要看**FrameLayout的onMeasure**，由于此时宽高都是精确值，可以简化成如下：

  ```java
  #FrameLayout.java
      @Override
      protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
          int count = getChildCount();
          // ...
          for (int i = 0; i < count; i++) {
              final View child = getChildAt(i);
              if (mMeasureAllChildren || child.getVisibility() != GONE) {
                  measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                  // ...
              }
          }
          // ...
          setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                  resolveSizeAndState(maxHeight, heightMeasureSpec,
                          childState << MEASURED_HEIGHT_STATE_SHIFT));
  
          // ...
      }
  ```

  ```java
  #View.java
      public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
          final int specMode = MeasureSpec.getMode(measureSpec);
          final int specSize = MeasureSpec.getSize(measureSpec);
          final int result;
          switch (specMode) {
              case MeasureSpec.AT_MOST:
                  if (specSize < size) {
                      result = specSize | MEASURED_STATE_TOO_SMALL;
                  } else {
                      result = size;
                  }
                  break;
              case MeasureSpec.EXACTLY:
                  result = specSize;
                  break;
              case MeasureSpec.UNSPECIFIED:
              default:
                  result = size;
          }
          return result | (childMeasuredState & MEASURED_STATE_MASK);
      }
  ```

- 其中的重点是**measureChildWithMargins**，它是FrameLayout继承自**ViewGroup**的方法，具体如下：

  ```java
  #ViewGroup.java
      protected void measureChildWithMargins(View child,
              int parentWidthMeasureSpec, int widthUsed,
              int parentHeightMeasureSpec, int heightUsed) {
          final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
  
          final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                  mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                          + widthUsed, lp.width);//----------------计算左右padding和margin
          final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                  mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                          + heightUsed, lp.height);//----------------计算上下padding和margin
  
          child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
      }
      public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
          int specMode = MeasureSpec.getMode(spec);
          int specSize = MeasureSpec.getSize(spec);
          int size = Math.max(0, specSize - padding);//----计算本层能给的最大值：即本层大小减去相应的边距
          int resultSize = 0;
          int resultMode = 0;
          switch (specMode) {
          // Parent has imposed an exact size on us
          case MeasureSpec.EXACTLY:
              if (childDimension >= 0) {
                  resultSize = childDimension;
                  resultMode = MeasureSpec.EXACTLY;
              } else if (childDimension == LayoutParams.MATCH_PARENT) {
                  // Child wants to be our size. So be it.
                  resultSize = size;
                  resultMode = MeasureSpec.EXACTLY;
              } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                  // Child wants to determine its own size. It can't be
                  // bigger than us.
                  resultSize = size;
                  resultMode = MeasureSpec.AT_MOST;
              }
              break;
          // Parent has imposed a maximum size on us
          case MeasureSpec.AT_MOST:
              if (childDimension >= 0) {
                  // Child wants a specific size... so be it
                  resultSize = childDimension;
                  resultMode = MeasureSpec.EXACTLY;
              } else if (childDimension == LayoutParams.MATCH_PARENT) {
                  // Child wants to be our size, but our size is not fixed.
                  // Constrain child to not be bigger than us.
                  resultSize = size;
                  resultMode = MeasureSpec.AT_MOST;
              } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                  // Child wants to determine its own size. It can't be
                  // bigger than us.
                  resultSize = size;
                  resultMode = MeasureSpec.AT_MOST;
              }
              break;
          // Parent asked to see how big we want to be
          case MeasureSpec.UNSPECIFIED:
              if (childDimension >= 0) {
                  // Child wants a specific size... let him have it
                  resultSize = childDimension;
                  resultMode = MeasureSpec.EXACTLY;
              } else if (childDimension == LayoutParams.MATCH_PARENT) {
                  // Child wants to be our size... find out how big it should
                  // be
                  resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                  resultMode = MeasureSpec.UNSPECIFIED;
              } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                  // Child wants to determine its own size.... find out how
                  // big it should be
                  resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                  resultMode = MeasureSpec.UNSPECIFIED;
              }
              break;
          }
          //noinspection ResourceType
          return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
      }
  ```

- 重点是**getChildMeasureSpec**方法：**根据本层的MeasureSpec和子View设置的LayoutParams来决定分给子View的MeasureSpec**，总共9种情况，常见的几类用伪代码总结如下：

  ```java
  int size = Math.max(0, specSize - padding);//----计算本层能给的最大值：即本层大小减去相应的边距
  //---------------------
  if(childDimension > 0 ){
      //即如果子View写了精确的dp值
      //则不管本层MeasureSpec是什么，通通按照子View的精确dp值给
      resultSize = childDimension;
      resultMode = MeasureSpec.EXACTLY;
  }
  //---------------------
  if( (本层是EXACTLY && 子View是WRAP_CONTENT) || 
      (本层是AT_MOST && 子View是MATCH_PARENT或者WRAP_CONTENT)
    ){
      //那就给个本层能给的最大值
      resultSize = size;
      resultMode = MeasureSpec.AT_MOST;
  }
  //---------------------
  if( 本层是EXACTLY && 子View是MATCH_PARENT ){
      //本层有精确值，子View要填充父布局，那就给子View保持跟本层一致
      resultSize = size;
      resultMode = MeasureSpec.EXACTLY;
  }
  ```

  

**此时回顾页面的布局结构**：

- **DecorView**------------------------------ **最顶层布局，宽高有精确值**（411dp、731dp）

  - **LinearLayout**----------------------**宽高都是match_parent，精确传递**，因此宽度=Parent=411dp，高度 = 731-48 = 683dp（paddingTop：24dp，layout_marginBottom：48dp）
    - **content**------------------------**宽高都是match_parent，精确传递**，因此宽度=Parent=411dp，高度 = 683-24 = 659dp
      - **MyView**------------------**宽高都是wrap_content，AT_MOST模式**，因此最大宽度是411dp，最大高度是659dp

- 重点看最后一步的MyView，View的Measure过程：**由于上层传进来的是AT_MOST模式，并且MyView没有重写onMeasure，因此宽高都是取上层的最大值**，相应的源码如下：

  ```java
  #View.java
      protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
          setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                  getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
      }
      public static int getDefaultSize(int size, int measureSpec) {
          int result = size;
          int specMode = MeasureSpec.getMode(measureSpec);
          int specSize = MeasureSpec.getSize(measureSpec);
          switch (specMode) {
          case MeasureSpec.UNSPECIFIED://--------UNSPECIFIED模式：用自己的大小
              result = size;
              break;
          case MeasureSpec.AT_MOST://--------AT_MOST和EXACTLY模式都取上一层View分配给它的推荐值
          case MeasureSpec.EXACTLY:
              result = specSize;
              break;
          }
          return result;
      }
      protected int getSuggestedMinimumWidth() {
          return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
      }
      protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
          boolean optical = isLayoutModeOptical(this);
          if (optical != isLayoutModeOptical(mParent)) {
              Insets insets = getOpticalInsets();
              int opticalWidth  = insets.left + insets.right;
              int opticalHeight = insets.top  + insets.bottom;
  
              measuredWidth  += optical ? opticalWidth  : -opticalWidth;
              measuredHeight += optical ? opticalHeight : -opticalHeight;
          }
          setMeasuredDimensionRaw(measuredWidth, measuredHeight);
      }
      private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
          mMeasuredWidth = measuredWidth;
          mMeasuredHeight = measuredHeight;
  
          mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
      }
  ```

  

## 致谢

​		再次感谢秉承开源和分享精神的前辈：

- ##### 小鱼人爱编程

  - https://github.com/fishforest
  - https://blog.csdn.net/wekajava
  - https://www.jianshu.com/u/c3187f5a9eb1
  - https://juejin.cn/user/3658822686609774
    - https://www.jianshu.com/nb/51480743
    - https://github.com/fishforest/AndroidDemo

- ##### Carson带你学Android

  - https://github.com/Carson-Ho
  
  - https://carsonho.blog.csdn.net/
  
  - https://blog.csdn.net/carson_ho
  
  - https://www.jianshu.com/u/383970bef0a0
  
  - https://juejin.cn/user/2524134385917293
    - https://github.com/Carson-Ho/DIY_View
  
- ##### 徐宜生/eclipse_xu（Android群英传）

  - https://xuyisheng.top/
  - https://github.com/xuyisheng
  - https://blog.csdn.net/eclipsexys
  - https://juejin.cn/user/43636194286093
  - https://www.jianshu.com/u/dfc0ed52c22b
    - Android群英传源码：https://github.com/xuyisheng/AndroidHeroes
