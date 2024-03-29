## PHP扩展及嵌入(Extending and Embedding PHP)
  仅仅几年的时间，PHP已经从小众语言转变为强大的web开发工具。现在使用量超过1400万的网站使用它，PHP变得比其他语言更加稳定和易于扩展。 然而，没有关于如何扩展PHP的文档; 开发者寻求构建PHP扩展来提升其应用程序的性能和功能，只有通过PHP内核口口相传的，得过且过的一些词汇，而没有系统的有用的帮助文档。 虽然基本扩展的开发相当容易掌握， 但是更加高级的特性具有更加艰苦的学习曲线，很难克服。这在中等到较高访问网站是普遍存在的，公司只能通过雇佣更高级的人才，以及更高代价的开发人员来提升他们网站性能。 而Extending and Embedding PHP, Sara Golemon让每个PHP开发者都能掌握PHP扩展的开发， 通过PHP棘手的内核来给读者以指导。
  
  
### 简介
#### 你该不该读此书?
  你可能从书架上拿起这本书，因为你对PHP语言有一定级别的兴趣。 如果你是一名新入门的程序员，想从简单语言深入到这个领域，这本书不是你的菜。 看一看PHP and MySQL Web Development或者Teach Yourself PHP in 24 Hours. 这两本书都能让你习惯使用PHP，并且能在任何时候写应用程序。
  
  当你越来越熟悉PHP脚本的语法和结构， 你可能准备好了深入到这个主题。 用户空间函数在PHP中可用的百科知识不是必须的，但是它能帮助你了解哪些轮子无需重复制造，以及哪些被证明的设计概念可以遵循。
  
  因为PHP解释器是C实现的， 它的扩展和嵌入API也是从C语言的视角实现的。 虽然它也当然可以通过其他语言来扩展和嵌入， 这些不在本书的范围之内。 了解基本的C语法，数据类型，指针管理很重要。
  
  如果你熟悉autoconf语法，也会非常有帮助。 如果不了解，也不要担心， 你只需要指导很少一部分经验规则，在后面第17章"配置和链接"以及第18章"扩展生成器"会介绍这些规则。
  
#### 为什么需要读本书?
  该书的目标是教会你两件事情。 首先， 会展示如何通过添加函数、类、资源以及流实现来扩展PHP语言。第二，教会你如何将PHP嵌入到其他应用程序中， 使它们对用户和客户更加灵活和有用。
  
##### 为什么要扩展你的PHP?
  有四个常见的需要扩展PHP的原因。到目前为止，最常见的原因是链接外部库并暴露它的API给用户空间脚本。 这个动机在mysql扩展中可看到， 它就是将libmysqlclient类库来提供mysql_*()家族的函数给PHP脚本。
  
  这种类型的扩展就是描述PHP为胶水(glue)的开发者所引用的。扩展执行的组成代码和自身的执行没有明显的度；二十，仅仅创建了PHP扩展API和类库暴露的API间的桥梁。没有这些，PHP和诸如libmysqlclient之类的类库将不能在同一个层级通信。 下图就展示了这种类型扩展架起了第三方类库和PHP核心之间的桥梁。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/php/glue_extensions.png)

  另一个常见原因是扩展PHP执行特殊的内核操作，例如声明超级全局，这个不能在用户空间实现，因为安全限制或设计的限制。 例如apd(Advanced PHP debugger)之类的扩展，runkit执行这种仅仅内部工作，通过暴露虚拟机器执行栈的位，当然这些是在视图层是隐藏的。
  
  第三种是为了速度。PHP代码需要token化，编译，然后转向虚拟机环境， 当然不比自然代码快。特定工具集(所谓的Opcode缓存)允许脚本跳过token化和编译过程， 但是它们不能加速执行的阶段。 通过翻译为C代码，维护人员牺牲一些设计的便利，使得PHP更加强大， 但是获得的是速度增加的好多倍。
  
  最后一种，就是作者可能将多年的工作转换为一种子程序，现在想要卖给另外一个组织，但是它不希望暴露源代码。 一种方法可以使用opcode编码程序；然而，这种程序比机器代码扩展更容易被解码。 最终，为了对授权的组织有用，它们的PHP构建必须，至少，有能力访问编译的字节码。解密后的字节码是在内存中，它是从磁盘提取并显示代码的一种路子。 字节码，实际上更加容易解析到源脚本比自然二进制。更加不好的情况，比具有高速优势，实际上稍微慢一点，因为解码的阶段。
  PHP扩展的目的总结如下:
  * 作为其他类库的粘合胶水(glue), 比如libmysqlclient的粘合扩展mysql_*
  * 执行特殊的内核操作，比如超全局声明，用户空间无法实现，需要借助特殊扩展实现。
  * 提升应用速度， 牺牲一定的设计， 获得数倍的效率提升
  * 加密源代码， 授权而不售源代码
  
#### 嵌入实际上完成的什么?
  比如说你写了一个完整的应用程序，超棒、效率高、精益，通过比如C语言编译的。为了让你的应用对用户和客户端更加有用，你可能要提供一个等价的脚本， 特定行为使用比较简单的高级语言实现， 那么这些都不用担心内存管理，指针，链接或者其他复杂的部分考虑。

  如果这样的特性不很明显，考虑下你的办公产品应用，如果没有宏、批处理命令shell，那么就没有那么有用了。如果浏览器中没有javascript支持，哪些行为不可能实现? 
  
  现在你想构建自定义脚本到你的应用中， 你可以写你自己的编译器，构建一个执行框架，以及花数千个小时来调试， 或者你可以使用现成的企业级语言比如PHP，然后嵌入它的解释器到你的应用中。艰难的选择，不是吗?
  
#### What's Inside?
  本书分为三个基本话题。第一部分从内到外介绍下PHP。“彻底再了解一下PHP”。
  
  你会看到PHP解释器如何共同构建块， 了解到熟悉的用户空间概念映射到它们的内部表现。
  
  第二部分， 扩展， 你会开始构建功能性的PHP扩展，了解到如何使用PHP API额外的特性。这一部分的结束，你有能力翻译几乎所有的PHP脚本到快速的，精益的c代码。你也准备好链接在用户空间不可能的外部库和执行操作。
  
  第三部分， 嵌入， 你会从相反的角度探讨PHP。这里，你会从一个简单的应用开始，然后添加PHP脚本的支持到上面。 你会学到如何权衡safe_mode和其他的安全特性来安全执行用户提供的代码，以及多个并发请求的协调。
  
  最后，有一系列关于API调用的附录， 针对通用问题解决，以及如何查找既有扩展。
  
#### PHP与Zend
  关于PHP首要了解的是，实际上它是由五个独立部分组成的，如下图:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/php/php_anatomy.png)
  
  在堆底是SAPI(Server API)层， 协调进程的生命周期，在第一章会看到，"PHP的生命周期"。 这一层是对Web Server(比如apache， mod_php5.so等)或命令行(bin/php)的一个接口层。在第三部分，你将基于嵌入SAPI在这一层进行链接。
  
  SAPI层之上是PHP核心。 这个核心提供了关键时间和处理特定底层操作的绑定层，比如文件流、错误处理以及启动/停止触发等。
  
  核心层右边是Zend引擎， 这里是解析和编译人类可读脚本到机器可读字节码的地方。Zend也在虚拟机中执行这些字节码，在这里会读取和写入用户空间变量，管理程序流，以及定期将控制权给其他层，例如在函数调用过程中，Zend也提供每个请求的内存管理，以及一种用于环境操作的鲁莽接口。
  
  PHP核心和Zend层之上的是扩展层，这里你可以发现很多在用户空间可用的函数。有些扩展是被默认编译的(例如standard, pcre, session等)，通常也不被认为是扩展。 其他的扩展是可选的构建到PHP中的，使用./configure选项比如with-mysql, enable-sockets,或者构建共享模块，然后在php.ini中加载，使用extension=xxx.so或者在用户空间使用dl()返回加载。将在后面第二部分和第三部分开开发这一层。同时执行扩展和嵌入。
  
  包围整个的是线程安全层，即TSRM(thread safe resource management)层。 PHP解释器的这部分允许单个PHP实例同时执行多个独立的请求而互不影响。幸运的是，这一层在视图层都是通过宏功能隐藏的，在这本书中你会逐渐熟悉这块的。
  
#### 扩展到底是什么?
  扩展就是一包离散代码，可以插入到PHP解释器提供功能的脚本集合。扩展一般暴露至少一个函数、类、资源类型或流实现，通常是这些数个这些的组合。
  
  最常用的扩展是standard, 定义了超过500个函数， 10个资源类型， 2个类以及5个流包装。 这个扩展和zend_buildin_functions扩展，通常编译到PHP解释器而无需考虑编译配置项。 另外的扩展，比如session, spl, pcre, mysql, sockets， 可以通过编译配置项目启用或禁用， 或者可以使用phpize工具单独编译进去。
  
  每个扩展(或模块)共享的一个结构是zend_module_entry结构，定义在PHP代码包的Zend/zend_modules.h中，这个结构是PHP引入扩展以及定义启动，关闭方法的入口点，在第一章会详细介绍。这个结构也引用了一个zend_function_entry结构的数组， 定义在Zend/zend_API.h中，这个数组，正如数据类型所示， 列出了扩展暴露的函数列表。
  
  ![](https://github.com/walkerqiao/walkman/blob/master/images/php/php_extensions_starting_point.png)
  
  在第六章你可以升入检查下这个结构， 当你构建自己的扩展的时候。
  
#### 如何使用PHP完成嵌入?
  通常，PHP解释器被链接到将脚本请求穿梭到解释器的进程，然后解析结果返回。
  
  CLI SAPI通过对解释器和命令行进行简单的包装实现的，而Apache SAPI暴露的是正确的钩子，比如apxs模块。
  
  可能会尝试嵌入PHP到应用程序中，使用自定义的SAPI模块。幸运的是，完全没有必要。从4.3版本以来， 标准PHP都包含一个embed的SAPI, 它允许PHP解释器扮演一个普通的动态链接库，可以包含在任何应用程序中。
  
  在第三部分，你会看到任何应用可以利用这个强大而又便利的PHP代码，通过使用这个简单简洁的类库。
  
#### 整书使用的术语集
  * PHP: 引用整个PHP解释器，包括Zend, TSRM, SAPI层以及扩展
  * PHP Core: PHP解释器的子集，参见上图
  * Zend: Zend引擎
  * PEAR:
  * PECL
  * PHP扩展
  * Zend扩展

------------------------------

## 目录
  * 第一章 [PHP生命周期](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_01.md)
  * 第二章 [变量的里里外外](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_02.md)
  * 第三章 [内存管理](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_03.md)
  * 第四章 [设置构建环境](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_04.md)
  * 第五章 [第一个扩展](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_05.md)
  * 第六章 [返回值](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_06.md)
  * 第七章 [接收参数](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_07.md)
  * 第八章 [数组和哈希表使用](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_08.md)
  * 第九章 [资源数据类型](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_09.md)
  * 第十章 [PHP4对象](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_10.md)
  * 第十一章 [PHP5对象](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_11.md)
  * 第十二章 [启动、停止以及之间的一些点](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_12.md)
  * 第十三章 [INI设置](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_13.md)
  * 第十四章 [流访问](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_14.md)
  * 第十五章 [流实现](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_15.md)
  * 第十六章 [流分流](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_16.md)
  * 第十七章 [配置和链接](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_17.md)
  * 第十八章 [扩展生成器](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_18.md)
  * 第十九章 [设置主机环境](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_19.md)
  * 第二十章 [高级嵌入](https://github.com/walkerqiao/walkman/blob/master/docs/php/chapter_20.md)
  * 附录A: Zend API参考
  * 附录B: PHP API
  * 附录C: 扩展和嵌入菜谱
  * 附录D: 额外资源
