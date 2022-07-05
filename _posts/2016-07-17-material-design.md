---
layout: post
title: 西安GDG上《以开发者的角度再聊Material Design》的总结
tags: Material Design
categories: Android
date: 2016-07-17
---

## 概述
谷歌在2014年I/O大会上推出了Material Design,旨在为手机、平板电脑、台式机和“其他平台”提供更一致、更广泛的“外观和感觉”。在国内有好几种版本的翻译：材料设计/材质设计/质感设计（官方文档）/原质设计（国内设计师更倾向于这个）。

## 三大设计原则

### 隐喻

通过纸墨做比，光影打造空间层次和符合客观规律的特效来隐喻表面质感、光效以及运动感。


### 鲜明、形象、深思熟虑

借鉴了传统的印刷设计，从排版、网格、空间、比例、配色、图像使用上做了精心处理，尤其对色彩、图像、字体、留白明确了规范，力求构建鲜明的用户界面。

### 有意义的动画效果

通过符合客观运动规律的特效效果，让物体的变化以更连续、更平滑的方式呈现给用户，从而吸引用户的注意。比如转场、触摸反馈、循环揭示等

关于界面的层次：
![image](/assets/images/md1.png)

从下往上依次可以分为：

* 界面的内容
* 顶部导航条 App Bar
* 浮动按钮  FAB
* 状态栏&底部虚拟导航键
* 抽屉菜单
* 通知栏等系统信息

## 六大部分
官方文档分别从动画、样式、布局、组件、模式、可用性六个方面对Material Design的设计规范进行了详细阐释。

这里主要列一些5.0以后引入的新特性：

### 1.触控涟漪

![image](/assets/images/md2.gif)

首先在视图 XML 中应用此功能

* ?android:attr/selectableItemBackground 指定有界的波纹。
* ?android:attr/selectableItemBackgroundBorderless 指定越过视图边界的波纹。 它将由一个非空背景的视图的最近父项所绘制和设定边界。注意这个是API21引入的新特性

如果要改变默认触摸反馈颜色，可以在我们自定义的主题下面主题的添加 android:colorControlHighlight 属性。

相关类是RippleDrawable。

在drawable-v21中：

```
<?xml version="1.0" encoding="utf-8"?>
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="@android:color/darker_gray">
    <item>
        <shape android:shape="rectangle">
            <solid android:color="@color/colorAccent" />
        </shape>
    </item>
</ripple>
```

如果要兼容5.0以下版本，我们可以使用第三方开源库[RippleEffect](https://github.com/traex/RippleEffect)

否则可以在drawable中定义selector指定不同的状态即可

### 2. View状态改变动画

StateListAnimator定义当视图的状态改变的时候运行动画，如下代码：

```
<selector xmlns:android="http://schemas.android.com/apk/res/android">
  <item android:state_pressed="true">
    <set>
      <objectAnimator android:propertyName="translationZ"
        android:duration="@android:integer/config_shortAnimTim"
        android:valueTo="2dp"
        android:valueType="floatType"/>
    </set>
  </item>
  <item android:state_enabled="true"
    android:state_pressed="false"
    android:state_focused="true">
    <set>
      <objectAnimator android:propertyName="translationZ"
        android:duration="100"
        android:valueTo="0"
        android:valueType="floatType"/>
    </set>
  </item>
</selector>
```

给视图分配动画使用android:stateListAnimator属性,代码实现可以借助AnimationInflater.loadStateListAnimator()和View.setStateListAnimator()方法。

* 注意当你的主题是继承的Material主题，按钮默认有一个Z轴动画。如果需要避免这个动画，设置android:stateListAnimator属性为@null即可。

与此相似的是animated-selector：android5.0的一些系统组件默认使用这些动画。

```
<animated-selector
    xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- provide a different drawable for each state-->
    <item android:id="@+id/pressed" android:drawable="@drawable/drawable_pressed"
        android:state_pressed="true"/>
    <item android:id="@+id/focused" android:drawable="@drawable/drawable_focused"
        android:state_focused="true"/>
    <item android:id="@id/default"
        android:drawable="@drawable/drawable_default"/>

    <!-- specify a transition -->
    <transition android:fromId="@+id/default" android:toId="@+id/pressed">
        <animation-list>
            <item android:duration="15" android:drawable="@drawable/drawable1"/>
            <item android:duration="15" android:drawable="@drawable/drawable2"/>
            ...
        </animation-list>
    </transition>
    ...
</animated-selector>
```

### 3.循环揭示
它提供视觉上的持续性挡显示或者隐藏一组界面元素。通过ViewAnimationUtils.createCircularReveal()方法可以使用动画效果来揭示或者隐藏一个视图。

![image](/assets/images/md3.gif)

关键代码
揭示一个先前隐藏的视图

```
Animator anim =
    ViewAnimationUtils.createCircularReveal(myView, cx, cy, 0, finalRadius);
myView.setVisibility(View.VISIBLE);
anim.start();
```

隐藏一个先前显示的视图

```
Animator anim =
    ViewAnimationUtils.createCircularReveal(myView, cx, cy, initialRadius, 0);

anim.addListener(new AnimatorListenerAdapter() {
    @Override
    public void onAnimationEnd(Animator animation) {
        super.onAnimationEnd(animation);
        myView.setVisibility(View.INVISIBLE);
    }
});

anim.start();
```

针对5.0以下如果实现，可以借用第三方库：[CircularReveal](https://github.com/ozodrukh/CircularReveal)

### 共享元素转场

android 5.0(api 21)提供以下进入和退出效果：
* explode(分解)  
* slide(滑动)   
* fade(淡入淡出)

接下来猪脚登场,共享元素过渡效果：

* changeBounds - 改变目标视图的布局边界
* changeClipBounds - 裁剪目标视图边界
* changeTransform - 改变目标视图的缩放比例和旋转角度
* changeImageTransform - 改变目标图片的大小和缩放比例

![image](/assets/images/SceneTransition.png)

首先在我们继承的主题上，使用android:windowContentTransitions属性开启窗口内内容过渡效果

* 以共享元素启动一个操作行为

```
ActivityOptionsCompat options = ActivityOptionsCompat.
    makeSceneTransitionAnimation(this, (View)ivAvatar, "avatar");
startActivity(intent, options.toBundle());
```

* 以多个共享元素启动一个操作行为

```
Pair<View, String> p1 = Pair.create((View)ivProfile, "profile");
Pair<View, String> p2 = Pair.create(vPalette, "palette");
Pair<View, String> p3 = Pair.create((View)tvName, "text");
ActivityOptionsCompat options = ActivityOptionsCompat.
    makeSceneTransitionAnimation(this, p1, p2, p3);
startActivity(intent, options.toBundle());
```

### Scrolling Animations

放到后面

### 高度和阴影

![image](/assets/images/md4.png)

Z = elevation + translationZ

主要借助elevation属性

### 着色

利用 Android 5.0（API 级别 21）及更高版本,可为位图以及定义为 Alpha 蒙版的点九图着色，主要借助android:tint 以及 android:tintMode 属性设置您的布局中的着色颜色和模式。

### 裁剪视图

![image](/assets/images/md5.png)

* 扩展 ViewOutlineProvider 类别。
* 重写 getOutline() 方法。
* 利用 View.setOutlineProvider() 方法设置轮廓。

### 矢量图片添加动画


先是xml文件，对应类是VectorDrawable
```
<vector xmlns:android="http://schemas.android.com/apk/res/android"
     android:height="64dp"
     android:width="64dp"
     android:viewportHeight="600"
     android:viewportWidth="600" >
     <group
         android:name="rotationGroup"
         android:pivotX="300.0"
         android:pivotY="300.0"
         android:rotation="45.0" >
         <path
             android:name="v"
             android:fillColor="#000000"
             android:pathData="M300,70 l 0,-70 70,70 0,0 -70,70z" />
     </group>
 </vector>
```

设置drawable，对应类是AnimatedVectorDrawable
```
<animated-vector xmlns:android="http://schemas.android.com/apk/res/android"
   android:drawable="@drawable/vectordrawable" >
     <target
         android:name="rotationGroup"
         android:animation="@anim/rotation" />
     <target
         android:name="v"
         android:animation="@anim/path_morph" />
 </animated-vector>
```

上面是使用了两个属性动画，对应类是ObjectAnimator

其中旋转动画和变形动画：
```
<objectAnimator  
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="6000"  
    android:propertyName="rotation"  
    android:valueFrom="0"  
    android:valueTo="360"/>

<objectAnimator  
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="3000"  
    android:propertyName="pathData"  
    android:valueFrom="M300,70 l 0,-70 70,70 0,0   -70,70z"  
    android:valueTo="M300,70 l 0,-70 70,0  0,140 -70,0 z"  
    android:valueType="pathType"/>   
```

### 从图像萃取颜色

如果要萃取这些颜色，可以通过如下函数：

![image](/assets/images/md6.png)

```
Palette.from(bitmap).generate()
Palette.generateAsync(bitmap,paletteAsyncListener)
```

然后可以通过 Palette.getVibrantColor方法获取色值

## 样式

### 颜色

颜色不宜过多。选取一种主色、一种辅助色（非必需），在此基础上进行明度、饱和度变化，构成配色方案。(https://www.materialpalette.com)

### 图标

建议模仿现实中的折纸效果，通过扁平色彩表现空间和光影。这里也推荐下这个插件：https://github.com/konifar/android-material-design-icon-generator-plugin

### 图片

优先使用图像。然后可以考虑使用插画。

### 文字

英文官方建议Roboto字体，中文推荐Noto(思源黑体)，另外官方也开源了[字体库](http://fonts.google.com/),需翻墙

## 布局

这里列出部分尺寸：

* 所有可操作元素最小点击区域尺寸：48dp X 48dp。
* 栅格系统的最小单位是8dp，一切距离、尺寸都应该是8dp的整数倍。以下是一些常见的尺寸与距离：（推荐网格校正工具：keyline pushing）
* 顶部状态栏高度：24dp
* Appbar最小高度：56dp
* 底部导航栏高度：48dp
* 悬浮按钮尺寸：56x56dp/40x40dp
* 用户头像尺寸：64x64dp/40x40dp
* 小图标点击区域：48x48dp
* 侧边抽屉到屏幕右边的距离：56dp
* 卡片间距：8dp
* 分隔线上下留白：8dp
* 大多元素的留白距离：16dp

推荐国外开发者贡献的一个开源库：https://github.com/DmitryMalkovich/material-design-dimens  里面总结了一些Material Design的设计规范，另外也推荐一款网格校正工具：[keyline pushing](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=0ahUKEwiB17eglZvOAhVJ72MKHdcIADUQFggfMAA&url=https%3A%2F%2Fplay.google.com%2Fstore%2Fapps%2Fdetails%3Fid%3Dcom.faizmalkani.keylines%26hl%3Dzh_CN&usg=AFQjCNFaXp2sbsfOHk8bifBkhuS38VukCw&sig2=pEOMduInSxFH9v2qvjPcYw)

## 组件

这里列出来一些官方组件，可供我们开发Material Design时使用

* RecyclerView
* CardView
* DrawerLayout
* NavigationView
* Toolbar
* FloatingActionButton
* Snackbar
* TextInputLayout
* TextInputEditText
* CoordinatorLayout
* TabLayout
* AppBarLayout
* SwipeRefreshLayout
* CollapsingToolbarLayout
* BottomSheetBehavior
* PercentRelativeLayout
* Palette
* PagerTabStrip
* PagerTitleStrip
* SlidingPanelLayout

### RecyclerView

关于RecyclerView的几点总结：

* RecyclerView.LayoutManager	:负责Item视图的布局的显示管理
* RecyclerView.ItemDecoration	:给每一项Item视图添加子View,如加分割线等
* RecyclerView.ItemAnimator:负责处理数据添加或者删除时候的动画效果
* RecyclerView.Adapter:为每一项Item创建视图
* RecyclerView.ViewHolder:承载Item视图的子布局

与ListView的对比：

* RecyclerView需借助ViewHolder模式来实现Adapter
* RecyclerView可以自定义Item布局：listview只能以垂直线性排列的方式来布局Item,而RecyclerView借助RecyclerView.LayoutManager可以实现多种布局，如网格、瀑布流等
* Item 动画：借助RecyclerView.ItemAnimator很容易实现item动画
* 数据源：listview针对不同的数据源可以有不同的适配器，而RecyclerView需要借助RecyclerView.Adapter自定义实现适配器来提供数据。
* Item Decoration：listview可以通过android:divider很容易实现分割线等，而RecycerView则需要借助RecyclerView.ItemDecoration
* Item Click : listview可以通过AdapterView.OnItemClickListener来实现，而RecyclerView则只提供了RecyclerView.OnItemTouchListener

### CardView

![image](/assets/images/md7.png)

两个属性：

```
app:cardCornerRadius
app:cardBackgroundColor
```

### Toolbar

用来替代ActionBar，可以当做一个普通的ViewGroup来使用，所以来说比前者更灵活

### AppBarLayout

常作为Toolbar和其他View的父View配合CoordinatorLayout使用来实现Scrolling Animation

### FloatingActionButton

浮动Button,也可以加入二级菜单

### Snackbar

带有动作的Toast
### TabLayout

常和ViewPager搭配使用，比较方便

### CoordinatorLayout.Behavior

CoordinatorLayout是一个比较重要的类，大致主要分这几点：

* 一个抽象内部类
* 利用泛型是指定我们应用这个Behavior的view的类型
* 自定义Behavior：某个View监听另一个view的状态变化，例如大小、位置、显示状态等：layoutDependendsOn和onDependentViewChanged方法；
* 某个view监听CoordinatorLayout里的滑动状态：onStartNestedScroll和onNestedPreScroll方法。
* CoordinatorLayout 所做的事情就是当成一个通信的 桥梁 ，连接不同的view。使用 Behavior 对象进行通信。

两个常见用例：

* CoordinatorLayout与悬浮操作按钮

1. CoordinatorLayout是用来协调其子view们之间动作的一个父view，而Behavior就是用来给CoordinatorLayout的子view们实现交互的。

2. FloatingActionButton作为一个子View添加进CoordinatorLayout并且将CoordinatorLayout传递给 Snackbar.make()

* CoordinatorLayout与app bar

1. 使用AppBarLayout可以让你的Toolbar与其他view（比如TabLayout的选项卡）能响应被标记了ScrollingViewBehavior的View的滚动事件

2. 这里使用了CollapsingToolbarLayout的app:layout_collapseMode="pin"来确保Toolbar在view折叠的时候仍然被固定在屏幕的顶部。借助app:layout_collapseMode="parallax"（以及使用app:layout_collapseParallaxMultiplier="0.7"来设置视差因子）来实现视差滚动效果（比如CollapsingToolbarLayout里面的一个ImageView），这种情况和CollapsingToolbarLayout的app:contentScrim="?attr/colorPrimary"属性一起配合更完美。

总结：

* CoordinatorLayout必须作为整个布局的父布局容器。
* 给需要滑动的组件设置 app:layout_scrollFlags=”scroll|enterAlways” 属性。
* 给你的可滑动的组件，也就是RecyclerView 或者 NestedScrollView 设置如下属性:app:layout_behavior = @string/appbar_scrolling_view_behavior
* 给需要有折叠效果的组件设置 layout_collapseMode属性。

### Chris Banes

android.support:design库作者

github: https://github.com/chrisbanes
google+: https://plus.google.com/+ChrisBanes
twitter: https://twitter.com/chrisbanes
blog: http://chris.banes.me/

## 总结

* 阴影：android:elevation 和 android:translationZ

* 调色：通过android:colorPrimary 和 android:colorAccent、Palette萃取等

* 图标：使用遵循material design spec的icon

* 尺寸：注意8dp的整数倍

* 动效：波纹动画、循环揭示、共享元素转场、滑动、SVG动画等

* 组件：FAB、Appbar、Tabs、Cards、Lists等

## 兼容

### 样式与布局兼容

> 可以写对应的样式文件和布局文件
  * res/values/styles.xml.
  * res/values-v21/styles.xml.
  * res/layout/my_activity.xml
  * res/layout-v21/my_activity.xml

### 使用支持库提供的组件

### 使用兼容的主题：Theme.AppCompat

> 以下组件可以借助兼容主题来实现已有样式
* EditText
* Spinner
* CheckBox
* RadioButton
* SwitchCompat
* CheckedTextView

### 调色板

![image](/assets/images/md8.png)

```
<resources>
 <style name="AppTheme" parent="android:Theme.Material">
   <item name="android:colorPrimary">@color/primary</item>
   <item name="android:colorPrimaryDark">@color/primary_dark</item>
   <item name="android:colorAccent">@color/accent</item>
  </style>
</resources>

```

### 使用第三方组件库

### 自己造轮子

先[熟悉API](http://developer.android.com/design/material/index.html )，再查看官方设计指南，利用现有API进行封装

## Best-In-Class Android Design

Material Design团队整理的Google Play上比较好的MD风格的App

![image](/assets/images/md9.png)

https://play.google.com/store/apps/collection/promotion_3001769_io_awards


## 附录

这次的PPT[下载地址](https://pan.baidu.com/s/1dFLqrs9)
