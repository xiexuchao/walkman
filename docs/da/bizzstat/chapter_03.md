## 描述数据: 集中趋势的度量

### 学习目标
  通过本章学习，你应该能够:
  1. 计算算术平均数、加权平均数、中位数、众数和几何平均数
  2. 解释各种度量的特点、用途、优点和缺点
  3. 对于对称与有偏分布， 识别算术平均、中位数和众数的位置

### 引言
  本章将通过寻找描述一组数据的单一的典型值来进一步学习描述数据的方法。 我们称这个典型值为集中趋势的度量(measure of central tendency).
> 集中趋势的度量 概括一组数据的一个单一数值。 它位于所有数据的中心。

  集中趋势的度量不只一种，实际上，它的种类很多。我们将要介绍其中的5种: 算术平均值、加权平均值、中位数、众数和几何平均数。

### 总体均值
  许多研究涉及到一个总体种的全部数据。对于原始数据，也就是还未分组前作出频数分布或者茎叶图的数据， 总体均值就是将总体种的全部数据求和然后再除以总体中数值的个数。 为了计算总体均值，我们使用下面的公式:
  总体均值 = 总体中的全部值之和/总体中数值的个数
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/bizzstat_total_avg.png)
  
  其中:
  μ: 总体均值。 它是小写的希腊字幕"mu"
  N: 总体中单元的个数
  X: 任意的特定值
  ∑: 大写希腊字母"sigma", 它表示求和运算
  ∑X: 表示X值的总和
  
  总体的任何可以度量的特征都称作参数(parameter). 总体的均值就是一个参数。
> 参数 总体的一个特征

### 样本均值
  未分组样本数据的均值
  样本均值 = 样本中全部数值之和/样本中数值的个数
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/bizzstat_sample_avg.png)
  X-bar表示样本均值。小写n表示样本容量。
  
  样本的均值以及其他基于样本数据的度量称作统计量(statistic). 如果一个滚球轴承样本的外径平均值未0.625英寸，这就是一个统计量的例子。
> 统计量 样本的一个特征

### 算术平均值的性质
  算术平均值是一个广泛使用的集中趋势的度量尺度。 它具有一下的重要性质:
  1. 所有定距尺度的数据都有一个均值。 比如收入、年龄和重量之类的数据
  2. 在计算均值时，所有的数据都包含在内。
  3. 一组数据只有一个均值，均值具有唯一性。
  4. 均值时比较两个或多个总体时一个非常有用的工具。
  5. 每一数据相对于平均值的偏离之和总是为0，算术平均值是唯一一个具有词性质的集中趋势的度量方法。 用符号表示为:∑(X - X-bar) = 0
  均值作为平衡点
  因此我们可以将均值视作一组数据的平衡点。

  缺点是: 
  1. 均值受到极端大或者极端小的值影响。
  2. 开口组的数据无法确定均值， 比如100 000美元以上

### 加权平均数
  一般的，用X<sub>1</sub>, X<sub>2</sub>, ..., X<sub>n</sub>表示的一组数据，他们相应的权术分别为w<sub>1</sub>, w<sub>2</sub>, ..., w<sub>n</sub>, 则它们的加权平均数的计算公式为:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/bizzstat_avg_with_wight.png)
  
  上面的公式可以简化为:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/bizzstat_avg_with_wight_simple.png)

### 中位数
  前面的内容中，我们已经提及，当一组数据存在一两个过大或过小的值时，算术平均值就不具有代表性了。 对于这样的数据，可以使用另外一种度量方法来描述其集中趋势， 这种方法称为中位数(median). 
  
  为了说明有时需要除了算术平均值以外的集中趋势度量方法，假设你正打算在Palm Aire购买住户共有公寓。 你的房地产代理人说现在每个单元的平均价格为110 000美元，你是否还要进一步考察呢? 如果你所做的预算中最高价格在60 000美元~75 000美元，你可能会认为报价超出了你的价格限制。但是，考察各个单元的价格你可能会令你改变想法，它们分别为60 000, 65 000, 70 000, 80 000美元，以及最为豪华的顶层住宅的275 000美元。正如房地产代理人所说的，价格的算术平均值为110 000美元，但是一个价格275 000将算术平均值大大提升了，使其成为了一个不具有代表性的平均数。确实，在65 000～75 000美元的价格才会是一个具有代表性的平均数。在上述的这种情况下，中位数将作出对集中趋势的更精确的度量。
  
> 中位数 将一组数据按照生序或者降序排列后，位于中点的值。 50%的观察在中卫数以上， 50%的观察在中位数以下。数据必须至少为定序尺度的数据。

  要得到中位数，首先需要按一种序列对数据进行排序，然后找到中位数。
  
  中位数不受极值影响 中位数以上的价格和中位数以下的价格个数相等或相当。
  
  当然观察数个数为奇数的时候，直接可以取中位数， 而个数为偶数的时候，取中间的两个观察数的算术平均值。因此个数为偶数的时候可能不是任何一个观察数其中的一个。
  
  中位数的主要性质是:
  1. 中位数具有唯一性。 也就是说，同均值一样一组数据只有一个中位数。
  2. 中位数不受极端大值或极端小值影响。所以当出现这类值时，它是集中趋势的一个十分有价值的度量方法。
  3. 对于含开口组的频数分布也可计算中位数，只要中位数不在开口组中即可。(后面我们将介绍如何计算分组数据的中位数) 除列名尺度的数据外，其他水平的数据都可计算中位数。
  4. 定比、定距和定序尺度的数据都可以计算中位数。 比如，一个人认为很好，一个人认为好， 一个认为一般，一个认为差， 那么中位数是一般， 50%的回答在好以上，50%的回答在好以下。

### 众数(mode)
  众数是另外一种集中趋势的度量方法。即出现频率最高的观察值。
  
  众数在对列名尺度和定序尺度的数据的描述中尤为有用。
  例如，一家公司开发了5种洗浴油， 其中喜欢Lamoure的人最多，因此Lamoure就是众数。
  
  总之，我们可以确定所有尺度水平(列名、定序、定距以及定比)的数据的众数。 众数也具有不受极大值或极小值影响的优点， 可以作为含有开口组的分布的集中趋势的度量方法。
  
  众数的缺陷:
  众数有一定的缺陷，因此相对于均值和中位数应用的要少些。 对于许多销售数据众数不存在，因为所有的观察者都值出现一次。而有些情况下，众数可能是多个值， 多个值同时出现的频率相同， 这时候，众数就让人比较困惑。

### 几何平均数
  几何平均数总是不大于算术平均数。
  在计算百分比、定比、指数以及增长率的平均值时，几何平均数(geometric mean)非常有用。 由于我们常常对销售、工资或者GNP等经济数字的变化百分比很感兴趣，所以几何平均数在商业商和经济商有着十分广泛的应用。 n个整数的几何平均数被定义为这n个值的乘积的n次方根， 其计算公式可以表示为
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/bizzstat_goemetric_mean.png)
  
  几何平均数总是小于或等于(从不大于)算术平均数。此外，在计算几何平均数时所有的数值必须为正。
  
  假设你今年的薪水增加了5%， 明年将增加15%, 则平均的增长是9.886%而不是10%, 为什么呢? 我们首先计算几何平均值，工资增长了5%,就得到1.05， 1.05 * 1.15, 然后再对结果取2次方根，得到1.09886.
  
  假设起薪为3000， 则两次加薪幅度分别为5%和15%， 那么:
```
  3000 * 0.05 = 150.00
  3150 * 0.15 = 472.50
-----------------------
总计            622.50

总共加薪 622.50美元，这相当于:
  3000.00 * 0.098 86 = 296.58
  3296.58 * 0.098 86 = 325.90
-----------------------------
总计                   622.48
```

  几何平均数一般用于对百分比、定比、指数以及增长率之类的观察值进行求平均数。
  
  几何平均数的第二个应用是计算一段时间内平均的增长百分比。例如，你在1990年的粘性为30 000美元，2000年的年薪为50 000美元，则在该段时间内你的薪资的年增长率是多少? 使用以下的公式可以计算出增长率。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/bizzstat_period_grow_rate.png)
  
  在上面的表达式中， n代表时期数。

### 分组数据的均值、中位数和众数
  收入、年龄等数据经常被分组并且以频数分布的形式出现，原始数据常常无法得到。因此，如果我们对数据的某些代表性数值感兴趣，就必须基于频数对其进行估计。
  
#### 算术平均数
  要估计以频数分布的形式出现的数据的算术平均数，我们首先假定每组内的数据都可以由改组的组中值代表。频数分布样本数据的均值按下面的公式计算:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/bizzstat_group_data_avg.png)
  其中:
  X-bar: 算术平均数的符号
  X: 各组的组中值
  f: 各组的频数
  fX: 各组的频数与组中值的乘积
  ∑fX: 组中值与频数乘积求和
  n: 频数的总和
  
#### 中位数
  中位数被定义为一般数值在其之上，另一半值在其之下的那个值。 当原始数据被分组形成频数分布后，一些信息就无法识别了。因此我们无法确定实际的中位数，但是可以通过估计得到它:
  1. 确定中位数所在的组
  2. 在改组内通过插值得到中位数。
  
  这个方法的基本原理是中位数所在组的各个数值被假定为在组内是均匀分布的， 计算公式如下:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/bizzstat_group_data_midum.png)

  包含有开口组的频数分布也可以确定中位数
  
  最后，一个值得注意的地方是，中位数仅仅基于中位数所在组的频数和组限。在最末端出现的开口组很少被用到。因此，包含有开口组的频数分布的中位数也可以被确定。包含有开口组的频数分布数据的算术平均值就无法精确计算。 当然，除非开口组的中点可以估计。进一步，如果给出了频数百分比而非实际的频数，中位数也可以确定。这是因为中位数是这样的一个值--50%的值在其之上，50%的值在其之下，它并不依赖于实际的频数。百分比可以视作为实际频数的替代品。 在某种意义下，可以认为它们就是总和为100的实际频数。
  
#### 众数
  众数是出现频率最多的值。 对于分组的频率分布数据，可以用频数最高组的组中值来近似。
  有时可能有两个数值出现的频数都最大，这样的分布被称为双峰分布(multimodal distribution). 

### 均值、中位数和众数的相对位置
  对于对称的钟形分布，均值、中位数位于中间并且总是相同的。从逻辑上来说，这三个度量值中任何一个作为该分布的代表值都是恰当的。
  
  一个有偏的分布是不对称的
  如果一组数据不是对称的，或者称为有偏的(skewed), 这三个度量值之间的关系就会发生改变。 在正偏分布(positively skewed distribution)中，算术平均值是三者中最大的。为什么呢? 因为比起中位数和众数，它受到少数极端大值的影响最大。中位数通常在正偏的分布中是第二大的度量值。众数在三者中最小。
  
  正偏分布 众数 < 中位数 < 算术平均值
  如果偏移程度很大， 均值将不能作为一个很好的度量。 相对而言，中位数和众数的代表性更强。
  
  相反的，在一个负偏(negatively skewed)的分布中，均值在三者之中是最小的。 显然，均值将受到极端小的观察值的影响。中位数比均值大， 众数在三者之中是最大的。 同样的，如果分布的偏斜程度很大， 均值不应作为数据的代表使用。
  
### 本章小节
  

