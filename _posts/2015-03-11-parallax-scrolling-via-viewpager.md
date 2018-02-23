---
layout: post
title: ViewPager实现APP引导页视差滚动(Parallax Scrolling)效果
tags: [Android, ViewPager, 引导页, 视差滚动]
---

## 1. 什么是视差滚动？
&emsp;&emsp;Parallax Scrolling(视差滚动)，是一种常见的动画效果。视差一词来源于天文学，但在日常生活中也有它的身影。在疾驰的动车上看风景时，会发现越是离得近的，相对运动速度越快，而远处的山川河流只是缓慢的移动着，这就是最常见的视差效果。视差动画独有的层次感能带来极为逼真的视觉体验，iOS、Android Launcher、Website都将视差动画作为提升用户视觉愉悦度的不二选择。
视差滚动效果一般是指，ViewPager滑动时其中的内容元素滚动速率不一，形成的元素动画错位效果。APP第一次打开出现引导页也不是什么新鲜的事儿，ViewPager配上几张设计师精心绘制的图片，分分钟即可了事。但是总有人把平凡的事情做到不平凡，现在市场上的众多应用里都出现了视差动画的身影，随着用户手指的滑动，反馈以灵动、贴近真实的视觉以及操作体验，对应用的初始印象登时被提升到一个极高的点。
## 2. 视差滚动效果是怎样的？
![这里写图片描述](http://img.blog.csdn.net/20150313164137974)
## 3. 如何实现视差滚动效果？
### 1). 通过使用ViewPager实现页面的左右滑动
&emsp;&emsp;相信大部分Android开发者对ViewPager控件并不陌生，它可以实现简单的页面左右滑动效果，包含在android-support-v4.jar兼容包中。其实使用ViewPager也可以实现简单的滑动引导页，但想要做到有视差滚动效果的话，可能要深挖一下ViewPager的API了。
### 2). ViewPager有办法实现ParallaxScrolling吗？
&emsp;&emsp;视差滚动效果，主要表现为内容元素滚动速率的差异上。比如在ViewPager中滑动了1px，而A元素移动2px，B元素移动1.5px，这种移动差距的比率，或称之为parallaxCofficient，即视差系数或者视差速率，正是同一个界面中的元素，由于层级不同，赋予的视差系数不同，在移动速度上的差异形成了视差的错觉，这就是我们要追求的效果。

&emsp;&emsp;知道原理就好办了，使用ViewPager.OnPageChangeListener监听接口，实现onPageScrolled、onPageSelected、onPageScrollStateChanged三个方法，然后动态计算不就得了。
NO！如果我们在这三个方法里面动态计算视差系数，又如何使这些值作用于页面的每个子View上面呢？难道要为每个子View设置一些translate动画？那什么时候该是第一个子View的动画开始呢？那么第二个呢？等等，这个方法明显不靠谱！那有什么更好的解决方案呢？
所幸Android3.0以后为ViewPager准备好了变形接口ViewPager.PageTransformer，它提供了一个方法transformPage(View page, float position)，正是为我们完成视差动画量身定制的。让我们来看看它的源码：
```java
/**
  * A PageTransformer is invoked whenever a visible/attached page is scrolled.
  * This offers an opportunity for the application to apply a custom transformation
  * to the page views using animation properties.
  *
  * <p>As property animation is only supported as of Android 3.0 and forward,
  * setting a PageTransformer on a ViewPager on earlier platform versions will
  * be ignored.</p>
  */
 public interface PageTransformer {
     /**
      * Apply a property transformation to the given page.
      *
      * @param page Apply the transformation to this page
      * @param position Position of page relative to the current front-and-center
      *                 position of the pager. 0 is front and center. 1 is one full
      *                 page position to the right, and -1 is one page position to the left.
      */
     public void transformPage(View page, float position);
 }
```
&emsp;&emsp;从源码中不难看到，PageTransformer在ViewPager滑动时被触发，它为我们自定义页面中进行视图变换打开了一扇大门。PagerTransformer的注释已经说明了它的用处——它给应用提供了一个为页面滑动设置自定义动画属性的机会。That's it！
在ViewPager源码中，我们可以很直观的看到它的调用过程：
```java
// ViewPager#onPageScrolled
if (mPageTransformer != null) {
    final int scrollX = getScrollX();
    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View child = getChildAt(i);
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();

        if (lp.isDecor) continue;

        final float transformPos = (float) (child.getLeft() - scrollX) / getClientWidth();
        mPageTransformer.transformPage(child, transformPos);
    }
}
```
### 3). ViewPager.PageTransformer如何使用？
&emsp;&emsp;首先，我们要弄懂transformPage方法的两个参数的含义。page当然指的就是滑动中的那个view，position这里定义为float，不是平时理解的int位置信息，而是当前滑动状态的一个表示，比如当滑动到正全屏时，position是0，而向左滑动，使得右边刚好有一部被进入屏幕时，position是1，如果前一夜和下一页基本各在屏幕占一半时，前一页的position是-0.5，后一页的posiotn是0.5，所以根据position的值我们就可以自行设置需要的alpha，x/y信息。

**Param 1: View page**

&emsp;&emsp;从上面的代码中，不难看出，page就是当前被滑动的页面，调试得知，每一个child view被NoSaveStateFrameLayout包装，也就是说page.getChildAt(0)即是每个page实际的child view。

**Param 2: float position**

&emsp;&emsp;position这个参数不看代码或者文档，总会误以为就是我们熟知的integer position，不过它实际上是滑动页面的一个相对比例，本质跟 1、2、3、4 这种position是一样的。
比如知乎启动页共有6个页面，分别是A，B，C，D，E，F初始状态也就是A页面静止时，A页面的position正好是0，B页面是1。而后滑动页面（B -> A），在这个过程中A的position是间于[-1, 0]，B页面则是间于[0, 1]。不过这个参数的文档却是简单不够直观，对照上面的例子，现在应该很清晰了。
根据上面的分析，我们可以得出一个相对简单的自定义的transformer，对page（view）进行遍历，递增或者递减其parallaxCofficient，以得到我们预期的效果，具体的系数设置请参考代码。
```java
class ParallaxTransformer implements ViewPager.PageTransformer {

    float parallaxCoefficient;
    float distanceCoefficient;

    public ParallaxTransformer(float parallaxCoefficient, float distanceCoefficient) {
        this.parallaxCoefficient = parallaxCoefficient;
        this.distanceCoefficient = distanceCoefficient;
    }

    @Override
    public void transformPage(View page, float position) {
        float scrollXOffset = page.getWidth() * parallaxCoefficient;

        // ...
        // layer is the id collection of views in this page
        for (int id : layer) {
            View view = page.findViewById(id);
            if (view != null) {
                view.setTranslationX(scrollXOffset * position);
            }
            scrollXOffset *= distanceCoefficient;
        }
    }
}
```
在ViewPager初始化的时候设置一下，便可以了。
```java
viewPager = (ViewPager) findViewById(R.id.pager);
viewPager.setAdapter(pagerAdapter);
viewPager.setPageTransformer(true, new ParallaxTransformer());
```
## 4. 由视差滚动效果引起的思考
&emsp;&emsp;通过上面的探索，其实我个人觉得ViewPager.PageTransformer上面大有可为，而不仅仅是可以做到视差滚动效果。比如我们做视差动画的时候是对整个页面View的child view设置一些视差系数以达到滚动速率差异的错位效果，然而我们其实可以仅仅对整个页面View设置一些过场动画，以达到生动的过渡效果。下面是一个页面放大的过渡效果实现，欢迎大家提出更多拓展方案。
```java
public class ZoomOutPageTransformer implements ViewPager.PageTransformer {
    private static final float MIN_SCALE = 0.85f;
    private static final float MIN_ALPHA = 0.5f;

    @SuppressLint("NewApi")
	public void transformPage(View view, float position) {
        int pageWidth = view.getWidth();
        int pageHeight = view.getHeight();

        if (position < -1) { // [-Infinity,-1)
            // This page is way off-screen to the left.
            view.setAlpha(0);

        } else if (position <= 1) { // [-1,1]
            // Modify the default slide transition to shrink the page as well
            float scaleFactor = Math.max(MIN_SCALE, 1 - Math.abs(position));
            float vertMargin = pageHeight * (1 - scaleFactor) / 2;
            float horzMargin = pageWidth * (1 - scaleFactor) / 2;
            if (position < 0) {
                view.setTranslationX(horzMargin - vertMargin / 2);
            } else {
                view.setTranslationX(-horzMargin + vertMargin / 2);
            }

            // Scale the page down (between MIN_SCALE and 1)
            view.setScaleX(scaleFactor);
            view.setScaleY(scaleFactor);

            // Fade the page relative to its size.
            view.setAlpha(MIN_ALPHA +
                    (scaleFactor - MIN_SCALE) /
                            (1 - MIN_SCALE) * (1 - MIN_ALPHA));

        } else { // (1,+Infinity]
            // This page is way off-screen to the right.
            view.setAlpha(0);
        }
    }
}
```
