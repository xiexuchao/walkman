### 第三章 描述性统计: 数值度量

  内容
  统计实践: SMALL FRY DESIGN
  3.1 位置度量
  * 均值(Mean)
  * 中位值(Median)
  * 众数(Mode)
  * 百分位数(Percentiles)
  * 四分位数(Quartiles)
  
  3.2 变化率度量
  * 范围(Range)
  * 四分位范围(Interquartile Range)
  * 方差(Variance)
  * 标准差(Standard Deviation)
  * 方差系数(Coefficient of Variation)
  
  3.3 分布形状、相对位置以及极值检测度量
  * 分布形状
  * z评分
  * 契比雪夫定理
  * 经验法则
  * 极值检测
  
  3.4 探索性数据分析
  五数概括箱线图(最小值、第1四分位数(Q1)、中位数(Q2)、第3四分位数(Q3)、最大值)
  
  3.5 两变量间关系度量
  * 协方差(Covariance)
  * 协方差解释
  * 相关系数(Correlation Coefficient)
  * 相关系数解释
  
  3.6 分组数据的加权均值
  * 加权平均数
  * 分组数据
  
### 3.1 位置度量
  均值: 提供数据中心位置的度量方法。对于样本来说，均值使用x̄表示，而对于总体来说使用μ表示。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_sample_mean_formula.png)
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_population_mean_formula.png)

  中位数: 数据按生序排列后位于中间的值。奇数个元素，中位值就是中间的那个元素的值；对于偶数个元素，中位值就是中间两个元素的均值。
  
  众数(Mode): 出现频率最多的值。
  众数缺点:可能存在多个众数。
  
  百分位数: 提供了关于数据从最小值到最大值是如何分散的信息。适合于没有重复值的序列，第p百分位数将数据分为两部分: 百分之p的数值小于第p百分位数；百分之100-p的数值大于第p百分位数。
  
  第p百分位数是这样一个值， 至少有百分之p的观察值小于这个值，而至少百分之100-p的观察值大于这个值。
  
  第p百分位数计算:
  1. 将数据按从小到大排序
  2. 计算索引i: i = (p/100) * n, 这里的p就是感兴趣的p百分位数，n是观察量的数量。
  3. 如果i不是整数(小数)，那么round up这个值，获取比该值大的最小整数就是p百分位数；如果i是整数，那么p百分位数就是第i个观察值和第i+1个观察值的均值。
  
  四分位数(Quartiles): 是p百分位数的特例。将整个观察值列表分为四个均匀的等份。分别用Q1, Q2, Q3表示25百分位数，50百分位数(中位数)，75百分位数。

> 注意
> 如果数据存在极大值或极小值，使用中位值比使用均值要更好反映观察值的特性。 有时候另外一种度量方法是修剪的均值。比如:
> 5%修剪均值是通过将5%最小的和5%最大的之后计算的新序列的均值。

### 3.2 差异度量
  除了位置度量，通常还希望对变量差异或离散程度进行度量。
  
  范围度量(Range): 最简单的差异度量方式。 Range = 最大值 - 最小值
  
  四分位范围(Interquartile Range): IQR 克服对极值依赖性的一种差异度量方式是使用四分位范围。
  四分位范围是Q3和Q1之间的差异。 换句话说就是中间50%的数据的范围度量。 IQR = Q3 - Q1
  
  方差(Variance): 利用所用数据的差异度量方法。 方差是基于每个值和均值的差异。每个xi和均值的差异叫做平均偏差.
  
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_population_variance_formula.png)
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_sample_variance_formula.png)
  
### 3.3 分布形状、相对位置以及极值检测度量。
  一个重要的分布形状的数字化度量叫做偏度系数(skewness)。
  偏度系数是描述分布偏离对称性程度的一个特征数。当分布左右对称时，偏度系数为0。当偏度系数大于0时，即重尾在右侧时，该分布为右偏。当偏度系数小于0时，即重尾在左侧时，该分布左偏。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_sample_skewness_formula.png)
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_skewness_demo.png)
  
  除了对位置、变异性以及形状的度量，我们也会对值在集合中的相对位置感兴趣。 度量相对位置可以帮助我们确定特定值与均值的距离。
  
  通过使用均值和标准差，我们能确定任何观察量的相对位置。
  
  在统计中，变量值与其平均数的离差除以标准差后的值，称为标准分数(standard score),也称为标准化值或Z分数。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_zscore_formula.png)
  
  切比雪夫定理
  切比雪夫定理让我们能够确定多少比例的数据值位于均值指定数量个标准差之内。

> 切比雪夫定理 至少有(1 - 1/z^2)的数据值必须位于均值的z个标准差之间， z是任意大于1的值。

  这个定理蕴含着下面的一些情况，z = 2, 3, 4个标准差的情况:
  * 至少有75%的数据值必须位于均值的2个标准差之间
  * 至少有89%的数据值必须位于均值的3个标准差之间
  * 至少有94%的数据值必须位于均值的4个标准差之间

  例如使用切比雪夫定理， 假设100名学生的期中考试成绩平均为70分， 标准差为5， 那么分数在60到80之间的学生比率有多少呢? 
  这个例子中， z = 10 / 5 = 2, 那么应该至少有75%的学生成绩在60到80之间。
  
  经验法则(Empirical Rule)
  对于数据分布为钟形的，可以使用经验法则来确定多少比例的数据必须位于均值的特定数量的标准差之间。
  
  经验法则:(具有钟形分布的数据)
  * 68%的数据位于均值的1个标准差之间
  * 95%的数据位于均值的2个标准差之间
  * 几乎所有的数据位于均值的3个标准差之间
  
  极值检测(Detecting Outliers)
  有时候数据集有一个或多个观察值具有异常大或异常小的值。 

  这些极端值叫做极值。有经验的统计学家采取步骤来标示极值，然后仔细对待每个极值。
  极值可能是不正确的值。如果是的话，可在将来分析的之前修正它们。
  极值可能是观察值不正确的包含进来的，如果是这样的话，可以删除掉。
  极值也可能是记录正确，是包含在数据集中的不常见值。这种情况需要保留。

  标准值(z分数)可以用来标示这些极值。 回忆前面的经验法则， 钟形分布的数据集几乎所有的值都包含在均值3个标准差之间， 我们推荐任意数据值的z分数小于-3或大于3的称为极值。
  
  使用z分数可以知道标准值的z分数位于-3, 3之间的是好的， 但是不能确定极值出现在类尺寸内的数据。
  
### 3.4探索性数据分析
  在第二章中，我们介绍了茎叶图作为一种探索性分析的技术。回忆一下探索性数据分析可以让我们使用简单的算术和容易绘制的图来总结数据。 这一节我们继续探索性数据分析， 下面考虑五数概括和箱线法。
  
  五数概括(Five-Number Summary)
  在五数概括中，下面的五个数字用于汇总数据:
  * 最小值
  * 四分之一数Q1
  * 中值数Q2
  * 四分之三数Q3
  * 最大值
  
  最简单的开发五数概括的方法是将数据集已升序方式排列，找出最小值、最大值以及三个Q1,Q2,Q3数。

  箱线图
  箱线是一种图形化的数据总结，基于五数汇总。 开发箱线图的关键在于计算中位值和四分数Q1,Q3. 四分范围IRR = Q3 - Q2, 也被使用。图3.5是一个箱线图的例子。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_box_plot_example.png)
  
### 3.5 两变量间的相关度量
  到目前为止，我们都是以一次一个变量的形式总结数据的。 通常对于管理者来说，对两个变量之间的关系会比较感兴趣。 这一节我们将介绍协方差和相关系数来描述多个变量之间的关系。
  
  协方差(Covariance)
  对于样本具有n个观察值(x1, y1), (x2, y2),等等，样本的协方差定义如下:
  
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_covariance_sample.png)
  
  总体的协方差定义:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_covariance_sample.png)
  
  协方差的解释
  为了解释样本协方差，考虑下面的图3.9. 这个是和3.7图的三点图是一样的， 不过添加了两条附加线.
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_scatter_for_covariance.png)
  
  数据点被分成四个象限， 其中第一象限的x大于x的均值，y大于y的均值；第二象限中x小于x的均值，但y大于y的均值；第三象限，x,y均小于相应的均值；第四象限x大于x均值，但y小于y的均值。
  
  如果协方差s<sub>xy</sub>为正数，那么对这个协方差影响较大的点必须位于第一象限和第三象限。 因此协方差的正值表明x和y的正线性相关，也就是说随着x的增加，y也随着增加。
  如果s<sub>xy</sub>为负数， 那么对这个协方差影响较大的点必须在第二象限或第四象限。因此负的协方差表明x和y之间是负线性相关的，随着x的增加，y减少。
  最后一种情况。如果点均匀分布到四个象限中，那么协方差接近0， 表明x和y之间没有线性相关。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_covariance_interpretation.png)
  
  通过前面的讨论，很明显较大的正协方差表明较强的正线性相关；而较大的负协方差表明较强的负线性相关。然而，使用协方差存在一个问题，就是使用它度量两个变量之间的线性关系，协方差的值和两个变量的度量单位有关系。 例如，假设我们对高度x和体重y的关系比较感兴趣。 显然关系强度应该相同，而不管度量重量的单位是英尺还是英寸。 度量高度用英寸， 然而，那么我们就会获取较大的x-x_avg,相较于体重使用英尺。 因此可能获取较大的协方差，但事实上，两者的关系却是不变的。
  
  一句话说，使用不同的单位得到同一种关系的两种不同的协方差值。 这个时候似乎用协方差不能很好的体现两者关系。那么我们提供了另外一种对使用度量单位无关的两者线性关系度量的方式，称为相关系数。
  
  相关系数(Correlation Coefficient)
  对于样本数据，皮尔森积矩相关系数定义如下:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_pearson_product_moment_correlation_coefficient.png)
  
  相关系数解释
  一般来说，如果数据所有点都落到正的斜率的直线上，那么样本相关系数就是+1. 也就是说，样本相关系数为+1, 相应于x和y的完美正线性相关。
  如果点都落在一个具有负的斜率的直线上，那么样本相关系数为-1. 样本相关系数是-1, 表明x,y是完美的负线性相关。
  
  下面我们假设特定数据集表明x,y之间是正线性相关，但是关系不是最完美的线性相关。相关系数小于1，表明在散点图中的点步完全在一条直线上。 随着点越来越偏移完美线性相关，r<sub>xy</sub>的值变得越来越小。 r<sub>xy</sub>等于0就表明x和y之间不存在线性相关， 相关系数越接近0就越表明两个变量之间是越弱的线性相关。
  
  结束的时候，我们注意到，相关性提供了一种线性相关的度量方法，但是并不是必然的因果关系。
  两个变量之间较高的相关性并不代表一个变量的改变会导致另外一个变量的改变。 
  
### 3.6 加权平均值以及分组数据的情况
  加权均值的公式:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_weighted_mean.png)
  
  分组数据
  很多情况下，位置和变异性度量，都是使用独立数据值计算的。但有些时候，只有分组数据或频率分布形式的数据可用。下面的讨论，我们将展示加权均值公式是如何用于获取分组数据大概的均值、离散性以及标准差。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_sample_grouped_data_mean.png)
  
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_sample_grouped_data_variance.png)
  
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_population_grouped_data_mean_variance.png)
  
  
### 总结
  本章我们介绍了几种描述性统计可用于位置、差异以及数据分布形状方面的汇总。下面是总结列表:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/da/sfbe_summary_3_statistics.png)
  
  
