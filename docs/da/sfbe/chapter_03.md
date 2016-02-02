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
  
