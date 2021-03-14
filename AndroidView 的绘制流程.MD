#### Android View 绘制原理之 onMeasure()

从上篇内容我们知道在ViewRootImpl 里面的performMeasure 开始View 测量  
```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        if (mView == null) {
            return;
        }
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }

```
ViewRootImpl 里面的mView 就是我们之前介绍的```DecorView ``` , ```DecorView```是继承```View``` , 所以我们直接看```View``` 里面的```measure()``` 方法
在使用View的时候 是是直接设置LayoutParams ，但是在测量view 的时候， 系统会将LayoutParams 在父容器的约束下来决定具体的MeasureSpec ， 然后根据这个MeasureSpec 来确定view 测量后的宽高
，所以MeasureSpec 是需要父容器一起来决定的， 对于最顶级的View 也就是上面的DecoreView 比较特殊， 他的MeasureSpec 是个根据屏幕大小和LayoutParams 来决定的， 其中DecoreView 的LayoutParams 默认
是march_parent 的， 普通的view 的measureSpec 是由父容器的MeasureSpec 和LayoutParams 来决定的 , 如下ViewRootImpl的代码

```java
                    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);

                    if (DEBUG_LAYOUT) Log.v(mTag, "Ooops, something changed!  mWidth="
                            + mWidth + " measuredWidth=" + host.getMeasuredWidth()
                            + " mHeight=" + mHeight
                            + " measuredHeight=" + host.getMeasuredHeight()
                            + " coveredInsetsChanged=" + contentInsetsChanged);

                     // Ask host how big it wants to be
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

```
我们看getRootMeasureSpec  , 这里的rootDimension 是ViewGroup.LayoutParams.MATCH_PARENT，所以
measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY); 
```java
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
然后继续看到MeasureSpec.makeMeasureSpec  ,   makeSpec 是通过specSize 和specMode 来共同决定的，measureSpec 是一个int 类型的数据，其中由高两位来表示specMode, 底30 位来表示spec size 


SpecMode 有三类，每一类都表示特殊的含义：

UNSPECIFIED 父容器不对 View 有任何的限制，要多大给多大，这种情况下一般用于系统内部，表示一种测量的状态。
EXACTLY 父容器已经检测出 View 所需要的精确大小，这个时候 View 的最终大小就是 SpecSize 所指定的值，它对应于LayoutParams 中的 match_parent 和具体的数值这两种模式
AT_MOST 父容器指定了一个可用大小即 SpecSize，View 的大小不能大于这个值，具体是什么值要看不同 View 的具体实现。它对应于 LayoutParams 中的 wrap_content。

```java
public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,
                                          @MeasureSpecMode int mode) {
            if (sUseBrokenMakeMeasureSpec) {
                return size + mode;
            } else {
                return (size & ~MODE_MASK) | (mode & MODE_MASK);
            }
        }

``` 


对于普通的View 的measure 的调用，是由器父容器传递而来的，我们可以来看下ViewGroup 的measureChildWithMargins 方法：

```java
* @param child 要被测量的 View
 * @param parentWidthMeasureSpec 父容器的 WidthMeasureSpec
 * @param widthUsed 父容器水平方向已经被占用的空间，比如被父容器的其他子 view 所占用的空间
 * @param parentHeightMeasureSpec 父容器的 HeightMeasureSpec
 * @param heightUsed 父容器竖直已经被占用的空间，比如被父容器的其他子 view 所占用的空间
 */
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
   //第一步，获取子 View 的 LayoutParams
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
   //第二步，获取子 view 的 WidthMeasureSpec，其中传入的几个参数说明：
   //parentWidthMeasureSpec 父容器的 WidthMeasureSpec
   //mPaddingLeft + mPaddingRight view 本身的 Padding 值，即内边距值
   //lp.leftMargin + lp.rightMargin view 本身的 Margin 值，即外边距值
   //widthUsed 父容器已经被占用空间值
   // lp.width view 本身期望的宽度 with 值
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
     //获取子 view 的 HeightMeasureSpec
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);
// 第三步，根据获取的子 veiw 的 WidthMeasureSpec 和 HeightMeasureSpec 
   //对子 view 进行测量
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```
从上代码第二步可以看出，子 view 的 MeasureSpec 的创建与父容器的 MeasureSpec 、子 view 本身的 LayoutParams 有关，此外还与 view 本身的 margin 和 padding 值有关，具体看一下 getChildMeasureSpec 方法：

```java
 /*
     * @param spec 父容器的 MeasureSpec，是对子 View 的约束条件
     * @param padding 当前 View 的 padding、margins 和父容器已经被占用空间值
     * @param childDimension View 期望大小值，即layout文件里设置的大小:可以是MATCH_PARENT,
     *WRAP_CONTENT或者具体大小, 代码中分别对三种做不同的处理
     * @return 返回 View 的 MeasureSpec 值
     */
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
  // 获取父容器的 specMode，父容器的测量模式影响子 View  的测量模式
    int specMode = MeasureSpec.getMode(spec);
         // 获取父容器的 specSize 尺寸，这个尺寸是父容器用来约束子 View 大小的
    int specSize = MeasureSpec.getSize(spec);
// 父容器尺寸减掉已经被用掉的尺寸
    int size = Math.max(0, specSize - padding);
    int resultSize = 0;
    int resultMode = 0;
    switch (specMode) {
    // 如果父容器是 EXACTLY 精准测量模式
    case MeasureSpec.EXACTLY:
        //如果子 View 期望尺寸为大于 0 的固定值，对应着 xml 文件中给定了 View 的具体尺寸大小
        //如 android:layout_width="100dp"
        if (childDimension >= 0) {
          //那么子 View 尺寸为期望值固定尺寸，测量模式为精准测量模式 EXACTLY
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
             //如果子 View 期望尺寸为 MATCH_PARENT 填充父布局
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // 那么子 View 尺寸为 size 最大值，即父容器剩余空间尺寸，为精准测量模式 EXACTLY
          //即子 View 填的是 Match_parent, 那么父 View 就给子 View 自己的size(去掉padding)，
          //即剩余全部未占用的尺寸, 然后告诉子 View 这是 Exactly 精准的大小，你就按照这个大小来设定自己的尺寸
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
          //如果子 View 期望尺寸为 WRAP_CONTENT ，包裹内容
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
          //子 View 尺寸为 size  最大值，即父容器剩余空间尺寸 ，测量模式为 AT_MOST 最大测量模式
          //即子 View 填的是 wrap_Content,那么父 View 就告诉子 View 自己的size(去掉padding),
          //即剩余全部未占用的尺寸,然后告诉子 View, 你最大的尺寸就这么多，不能超过这个值, 
          //具体大小，你自己根据自身情况决定最终大小。一般当我们继承 View 基类进行自定义 View  的时候
          //需要在这种情况下计算给定 View 一个尺寸，否则当使用自定义的 View 的时候，使用 
          // android:layout_width="wrap_content" 属性就会失效
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    // 父容器为 AT_MOST 最大测量模式
    case MeasureSpec.AT_MOST:
           // 子 View 期望尺寸为一个大于 0的具体值，对应着 xml 文件中给定了 View 的具体尺寸大小
        //如 android:layout_width="100dp"
        if (childDimension >= 0) {
           //那么子 View 尺寸为期望固定值尺寸，为精准测量模式 EXACTLY
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
          //如果子 View 期望尺寸为 MATCH_PARENT 最大测量模式
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
             //子 View 尺寸为 size，测量模式为 AT_MOST  最大测量模式
          //即如果子 View 是 Match_parent,那么父 View 就会告诉子 View, 
          //你的尺寸最大为 size 这么大（父容器尺寸减掉已经被用掉的尺寸，即父容器剩余未占用尺寸），
          //你最多有父 View的 size 这么大，不能超过这个尺寸，至于具体多大，你自己根据自身情况决定。
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
             //同上
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    // 父容器为 UNSPECIFIED 模式
    case MeasureSpec.UNSPECIFIED:
           // 子 View 期望尺寸为一个大于 0的具体值
        if (childDimension >= 0) {
             //那么子 View 尺寸为期望值固定尺寸，为精准测量模式 EXACTLY
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
           //如果子 View 期望尺寸为 MATCH_PARENT 最大测量模式
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
              //子 View 尺寸为 0，测量模式为 UNSPECIFIED
           // 父容器不对 View 有任何的限制，要多大给多大
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
           //如果子 View 期望尺寸为 WRAP_CONTENT ，包裹内容
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
             //子 View 尺寸为 0，测量模式为 UNSPECIFIED
             // 父容器不对 View 有任何的限制，要多大给多大
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}

```
以上代码主要作用就是根据父容器的 MeasureSpec 和 view 本身的 LayoutParams 来确定子元素的 MeasureSpec 的整个过程，这个过程清楚的展示了普通 view 的 MeasureSpec 的创建规则，整理一下可得到如下表格（来源艺术探索截图）：

|  childLauyoutParams/parentSpecMode   | EXACTLY  | AT_MOST|  UNSPECIFIED  |
|  ----  | ----  | ----  | ----  |
| dp/px  | EXACTLY/childSize |EXACTLY/childSize |  UNSPECIFIED/childSize  |
| match_parent  | EXACTLY/parentSize | AT_MOST/parentSize|  UNSPECIFIED/0  |
| wrap_parent  | AT_MOST/parentSize | AT_MOST/parentSize|  UNSPECIFIED/0  |





```java

    * @see #onMeasure(int, int)
     */
    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int oWidth  = insets.left + insets.right;
            int oHeight = insets.top  + insets.bottom;
            widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
            heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }

        // Suppress sign extension for the low bytes
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
        if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);

        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;

        // Optimize layout by avoiding an extra EXACTLY pass when the view is
        // already measured as the correct size. In API 23 and below, this
        // extra pass is required to make LinearLayout re-distribute weight.
        .....
        if (forceLayout || needsLayout) {
            // first clears the measured dimension flag
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

            resolveRtlPropertiesIfNeeded();

            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }

           .....
        }

        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;

        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }

```
在``` view measure ``` 里直接调用的是子View 重写的onMeasure() 方法， 这里的我们来看FrameLayout 的onMeasure()。

```
```