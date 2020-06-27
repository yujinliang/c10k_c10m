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
> ###### 
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
> 当然，如果你只观察一个整数，那么这个值出错的几率很高。这就是为什么该算法将流分成“m”个独立的子流(桶)，并保持每个子流(桶)的前缀“00…1”的最大长度。然后，通过取每个子流基数估值的平均值来估计最终值, 当然还要误差修正。这就是这个算法的主要思想。
> ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
> 假设您有一个长度为m的字符串，该字符串由{0，1}组成，概率相等。它以0开始，以2个0开始，以k个0开始的概率是多少？它是1/2，1/4和1/2^k。这意味着如果你遇到了一个k个零的字符串，你大概已经看过了2^k个元素。所以这是一个很好的起点。有一个在0和2^k-1之间均匀分布的元素列表，您可以在二进制表示中计算最大前缀零的最大数目，这将为您提供一个合理的估计。
> ```
>
> 好文章推荐：`https://www.cnblogs.com/linguanh/p/10460421.html` 和`https://stackoverflow.com/questions/12327004/how-does-the-hyperloglog-algorithm-work` 



- HLL算法描述

```shell
#HLL 核心算法 #摘引自`https://www.jianshu.com/p/55defda6dcd2` ， 感谢原作者，非常棒的技术文章！
#hash(a element) = 0b1110100110...  之所以首先对每一个集合元素执行hash , 因为hash结果为整数，并且均匀分布在一定范围之内！
#若分布不均匀，则直接导致估值不准确！！！
#默认小端字节序,其实无关大小端字节序上面的HLL数学原理和结论都是一样的！
#-------------------------------------------------------------------------------------------------------------------------------------------------------
输入：一个集合
输出：集合的基数
算法：
     max = 0
     对于集合中的每个元素：
               hashCode = hash(元素)
               num = hashCode二进制表示中最前面连续的0的数量
               if num > max:
                   max = num
     最后的结果（集合的基数）是2的(max + 1)次幂  
```



- 优化-分桶

> `最简单的一种优化方法显然就是把数据分成m个均等的部分，分别估计其总数求平均后再乘以m，称之为分桶。对应到前面抛硬币的例子，其实就是把硬币序列分成m个均等的部分，分别用之前提到的那个方法估计总数求平均后再乘以m，这样就能一定程度上避免单一突发事件造成的误差。
>  具体要怎么分桶呢？我们可以将每个元素的hash值的二进制表示的前几位用来指示数据属于哪个桶，然后把剩下的部分再按照之前最简单的想法处理。
>  还是以刚刚的那个集合`{ele1,ele2}`为例，假设我要分2个桶，那么我只要去ele1的hash值的左边第一位来确定其分桶即可，之后用剩下的部分进行前导零的计算，如下图：
>  假设ele1和ele2的hash值二进制表示如下：`
>
> ```shell
> hash(ele1) = 00110111 => (0)0110111 => 桶0 => 前导0个数:  1
> hash(ele2) = 10010001 => (1)0010001 => 桶1 => 前导0个数:  2
> ```

> LogLog算法完整的基数计算公式：
> $$
> DV_L {_L} = constant * m*2^\overline{ R }
> $$
> 注：m：桶数， R上划线代表每个桶中数据的最长前导零个数 + 1 的几何平均值。
>
> 分桶数只能是2的整数次幂.



> ＨLL 调和平均数公式：
> $$
> H_n = \frac{n}{\sum_{i=1}^{n}{ \frac{1}{R_i}}}
> $$
> `注：　Ri代表第ｉ个桶中的数据的最大前导零个数＋１.`



> 分桶平均的基本原理是将统计数据划分为m*m*个桶，每个桶分别统计各自的{k_{max}}*k**m**a**x*并能得到各自的基数预估值 \hat{n}*n*^ ，最终对这些 \hat{n}*n*^ 求平均得到整体的基数估计值。LLC中使用几何平均数预估整体的基数值，但是当统计数据量较小时误差较大；HLL在LLC基础上做了改进，采用调和平均数，调和平均数的优点是可以过滤掉不健康的统计值。
>
> 影响LogLog算法精度的一个重要因素就是，hash值的前导零的数量显然是有很大的偶然性的，经常会出现一两数据前导零的数目比较多的情况，所以HyperLogLog算法相比LogLog算法一个重要的改进就是使用`调和平均数`而不是`几何平均数`来聚合每个桶中的结果，HyperLogLog算法的公式如下：
> $$
> DV_H{_L}{_L} = constant *m*（\frac{m}{\sum_{i=1}^{m}{\frac{1}{2^{R_i}}}}）
> $$

> 注意：m之后圆括号部分代表：所有桶估计值的调和平均数。



> [当数据总量较少时，预测估值偏大]，修正方法如下：
>
> `if DV < (5 /2 ) *m {`
>
> ​		`DV = m * log( m / V )`
>
> `}`
>
> 注意：DV代表基数估值，m代表桶数，　V代表结果为０的桶数，　log代表自然对数
>
> 
>
> [constant的选择]
> $$
> p = log_2 (m)
> $$
> 注意：m代表桶数, p代表桶数的二进制位数。
>
> ```c
> switch (p) {
>    case 4:
>        constant = 0.673 * m * m;
>    case 5:
>        constant = 0.697 * m * m;
>    case 6:
>        constant = 0.709 * m * m;
>    default:
>        constant = (0.7213 / (1 + 1.079 / m)) * m * m;
> }
> ```
>
> [m桶数选择]
> $$
> RSD = \frac{1.04}{\sqrt{m}}
> $$
> 注意：RSD(relative standard deviation相对标准误差)　,RSD代表我们希望估值达到的精度，所以给出RSD可以求出m（桶数）。
>
> 　

> [合并]
>
> ```shell
> 数据流a："a" "b" "c" "d"  基数：4
> 数据流b："b" "c" "d" "e"  基数：4
> 两个数据流的总体基数：5
> ```
>
> [合并算法]
>
> ```shell
> 输入：桶数组a，b。它们的长度都是n
> 输出：新的桶数组c
> 算法：
>   c = c[n];
>   for (i=0; i<n; i++){
>       c[i]=max(a[i], b[i]);
>   }
>   return c;
> ```
>
> #注意以上内容摘取自`https://www.jianshu.com/p/55defda6dcd2` ， 精彩好文，可深入学习之！

> Redis对于一个输入的字符串，首先得到64位的hash值，用前14位来定位桶的位置（共有 ![[公式]](https://www.zhihu.com/equation?tex=2%5E%7B14%7D) ，即16384个桶）。后面50位即为伯努利过程，每个桶有6bit，记录第一次出现1的位置count，如果count>oldcount，就用count替换oldcount。
>
> 注意：64位二进制hash值中，第一次出现１的位置（以右端或左端为基准）有６４个，　所以每个桶内只需要６个bit就可以记录了！！！ln64 = 6 , 因为hash值截取１４位作为桶编号，　所以只只需对剩下的５０bit来统计第一次出现１的位置，　有５０个位置，５个bit的桶最多记录３２个位置，所以向上ceil取整，６个bit可以记录６４个位置，　最终定位６bit桶。





- HLL 实现代码分析

```java
//https://www.jianshu.com/p/55defda6dcd2
////理解下面代码的关键在于：他是以二进制的左端为基准，比如一个二进制value, 结构为： 桶index二进制部分　＋　 hash值其余二进制部分：左端
    private static double rsd(int log2m) {
        return 1.106 / Math.sqrt(Math.exp(log2m * Math.log(2)));
    }

    public boolean offerHashed(int hashedValue) {
        // j 代表第几个桶,取hashedValue的前log2m位即可
        // j 介于 0 到 m
        //无符号数右移运算符>>>，空位补０　
        //p=log2m 其实就是可以存储｀桶index｀的二进制bit位数
        //这行就是通过右移将｀桶index｀部分提取出来，其他部分右移位丢掉。
        final int j = hashedValue >>> (Integer.SIZE - log2m);
        
        
        // r代表 除去前log2m位剩下部分的前导零 + 1
        //左移１bit 等于乘以２，　右端补０
        //没看懂，因为(hashedValue << this.log2m)就够了，为什么还要或还要加１？？？
        final int r = Integer.numberOfLeadingZeros((hashedValue << this.log2m) | (1 << (this.log2m - 1)) + 1) + 1;
        return registerSet.updateIfGreater(j, r);
    }

public class RegisterSet {

    //这里定义为常量，不太理解，　按理说流程应该如此：　rsd => m => log2m => 既是桶index的二进制bit位数，
    //再加上已知hash value 是64bit长，　=> 64 - p => 每个桶的二进制bit位数（必须可以存储｀除去桶部分后剩余hash串的左端前导０最大个数｀）　
    public final static int LOG2_BITS_PER_WORD = 6;  //2的6次方是64
    public final static int REGISTER_SIZE = 5; 
            /**
             * 分配(m / 6)个int32给M
             *
             * 因为一个register占五位，所以每个int（32位）有6个register
             */
    public void set(int position, int value) {
        
        int bucketPos = position / LOG2_BITS_PER_WORD; //定位到对应int32, 一个int32包含６个桶，每个桶５bit.
        //在int32中定位桶，　我觉的把M定义为：[u8]的数组更易操作，只是可能浪费一点内存！
        int shift = REGISTER_SIZE * (position - (bucketPos * LOG2_BITS_PER_WORD));
        this.M[bucketPos] = (this.M[bucketPos] & ~(0x1f << shift)) | (value << shift);
    }
}
```

> 理解上面代码需要了解对数公式：
> $$
> Math.sqrt(x) = \sqrt{x}   
> $$
>
> $$
> Math.exp(x) = e^x
> $$
>
> $$
> Math.log(x) = log_e x = ln x
> $$
>
> $$
> log_a M^n = nlog_a M  -> \ln M^n = n\ln M
> $$
>
> $$
> a^{log_a b} = b
> $$
>
> 所以函数 private static double rsd(int log2m) 表达的数学意思为：
> $$
> p = log_2(m) -> m = 2^p
> $$
> 
> $$
> rsd = \frac{1.106}{ \sqrt{e^{p * \ln 2}}}
> $$
>   
> $$
> rsd = \frac{1.106}{ \sqrt{e^{\ln{2^p}}}} =  \frac{1.106}{\sqrt{2^p}} = \frac{1.106}{\sqrt{m}}
> $$
> 注意：原来公式中常量为：１.04，　而实现者采用1.106 为常量，可能这样微调可以提高估值精度，不必纠结。
>
>   `public void set(int position, int value)`函数中位操作的意思：
>
>   where: position: [0,m) , int -> int32
>
> 比如：positon = 1, value = 5 , 计算 `int shift = REGISTER_SIZE * (position - (bucketPos * LOG2_BITS_PER_WORD));`
>
> 则shift = 5 * ( 1- (0*6) ); => 5;
>
> 现在开始位移操作定位到桶： `this.M[bucketPos] = (this.M[bucketPos] & ~(0x1f << shift)) | (value << shift);`
>
> 0x1f => 32bit integer => (00000000) (00000000) (00000000)(00011111) => 左移５bit 右补０　＝> (00000000) (00000000) (00000011) (11100000)
>
> => 看，已经定位到第１个桶（５个１），其右侧５个０为第０个桶，　其左侧为其他桶　=> 取反，消除第一个桶旧值,其他桶与１按位与，则保持不变。　
>
> ＝> (11111111) (11111111)(11111100)(00011111)　＝> value = 5 => int32 （00000000）(00000000) (00000000) (00000101) => 左移５bit,右补０
>
> => (00000000) (00000000) (00000000) (10100000) => 看，value的值已经被移动到第１个桶的位置，５bit一一对应，只要按位或就实现了将value值存入对应桶中的操作。



------

- 字节序-大小端

> 一个数字类型的长度必须>=2字节时，才有字节序的问题！比如一个4字节十六进制数：0x12abcdef，　f最低有效位，　向左依次升高，直到最高有效位１　，　这是数字本身的纸面定义与大小端字节序无关，　涉及３种情况时才需要考虑字节序：存储、传输、读写内存中数字类型值中的某个字节；虽然以16进制数举例，但是对于二进制、八进制、十进制等也是一样的！比如一个u32类型，有４个字节，每个字节８bit ,  平时编程，定义　val:u32 = 0x12abcdef; 如果我们只是整体操作val , 比如位操作，val & 0x1F等，　此时不必考虑字节序的问题，　只有当读写val中的某个字节时才需要考虑字节序的问题，　如此才能确定这个内存字节中存储了val数值的那部分？！又比如网络传输val的u32类型值需要考虑字节序的问题：首先明确网络字节序规定为大端字节序！　收到的字节流需要按本地平台字节序重新拼装才能正确排列存入val的本地内存，　如此通过变量val才能读写到正确的数值。
>
> 【大小端字节序的区分】
>
> 比如：`0x12abcdef` 这是纸面上书写的一个u32数值的十六进制形式，可是如果要存入内存中就需要考虑平台相关的字节序问题，如果平台是`大端字节序`：比如内存地址序列从低到高依次存储：  0(12) , 1(ab) , 2(cd), 3(ef) 　，　也即是说纸面数值的最高有效位存入内存最低字节地址，　而最低有效位存入内存最高字节地址！俗称从最低内存字节地址开始，按数值纸面书写顺序依次存入内存！遵循纸面书写顺序，总结为：高对低，　低对高。
>
> `小端字节序`　遵循数值有效位顺序从低到高： 0(ef), 1(cd), 2(ab), 3(12) ,　纸面数值最低有效位对应最低内存字节地址，　而纸面数值的最高有效位对应最高内存字节地址，　总结为：高对高，　低对低！
>
> HLL的数学原理与大小端字节序无关，都成立，　实际编程中bit操作粒度为数字类型整体时也不必考虑字节序，　按纸面书写顺序手动计算为准。
>
> 检测方法核心C伪码表示：　`u16 x = 0xeeff;    u8 *ptr = &x; if ( ptr[0] = 0xff) 　｛小端字节序｝　else {大端字节序}`
>
> 注意：&x 取到的内存地址始终为此变量的最低字节地址。



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
>
> `https://www.jianshu.com/p/e74eb43960a1`
>
> `https://blog.csdn.net/weixin_46233323/article/details/105242847`
>
> `https://blog.csdn.net/zdk930519/article/details/54137476`
>
> `https://zhuanlan.zhihu.com/p/58519480`





