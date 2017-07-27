---
title: Android 动画
date: 2017-07-27 09:07:30
categories: [Android, Animation]
tags: [Animator, 属性动画, 自定义动画]
---
在日常的Android开发过程中，或多或少的会接触到动画效果，这些都得益于Android系统为开发者提供了一套丰富的动画APIs。接下来是对Android动画的一些理解，分析和总结。
### Android 动画分类
Android提供了非常丰富的APIs来实现View的动画效果。主要划分如下类别：
* **Property Animation** Android 3.0（API 11）才引入，这种动画可以设置给任何Object，包括那些还没有渲染到屏幕上的对象。这种动画是可扩展的，可以让你自定义任何类型和属性的动画。
* **View Animation** 视图动画在Android系统一开始就存在了， 它只能运用在View上。
* **Drawable Animation**  帧动画，专门用来一个一个的显示Drawable的resources，就像放幻灯片一样。

如果想对以上Android类别有详细了解可以直接查看Andoid 官方文档 [Animation and Graphics](https://developer.android.google.cn/guide/topics/graphics/overview.html)。

### View Animation（视图动画）
视图动画，也叫补间动画，它是使View在试图容器内实现不同的变换(位移，缩放，渐变，旋转)。
补间动画的实现，一般会采用xml 文件的形式；代码会更容易书写和阅读，同时也更容易复用。也可以动Java代码实现。

#### XML实现
动画文件需要放在res/anim/文件夹下面。

新建xml文件，配置animation类型以及相关属性。单个xml可以由单一类型的animation也可以是一个Animation的集合(set)组成。
```java
<?xml version="1.0" encoding="utf-8"?>
// 动画集合：设置插值器，是否共享插值器等设置
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:interpolator="@[package:]anim/interpolator_resource"
    android:shareInterpolator=["true" | "false"] >
    // 颜色渐变，从一个Alpha到另一个Alpha值
    <alpha
        android:fromAlpha="float"  // 动画开始的透明度（0.0到1.0，0.0是全透明，1.0是不透明）
        android:toAlpha="float" /> //	动画结束的透明度
       
    //缩放动画   
    <scale
        android:fromXScale="float" // 开始的X轴缩放比
        android:toXScale="float"   // 开始的Y轴缩放比
        android:fromYScale="float" // 结束的X轴缩放比
        android:toYScale="float"   // 结束的X轴缩放比
        android:pivotX="float"     // 中心点X坐标
        android:pivotY="float" />  // 中心点Y坐标
    
    // 位移动画
    <translate
        android:fromXDelta="float"  // 起始点X坐标
        android:toXDelta="float"    // 起始点Y坐标
        android:fromYDelta="float"  // 结束点X坐标
        android:toYDelta="float" /> // 结束点Y坐标

    // 旋转动画
    <rotate
        android:fromDegrees="float" // 旋转开始角度
        android:toDegrees="float"   // 旋转结束角度
        android:pivotX="float"      // 旋转中心点X坐标
        android:pivotY="float" />   // 旋转中心点Y坐标
   
    // 动画集合
    <set>
        ...
    </set>
</set>

```

** Interpolator **
> Interpolator 被用来修饰动画效果，定义动画的变化率，可以使存在的动画效果accelerated(加速)，decelerated(减速),repeated(重复),bounced(弹跳)等。

Android 系统为开发者已经提供了如下的 Interpolator：

  1. AccelerateInterpolator  在动画开始的地方速率改变比较慢，然后开始加速
  2. AnticipateInterpolator 开始的时候向后然后向前甩
  3.  AnticipateOvershootInterpolator 开始的时候向后然后向前甩一定值后返回最后的值。
  4.  BounceInterpolator   动画结束的时候弹起
  5.  CycleInterpolator 动画循环播放特定的次数，速率改变沿着正弦曲线
  6.   DecelerateInterpolator 在动画开始的地方快然后慢
  7.   LinearInterpolator   以常量速率改变
  8.  OvershootInterpolator    向前甩一定值后再回到原来位置
  9.  AccelerateDecelerateInterpolator，动画始末速率较慢，中间加速

如果以上Interpolator无法满足需求，是可以自定义Interpolator的，这个后面还会提到。

** pivot **
> pivot 决定了当前动画执行的参考位置

pivot 在scale 和rotate动画中用作设置缩放和旋转的中心点，它可以为具体的数值（10）, 也可用百分比（50%），同时也可以带相对位置(50%p).
以pivotX为例:
50 ： 距离动画所在view自身左边缘50像素
50% ： 距离动画所在view自身左边缘 的距离是整个view宽度的50%
50%p ： 距离动画所在view自身左边缘 的距离是整个view宽度的50%
** 注意%号带p和不带的区别在于相对谁的位置 **。

#### Java code 实现

1. 从xml中加载动画
```java
// 通过AnimationUtils加载xml中定义的动画
AlphaAnimation alphaAnimation = (AlphaAnimation) AnimationUtils.loadAnimation(this, R.anim.alpha_anim);
alphaAnimation.setRepeatMode(Animation.INFINITE);
alphaAnimation.start();
```
2. 新建Animation动画
```java
 ScaleAnimation scaleAnimation = new ScaleAnimation(0.0f, 1.0f, 0.0f, 1.0f, Animation.RELATIVE_TO_SELF, Animation.RELATIVE_TO_SELF );
scaleAnimation.setDuration(1000);
// 设置动画重复播放模式(INFINITE 循环)
scaleAnimation.setRepeatMode(Animation.INFINITE);
// -1 代码无限循环
scaleAnimation.setRepeatCount(-1);
// 动画结束后停留在最后一帧
scaleAnimation.setFillAfter(true);
scaleAnimation.start();
```
####  视图动画相关的类的关系
| Java类名	      |   xml关键字	           |描述信息|
| :-------------: |:-------------:| :-----:|
|AlphaAnimation	|alpha| 放置在res/anim/目录下	渐变透明度动画效果|  
|RotateAnimation|	rotate| 放置在res/anim/目录下	画面转移旋转动画效果|
|ScaleAnimation	|scale |放置在res/anim/目录下	渐变尺寸伸缩动画效果|
|TranslateAnimation	|translate|放置在res/anim/目录下	画面转换位置移动动画效果|
|AnimationSet	|set| 放置在res/anim/目录下	一个持有其它动画元素alpha、scale、translate、rotate或者其它set元素的容器|

> 需要注意的是Tween动画有个特点就是虽然View确实发生了变化，但是View的实际位置却一直没有改变，比如说View从A点移动到B点（设置了setFillAfter(true)），View看起来在B点，但是View的点击响应区域还在原来位置，并不在B点，这点要注意，

### Drawable帧动画
Drawable动画其实就是Frame动画（帧动画），它允许你实现像播放幻灯片一样的效果，这种动画的实质其实是Drawable，所以这种动画的XML定义方式文件一般放在res/drawable/目录下。
```xml
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="true">
    <item android:drawable="@drawable/rocket_thrust1" android:duration="200" />
    <item android:drawable="@drawable/rocket_thrust2" android:duration="200" />
    <item android:drawable="@drawable/rocket_thrust3" android:duration="200" />
</animation-list>
```
* android:oneshot true代表只执行一次, 最后停在最后一帧，false循环执行。
*  item： 对应的动画的一帧，动画是从前到后逐一播放。
*  此动画可以应用为View的背景。

在Java代码中使用:
```java
 ImageView rocketImage = (ImageView) findViewById(R.id.rocket_image);
 // 作为ImageView的背景
  rocketImage.setBackgroundResource(R.drawable.rocket_thrust);
  // 获取AnimationDrawable对象
  rocketAnimation = (AnimationDrawable) rocketImage.getBackground();

  // 该方法不能在Activity onCreate() 方法中调用，因为AnimationDrawable还没有Attach到Window上，可以在onWindowFocusChanged() 方法中调用。
  rocketAnimation.start();
```

### Property Animation（属性动画）
属性动画，顾名思义就是通过修改对象的属性来带到动画的效果。Android 3.0（API 11）以后引入了属性动画，如果想要在11之前使用属性动画，可以使用第三方库 [NineOldAndroids](https://github.com/JakeWharton/NineOldAndroids/)。

#### 属性动画概述
```java
具体先看下类关系：

/**
 * This is the superclass for classes which provide basic support for animations which can be
 * started, ended, and have <code>AnimatorListeners</code> added to them.
 */
public abstract class Animator implements Cloneable {
    ......
}
```

所有的属性动画的抽象基类就是它。我们看下他的实现子类：
* ValueAnimator  ** animator>** 放置在res/animator/目录下
* ObjectAnimator **objectAnimator**` 放置在res/animator/目录下
* AnimatorSet 	**set** 放置在res/animator/目录下

#####  ValueAnimator
ValueAnimator是整个属性动画机制当中最核心的一个类, 属性动画的运行机制是通过不断地对值进行操作来实现的，而初始值和结束值之间的动画过渡就是由ValueAnimator这个类来负责计算的。
```java
ValueAnimator anim = ValueAnimator.ofFloat(0f, 1f);
anim.setDuration(300);
anim.start();
```
调用ValueAnimator的ofFloat()方法就可以构建出一个ValueAnimator的实例，ofFloat()方法当中允许传入多个float类型的参数，这里传入0和1就表示将值从0平滑过渡到1，然后调用ValueAnimator的setDuration()方法来设置动画运行的时长，最后调用start()方法启动动画。


##### ObjectAnimator 
相比于ValueAnimator，ObjectAnimator可能才是我们最常接触到的类，因为ValueAnimator只不过是对值进行了一个平滑的动画过渡，但我们实际使用到这种功能的场景好像并不多。而ObjectAnimator则就不同了，它是可以直接对任意对象的任意属性进行动画操作的，比如说View的alpha属性。

```java
ObjectAnimator animator = ObjectAnimator.ofFloat(textview, "alpha", 1f, 0f, 1f);
animator.setDuration(5000);
animator.start();
```

##### AnimatorSet 
独立的动画能够实现的视觉效果毕竟是相当有限的，因此将多个动画组合到一起播放就显得尤为重要。
实现组合动画功能主要需要借助AnimatorSet这个类，这个类提供了一个play()方法，如果我们向这个方法中传入一个Animator对象(ValueAnimator或ObjectAnimator)将会返回一个AnimatorSet.Builder的实例，AnimatorSet.Builder中包括以下四个方法：

* after(Animator anim)   将现有动画插入到传入的动画之后执行
* after(long delay)   将现有动画延迟指定毫秒后执行
* before(Animator anim)   将现有动画插入到传入的动画之前执行
* with(Animator anim)   将现有动画和传入的动画同时执行

好的，有了这四个方法，我们就可以完成组合动画的逻辑了，那么比如说我们想要让TextView先从屏幕外移动进屏幕，然后开始旋转360度，旋转的同时进行淡入淡出操作，就可以这样写：
```java
ObjectAnimator moveIn = ObjectAnimator.ofFloat(textview, "translationX", -500f, 0f);
ObjectAnimator rotate = ObjectAnimator.ofFloat(textview, "rotation", 0f, 360f);
ObjectAnimator fadeInOut = ObjectAnimator.ofFloat(textview, "alpha", 1f, 0f, 1f);
AnimatorSet animSet = new AnimatorSet();
animSet.play(rotate).with(fadeInOut).after(moveIn);
animSet.setDuration(5000);
animSet.start()
```
可以看到，这里我们先是把三个动画的对象全部创建出来，然后new出一个AnimatorSet对象之后将这三个动画对象进行播放排序，让旋转和淡入淡出动画同时进行，并把它们插入到了平移动画的后面，最后是设置动画时长以及启动动画。

ViewPropertyAnimator
有时候需要在一个动画里面同时修改几个属性的时候可以使用ViewPropertyAnimator

```java
PropertyValuesHolder pvhX = PropertyValuesHolder.ofFloat("x", 50f);
PropertyValuesHolder pvhY = PropertyValuesHolder.ofFloat("y", 100f);
ObjectAnimator.ofPropertyValuesHolder(myView, pvhX, pvyY).start();
```

以上动画同时修改属性x，y ，如果需要再同时修改属性z，只要再加一个ViewPropertyAnimator即可。

#### XML中使用Animator
首先属性动画需要放在res/animator/文件夹下面，它和普通的View动画存放不同文件夹里面。
```java
<set
  android:ordering=["together" | "sequentially"]> // 动画播放顺序

    <objectAnimator // 代表类别ObjectAnimator
        android:propertyName="string" // 属性名称
        android:duration="int"        // 动画播放时间
        android:valueFrom="float | int | color" // 起始属性
        android:valueTo="float | int | color"   // 结束属性属性
        android:startOffset="int"  // 动画延迟时间(milliseconds)
        android:repeatCount="int" // 重复次数， -1 代表无限循环
        android:repeatMode=["repeat" | "reverse"] // 重复方式
        android:valueType=["intType" | "floatType"]/> // 属性值的类型，默认为floatType

    <animator // 代表类别ValueAnimator， 其他属性同上
        android:duration="int"
        android:valueFrom="float | int | color"
        android:valueTo="float | int | color"
        android:startOffset="int"
        android:repeatCount="int"
        android:repeatMode=["repeat" | "reverse"]
        android:valueType=["intType" | "floatType"]/>

    <set> // 代表类别为AnimatorSet.
        ...
    </set>
</set>
```
更多关于属性的解释，[参考](https://developer.android.google.cn/guide/topics/resources/animation-resource.html#Property)

在java code中使用上述动画:
```java
// Load ValueAnimator
ValueAnimator xmlAnimator = (ValueAnimator) AnimatorInflater.loadAnimator(this,
        R.animator.animator);
xmlAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
    @Override
    public void onAnimationUpdate(ValueAnimator updatedAnimation) {
        float animatedValue = (float)updatedAnimation.getAnimatedValue();
        textView.setTranslationX(animatedValue);
    }
});

xmlAnimator.start();

// Load ObjectAnimatorSet
AnimatorSet set = (AnimatorSet) AnimatorInflater.loadAnimator(myContext,
    R.anim.property_animator);
set.setTarget(myObject);
set.start();
```

####  View 常用的动画属性值
* translationX, translationY
* rotation, rotationX, and rotationY
* scaleX and scaleY
* pivotX and pivotY
* x and y
* alpha

#### Interpolator和TypeEvaluator
ValueAnimator怎样实现从初始值平滑过渡到结束值的呢？这个就是由TypeEvaluator 和TimeInterpolator 共同决定的。
** TimeInterpolator **
> TimeInterpolator 中文意思是时间插值器。它的作用是根据时间的流逝的百分比计算出当前属性值改变的百分比，它反映的是属性变化的速率。Android内置了几个插值器，上面有提到过。

** TypeEvaluator**
> TypeEvaluator 翻译成中文为类型估值器。它的作用是根据当前属性变化的百分比计算当前的属性值，系统内置的估值器有：IntEvaluator， FloatEvluator，ArgbEvluator。

系统提供了几个插值器和插值器，如果在实际项目中无法满足需求，可以自定义。自定义插值器需要实现TimeInterpolator 或Interpolator，如果自定义估值器，需要实现TypeEvaluator。

#### 自定义插值器
```java
// BaseInterpolator 继承Interpolator， Interpolator继承TimeInterpolator
public class AccelerateDecelerateInterpolator extends BaseInterpolator
        implements NativeInterpolatorFactory {
    public AccelerateDecelerateInterpolator() {
    }
...
   
   // 获取插值结果，
    public float getInterpolation(float input) {
        return (float)(Math.cos((input + 1) * Math.PI) / 2.0f) + 0.5f;
    }

    ...
    }
```
以上是一个先加速后减速的插值器，我们主要是要实现它的getInterpolation()方法，它为什么要要这样实现呢？
首先先加速后减速的插值器，它可以用一个图来解释：
![](http://s3.51cto.com/wyfs02/M00/2F/4D/wKioL1OfBXviFOI8AACS1BUkSAc421.jpg)

这张图反映了时间t和插值之间的关系，t代表时间，纵坐标代表动画完成的百分比。这张图刚开始随着时间推移插值变化很快，时间临近结尾的是插值变化不大，这不正是符合先加速后减速的效果。
所以根据这个曲线公式  ** 公式: y=cos((t+1)π)/2+0.5 **
实现上述公式就是可以实现getInterpolation()方法了。
其他查找器描述图，可以参考：[More on Interpolators](http://cogitolearning.co.uk/?p=1078)

所以要自定义插值器之前，先要构思一个上面的类似的图，然后找到其对应的数学公式。最好就是将公式在getInterpolation()方法中实现了。

#### 自定义估值器 

拿FloatEvaluator源码看看:  

```java 
public class FloatEvaluator implements TypeEvaluator<Number> {

    // fraction 为完成百分比 0.0f-1.0f的范围
    // startValue，起始值，类型有由泛型定义
    // endValue： 结束值
    public Float evaluate(float fraction, Number startValue, Number endValue) {
        float startFloat = startValue.floatValue();
        return startFloat + fraction * (endValue.floatValue() - startFloat);
    }
}

```

以上代码 可以看出来，获取要重写估值器，需要实现TypeEvaluator，并且实现它的evaluate()方法。
例如可以使用贝塞尔曲线，就拿二阶为例：
![](https://pic1.zhimg.com/87a5997cdd29dcf7620f11b53015fcc8_b.jpg)

首先定义个JaveBean来记录需要的点，例如起点，控制点，结束点
```java

public class BezierPoint {
    public final float mX;
    public final float mY;

    public float mCtrX;
    public float mCtrY;

    public BezierPoint(float x, float y) {
        mX = x;
        mY = y;
    }
}
```
自定义TypeEvaluator，指定类型为BezierPoint 
```java
public class BezierEvluate implements TypeEvaluator<BezierPoint> {

    @Override
    public BezierPoint evaluate(float t, BezierPoint startValue, BezierPoint endValue) {
        // 获取控制点坐标
        float ctrX = endValue.mCtrX;
        float ctrY = endValue.mCtrY;

        // 取反
        float minusT = 1 - t;

        // 根据二阶贝塞尔曲线公式计算
        float x = minusT * minusT * startValue.mX + 2 * t * minusT * ctrX + t * t * endValue.mX;
        float y = minusT * minusT * startValue.mY + 2 * t * minusT * ctrY + t * t * endValue.mY;

        return new BezierPoint(x, y);
    }
}
```
通过以上自定义插值器，可以按照贝塞尔曲线算法获取对应的点坐标，如果将该插值器运用到某一View上，该View就可以按照该曲线路径运动了。

#### 对任意属性做动画
ObjectAnimator对任意属性都可以加动画，但是必须满足一定条件：
* Object 必须给属性提供set方法，比如属性abc，必须有setAbc方法。
* Object的属性值发生变化必须通过某种方式反映出变化，比如带来UI的改变，类似发生位移，缩放等。

```java
 private void animation() {
        ViewWappper wappper = new ViewWappper(mPullScrollView);
        ObjectAnimator objectAnimator = ObjectAnimator.ofInt(wappper, "abc", 100);
        objectAnimator.setDuration(5000).start();
    }
    
public static class ViewWappper {

        private int abc;
        private View mTarget;

        public ViewWappper(View view) {
            mTarget = view;
        }

        public void setAbc(int value) {
            mTarget.setTranslationX(value);
        }
    }
```

### 动画几个特殊使用场景

#### LayoutAnimation
LayoutAnimation 作用于ViewGroup，给一个指定一个动画，它的子View都具有同样的动画效果。用过“OFO” App的应该知道，其用户界面点开时，所有子View依次执行一个从右往左的动画，应该就是用LayoutAnimation实现的。
```java
<layoutAnimation xmlns:android="http://schemas.android.com/apk/res/android"
        android:delay="30%" // 表示动画播放的延时，既可以是百分比，也可以是float小数 
        android:animationOrder="random" //表示动画的播放顺序，有三个取值normal(顺序)、reverse(反序)、random(随机)。
        android:animation="@anim/slide_right" /> // 子View动画
```
将layout-animation应用到ViewGroup中 **  android:layoutAnimation="@anim/list_anim_layout"** 这样在加载布局的时候就会自动播放layout-animtion。


#### Activity 切换效果
Activity有默认的切换效果，它是通过设置主题实现，同时也可以实现自定义。主要用到是overridePendingTransition这个方法实现的。这个方法必须在startActivity()和finish() 之后调用才有效。

```java
startActivity(new Intent(this, ProfileActivity.class));
overridePendingTransition(R.anim.push_right_in, R.anim.fade_out);

private void finishActivity() {
        finish();
        overridePendingTransition(R.anim.fade_in, R.anim.push_right_out);
   }
   
```

#### 颜色估值方法
在实际项目组有可能用到颜色渐变效果，可以查看ArgbEvaluator的evaluate方法，拿到渐变颜色.
```java
public Object evaluate(float fraction, Object startValue, Object endValue) {
        int startInt = (Integer) startValue;
        int startA = (startInt >> 24) & 0xff;
        int startR = (startInt >> 16) & 0xff;
        int startG = (startInt >> 8) & 0xff;
        int startB = startInt & 0xff;

        int endInt = (Integer) endValue;
        int endA = (endInt >> 24) & 0xff;
        int endR = (endInt >> 16) & 0xff;
        int endG = (endInt >> 8) & 0xff;
        int endB = endInt & 0xff;

        return (int)((startA + (int)(fraction * (endA - startA))) << 24) |
                (int)((startR + (int)(fraction * (endR - startR))) << 16) |
                (int)((startG + (int)(fraction * (endG - startG))) << 8) |
                (int)((startB + (int)(fraction * (endB - startB))));
    }
```


### 使用动画注意事项
* OOM, 在使用帧动画是避免使用大图片
* 内存泄漏，特别是循环播放的动画，需要在Activity退出时停止动画。
* 兼容性问题，属性动画在3.0以下不支持，需要做适配。
* 动画元素的交互，前面提到过View动画可见位置发生改变但是单击有效位置没法发生改变。
* 硬件加速，建议开启硬件加速，提高动画流畅性。

### 结尾
Android动画分析到此为止，文中如有出错，请多多指正。

参考
[Android Animations Tutorial](http://cogitolearning.co.uk/?p=877)
[官方文档](https://developer.android.google.cn/guide/topics/graphics/overview.html)
[Android 动画总结](http://www.jianshu.com/p/420629118c10)