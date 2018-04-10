> 作者：林冠宏 / 指尖下的幽灵

> 掘金：https://juejin.im/user/587f0dfe128fe100570ce2d8

> 博客：http://www.cnblogs.com/linguanh/

> GitHub ： https://github.com/af913337456/

> 腾讯云专栏：  https://cloud.tencent.com/developer/user/1148436/activities

---

### 首先是-- Android SDK 自带的 `BottomSheetDialog`
下面的 gif 图是一个Android SDK 自带的 `BottomSheetDialog` 内部加了 `RecyclerView` 列表控件的效果

![](https://user-gold-cdn.xitu.io/2018/4/10/162ad713f3944011?w=720&h=1280&f=gif&s=1602223)

可以看出：
* 下滑动作会收起，隐藏掉 `dialog`
* 上滑会完全展开
* 展开后，才能滑动 `RecyclerView` 内部

其次
* 如果你内部使用的是 `ListView` 列表控件，你会发现会有其他奇怪的情况。

### 然后是--网易云音乐 的 `BottomSheetDialog`
下面的 gif 图是一个Android 版 `网易云音乐`的`BottomSheetDialog`效果
![](https://user-gold-cdn.xitu.io/2018/4/10/162ad896c58dee0f?w=482&h=850&f=gif&s=5148050)

可以看出：
* 下滑动作会有`范围`回弹，也就是下滑到一定距离才会收起，隐藏掉 `dialog`
* 上滑不给展开
* 能够在半展开的情况下，内嵌滑动列表控件，例如 `listView`
* 和列表控件滑动不冲突，在`列表控件`滑尽的时候，可以下滑隐藏`dialog`

### 最后是--我开源 的仿网易云音乐 `BottomSheetDialog`
![](https://user-gold-cdn.xitu.io/2018/4/10/162ad73525847cce?w=720&h=1280&f=gif&s=2398240)

可以看出，效果和`网易云的一样`

## 核心代码简述

SDK 的 BottomSheetDialog 内部布局的结构如下：
```java
--FrameLayout
--|--CoordinatorLayout
--|--|--FrameLayout
--|--|--|--Our ContentView // 最后是我们设置的 ContentView

```
CoordinatorLayout 在 `Action_Move` 事件时，必要的时候对其子 View 进行事件拦截，所以有第一个 gif 看到的效果，具体不详说。

#### 第一个步骤 --- 防止 `CoordinatorLayout` 对 `Our ContentView` 拦截事件

这里使用 ListView 做例子，设置`onTouch`，在内部做适当时候的适当阻止`CoordinatorLayout` 拦截事件。

```java
// ListView
setOnTouchListener(
    new OnTouchListener() {
        @SuppressLint("ClickableViewAccessibility")
        @Override
        public boolean onTouch(View v, MotionEvent event) {
            if (bottomCoordinator == null)
                return false;
            // 拿出当前列表第一个可见 item 的 pos
            int firstVisiblePos = getFirstVisiblePosition();
            switch (event.getAction()) {
                case MotionEvent.ACTION_DOWN:
                    downY = event.getRawY();
                    bottomCoordinator.requestDisallowInterceptTouchEvent(true);
                    break;
                case MotionEvent.ACTION_MOVE:
                    moveY = event.getRawY();
                    if ((moveY - downY) > 10) {
                        // 下滑情况
                        if (firstVisiblePos == 0 && isOverScroll) {
                            // 列表控件，例如 listView 已经滑到头了，允许被拦截
                            bottomCoordinator.requestDisallowInterceptTouchEvent(false);
                            break;
                        }
                    }
                    // 上滑时，总是不允许被拦截，listView 消耗当前事件
                    bottomCoordinator.requestDisallowInterceptTouchEvent(true);
                    break;
                case MotionEvent.ACTION_UP:
                    break;
            }
            return false;
        }
    }
);
```

### 第二个步骤，让 `ListView` 能在半展开的情况下，显示完整的数据条数

重写 `onMeasure`，使用自定义的测量模式。

```java
// ListView
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    if(bottomCoordinator == null){
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        return;
    }
    // 以黄金分割的尺寸来显示 listView 的高度
    int size = (int)((float)(getResources().getDisplayMetrics().heightPixels*0.618));
    int newHeightSpec = MeasureSpec.makeMeasureSpec(
            size,
            // mode，非法的情况，super 直接使用 size 做高，看源码后，你会发现也可以使用 exact 模式
            Integer.MIN_VALUE
    );
    super.onMeasure(widthMeasureSpec, newHeightSpec);
}
```

### 第三个步骤，实现 `BottomSheetDialog` 范围回弹

```java
/**
 * 添加 top 距离顶部多少的时候触发收缩效果
 * @param targetLimitH int 高度限制
 */
@SuppressWarnings("all")
public void addSpringBackDisLimit(final int targetLimitH){
    if(coordinator == null)
        return;
    // totalHeight 屏幕的总像素高度
    final int totalHeight = getContext().getResources().getDisplayMetrics().heightPixels;
    // currentH 当前我们的 列表控件 展开的高度
    final int currentH = (int) ((float)totalHeight*0.618); // 0.618 是黄金分割点，随便自定义，对应 contentView
    final int leftH    = totalHeight - currentH;
    coordinator.setOnTouchListener(
            new View.OnTouchListener() {
                @Override
                public boolean onTouch(View v, MotionEvent event) {
                    switch (event.getAction()){
                        case MotionEvent.ACTION_MOVE:
                            // 计算相对于屏幕的 坐标
                            bottomSheet.getGlobalVisibleRect(r);
                            break;
                        case MotionEvent.ACTION_UP:
                            // 抬手的时候判断
                            int limitH;
                            if(targetLimitH < 0)
                                limitH = (leftH + currentH/3);
                            else
                                limitH = targetLimitH;
                            if(r.top <= limitH)
                                if (mBehavior != null)
                                    // 范围内，让它继续是 半展开的状态
                                    mBehavior.setState(BottomSheetBehavior.STATE_COLLAPSED);
                            break;
                    }
                    return false;
                }
            }
    );
}
```
