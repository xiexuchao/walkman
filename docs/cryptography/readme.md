## 现代密码学理论与实践

### 第一部分 引言
  本书第一部分由两个引论性的篇章组成。主要为我们介绍密码学和信息安全中的一些基本概念；我们进行通信与处理敏感信息的环境；在该环境下参与演出的几位著名"人物"以及其中扮演坏家伙的标准伎俩；密码学和信息安全系统研究与发展领域的社团文化，以及这些系统极为容易出错的现实。
  
  作为初级引论，这一部分所针对的读者是希望进入这一领域的初学者。

#### 第1章 [一个简单的通信游戏](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_01.md)

#### 第2章 [防守与攻击](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_02.md)

### 第二部分 [数学基础]
  这一部分收集了一些数学资料，给出了基本概念、方法、代数运算的基础、算法过程的基本模块，以及在本书其余部分出现的有关建模、说明、分析、变化和解决不同问题的参考。
  
  该部分包括四章:概率论和信息论(第3章)、计算复杂度(第4章)、代数基础(第5章)和数论(第6章)。这部分内容是一个自足的数学参考指南。在本书的其余章节中，我们所遇到的重要数学问题，都可以在这四章的确切位置上找到该问题的支持事实或基础。因此，本书中我们选用的数学材料的方式有助于读者以主动、交互的方式学习现代密码学的数学基础。
  
  对于那些在现代密码学的理论基础和实际应用中非常重要的算法和定理，我们将予以特别注意，并对其进行足够详细的说明。如果我们认为某个定义的证明能够帮助读者提高与学习本书中的密码学主题相关的技巧，我们将给出该定理的证明。有时，我们的数学主题发展到不得不利用来自于其他数学分支的事实(如线性代数)，而这些内容与本书所讲的密码学技巧没有直接的关系；在这种情况下，我们将简单地实用所需地事实，而不给出证明。
  
  标准符号
  下面这些标准符号通用于本书的其余部分。一些符号将在其第一次使用时给出定义，另一些符号将不再进一步说明。
  * ∅: 空集
  * S U T: 集合S和T的并集
  * S ∩ T: 集合S和T的交集
  * S \ T: 集合S和T的差集
  * S ⊆ T: S是T的子集
  * #S: 集合S中的元素数目(比如#∅ = 0)
  * x ∈ S, x ∉ S: 元素x属于(不属于)集合S
  * x ∈<sub>u</sub> S: 均匀随机的在集合S中选取元素x
  * x ∈ (a, b), x ∈ [a, b]: x属于开区间(a, b)(x属于比区间[a, b])
  * N, Z, Q, R, C: 自然数集、整数集、有理数集、实数集和复数集
  * Z<sub>n</sub>: 模n的整数
  * Z<sup>*</sup><sub>n</sub>: 模n的整数乘法群
  * F<sub>q</sub>: q个元素的有限域
  * desc(A): 代数结构A的描述
  * x←D: 根据分布D进行赋值
  * x←<sub>u</sub>S: 按S为均匀分布进行赋值
  * a(mod b): 模运算:a被b除所得的余数
  * x|y, xΧy: 整数y可被(不可被)整数x整除
  * ≡: 定义为
  * ∀: 对所有的
  * ∃: 存在
  * gcd(x, y): x和y的最大公因子
  * lcm(x, y): x和y的最小公倍数
  * log<sub>b</sub>x: x以b为底的对数；b省略则表示自然对数
  * ⌊x⌋: 小于或等于x的最大整数
  * ⌈x⌉: 大于或等于x的最小整数
  * |x|: 整数x的长度(= 1 + ⌊log<sub>2</sub>x⌋, x≥1), 也表示x的绝对值
  * Φ(n): n的欧拉函数(Euler)
  * λ(n): n的

#### 第3章 [概率论和信息论](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_03.md)

#### 第4章 [计算复杂性](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_04.md)

#### 第5章 [代数学基础](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_05.md)

#### 第6章 [数论](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_06.md)

### 第三部分 基本的密码学技术

#### 第7章 [加密--对称技术](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_07.md)

#### 第8章 [加密--非对称技术](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_08.md)

#### 第9章 [理想情况下基本公匙密码函数的比特安全性](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_09.md)

#### 第10章 [数据完整性技术](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_10.md)

### 第四部分 认证

#### 第11章 [认证协议--原理篇](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_11.md)

#### 第12章 [认证协议--实践篇](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_12.md)

#### 第13章 [公匙密码的认证框架](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_13.md)

### 第五部分 建立安全性的形式化方法

#### 第14章 [公匙密码体制的形式化强安全性定义](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_14.md)

#### 第15章 [可证明安全的有效公匙密码体制](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_15.md)

#### 第16章 [强可证明安全的实用公匙密码体制的文献注记](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_16.md)

#### 第17章 [分析认证协议的形式化方法](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_17.md)

### 第六部分 密码学协议

#### 第18章 [零知识协议](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_18.md)

#### 第19章 [回到"电话掷币"协议](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_19.md)

#### 第20章 [结束语](https://github.com/walkerqiao/walkman/blob/master/docs/cryptography/chapter_20.md)
