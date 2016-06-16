title: collectionView自定义layout
date: 2016-06-06 14:13:54
categories: collectionView
tags: [layout]

---
自定义layout前，先来看看UICollectionView。

## 什么是UICollectionView

UICollectionView是一种新的数据展示方式，简单来说可以把他理解成多列的UITableView(请一定注意这是UICollectionView的最最简单的形式)。

<!--more-->

## 结构

首先回顾一下Collection View的构成，我们能看到的有三个部分：

* Cells 用于展示内容的主体，对于不同的cell可以指定不同尺寸和不同的内容
* Supplementary Views 追加视图 （类似UITableView Header或者Footer，但是只发挥了10%的功力）
* Decoration Views 装饰视图 （用作背景展示）

### 关于重用
为了得到高效的View，对于cell的重用是必须的，避免了不断生成和销毁对象的操作，这与在UITableView中的情况是一致的。但值得注意的时，在UICollectionView中，不仅cell可以重用，Supplementary View和Decoration View也是可以并且应当被重用的。

### 关于Cell

SDK提供给我们的默认的UICollectionViewCell结构上相对比较简单，由下至上：

- 首先是cell本身作为容器view
- 然后是一个大小自动适应整个cell的backgroundView，用作cell平时的背景
- 在其上是selectedBackgroundView，是cell被选中时的背景
- 最后是一个contentView，自定义内容应被加在这个view上

### UICollectionViewLayout

终于到UICollectionView的精髓了…这也是UICollectionView和UITableView最大的不同。UICollectionViewLayout可以说是UICollectionView的大脑和中枢，Layout决定了UICollectionView是如何显示在界面上的, 每个cell view、supplemental view和decoration view 都有layout属性。想要知道layouts如何灵活，只需看看 UICollectionViewLayoutAttributes 对象的特性就知道了：

* frame
* center
* size
* transform3D
* alpha
* zIndex
* hidden

这是最酷的方法：

* -layoutAttributesForElementsInRect:

在展示之前，一般需要生成合适的UICollectionViewLayout子类对象，并将其赋予CollectionView的collectionViewLayout属性。

### 自定义layout

#### 继承UICollectionViewLayout

##### 理解layout的处理过程

1. prepareLayout  提供layout信息的一些前期准备
2. collectionViewContentSize  基于初始化计算得出的整个内容的size
3. layoutAttributesForElementsInRect: 返回在指定rectangle区域的cells和views的attributes

布置自定义内容:
![](https://developer.apple.com/library/ios/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/cv_layout_process_2x.png)

##### 创建Layout Attributes

用下面类方法创建UICollectionViewLayoutAttributes的类实例：

* -layoutAttributesForItemAtIndexPath:
* -layoutAttributesForSupplementaryViewOfKind:atIndexPath:
* -layoutAttributesForDecorationViewOfKind:atIndexPath:

##### prepareLayout

layout处理前，有机会做一些初始化计算。

##### 在给定Rectangle区域提供Items的Layout Attributes
在layout处理流程的最后一步，collectionView将会调用layoutAttributesForElementsInRect:。目的是为和指定rectangle交叉的每一个cell和supplementary、decoration视图提供layout attributes。

布局看得见的元素：
![](https://developer.apple.com/library/ios/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/Art/cv_visible_elements_2x.png)

### 总结
跟UITableView有些类似，一个UICollectionView的实现包括两个必要部分：UICollectionViewDataSource和UICollectionViewLayout，和一个交互部分：UICollectionViewDelegate。而Apple给出的UICollectionViewFlowLayout已经是一个很强力的layout方案了。

## 自定义layout几种实现([点击查看具体代码](https://github.com/meigesir/TTCollectionView.git))

![自定义layout](https://github.com/meigesir/TTCollectionView/raw/master/Resources/home.gif)

### 1. 无限轮播

![无限轮播](https://github.com/meigesir/TTCollectionView/raw/master/Resources/InfiniteCarouse.gif)

优点，相比ScrollView,可以重用cell，优化性能

#### 实现
1. 严格意义上说并没有自定义layout，而是使用的UICollectionViewFlowLayout，设置水平滚动方向，设置minimumLineSpacing为0
2. collectionView：设置pagingEnabled为YES
3. 以上可以达到轮播的目的，无限轮播实现方式，就是在第一张轮播视图的前面插入最后一张轮播，最后一张轮播视图的后面插入第一张轮播，这样在滑动到第一个或最后一个位置的时候，分别无动画的切换到原始数据的最后一张或第一张。再加入timer定时滚动。

### 2. 瀑布流（CHTCollectionViewWaterfallLayout）

![瀑布流](https://github.com/meigesir/TTCollectionView/raw/master/Resources/WaterFall.gif)

瀑布流比较有代表的是app是Pinterest。特点是，每个元素都是等宽不等高，而且每当排列一个元素的时候，都是从最短的那一列开始排。这里我就不重复造轮了，因为有一个比较好的开源项目(CHTCollectionViewWaterfallLayout，注释也很充分)。但是作者有一个地方有欠缺(CHTFloorCGFloat)，但是可以忽略不计。

#### 实现
1. prepareLayout
计算好每一列的高度，计算每一个元素的Layout Attributes。作者在此方法最后有一个亮点，就是将Attributes每20个分组，后面调用layoutAttributesForElementsInRect:的时候使用，可以省掉一些运算，从而优化性能。
2. collectionViewContentSize
算出真个内容的高度，也就是最后一个section的列的高度(最后保持一致，因为已经展示完了)
3. layoutAttributesForElementsInRect:
先锁定Attributes组的范围，然后再具体求解，优化了性能。

### 3. 弹性header

![弹性header](https://github.com/meigesir/TTCollectionView/raw/master/Resources/StretchyHeader.gif)

下拉collectionView的时候，header可以被拉伸，从而可以更全面地查看图片更多细节，手势松开，又可以反弹到初始状态。

#### 实现
1. 展示时，在layoutAttributesForElementsInRect:方法中获取header的attributes
2. 然后设置attributes的y和height值，从而达到相应的效果
3. 弹性：调用collectionView的setAlwaysBounceVertical:，参数为YES
4. 设置图片展示方式为UIViewContentModeScaleAspectFill

### 4. 水平滚动放大

![水平滚动放大](https://github.com/meigesir/TTCollectionView/raw/master/Resources/SlideZoomIn.gif)

水平滚动cell，当到屏幕中间的时候，放到最大，并会自动校正位置

#### 实现

1. 展示时，layoutAttributesForElementsInRect:，设置每个item的Attributes，通过item中心点x和collectionView屏幕中心点的x值的间距来设置item的缩放
2. 校正位置：重写targetContentOffsetForProposedContentOffset:withScrollingVelocity:，在item不在中心位置的时候自动居中

### 5. 拖动排序item

![拖动排序item](https://github.com/meigesir/TTCollectionView/raw/master/Resources/ReOrder.gif)

手动长按item的时候，item稍微放大一些，提示用户当前可以做出拖动操作。然后移动item，可以将item任意的移动位置。iOS9已经有相应的API支持，而且比较简单，这里为了兼容iOS7以上，所以需要自定义layout。由于也有相应的轮子LXReorderableCollectionViewFlowLayout，这里就不重新造轮子了，局限是不适用于自定义layout。

### 实现

1. 添加长按手势，有可能手势冲突，比如iOS9添加长按手势实现移动元素，所以需要设置我们自定义长按手势优先级高(requireGestureRecognizerToFail:)。
2. 长按手势开始时，基于原有item的frame创建view，添加正常、高亮时的截图，然后放大view到1.1倍；长按手势结束时，view缩放到正常大小，然后view设置nil，拖动结束；两次相应的委托调用。
3. 拖动需要添加pan手势，拖动改变时，改变view的中心点即可。
4. 在layoutAttributesForElementsInRect:中，判断如果是当前操作的item，Attributes的hidden设为YES，从而隐藏原有的item。
5. invalidateLayoutIfNecessary，实现没有操作在item上面时不生效，及其相应的删除、插入操作，及其相应的委托方法。


参考：
1. [UICollection​View](http://nshipster.cn/uicollectionview/)
2. [WWDC 2012 Session笔记——205 Introducing Collection Views](https://onevcat.com/2012/06/introducing-collection-views/)
3. [Creating Custom Layouts](https://developer.apple.com/library/ios/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html)
4. [理解UIScrollView的三个属性：contentOffset、contentSize以及contentInset](http://pandajohn.com/jin-bu-li-jie-uiscrollviewde-san-ge-shu-xing-contentoffset-contentsizeyi-ji-contentinset/)
5. [Stretchy UICollectionView headers](https://nrj.io/stretchy-uicollectionview-headers/)
