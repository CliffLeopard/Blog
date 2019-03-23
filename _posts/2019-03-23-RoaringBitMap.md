---
layout: post
title: "RoaringBitMap"
date: 2019-03-23 16:29:00.000000000 +09:00
tags: BigData
---

RoaringBitmap的存储原理，计算方式。以及使用场景

## 一、存储

RoaringBitmap用来存储值各不相同的32位整型Int值。将Int值划分为高16位和低16位。内部的存储结构是一个RoaringArray。
RoaringArray由以下构成:

* short[] keys
* Container[] values
* int size = 0

### 1. 高位

keys: 存储高16位，是有序数组，从小到大排序。

values:存储低16位,按照单个Container中数据的存储个数的不同，低16位使用不同的存储方式。即Container的三个子类。ArrayContainter,BitMapContainer,RunContainer(下文会详述)

size:存储当前存储的高位的个数，当然也是values的size.

显然keys的index与values的Index是对应的。即高位keys[i]对应的所有数据都在values[i]对应的Container中，当然一个Container存储了一条或者多条低位数据。

显然高位数据的存储非常简单，使用了short 类型，所以高位数据可存储的最大值为pow(2,16)-1即65535个，最小值为0。所以最多可存储pow(2,16)=65536条数据。

short占用2个字节，则高位keys填充满的情况下，高位最多占用内存数为2*pow(2,16)/pow(2,10)=pow(2,7)=128KB。

### 2. 低位

低位数据的存储，稍微复杂一些。我们来看以下Container的子类

#### 2.1 ArrayContainer

* short[] content; 低位数据存储结构。每个低位都是一个short。这里content也是有序的，从小到大排序
* protected int cardinality = 0; 我们在下文讲解运算时再详细介绍
* static final int DEFAULT_MAX_SIZE = 4096 content中最多存储个数。

ArrayContainer的结构比较简单。进来一个低位数据，就把它存储到content中。DEFAULT_MAX_SIZE中规定了content中最多存储的数据个数。一旦数据量大于这个值，将转化为BitMapContainer来存储。

假如存储n个数据，占用内存量是多少呢？ 很简单，short占用两个字节，2*n/pow(2,10)=n/pow(2,-9)KB。显然存储空间的占有量是随着数据量的增加线性增加的。 n=4096时 内存占用量8Kb。

#### 2.2 BitMapContainer

* final long[] bitmap;
* int cardinality;

* 其实BitMapContainer也比较简单。类似于Hash排序。低16位最多存储的不同数据的个数为pow(2,16)=65536个。BitMapContainer使用pow(2,16) bit的存储空间，每一个bit代表一个数，如果此数存在则标记位1，否则标记为0。(即从小到大排序，坐标值代表数字)

* long类型64位，要存储pow(2,16)位，需要pow(2,10)=1024个long(long类型64位，8个字节)的存储空间。所以显然bitmap.size=1024。

* 占用内存位pow(2,16)/pow(2,3)/pow(2,10) = 8KB。显然一个bitmap占用的内存量是恒定的8KB，不会随着存储数据的多少而变化。这也就是ArrayContainer设置DEFAULT_MAX_SIZE = 4096的原因，当n<4096时ArrayContainer的存储效率高于BitMapContainer，而数据量增大时BitMapContainer更优。

#### 2.3 RunContainer

RunContainer使用了run-length数据压缩算法。对于连续型数据压缩率很高，但是对于非连续型有可能会负压缩。

比如数据位 18，19，20，27 --> 18,2,27,0 将后面连续的数字个数记下。

RunContainer的存储结构会在手动调用runOptimize()函数时，或者当BitMapContainer中的数据满时(pow(2,16)=65536)自动进行压缩。

### 3. 内存占用

一个RoaringArray在存满的情况下，存储的数据条数是多少？占用多少内存？

* 最多存储的不同数据的个数为：高位pow(2,16)条，每个高位存储一个bitmap可存储pow(2,16)条。 总个数为:pow(2,32)个。
* 占用内存: 高位128KB + 低位 pow(2,16)*8KB/pow(2,10) = 512MB+128KB。还是挺大的。
  
那么，如果我们不使用RoaringBitmap,而是使用数组存储呢？(不考虑unsigned转化的情况)

pow(2,32)个Int值，每个Int,32位4个字节。pow(2,32)*4/pow(2,30)=pow(2,4)G=16G。

经过RoaringBitMap算法数据压缩到了1/32。平均每个Int占用内存1bit。(高位存储最高为128KB，比较小，忽略不计)

当然，在数据存满的情况下，数据连续性会很普遍，经过RunContainer优化，压缩率会非常高。

## 二、运算

由于运算略复杂。我们从基础数据结构的运算开始看。当然它们都是为RoaringBitMap的各种运算服务的。最后我们在通过讲解RoaringBitMap的各种运算，将所有细节串联起来。

### 1.RoaringBitmap运算

#### 1.1. void add(final int x)

```java

  @Override
  public void add(final int x) {
    final short hb = Util.highbits(x);
    final int i = highLowContainer.getIndex(hb);
    if (i >= 0) {
      highLowContainer.setContainerAtIndex(i,
          highLowContainer.getContainerAtIndex(i).add(Util.lowbits(x)));
    } else {
      final ArrayContainer newac = new ArrayContainer();
      highLowContainer.insertNewKeyValueAt(-i - 1, hb, newac.add(Util.lowbits(x)));
    }
  }
```

这是最基础的运算，也体现了构造存储结构的过程。

* 得到x的高16位short:`hb`
* 计算x的低16位short:`lb`
* 计算`hb`在`keys`中需要插入的位置`index`。
  
Index 的计算和插入方式：

* 当index>0时，表示高位数据曾经存在过，已经初始化。则找到对应的Container:`values[index]`,将`lb`添加到此container中。找到的Container可能是`ArrayContainer`,也可能是`BitMapContainer`,两者的添加方式不同。下文`2.1`和`3.1`详细讲述。
* 当index<0时表示对应的高位还未初始化(`keys`中不存在此`hb`)。通过二分查找，得到的合适的插入位置x，则`index = -x-1`。 这样做是为了避免与size=0和`hb`已经存在的情况混淆，方便后面通过index进行计算。计算时(-index-1)得到x。然后ArrayContainer `newac` = new ArrayContainer,将`lb`插入到创建的newac中。 通过copy数组，将`hb`插入到`keys[x]`位置。同时将`newac`插入到`values[x]`位置。

### 2. BitMapContainer中的计算。

对照下文的ArrayContainer中的运算一起看会比较容易理解。因为二者基本都会有相同的计算任务。

#### 2.1 添加数据 add(final short i)

显然`i`为需要插入RoaringBitMap的整数的低位`lb`。(高位`hb`以short类型存储到RoaringArray的`keys`中)。

从上文我们可以知道，在BitMapContainer中要存储pow(2,16)个数据，使用了`pow(2,16)= (1 << 16)=65536`bit来存储。每个bit代表一个数。存在则标记为1，不存在则为0。而`bitmap`是一个long[]。一个long为64bit,所以需要
`(1 >> 16)/64 = 1024`个long。即`bitmap.length=1024`。当然`bitmap`在BitMpContainer构造函数中就已经初始化好为1024个。

当一个数据`i`进来之后。我们要确定表示`bitmap`中哪个long,以及long中的哪个bit来表述这个数字，然后将这个bit标记为1。则`i`就会被记录已存在，即`add`成功。

我们来看一下在源码中是如何实现的：

```java
  public Container add(final short i) {
    final int x = Util.toIntUnsigned(i);
    final long previous = bitmap[x / 64];
    long newval = previous | (1L << x);
    bitmap[x / 64] = newval;
    if (USE_BRANCHLESS) {
      cardinality += (previous ^ newval) >>> x;
    } else if (previous != newval) {
      ++cardinality;
    }
    return this;
  }
```

* 首先将`i`由16bit存储的short转化为32bit存储的int。用`x`来表示，显然此时`x`的高16位都为0,显然`x`中真正有效的值为低位16位。我们舍用第`bmi`个long:`bitmap[bmi]`,和`bitmap[bmi]`中第`bi`(`bi`<64)位置的bit来来表示`x`,则显然 `(bmi-1)*64+bi=x`   -->  `bmi`= [x/64]+(64-bi)/64,  又因为(0<=bmi<1024;0<=bi<64;bmi,bi都是正整数)，则`bmi=bitmap[x/64]` (x/64 是 x除以64的整数商，舍余)
* 所以显然在`i`加入之前表示此数值的long为`previous = bitmap[x / 64]`,我们只需要对`previous`的第`bi`位进行修改就可以了。
* `long newval = previous | (1L << x);` 这里一度让我非常困惑，怎么会对1L进行左移位x个，应该是左移位`bi`个啊，如果左移位x个，很可能会出现负数啊，完全没有意义。查了部分资料，1L进行左移位时，如果位数超过64,则取余操作。那么这里其实`1L << x == 1L << bi`。 那么很简单，移位的作用便是将1对应bi的位置，其余位置都为0.这样`|previous`之后`i`值就被成功添加。
* `bitmap[x / 64] = newval;` 将新long值赋值给bitmap。
* `cardinality`记录存储的数据个数，显然在添加操作中，前后long不同，就是添加了一个，否则此值之前已存在，不在计数。
* 这部分的移位操作略显飘逸，可能不太容易理解。然而看几遍之后就很简单。

### 3. ArrayContainer中的计算。

ArrayContainer 由于存储逻辑比较简单，所以运算也比价简单。但在计算后ArrayContainer和BitMapContainer的转化比较有意思。

#### 3.1 添加数据 add(final short i)

```java
  @Override
  public Container add(final short x) {
    int loc = Util.unsignedBinarySearch(content, 0, cardinality, x);
    if (loc < 0) {
      if (cardinality >= DEFAULT_MAX_SIZE) {
        BitmapContainer a = this.toBitmapContainer();
        a.add(x);
        return a;
      }
      if (cardinality >= this.content.length) {
        increaseCapacity();
      }
      System.arraycopy(content, -loc - 1, content, -loc, cardinality + loc + 1);
      content[-loc - 1] = x;
      ++cardinality;
    }
    return this;
  }
```

上文已讲ArrayContainer中的存储结构short[] content;是一个有序数组，从小到大排序。所以插入数据的时候先二分查找，知道需插入的位置，如果已存在则返回 loc = -index-1;否则返回应插入的位置loc= index。 当loc>0时执行插入操作。这部分逻辑略类似于RoaringArray高位的操作。

插入时，如果cardinality已经>=4096 则转化为BitMapContainer再添加。当然还有content(初始化长度4)的扩展操作，比较简单不再赘述。

**未完待续**
对于运算只详细讲述了add,但是通过add(),整体的存储结构和逻辑已经非常清晰。后期会继续添加各种其他操作的逻辑。
