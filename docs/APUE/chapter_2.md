## 第二章 Unix标准化及实现

### 2.1 引言(Introduction)
  Unix和C程序设计语言的标准化工作已经做了很多。虽然Unix应用程序在不同的Unix版本之间进行移植相当容易，但是80年代Unix版本的剧增以及它们之间的差别的扩大导致很多大用户要求对其进行标准化。
  
  本章将介绍正在进行的各种标准化工作,然后讨论这些标准对本书所列举的实际Unix实现的影响。所有标准化工作的一个重要部分是对每种实现必须定义的各种限制的说明,所以我们将说明这些限制以及确定它们值的多种方法。

### 2.2 Unix标准化
#### 2.2.1 ANSI C
  1989年后期, C程序设计语言的ANSI标准X3.159-1989得到批准[ANSI1989]。此标准已被采用为国际标准ISO/IEC 9899:1990.ANSI是美国国家标准学会(American National Standards Institute), 它是由制造商和用户组成的非赢利性组织。在美国, 它是全国性的无偿标准交换站, 在国际标准化组织(ISO)中是代表美国的成员。
  
  NSI C标准的意图是提供C程序的可移植性,使其能适合于大量不同的操作系统,而不只 是 U N I X 。此标准不仅定义了 C程序设计语言的语法和语义,也定义了其标准库〔 A N S I 1 9 8 9 第 4 章; P l a u g e r 1 9 9 2 ; K e r n i g h a n 及 R i t c h i e 1 9 8 8 中的附录 B 〕。因为很多新的 U N I X 系 统 ( 例 如 本 书 介绍的几个 U N I X 系统)都提供 C标 准 中 说 明 的 库 函 数 , 所 以 此 库 对 我 们 来 讲 是 很 重 要 的 。
  
  按 照 该 标 准 定 义 的 各 个 头 文 件 , 可 将 该 库 分 成 1 5 区。表 2 - 1 中列出了 C标 准 定 义 的 头 文 件 , 以及下面几节中说明的另外两个标准 ( P O S I X . 1 和 X P G 3 ) 定义的头文件。在其中也列举了 S V R 4 和 4 . 3 + B S D所支持的头文件。本章也将对这两种 U N I X 实现进行说明。
  
  ![ISO C](https://github.com/walkerqiao/walkman/blob/master/images/isoc_std.png)
  
  ![ISO C](https://github.com/walkerqiao/walkman/blob/master/images/iso_c1.png)
  ![ISO C](https://github.com/walkerqiao/walkman/blob/master/images/iso_c2.png)

### 2.3 Unix系统实现

### 2.4 标准化和实现之间的关系

### 2.5 限制

### 2.6 选项

### 2.7 特性测试宏

### 2.8 原始系统数据类型

### 2.9 标准间的差异

### 2.10 总结
