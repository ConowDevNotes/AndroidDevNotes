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
在Android中，将屏幕左上角的定点座位Android坐标系的原点，从这个点向右是X轴得正方向，这个点向下为Y轴正方向

![](https://github.com/ConowDevNotes/AndroidDevNotes/blob/master/res/img/zuobiao.png "Android坐标系")

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

### 2.3 菜单优化实战
首先来看下最终效果

![](https://github.com/ConowDevNotes/AndroidDevNotes/blob/master/res/img/modernView.png "最终效果")


#### 2.3.1 继承view，重写view的构造方法

```
public class ModernMenuView extends View {

    public ModernMenuView(Context context) {
        this(context, null);
    }

    public ModernMenuView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public ModernMenuView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }
}
```

#### 2.3.2 获取配置信息，并初始化

如果希望对控件进行一些灵活的配置，我们可以对view设置一些配置的参数,在这里我设置了一下几个属性，
并在构造方法中获取这些配置;上面说了画笔的初始化不要放在`onDraw()`中操作，应当放在这里。

```
//首先声明需要的配置参数，分别是内边距、小图标之前的空隙、圆角、小图标里面文字的大小
<declare-styleable name="ModernMenuView">
    <attr name="pad" format="dimension"/>
    <attr name="space" format="dimension"/>
    <attr name="cornerRadius" format="dimension"/>
    <attr name="iconSize" format="dimension"/>
</declare-styleable>
```

```
public ModernMenuView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
    mContext = context;

    //获取自定义的属性配置
    TypedArray typedArray = context.obtainStyledAttributes(attrs, R.styleable.ModernMenuView);
    PADDING = typedArray.getDimensionPixelSize(R.styleable.ModernMenuView_pad, 0); //padding
    //获取其他属性配置
    typedArray.recycle();

    //初始化矩形的画笔
    mRectPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    mRectPaint.setStyle(Paint.Style.FILL);

    //初始化图标文字的画笔
    mIconPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    mIconPaint.setStyle(Paint.Style.FILL);
    mIconPaint.setColor(Color.WHITE);
    mIconPaint.setTextAlign(Paint.Align.CENTER);
    mIconPaint.setTypeface(IconUtil.getTypeface(mContext));
    mIconPaint.setTextSize(ICON_SIZE);

    needReDraw = true;
}
```

#### 2.3.3 测量
测量要重写`onMeasure()`,在这里我主要是确保view的长宽比例是1:1而已,并计算view中每个小图标的大小

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
    //图标的大小
    BODY = (width - PADDING * 2 - SPACE * 2) / 3;
    if (needReDraw) buildDrawRectList();
}
```


#### 2.3.3 预先将一些创建对象的操作提前做好
在这里，我们将会绘制很多矩形，这种耗内存的操作必须提前做好。

```
/**
 * 预先计算好矩形的坐标
 */
private void buildDrawRectList() {
    for (int i = 0; i < dataList.size(); i++) {
        RectF mRectF = new RectF(PADDING + SPACE * (i % 3) + BODY * (i % 3),
                PADDING + SPACE * (i / 3) + BODY * (i / 3),
                PADDING + SPACE * (i % 3) + BODY * (i % 3 + 1),
                PADDING + SPACE * (i / 3) + BODY * (i / 3 + 1));
        mDrawRectList.add(mRectF);
    }
}
```

画矩形首先要确定好它的位置,也就是要确定它的坐标，分别是它的4个角的坐标
![](https://github.com/ConowDevNotes/AndroidDevNotes/blob/master/res/img/rectRect.png "矩形坐标")

#### 2.3.4 拿起画笔在画布上绘制

```
@Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        for (int i = 0; i < size; i++) {
            Map item = dataList.get(i);
            //画小矩形
            mRectPaint.setColor((Integer) item.get("COLOR"));
            canvas.drawRoundRect(mDrawRectList.get(i), CORNER_RADIUS, CORNER_RADIUS, mRectPaint);

            String text = (String) item.get("ICON");
            Paint.FontMetrics metrics = mIconPaint.getFontMetrics();
            float dy = -(metrics.descent + metrics.ascent) / 2;
            //画矩形上的文字图标
            canvas.drawText(text,
                    PADDING + SPACE * (i % 3) + BODY * (i % 3) + BODY * 0.5f,
                    PADDING + SPACE * (i / 3) + BODY * (i / 3) + BODY * 0.5f + dy,
                    mIconPaint);
        }
    }
```

#### 2.3.5 最终效果
![](https://github.com/ConowDevNotes/AndroidDevNotes/blob/master/res/img/finalView.jpg "现代菜单")



