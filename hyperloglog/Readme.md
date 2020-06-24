# 									HyperLogLog学习杂记



- What

> HyperLogLog是一种基于数学概率论的基数估计算法。基数估算就是为了估算在一批数据中，它的不重复元素有多少个。



- 背景需求

> `现实需求：统计 APP或网页 的一个页面，每天有多少用户点击进入的次数。同一个用户的反复点击进入记为 1 次`
>
> 编程解决上面的需求好像很简单， 可以想到很多种实现方法：HashMap、 B+ 树、Bitmap等等， 在一定数据规模下它们都可以有效解决问题， 但是当数据规模达到一定数量级，如：百万、千万、亿、十亿、百亿、千亿等等的时候， 上面的解决方案就需要占用大量的内存， 甚至现实中我们可能根本无法满足其内存需求量！此时强烈的需求就是： 如果可以不占用，或是尽可能少占用内存的条件下，就可以解决上面的`现实需求`， 那该多好！如此HyperLogLog就诞生了！据说：在 `Redis` 中实现的 `HyperLogLog`，只需要`12K`内存就能统计`2^64`个数据。





- 数学原理

> 数学概率论 - `伯努利试验`是数学`概率论`中的一部分内容，它的典故来源于`抛硬币`。
>
> 硬币拥有正反两面，一次的上抛至落下，最终出现正反面的概率都是50%。假设一直抛硬币，直到它出现正面为止，我们记录为一次完整的试验，间中可能抛了一次就出现了正面，也可能抛了4次才出现正面。无论抛了多少次，只要出现了正面，就记录为一次试验。这个试验就是`伯努利试验`。
>
> 那么对于多次的`伯努利试验`，假设这个多次为`n`次。就意味着出现了`n`次的正面。假设每次`伯努利试验`所经历了的抛掷次数为`k`。第一次`伯努利试验`，次数设为`k1`，以此类推，第`n`次对应的是`kn`。
>
> 其中，对于这`n`次`伯努利试验`中，必然会有一个最大的抛掷次数`k`，例如抛了12次才出现正面，那么称这个为`k_max`，代表抛了最多的次数。
>
> `伯努利试验`容易得出有以下结论：
>
> 1. n 次伯努利过程的投掷次数都不大于 k_max。
> 2. n 次伯努利过程，至少有一次投掷次数等于 k_max
>
> 最终结合极大似然估算的方法，发现在`n`和`k_max`中存在估算关联：`n = 2^(k_max)` 。这种通过局部信息预估整体数据流特性的方法似乎有些超出我们的基本认知，需要用概率和统计的方法才能推导和验证这种关联关系。
>
> 例如下面的样子：
>
> ```log
> 第一次试验: 抛了3次才出现正面，此时 k=3，n=1
> 第二次试验: 抛了2次才出现正面，此时 k=2，n=2
> 第三次试验: 抛了6次才出现正面，此时 k=6，n=3
> 第n 次试验：抛了12次才出现正面，此时我们估算， n = 2^12
> ```
>
> 假设上面例子中实验组数共3组，那么 k_max = 6，最终 n=3，我们放进估算公式中去，明显： 3 ≠ 2^6 。也即是说，当试验次数很小的时候，这种估算方法的误差是很大的。
>
> ###### 估算的优化：`HyperLogLog`和`LogLog`的区别就是，它采用的不是`平均数`，而是`调和平均数`。`调和平均数`比`平均数`的好处就是不容易受到大的数值的影响。
>
> 统计网页点击量和伯努利实验有什么关系呢？
>
> ```shell
> 通过hash函数，将数据转为比特串，例如输入5，便转为：101。为什么要这样转化呢？
> 
> 是因为要和抛硬币对应上，比特串中，0 代表了反面，1 代表了正面，如果一个数据最终被转化了 10010000，那么从右往左，从低位往高位看，我们可以认为，首次出现 1 的时候，就是正面。
> 
> 那么基于上面的估算结论，我们可以通过多次抛硬币实验的最大抛到正面的次数来预估总共进行了多少次实验，同样也就可以根据存入数据中，转化后的出现了 1 的最大的位置 k_max 来估算存入了多少数据。
> 
> 通过划分出多个桶，可以提升精度减小误差， 比如上面同一个hash函数值，可以分成两部分， 桶index + 其余hash值， 然后对所有桶求平均数， 最终估算出基数。
> ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
> 这个算法背后的主要技巧是，如果你观察到一个随机整数流，看到一个以某个已知前缀开头的整数，那么该流的基数很可能是2^（前缀的大小）。
> 也就是说，在随机整数流中，约50%的数字（二进制）以“1”开头，25%以“01”开头，12,5%以“001”开头。这意味着，如果您观察到一个随机流并看到一个“001”，则该流具有基数8的可能性更大。
> 
> 当然，如果你只观察一个整数，那么这个值出错的几率很高。这就是为什么该算法将流分成“m”个独立的子流(桶)，并保持每个子流(桶)的前缀“00…1”的最大长度。然后，通过取每个子流的平均值来估计最终值, 当然还要误差修正。这就是这个算法的主要思想。
> ```
>
> 好文章推荐：`https://www.cnblogs.com/linguanh/p/10460421.html` 和`https://stackoverflow.com/questions/12327004/how-does-the-hyperloglog-algorithm-work`



- HLL实现代码收集

> `https://github.com/kunigami/blog-examples/blob/master/hyper-log-log/hyperloglog/src/main.rs`
>
> `https://kunigami.blog/2018/03/31/hyperloglog-in-rust/`
>
> `https://github.com/kjgorman/hll.rs`
>
> `https://github.com/jedisct1/rust-hyperloglog`
>
> `https://github.com/antirez/redis/blob/unstable/src/hyperloglog.c`
>
> `https://www.jianshu.com/p/55defda6dcd2`







- Author

> 学习随笔，难免谬误，望请指教
>
> 作者：心尘了
>
> email: [285779289@qq.com](mailto:285779289@qq.com)
>
> git：https://github.com/yujinliang



- Reference

> `https://www.cnblogs.com/linguanh/p/10460421.html`
>
> `https://www.jianshu.com/p/412c96a76723`
>
> `https://blog.csdn.net/firenet1/article/details/77247649`
>
> `http://blog.codinglabs.org/articles/algorithms-for-cardinality-estimation-part-i.html`
>
> `https://www.zhihu.com/question/53416615`
>
> `https://github.com/jedisct1/rust-hyperloglog`
> `https://kunigami.blog/2018/03/31/hyperloglog-in-rust/`
> `https://github.com/kjgorman/hll.rs`
> `http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf`
> `https://stackoverflow.com/questions/12327004/how-does-the-hyperloglog-algorithm-work`
>
> `[http://www.rainybowe.com/blog/2017/07/13/%E7%A5%9E%E5%A5%87%E7%9A%84HyperLogLog%E7%AE%97%E6%B3%95/index.html](http://www.rainybowe.com/blog/2017/07/13/神奇的HyperLogLog算法/index.html)`
>
> `http://content.research.neustar.biz/blog/hll.html`
>
> `https://www.jianshu.com/p/55defda6dcd2`
>
> `https://github.com/antirez/redis/blob/unstable/src/hyperloglog.c`
>
> `https://github.com/kunigami/blog-examples/blob/master/hyper-log-log/hyperloglog/src/main.rs`





