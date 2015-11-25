## 如何编写一个简单的makefile
  程序设计的技术通常遵循着一个极为简单的惯例:编写源代码文件、将源代码编译成可执行的形式以及对成果进行调试。 尽管将源代码转换成可执行文件被视为惯例，但如果程序员的做法有误也可能会浪费大量的时间在调试上。大多数开发者都经历过这样的挫败: 修改号一个函数之后，运行新的程序代码时却发现缺陷并未被修正，接着发现再也无法执行这个经过修改的函数，因为其中某个程序有误，项无法重新编译源代码、重新链接可执行文件或者是重新建造jar文件。此外，当程序变得越来越复杂时，这些例行工作可能会因为需要(针对其他平台或程序库的其他版本等)未程序开发不同的版本而变得越来越容易发生错误。
  
  make程序可让"将源代码转换成可执行文件"之类的例行工作自动化。相较于脚本，make的优点是: 你可以把程序中各元素之间的关系告诉make, 然后make会根据这些关系和时间戳判断应该重新进行哪些步骤，以产生你所需要的程序。有了这个信息，make还可以优化编译过程，跳过非必要的步骤。
  
  GNU make(以及make的其他变体)可以准确完成此事。make定义了一种语言，可用来描述原文件、中间文件以及可执行文件之间的关系。它还提供了一些功能，可用来管理各种候选配置、实现可重用程序库的细节以及让用户以自定义宏将过程参数化，简言之，make常被视为开发过程的核心，因为它为应用程序的组件以及这些组件的搭配方式提供了一个可依循的准则。
  
  make一般会将工作细节放在一个名为makefile的文件中。下面事一个可用来编译传统hello world程序的makefile：
```
hello: hello.c
  gcc hello.c -o hello
```
  要编译此程序，可以在命令行提示符之后键入: make 以便执行make. 这将会使得make程序读入makefile文件，并且编译它在该处所找到的第一个工作目标(target):
  make ---> gcc hello.c -o hello
  
  如果将某个工作目标(target)指定成命令行参数(command-line argument), make就会特别针对该工作目标进行更新的动作；如果命令行上未指定任何工作目标，make就会采用makefile文件的第一个工作目标，称为默认目标(default target).
  
  在大多数makefile文件中，默认的目标一般就是编译程序，这通畅设计许多步骤。程序的源代码经常是不完整的，而且必须使用flex或bison之类的工具来产生源代码。接着，源代码必须被编译成二进制目标文件(binary object file)--.o文件用于C/C++、.class文件用于Java等。然后，对C/C++而言，连接器(通常调用自gcc)会将这些目标文件链接在一起形成一个可执行文件。
  
  修改源文件中国年的任何内容并重新调用make, 将会使得这些步骤中国年的某些(通常不是全部)被重新进行，因此源代码中国年的变更会被适当地并入可执行文件。这个规范说明书文件(specification file)或makefile文件中， 描述了源代码文件、中间文件以及可执行文件之间地关系，使得make能够以最少的工作量来完成更新可执行文件的工作。
  
  所以，make的主要价值在于它有能力完成编译应用程序时所需要的一些复杂步骤，以及当有可能缩短"编辑-编译-调试"(edit-compile-debug)周期对这些步骤进行优化的动作。此外,make极具灵活性，你可以在任何具有文件依存关系的地方使用它，范围从C/C++到Java、TeX、数据库管理等。
  
### 工作目标与必要条件
  基本上,makefile文件中包含了一组用来编译应用程序的规则。make所看到的一项规则，被视为默认规则(default rule)使用。一项规则可以分成三个部分: 工作目标(target)、它的必要条件(prerequisite)以及所要执行的命令(command)：
```
target: prerequisite1 prerequisite2 ...
  commands
```

  工作目标是一个必须建造的文件或进行的事情；必要条件或依存对象是工作目标得以被成功创建之前，必须事先存在的那些文件；而所要执行的命令则是必要条件成立时将会创建工作目标的那些shell命令。
  
  下面这项规则是用来将一个C文件foo.c编译成一个目标文件foo.o:
```
foo.o: foo.c foo.h
  gcc -c foo.c
```
  工作目标foo.o出现在冒号之前；必要条件foo.c, foo.h出现在冒号之后；命令脚本通常出现在后续的文本行上，而且会在tab键之后。
  
  当make被要求处理某项规则时，它首先会找出必要条件和工作目标中所指定的文件。如果必要条件中存在关联到其他规则的文件，则make会先完成相应规则的更新动作，然后才会考虑到工作目标。如果必要条件中存在时间戳在工作目标的时间戳之后的文件，则make会执行命令以便重新建立工作目标。脚本会被传递给shell并在其subshell中运行。如果其中的任何命令发生错误，则make会终止工作目标的建立动作并结束执行。
  
  下面这个程序会在它的输入中国年计算fee、fie、foe、fum等词汇出现的次数。这个程序(count_words.c)并不难，因为它使用了一个flex扫描程序:
```
#include <stdio.h>

extern int fee_count, fie_count, foe_count, fum_count;
extern int yylex( void );

int main( int argc, char ** argv )
{
  yylex();
  printf( "%d %d %d %d\n", fee_count, fie_count, foe_count, fum_count );
  exit( 0 );
}
```
  这个扫描程序(文件名lexer.l)相当简单:
```
  int fee_count = 0;
  int fie_count = 0;
  int foe_count = 0;
  int fum_count = 0;

%%
fee fee_count++;
fie fie_count++;
foe foe_count++;
fum fum_count++;
```

  用来编译这个程序的makefile也很简单:
```
count_words: count_words.o lexer.o -lfl
  gcc count_words.o lexer.o -lfl -o count_words
  
count_words.o: count_words.c
  gcc -c count_words.c

lexer.o: lexer.c
  gcc -c lexer.c
  
lexer.c: lexer.l
  flex -t lexer.l > lexer.c
```
  当这个makefile首次被执行时，我们会看到:
```
$ make
gcc -c count_words.c
flex -t lexer.l > lexer.c
gcc -c lexer.c
gcc count_words.o lexer.o -lfl -o count_words
```

  现在我们已经编译好了一个可执行的程序。当然，此处所举的例子有点简化，因为实际的程序通常由多个模块构成。此外，看过后面的章节你就会知道，这个makefile并未用到make大部分的特性，所以显得有点冗长。 不过，它仍不失为一个实用的makefile. 举例来说，这个范例的编写期间，为了测试程序，我执行了这个makefile不少于10次。
  
  当这个makefile范例在执行时，你可能会发现make执行命令的顺序几乎和它们出现在makefile中的顺序相反。这种"从上而下"的风格时makefile文件中最常见的手法。一般来说，通用形式的工作目录会现在makefile文件中被指定，而细节则会跟在后面。make程序对此风格的支持有许多方式，其中以make的两阶段执行模型以及递归变量最为重要。我们将会在稍后的章节深入探讨相关细节。
  
### 检查依存关系
  make如何知道自己该做什么事呢? 让我们继续探讨前一个范例。
  make首先注意到命令行上并未指定任何工作目标，所以会想要建立默认目标count_words. 当make检查其必要条件时看到了三个项目: count_words.o, lexer.o以及-lfl. 现在make会想要编译count_words.o并看到相应的规则。接着make会再次检查必要条件并注意到count_words.c并未关联到任何规则，但存在count_words.c这个文件，所以会执行相应的命令把count_words.c转换成count_words.o：`gcc -c count_words.c`
  
  这种从工作目标到必要条件，从必要条件到工作目标，再从工作目标到必要条件的链接机制就是make分析makefile决定要执行哪些命令的典型做法。
  
  必要条件的下一个项目会让make想要编译lexer.o. 规则链将make导向lexer.c, 但这次lexer.c并不存在。 make会从lexer.l找到产生lexer.c的规则，所以make会运行flex程序。现在lexer.c存在了，make会接着执行gcc命令。
  
  最后,make看到-lfl，其中-l是选项，用来要求gcc必须将其所指定的系统程序库链接进应用程序。此处指定了fl这个参数，代表实际的程序库名未libfl.a。GNU make对这个语法提供了特别的支持: 当-l<name>形式必要条件被发现时，make会搜索libname.so形式的文件；如果找不到相符的文件，make接着会搜索libname.a形式的文件。 在此例中，make会找到/usr/lib/libfl.a，而且会进行最后的动作--链接。
  
```
gcc -l<name> 会查找 libname.so | libname.a

在macbook下面没有libfl.a, 对应的是libl.a, 可以修改-lfl为 -ll或创建一个libfl.a软链到libl.a.

同时注意上面的lexer.l，前面几个变量的定义需要tab缩进。
```
### 尽量减少重新编译的工作量
  运行这个程序时，我们发现它除了会输出fee,fie,foe,fum等单词的出现次数，还会输出来自输入文件的其他文本。这并非我们想要的结果。 问题出现在我们忽略了词汇分析器(lexical analyzer)的一些规则，而且flex会将未被认出的文本送往输出。我们只要加入一条"any character"规则以及一条newline规则既可以解决这个问题:
```
  int fee_count = 0;
  int fie_count = 0;
  int foe_count = 0;
  int fum_count = 0;

%%
fee fee_count++;
fie fie_count++;
foe foe_count++;
fum fum_count++;
.
\n
```
  编辑这个文件之后，还需要重新编译应用程序以便测试我们所做的修正:
```
// 首次运行
gcc -c count_words.c
flex -t lexer.l > lexer.c
gcc -c lexer.c
gcc count_words.o lexer.o -ll -o count_words

// 修正后运行
flex -t lexer.l > lexer.c
gcc -c lexer.c
gcc count_words.o lexer.o -ll -o count_words
```
  注意，修正后count_words.c文件并未被重新编译。分析规则的时候，make发现count_words.o已存在， 而且该文件的时间戳在其必要条件count_words.c之后，所以不需要采取任何更新的动作。 不过，分析lexer.c的时候，make发现必要条件lexer.l的时间戳在其工作目标lexer.c的之后，所以make必须更新lexer.c. 这会一次引起lexer.o、count_words的更新。 运行这个重新编译的程序， 你会看到如下结果:
```
// 修改前
    int _count = 0;
    int _count = 0;
    int _count = 0;
    int _count = 0;
%%
 _count++;
 _count++;
 _count++;
 _count++;
3 3 3 3

// 修改后
3 3 3 3
```

### 调用make
  前面的范例做了以下假设:
  * 项目的所有程序代码以及make描述文件全都被放在单一目录中。
  * make描述文件的文件名为makefile、Makefile或GNUMakefile.
  * 执行make命令时，makefile就放在用户的当前目录中。

  当make在上述情况下被调用时，make会自动编译其找到的第一个工作目标。要更新另一个不同的工作目标(或多个工作目标)，请在命令行上指定目标的名称: `make lexer.c`
  
  当make被执行时，它会读取描述文件以及找出所要更新的工作目标。如果工作目标或其必要条件中的任一文件尚未更新(或不存在)，则会(以一次一个命令的方式)执行相应规则的命令脚本中的shell命令。 这些命令被执行之后，make会假设工作目标已完成更新动作，于是移往下一个工作目标或是结束执行。
  
  如果你所指定的工作目标已经更新(up to date), 则make除了告诉你此状况并立即结束以外，其他什么事也不做:
```
$ make lexer.c
make: `lexer.c' is up to date.
```
  如果你所指定的工作目标并未出现在makefile文件中，也不存在与之相应的隐含规则(implicit rule), 则make将会作出如下的响应:
```
bogon:count_words apple$ make test
make: *** No rule to make target `test'.  Stop.
```

  make提供了许多命令行选项。其中最有用的选项之一是--just-print(或-n)，用来要求make显示它将为特定工作目标执行的命令，但不要实际执行它们。当你编写makefile时，这个功能特别有用。你甚至还可以在命令行上设定几乎所有的makefile变量，来改写默认值或makefile文件中所设定的值。
```
bogon:count_words apple$ make -n
gcc -c count_words.c
flex -t lexer.l > lexer.c
gcc -c lexer.c
gcc count_words.o lexer.o -ll -o count_words
```

### Makefile的基本语法
  对make有了基本的认识之后，现在你差不多可以编写自己的makefile了。 这一节我们将会介绍makefile的基本语法和结构，让你得以开始使用make.
  
  makefile文件中一般采用“自上而下top-down”的结构，所以默认会更新最上层的工作目标(通常叫做all). 下层工作目标用来让上层工作目标保持在最新的状态，例如，用来删除无用的临时文件的clean工作目标应该放在最下层。正如你所猜测的，工作目标的名称并不一定非得是真实的文件名称不可，你可以使用任何名称。
  
  在前面的范例中，我们所看到的是经过简化的规则。下面是较完整的规则(但可能仍然不够完整):
```
target1 target2 target3: prerequisite1 prerequisite2
  command1
  command2
  command3
```
  冒号的左边可以出现一个或多个工作目标，而冒号的右边可以出现零个或多个必要条件。如果冒号的右边没有必要条件，那么只有在工作目标所代表的文件不存在时才会进行更新动作。更新工作目标所要执行的那组命令会被称为命令脚本,不过通常只被称为命令。
  
  每个命令必须以跳格符(tab)开头，这个(隐含的)语法用来要求make将紧跟在跳格符之后的内容传给subshell来执行。如果你不敬意地在非命令行的第一个字符前插入了一个跳格符，则在大多数情况下，make将会把其后的文字作为命令来解释。如果你很幸运，这个误入歧途的跳格符被视为语法错误，那么你会因此收到如下的信息:
```
$ make
Makefile:6: *** commands commence before first target. Stop.
```

  我们将会在第二章"规则"中讨论错综复杂的跳格符。
  
  make会将井号视为注释符，从井号开始到该行结束之间的所有文字都会被make忽略。你可以对作为注释的文本行进行缩排或前置空格。注释字符#并不会在代表命令的文本行中引入make的注释功能， 这一整行(包括#后面的字符)会被传给shell来执行。这行文字的处理方式取决于你所使用的shell。
  
  你可以使用标准的Unix转义字符--反斜线(\), 来延续过长的文本行。反斜线一般用来延续过长的命令，也可以用来延续必要条件。稍后我们会探讨处理过长必要条件的其他方法。
  
  到现在为止，你已经有能力编写简单的makefile了。 下一章我们探讨规则的细节。
  
