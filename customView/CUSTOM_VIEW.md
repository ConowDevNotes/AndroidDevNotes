# 自定义View
**自定义控件，谈谈九宫格菜单优化<br>**

## 1.什么是自定义View
Android里面基本上每时每刻都在和View打交道，app中每个页面都是由view组成的，
例如简单的一个Button就是一个View。

Adnroid里面所有的控件都是直接或者间接继承View的，当系统提供的控件不满足我们所要实现的，
那这时候就需要继承view或者viewGroup来创建一个view来满足我们的需求。

## 2.如何创建自定义View
自定义view其实就是一个画画的过程，画画少不了两样材料：画布和画笔。

### 2.1 Android 坐标系统

### 2.2 绘制流程

#### 2.2.1 onMeasure()
`onMeasure()`顾名思义，就是测量的意思。这是画画之前的准备，这是必须的，
因为画画首先要确定要画布的大小。

etc.
在自定义菜单view中，我在`onMeasure()`中设置了view的大小是1:1的，无论在xml中怎么设置，
view的大小比例永远保持1:1。

```
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    int width = getMeasuredWidth();
    int height = getMeasuredHeight();
    //设置大小比例为1:1
    if (width < height) {
        height = width;
    } else {
        width = height;
    }
    //重新设置画布的大小
    setMeasuredDimension(width, height);
}
```

#### 2.2.2 onLayout()
layout就是布局的意思，很容易理解`onLayout()`就是确定布局,也就是确定位置的意思，
通常在ViewGroup中用得比较多。因为`onLayout()`确定的位置是相对于父组件来说的。

#### 2.2.3 onDraw()
`onDraw()`这里就开始画画了，首先我们前期需要准备画笔，通常画笔在
View的构造函数中进行初始化，同理new对象的事千万不要在这里做。

`onDraw()`这个用得比较频繁，如果频繁在里面new对象的话，非常占用内存，频繁触发GC回收，
严重的话可能OOM