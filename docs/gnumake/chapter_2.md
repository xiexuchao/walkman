## 规则

  前一章中，我们编写了若干规则，用来编译与链接我们的单词计数(word counting)程序。 我们为每个规则定义了一个工作目标，也就是一个需要更新的文件。每个工作目标依存于一组必要条件，这组必要条件也都是文件。 当你要求更新某个工作目标时，如果必要条件中存在时间戳在工作目标的时间戳之后的文件， make就会执行相应规则里的命令脚本。因为某个规则的工作目标可以时另一个规则的必要条件，所以这样的工作目标和必要条件将会形成依存图(dependency graph). 建立即处理依存图，并据此更新特定的工作目标，就是make所要做的事情。
  
  规则对make而言十分重要，make允许你使用各种类型的规则。具体规则(explicit rule)，就像我们在上一章所编写的规则，用来指定需要更新的工作目标:如果必要条件中存在时间戳在此工作目标的时间戳之后的文件，make就会对它进行更新的动作。这将会是你最常使用的规则类型。 模式规则(pattern rule)中所使用的是通配符而不是明确的文件名称，这让make得以对模式相符的工作目标应用该规则，进行必要的更新动作。隐含规则(implicit rule)可以是模式规则，也可以是内置于make的后缀规则(suffix rule). 有了这些内置于make的规则可让makefile的编写变得更为容易，因为对于工作目标的更新，make已经知道许多常见文件类型、后缀以及更新工作目标的程序。 至于静态模式规则(static pattern rule), 它就像正规模式规则一样，只不过它们只能应用在一串特定的工作文件中。
  
  GNU make可作为许多其他版本的make的替代品，它特别针对兼容性提供了若干功能。后缀规则最初是make用来编写通则(general rule)的方法。尽管GNU make也支持后缀规则，不过为了更完整及一般化，它考虑以模式规则来替换。
  
### 具体规则
  你编写的规则多半会是具体规则，以特定的文件作为工作目标和必要条件。每个规则都可以有多个工作目标。这意味着，每个工作目标所具备的必要条件可以跟其他工作目标的一样。如果这些工作目标尚未被更新，则make将会为它们执行同一组更新动作。例如:
```
vpath.o variable.o: make.h config.h getopt.h gettext.h dep.h
```
  这代表vpath.o和variable.o与同一组C头文件具有依存关系。这一行等效于:
```
vpath.o: make.h config.h getopt.h gettext.h dep.h
variable.o: make.h config.h getopt.h gettext.h dep.h
```
  这两个工作目标将会被分开处理。只要有任一个目标文件尚未被更新(也就是说，任何一个头文件的时间戳在该目标文件的之后)，则make将会执行规则中所指定的命令以便更新该目标文件。
  
  你不必将规则一次定义完全(all at once). 每当make看到一个工作目标，就会将该工作目标与其必要条件加入依存图。 如果make所看到的工作目标已经存在于依存图中，则任何额外的必要条件都会被附加到该工作目标在依存图中的项目里。 对较简单的应用来说，这个特性可用来断开太长的规则以增进makefile的可读性:
```
vpath.o: make.h config.h getopt.h gettext.h dep.h
variable.o: make.h config.h getopt.h gettext.h dep.h
```
  对较复杂的应用来说，必要条件可以组成自看似无关的文件:
```
# 确定vpath.c被编译之前lexer.c已经创建好了
vpath.o: lexer.c
...

# 以特殊的标记来编译vpath.c
vpath.o: vpath.c
  $(COMPILE.c) $(RULE_FLAGS) $(OUTPUT_OPTION) $<
...

# 引入另一个程序所产生的依存关系
include auto-generated-dependencies.d
```

  第一个规则之处，每当lexer.c被更新后，vpath.o就必须被更新(这或许是因为产生lexer.c的过程中会有其他副作用)。 这个规则还可以用来确保必要条件的更新动作总是在工作目标之前被实施(注意规则的双向作用。就其正向作用而言，此规则之处，若lexer.c已经被更新，则需要对vpath.o执行更新的动作；就其反向作用而言，此规则之处，如果我们需要建立或使用vpath.o, 首先必须确定lexer.c已经更新)。 这个规则应该就近放在lexer.c处理规则的旁边，好让开发人员能够注意到这个微妙的关系。稍后，vpath.o的编译规则会被放到其他编译规则中。 此规则的命令用到了三个make变量。你将会看到更多的make变量，不过现在你只需要知道，一个变量可以是一个美元符号后面跟着单一字符(character)，也可以是一个美元符号后面紧跟着一个加圆括号的单词(稍后，我将会在本章做进一步的说明，并且会在第三章做更多的说明)。最后，.o文件和.h文件的依存关系是从另一个文件(这个文件产生自外部程序)引入到makefile的。
  
### 通配符
  当你有一长串文件要指定时，为了简化此过程，make提供了通配符, 此功能也被称为文件名模式匹配(globbing). make的通配符如同Bourne shell的~, *, ?, [...], [^...]。举例来说：
```
*.* : 表示文件名中包含点号的所有文件。
?: 表示代表任何单一字符
[...]: 代表一个字符集。
[^...]: [...]的补集。
~: 当前用户的主目录(home directory).
~username: username的主目录
```

  每当make在工作目标、必要条件或命令脚本等语境(context)中看到通配符，就会自动扩展通配符。在其他语境中，你可以通过函数的调用手动扩展通配符。如果你向创建适应能力较强的makefile, 统配符非常有用。举例来说，如果不想手动列出一个程序里的所有文件，你就可以使用通配符: 
```
prog: *.c
  $(CC) -o $@ $^
```
  不过通配符的使用务必谨慎为之，因为一不小心就会有误用的危险。比如: `*.o: constants.h`
  
  这个规则的意图很明显: 所有的目标文件皆依存于头文件constants.h。 不过，如果工作目录中当前并未包含任何目标文件，则通配符扩展后会变成下面这样: `: constants.h`
  这是个合法的make表达式，而且它本身并不会产生错误信息。实现此规则的正确方法，就是针对源文件使用通配符(因为它们总是存在的)以及将之转换成一串目标文件。
  
  最后值得注意的是，当模式出现在工作目标或必要条件中时，是由make进行通配符的扩展。然而，当模式出现子啊命令中时， 是由subshell进行扩展的动作。 区分这两种情况有时会变得很重要，因为make会在读取makefile的时候立即扩展通配符，但是shell只会在执行命令的时候扩展通配符。 当有许多复杂的文件操作需要进行时，这两种文件扩展动作将会有很大的差别。
  
### 假想工作目标
  到目前为止，我们所提到的工作目标以及必要条件都会进行文件的创建和更新的动作。尽管这是典型的用法，但是以工作目标充当标签来代表命令脚本，通常会有些用处。举例来说，稍早我们提到再许多makefile中，默认的首先要处理的标准工作目录称为all. 任何不代表文件的工作目标就叫做假想工作目标(phony target). 另一个标准的假想工作目标称为clean：
```
clean:
  rm -f *.o lexer.c
```

  通常，make总是会执行假想工作目标，因为对应于该规则的命令并不会创建以该工作目标为名称的文件。
  
  切记，make无法区分文件形式的工作目标与假想工作目标。如果当前目录中国年刚好出现与假想工作目标同名的文件，make则将会再它的相依图中建立该文件与假想工作目标的关系。 举例来说，如果你运行make clean时，工作目录中刚好存在clean这个文件，那么将会产生令人困惑的信息:
```
$ make clean
make: `clean` is up to date.
```

  因为大多数的假想工作目标并未指定必要条件，clean工作目标总是被视为已经更新，所以相应的命令永远不会被执行。
  
  为了避免这个问题，GNU make提供了一个特殊的工作目标--.PHONY, 告诉make, 该工作目标不是一个真正的文件。当你要声明假想工作目标时，只要将该工作目标指定成.PHONY的一个必要条件即可:
```
.PHONY: clean
clean:
  rm -f *.o lexer.c
```
  现在即使当前目录中存在名为clean的文件，make还是会执行对应于clean的命令。除了总是将工作目标标记为尚未更新，将一个工作目标声明为"假"之外，还会让make知道，不应该像处理一般规则那样，从源文件来建立以工作目标为名的文件。 因此，make可以优化它的一般规则搜索程序以提高性能。
  
  以假想工作目标作为实际文件的一个必要条件似乎不太有意义，因为假想工作目标总是尚未更新，这总会使得该实际文件(工作目标)被重新建立。 然而，以假想工作目标作为假想工作目标的必要条件通常会有些用处。 举例来说， all工作目标通常被用来指定要编译的一串程序:
```
.PHONY
all: bash bashbug
```
  其中，all工作目标将创建bash(一个shell程序)以及bashbug(一个错误报告工具)。
  
  你还可以将假想工作目标作为内置再makefile里的shell脚本来用。 以假想工作目标作为另一个工作目标的必要条件，可以让make在进行实际工作目标之前调用假想工作目标所代表的脚本。假如我们很在一磁盘空间的使用情况，因而在进行磁盘密集的工作之前，我们会想要显示磁盘尚有多少空间可供使用，我们可能会这么做:
```
.PHONY: make-documentation
make-documentation:
  df -k . | awk 'NR == 2 { printf( "%d available\n", $$4 ) }'
  javadoc ...
```
  这么做的问题是，我们最后可能会在不同的工作目标下多次指定df和awk命令，这回造成一个维护上的问题，因为如果我们在另一个系统上遇到了输出格式不同的df命令，那么我们必须到指定df和awk的每一处进行修改。此时，我们可以把df那一行放在它自己的假想工作目标里:
```
.PHONY: make-documentation
make-documentation: df
  javadoc ...
.PHONY: df
df:
  df -k . | awk 'NR == 2 { printf( "%d available\n", $$4 ) }'
```
  以df作为make-documentation的一个必要条件，可让make在产生文件之前先调用我们的df工作目标。可以这么做是因为make-documentation也是一个假想工作目标。现在即使我们在其他工作目标中重复使用df, 也不会造成什么维护上的问题。
  
  假想工作目标还有许多其他的好处。
  
  make的输出常会把想要进行阅读以及调试的人搞糊涂。 这时因为: 尽管makefile的编写是采用自上而下(top-down)的形式，不过make执行命令的方式却是采用从下而上(bottom-up)的形式；此外，你根本无法判断当前正在执行哪个规则。如果能够在make的输出中为主要工作目标加上注释，那么make的输出就会变得很容易阅读。 这就是假想工作目标可以派上用场的地方。 如下所示的例子摘录自bash的makefile：
```
$(Program): build_msg $(OBJECTS) $(BUILTINS_DEP) $(LIBDEP)
  $(RM) $@
  $(CC) $(LDFLAGS) -o $(Program) $(OBJECTS) $(LIBS)
  ls -l $(Program)
  size $(Program)
  
.PHONY: build_msg
build_msg:
  @printf "#\n# Build $(Program)\n#\n"
```
  因为printf位于假想工作目标之中，所以在任何必要条件被更新之前会立即输出信息。如果以build_msg作为$(Program)命令脚本的第一个命令，那么在所有编译结果和依存关系都产生之后才会执行该命令。切记，因为<strong>假想工作目标总是尚未更新</strong>，所以假想工作目标build_msg会导致$(Program)被重建--即使它已经被更新。这么做似乎是明智的选择，所有的计算工作在编译目标文件的时候大多已经完成，因此只有最后的链接工作一定会被执行。
  
  假想工作目标还可以用来改善makefile的用户接口。 工作目标通常是包含目录路径元素、额外文件名成分(比如版本编号)以及标准扩展名的复合字符串，这使得"在命令行上指定工作目标的文件名"称为一种挑战。你只要假如一个简单的假想工作目标，并以实际文件名作为它的必要条件，就可以避免这个问题。
  
  许多makefile多少都会包含一组标准的假想工作目标。下表列出这些标准的假想工作目标。
```
标准假想工作目标
工作目标      功能
all       执行编译应用程序的所有工作
install   从已编译的二进制文件进行应用程序的安装
clean     将产生自源代码的二进制文件删除
distclean 删除编译过程中所产生的任何文件(除了二进制文件，也包含configure所产生的Makefile)
TAGS      建立可供编辑器使用的标记表
info      从Textinfo源代码来创建GNU info文件
check     执行与应用程序相关的任何测试
```

  工作目标TAGS实际上不是一个假想工作目标，因为ctags和etags程序的输出就是名为TAGS的文件。此处之所以提到它，是因为就我们所知，它是绝无仅有的、标准的非假想工作目标(nonphony target).
  
### 空工作目标
  空工作目标(empty target)如同假想工作目标一样，可用来发挥make的潜在能力。假想工作目标总是尚未更新，所以它们总是被执行，并且总是会使得它们的依存对象(工作目标所关联到的必要条件)被创建。但假设我们有若干命令，它们不会输出任何文件，二十偶尔需要被执行以下，而且我们并不像让我们的依存对象被更新，怎么办? 此时，我们可以建立一个规则，它的工作目标是一个空文件(有时候称为cookie)：
```
prog: size prog.o
  $(CC) $(LDFLAGS) -o $@ $^
  
size: prog.o
  size $^
  touch size
```
  请注意，size规则在执行完之后，会使用touch创建一个名为size的空文件。这个空文件可作为它的时间戳，因此make只在prog.o被更新之后才会执行size规则。此外prog的必要条件size将不会导致prog的更新，除非它的目标文件的时间戳也在工作目标(的时间戳)之后。
  
  与自动变量$?并用时，空文件特别有用。我们将会在自动变量一节探讨自动变量，不过实现了解一下这个变量应该不会有什么问题。 对每个规则的命令脚本部分来说，make会将$?替换成一组必要条件，这组必要条件的时间戳在工作目标的时间戳之后。例如，下面的这个规则将会输出从上此执行make print之后，变更过的所有文件:
```
print: *.[hc]
  @echo $?
  touch $@
```
  make print首次执行，会打印所有的.h, .c文件，并创建print文件。 下次再执行print时，打印出比print时间戳要晚的.h,.c文件，即上次调用make print之后修改过的.h,.c文件，同时创建print文件。
  通常空文件可以用来标明最近发生了一个特殊的事件。
  
### 变量
  现在让我们来查看曾在范例中出现的若干变量。其中最简单的变量具有如下的语法:`$(variable-name)`
  
  这代表我们想要扩展名为variable-name的变量。任何文字都可以包含在变量之中，而且大多数字符(包括标点符号)都可以用在变量的名称上。例如，内含C编译命令的变量可以取名为COMPILE.c。 一般来说，你必须以$()或${}将变量名称扩住，这样make才会认得。有一个例外:变量名称若是单一字符则不需要为它加上圆括号。
  
  通常makefile文件中都会定义许多变量，不过其中有许多特殊变量是make自动定义的。这些变量中的若干变量可供用户用来控制make的行为，其余变量则是供make用来跟用户的makefile文件沟通。
  
### 自动变量
  当规则相符时，make会设定自动变量(automatic variable). 通过它们，你可以取用工作目标以及必要条件中的元素，所以你不必指明任何文件名称。要避免重复，自动变量就相当有用，它们也是定义较一般的模式规则时不可少的项目。
  下面是7个核心的自动变量:
  * `$@`: 工作目标的文件名
  * `$%`: 档案文件成员(archive member)结构中的文件名元素
  * `$<`: 第一个必要条件的文件名
  * `$?`: 时间戳在工作目标(的时间戳)之后的所有必要条件，并以空格隔开这些必要条件。
  * `$^`: 所有必要条件的文件名，并以空格隔开这些文件名。这份列表以删掉重复的文件名，因为对大多数的应用而言，比如编译、复制等，并不会用到重复的文件名。
  * `$+`: 如同`$^`,代表所有必要条件的文件名，并以空格隔开这些文件名。不过,`$+`包含重复的文件名。此变量会在特殊的状况下被创建，比如将自变量传递给链接器(linker)时重复的值是有意义的。
  * `$*`: 工作目标的主文件名。一个文件名成是由两个部分组成:主文件名(stem)和扩展名(suffix). 稍后我们将会在"模式规则"一节中探讨主文件名的处理方式，。请不要在模式规则以外使用此变量。
  
  此外，为了跟其他版本的make兼容，以上这六个变量都具有两个变体。其中一个变体只会返回值的目录部分，它的指定方式就是在原有的符号之后附加D这个字母，例如$(@D)、$(<D)等。另一个变体只会返回值的文件部分，它的指定方式就是在原有的符号之后附加F这个字母，比如$(@F), $(<F)等。 请注意，这些变体名称的字符长度超过一个，所以必须加上圆括号。GNU make还以dir和nodir函数提供较具可读性的替代方案。 我们将会在第四章进行函数的探讨。

  make会在规则与它的工作目标和必要条件相符之后设定自动变量，所以变量只能应用在规则中的命令脚本。
  
  现在，我们可以将之前明确指定文件名的makefile替换成适当的自动变量。
```
count_words: count_words.o counter.o lexer.o -lfl
  gcc $^ -o $@
  
count_words.o: count_words.c
  gcc -c $<
  
counter.o: counter.c
  gcc -c $<

lexer.o: lexer.c
  gcc -c $<

lexer.c: lexer.l
  flex -t $< > $@
```

### 以VPATH和vpath来查找文件
  到目前为止我们所举的例子都相当简单:makefile与源文件都存放在同一个目录下。真实世界的程序比较复杂(请问，你上一次开发只有一个目录的项目在什么时候?).  现在让我们重构先前的范例，进行较实际的文件布局。我们可以通过将main构造成一个名为counter的函数来修改我们的单词计数程序。
```
#include <lexer.h>
#include <counter.h>

void counter( int counts[4] )
{
    while ( yylex() )
        ;   

    counts[0] = fee_count;
    counts[1] = fie_count;
    counts[2] = foe_count;
    counts[3] = fum_count;
}
```
  一个可重复使用的程序库函数(library function),在头文件中应该要有一个声明(declaration), 所以让我们创建counter.h头文件来包含此声明:
```
#ifndef COUNTER_H_
#define COUNTER_H_

extern void counter( int counts[4] );

#endif
```
  我们还可以把lexer.l的声明放在lexer.h头文件中:
```
#ifndef LEXER_H_
#define LEXER_H_

extern int fee_count, fie_count, foe_count, fum_count;
extern int yylex( void );

#endif
```
  然后按源码树的布局惯例，头文件会被放在include目录中，而源文件会被放在src目录里。我们也这样做，并把makefile放在它们的上层目录。那么现在范例程序的布局如下:
```
+----- makefile
|----- include
_        |----- lexer.h
_        |----- counter.h
|----- src
_        |----- counter.c
_        |----- count_words.c
_        |----- lexer.l
```
  既然现在我们的源文件包含了头文件，这些新产生的依存关系就应该记录在我们的makefile文件中， 这样，当我们的头文件有所变动时，才会更新相应的目标文件。
```
count_words: count_words.o counter.o lexer.o -ll 
>---gcc $^ -o $@

count_words.o: count_words.c include/counter.h
>---gcc -c $<

counter.o: counter.c include/counter.h include/lexer.h
>---gcc -c $<

lexer.o: lexer.c include/lexer.h
>---gcc -c $<

lexer.c: lexer.l
>---flex -t $< > $@
```

  现在运行make, 将会看到如下的错误:
```
bogon:count_words2 apple$ make
make: *** No rule to make target `count_words.c', needed by `count_words.o'.  Stop.
```
  咦，发生了什么事情? makefile想要更新count_words.c, 不过那是一个源文件! 让我们来看看扮演make的角色。我们的第一个必要条件是count_words.o. 我们并未看到这个文件，所以我们会去查找一个规则以便创建这个文件。用来创建count_words.o的具体规则指向count_words.c， 但为何make找不到这个源文件呢? 因为这个源文件并非位于当前目录中，二十被放在了src中。除非你告诉make, 否则它只会在当前目录中找寻工作目标以及必要条件。我们要怎么做才能让make到src目录找寻到源文件?也就是说，要如何告诉make我们的源代码放在哪里?
  
  你可以使用VPATH和vpath来告诉make到不同的目录去查找源文件。要解决我们眼前的问题，可以在makefile文件中对VPATH进行如下的赋值动作: VAPTH = src
  
  这表示，如果make所需要的文件并未放在当前目录中，就应该到src目录去找。为了make的输出更为明确，makefile本身也做了调整,此时makefile会像下面这样:
```
VPATH = src include
count_words: count_words.o counter.o lexer.o -ll 
>---gcc $^ -o $@

count_words.o: count_words.c include/counter.h
>---gcc -c $< -o $@

counter.o: counter.c include/counter.h include/lexer.h
>---gcc -c $< -o $@

lexer.o: lexer.c include/lexer.h
>---gcc -c $< -o $@

lexer.c: lexer.l
>---flex -t $< > $@
```
  执行make会看到如下的结果:
```
bogon:count_words2 apple$ make
gcc -c src/count_words.c -o count_words.o
gcc -c src/counter.c -o counter.o
src/counter.c:1:10: fatal error: 'lexer.h' file not found
#include <lexer.h>
         ^
1 error generated.
make: *** [counter.o] Error 1
```
  请注意，现在make可以编译第一个文件了，因为它会未该文件正确填入相对路径。使用自动变量的另一个理由是: 如果你写出具体的文件名，make将无法未该文件填上正确的路径。可惜并未编译成功，因为gcc无法找到引入文件include file. 我们只要使用正确的-I选项来自定义隐含编译规则就可以解决这个问题了: CPPFLAGS = -I include
  
  请注意，由于头文件被放在include目录中，所以还必须调整VPATH。 VPATH = src include

  现在就可以编译通过了。
```

  [源代码包]()
CPPFLAGS = -I include
VPATH = src include
count_words: count_words.o counter.o lexer.o -ll 
>---gcc $^ -o $@

count_words.o: count_words.c include/counter.h
>---gcc -c $< -o $@

counter.o: counter.c include/counter.h include/lexer.h
>---gcc -c $< -o $@ $(CPPFLAGS)

lexer.o: lexer.c include/lexer.h
>---gcc -c $< -o $@

lexer.c: lexer.l
>---flex -t $< > $@

// 运行make
bogon:count_words2 apple$ make
gcc -c src/count_words.c -o count_words.o
gcc -c src/counter.c -o counter.o -I include
flex -t src/lexer.l > lexer.c
gcc -c lexer.c -o lexer.o
gcc count_words.o counter.o lexer.o /usr/lib/libl.a -o count_words
```

  [mac上可运行的源代码](https://github.com/walkerqiao/walkman/tree/master/sources/count_words2.tar.gz)
  
  VPATH变量的内容是一份目录列表，可供make搜索其所需要的文件。这份目录列表可用来搜索工作目标以及必要条件，但不包括脚本中所提及的文件。这份目录列表的分割符在Unix上可以是空格或冒号，在Windows上可以是空格或分号。 因此使用空格，支持各种系统。此外以空格为分割符将会使得目录将容易阅读。
  
  虽然VPATH变量可以解决以上的搜索问题，但是也有限制。make将会为它所需要的任何文件搜索VPATH列表中的每个目录，如果在多个目录中出现同名的文件，则make只会攫取第一个被找到的文件。有时这可能会造成问题。
  
  此时可以使用vpath指令。这个指令的语法: vpath pattern directory-list
  
  所以之前使用的VPATH可以改写成:
  vpath %.l %.c src
  vpath %h include
  
  现在，我们告诉了make应该在src下面搜索.c文件，我们还告诉它，应该在include下面查找.h文件。(所以我们可以从头文件必要条件中一处include/字样)。 在较复杂的应用程序中，这项控制功能可省去许多头痛和调试的时间。
  
  注意: 在macbook中， 需要将vpath %.l %.c src写成 vpath %.l src 和 vpath %.c src. 暂时没有去看原因。
  
  我们在此处使用vpath来解决源文件散布在多个目录中的问题。这个问题与源文件放在源代码树而目标文件放在二进制代码树时，要如何编译应用程序的问题，虽然相关但却是不同的。尽管适当地使用vpath也可以解决这个问题，不过整个工作很快就会复杂到单靠vpath无法处理的地步。我们将会在稍后详细探讨这个问题。
  
### 模式规则
  我们现在所看到的makefile范例已经有点长了。如果这时一个仅包含十几个或更少文件的小程序，我们可能并不担心；但如果这是一个包含成百上千个文件的大型程序，手动指定每个工作目录、必要条件以及命令脚本将会变得不切实际。此外，在我们的makefile中，这些命令脚本代表着重复的程序代码。如果这些命令脚本包含了一个缺陷或曾经被修改过，那么我们必须跟新所有相关的规则。这将会给维护带来困难，而且会称为各种缺陷的源头。
  
  许多程序在读取文件以及输出文件时都会依照惯例。例如，所有C编译器都会假设，文件若是以.c为扩展名，其所包含的就是C源代码，把扩展名从.c替换成.o(Windows下面为.obj)就可以得到目标文件的文件名。在前一章中国年，我们可以看到flex输入文件使用了.l这个扩展名，它的输出使用.c这个扩展名。
  
  这些惯例让make可以通过文件名模式的匹配来简化规则的建立，以及提供内置规则来处理它们。举例来说，通过这些内置的规则，我们可以把之前的这多行makefile缩减为7行:
```
VPATH = src include
CPPFLAGS = -I include

count_words: counter.o lexer.o -ll 
count_words.o: counter.h
counter.o: counter.h lexer.h
lexer.o: lexer.h
```
  所有内置规则都是模式规则的实例。一个模式规则看起来就像之前你所见过的一般规则，指示祝文件名(就是扩展名之前的部分)会被表示成%字符。上面这个makefile之所以可行是因为make里存在三项内置规则。第一项规则描述了如何从一个.c文件编译出一个.o文件:
```
%.o: %.c
  $(COMPILE.c) $(OUTPUT_OPTION) $<
```
  第二项规则描述了如何从.l文件产生一个.c文件:
```
%.c: %.l
  @$ (RM) $@
  $(LEX.l) $< > $@
```
  最后一项特殊的规则，描述了如何从.c文件产生一个布局扩展名(通常是一个可执行文件)的文件。
```
%: %.c
  $(LINK.c) $^ $(LOADLIBES) $(LDLIBS) -o $@
```
  我们将会进一步探讨这个语法的细节，不过首先让我们查看make的输出，看看它们时如何应用这些内置规则的。
```
bogon:count_words2 apple$ make
cc  -I include  -c -o count_words.o src/count_words.c
cc  -I include  -c -o counter.o src/counter.c
lex  -t src/lexer.l > lexer.c
cc  -I include  -c -o lexer.o lexer.c
cc   count_words.o counter.o lexer.o /usr/lib/libl.a   -o count_words
rm lexer.c
```
  首先，make会读取makefile, 并且将默认目标设置成count_words, 因为命令韩商并未指定任何工作目标。查看默认目标时，make发现了四个必要条件: count_words.o(makefile并未指定这个必要条件，它是由隐含规则提供的)、counter.o、lexer.o以及-ll。 接着，make会试着依次更新每个必要条件。
  
  当make检查第一个必要条件count_words.o时，并未发现可以处理它的具体规则(explicit rule), 不过却找到了隐含规则(implicit rule). 查看当前目录，make并未找到源文件，所以它开始搜索VPATH，而且在src目录中找到了一个相符的源文件。因为src/count_words.c没有其他必要条件，make可以自由更新count_words.o, 所以它会执行这个隐含规则。counter.o也是类似的情况，make检查lexer.o时，并未找到相应的源文件(即使在src目录中)，所以make会假设这(不存在的源文件)是一个中间文件，而且会查找"从其他源文件产生lexer.c文件"的方法。make找到了一个从.l文件产生.c文件的规则，并且注意到lexer.l的存在。因为不需要进行lexer.l的更新，所以make前往用来更新lexer.c的命令，这会产生flex命令行。接着，make会从C源文件来更新目标文件。 像这样使用一连串的规则来更新一个工作目标的动作称为规则链接.
  
  接下来，make会检查程序库规范-ll, 它会搜索系统的标准程序库，并且找到libl.a。
  
  现在make已经找到更新count_words时所需的每个必要条件，所以它会执行最后一个gcc命令。 最后, make发现自己创建了一个不必保存的中间文件，所以会对它进行清除操作。
  
  正如所见，在makefile文件中使用规则，可以略过许多细节。这些规则经过复杂的交互之后可产生极为强大的功能。尤其是，使用这些内置规则可大量简化makefile的规范工作。
  
  你可以通过在脚本中更改变量的值来定义内置规则。一个典型的规则包含一群变量，以所要执行的程序开头，并且包括用来设定主要命令行选项(比如输出文件、进行优化、进行调试等)的变量。你可以通过运行make --print-data-base列出make具有哪些默认规则(和变量)。

### 模式
  模式规则中的百分比字符(%)大体上等效于Unix Shell的星号，它可以代表任意多个字符。百分比字符可以放在模式中的任何地方，不过只能出现一次。百分比字符的正确用法如下:
```
%,v
s%.o
wrapper_%
```
  在文件名中，百分比以外的字符会按照字面进行匹配。一个模式可以包含一个前缀或一个后缀，或是这两者同时存在。当make搜索所要使用的模式规则时，它首先会查找相符的模式规则工作目标(pattern rule target). 模式规则工作目标必须以前缀开头并且后缀结尾(如果它们存在的话)。如果占到相符的模式规则工作目标，则前缀与后缀之间的字符会被作为文件名的词干(stem). 接着make会通过将词干替换到必要条件模式中来检查该模式规则的必要条件。如果所产生的文件名存在，或是可以应用另一项规则进行产生的工作，则会进行比较以及应用规则的动作。词干必须至少包含一个字符。
  
  事实上，你还有可能用到只有一个百分比字符的模式。此模式最常被用来编译Unix可执行程序。例如，下面就是GNU make用来编译程序的若干模式规则:
```
%: %.mod
  $(COMPILE.mod) -o $@ -e $@ $^
  
%: %.cpp
  $(LINK.cpp) $^ $(LOADLIBES) $(LDLIBS) -o $@
  
%: %.sh
  cat $< >$@
  chmod a+x $@
```
  这些模式会依次被用来从Modula源文件，经过与处理的C源文件和Bourne shell脚本产生出可执行文件。 我们将会在隐含规则库一节看到更多的隐含规则。
  
### 静态模式规则
  静态模式规则只能应用在特定的工作目标上。
```
$(OBJECTS): %.o: %.c
  $(CC) -c $(CFLAGS) $< -O $@
```
  此规则一般与模式规则的唯一差别时开头的$(OBJECTS):规范。这将使得该项规则能应用在$(OBJECTS)变量中所列举的文件上。
  
  此规则与模式规则十分相似。%.o模式会匹配$(OBJECTS)中所列举的每个目标文件并且取出其词干。然后该词干会被替换进%.c模式，以产生工作目标的必要条件。如果工作目标模式不存在，则make会发出警告。
  
  如果明确列出工作目标文件比较容易进行扩展名或其他模式的匹配，请使用静态模式规则。
  
### 后缀规则
  后缀规则使用来定义隐含规则的最初方法(也是过时的方法)。旧版的make可能不支持GNU make的模式规则语法，因此你仍然会在许多makefile文件中看到后缀规则，所以你最好能了解它的语法。尽管未目标系统编译GNU make可以解决makefile的可移植性问题，但是在一些罕见的情况下你仍旧需要使用后缀规则。
  
  后缀规则中的工作目标，可以是一个扩展名或两个被衔接在一起的扩展名:
```
.c.o:
  $(COMPILE.c) $(OUTPUT_OPTION) $<
```
  这令人有些困惑，因为必要条件的扩展名被摆到开头，而工作目标退居第二位。这个规则所匹配的工作目标以及必要条件跟下面的规则一样:
```
%.o: %.c
  $(COMPILE.c) $(OUTPUT_OPTION) $<
```
  后缀规则会通过移除工作目标的扩展名以形成文件的主文件名，以及通过将工作目标的扩展名替换成必要条件的扩展名以形成必要条件。make只会在这两个扩展名都列在已知扩展名列表中时，才将之视为后缀规则。
  
  上面的后缀规则就是所谓的双后缀规则(double-suffix rule), 因为它包含了两个扩展名。你还可以使用单后缀规则(single-suffix rule). 没错，单后缀规则只包含了一个扩展名，也就是源文件的扩展名。 这个规则可用来创建可执行文件， 因为Unix上的可执行文件不需要扩展名。
```
.p:
  $(LINK.p) $^ $(LOADLIBES) $(LDLIBS) -o $@
```

  这个规则将会从Pascl源文件产生出可执行图像(executable image). 这个规则的作用等效于下面的这个规则模式:
```
%: %.p
  $(LINK.p) $^ $(LOADLIBES) $(LDLIBS) -o $@
```
  已知扩展名列表时语法中最奇特的部分，你可以使用.SUFFIXES这个特别的工作目标来设定已知的扩展名。 下面是.SUFFIXES默认值的第一个部分:
```
.SUFFIXES: .out .a .ln .o .c .cc .C .cpp .p .f .F .r .y .l
```

  你只要在makefile文件中假如.SUFFIXES规则，就剋将自己的扩展名加入此列表。.SUFFIXES: .pdf .fo .html .xml
  
  如果要删除所有已知的扩展名(因为它们的存在干扰到了你的扩展名)，你只要在加入.SUFFIXES规则时不指定必要条件就行了:
  你还可以使用命令行选项--no-builtin-rules(-r).
  
  我们不会在本书的其余部分使用这个旧语法，因为make的模式规则较明显也比较一般化。
  
### 隐含规则
  GNU make 3.80具有90个隐含规则。隐含规则不是模式规则的形式就是后缀规则的形式。这些内置的模式规则可应用于C、C++、Pascal、FORTRAN、ratfor、Modula、Texinfo、TeX(包括Tangle和Weave)、Emacs Lisp、RCS以及SCCS. 此外，有些规则是应用在这些语言的支持程序上的，比如cpp、as、yacc、lex、tangle、weave以及dvi工具。
  
  如果你使用到这些工具，你很有可能会发现，内置规则中已经有你所需要的东西了。如果你用到了未受支持的语言，比如java或xml, 那么你就必须编写自己的规则。单不必担心，通常你需要若干规则旧可以支持一种语言了，而且此类规则相当容易编写。
  
  要查看make内置了哪些规则，可使用--print-data-base(-p)命令行选项。这将会产生1000行左右的输出。在版本与版权信息之后，make会输出变量的定义，而且会为每个变量前置一行注释以说明其用途。举例来说，一个变量可以是环境变量、默认值、启动变量等。变量之后，make会输出规则。这个规则若是GNU make内置的，则会使用如下的格式:
```
%: %.C
# commands to execute (built-in):
  $(LINK.C) $^ $(LOADLIBES) $(LDLIBS) -o $@
```

  这个规则若是在makefile中定义的，则会使用如下的格式(注释中将会包含文件名和行号):
```
%.html: %.xml
# commands to execute (from `Makefile` line 168):
  $(XMLTO) $(XMLTO_FLAGS) html-nochunks $<
```
  
### 隐含规则的使用
  当make检查一个工作目标时，如果找不到可以更新它的具体规则，就会使用隐含规则。隐含规则的使用很容易: 当你将工作目标加入makefile时，只要不指定脚本就行了。这会使得make搜索隐含规则以满足工作目标的需要。通常这就是你所要的结果，但是在极少的状况下，你的开发环境可能会引发问题。举例来说，假设你的语言环境掺杂了Lisp和C源代码，如果文件editor.l和editor.c存在于同一个目录中(假设其中一个时另一个所使用的低级实现)，则make将会把Lisp文件作为flex文件(因为flex文件是以.l为扩展名)并把C源文件作为flex命令的输出。如果工作目标是editor.o,而editor.l的时间戳在editor.c的之后，则make将试图以flex的输出更新C源文件，结果你的源代码被覆盖掉了。
  
  要解决这个问题，你可以从内置规则库中删除这两个规则:
```
%.o: %.l
%.c: %.l
```
  一个没有指定脚本的模式规则将会从make规则库删除相应的规则。尽管你很难遇到此类情况，不过有一件事情铭记在心:内置规则库中所包含的规则与你的makefile之间可能存在让你意外的交互。
  
  我们已经看过make在尝试更新工作目标时将规则链接在一起的几个例子。这样做可能会提高复杂度，我们会子啊此处查看一番。当make试着更新一个工作目标时，它会搜索隐含规则，试图找到与工作目标(target)相符的工作目标模式。对每个与工作目标相符的工作目标模式来说，make会查找相符的必要条件。也就是说，匹配工作目标模式之后，make会立即查找必要条件(源文件)。如果找到了相符的必要条件，就会使用相应的规则。有些工作目标模式具有多个可能的源文件。举例来说，一个.o的文件可能产生来自.c, .cc, .cpp, .p, .f, .r, .s以及.mod等文件，单如果搜索过所有可能的规则之后还是找不到源文件，结果会怎样呢? 此时，make会再sousuo规则依次，不过这一次会认为源文件的匹配是在跟新一个新的工作目标。通过递归地进行这样地搜索动作，make可以找到一串用来更新工作目标的规则链。我们可以再这个lexer.o范例中看到这个现象:即使.c这个中间文件不存在，通过调用从.l到.c的规则以及从.c到.o的规则，make仍旧能更新lexer.o这个工作目标。
  
  make可以从它的规则库中产生出卓越的处理程序。让我们做个试验:
  首先，创建一个空的yacc源文件以及使用ci向RCS登记(也就是说，我们需要一个有版本控制的yacc源文件)：
```
$ touch foo.y
$ ci foo.y
foo.y,v <-- foo.y
.
initial revision: 1.1
done
```
  现在，我们想问make要如何创建一个可执行foo. 我们可以用--just-print(-n)选项要求make回报它将会采取哪些行动，单不要实际执行它们。 请注意，此刻并没有makefile也没有源文件，只有一个RCS文件:
```
$ make -n foo
co coo.y, v foo.y
foo.y, v --> foo.y
revision 1.1
done
bison -y foo.y
mv -f y.tab.c foo.c
gcc -c -o foo.o foo.c
gcc foo.o -o foo
rm foo.c foo.o foo.y
```
  找到了隐含规则链之后，make可以作出如下决定: 如果目标文件foo.o存在，就可以创建foo;如果C源文件foo.c存在，就可创建foo.o; 如果yacc源码foo.y存在，就可以创建foo.c. 最后，make发现它可以通过从RCS文件foo.y,v中调出(check out)文件foo.y来创建该文件。一旦make将此计划公式化后，就会以co调出foo.y, 以bison将之转换成foo.c, 以gcc将之编译成foo.o, 最后再次以gcc将之链接成foo，以上这些步骤全部产生自隐含规则库。 酷极了。
  
  链接规则的过程中所产生的文件称为中间文件，make会对它们进行特别的处理。 首先，因为中间文件不会在工作目标中出现(否则它们旧不是中间文件), 所以make不会更新中间文件；其次，make创建中间文件本身旧有更新工作目标的副作用，所以make在结束运行之前会删除这些中间文件。
  
### 规则的结构
  内置规则具有标准的结构，好让它们容易自定义。现在让我们来查看此结构，并探讨有关"自定义"这方面的议题。下面是从C源文件来更新目标文件的规则:
```
%.o: %.c
  $(COMPILE.c) $(OUTPUT_OPTION) $<
```
  这个规则的自定义完全取决于其所用到的变量。我们在此处看到了两个变量，其中的COMPILE.c是由多个其他变量所定义而成的:
```
COMPILE.c = $(CC) $(CFLAGS) $(CPPFLAGS) $(TARGET_ARCH) -c
CC = gcc
OUTPUT_OPTION = -o $@
```
  你只要变更CC变量的设定值就可以更换C编译器。此外还包括用来设定编译选项的变量CFLAGS,用来设定预处理器选项的变量(CPPFLAGS)以及用来设定结构选项的变量TARGET_ARCH。
  
  在内置规则中使用变量的目的，就是让规则的自定义尽可能简单。因此，当你在makefile文件中设定这些变量时，务必谨慎。如果设定这些变量时随意为之，将会破坏终端用户自定义它们的能力。例如，你在makefile中做了如下的赋值动作:
  CPPFLAGS = -I project/include
  如果用户想再命令行上加入CPP的定义，它们一般会像这样来调用make: make CPPFLAGS=-DDEBUG
  如果他们真的这么做了，将会意外删除编译时所需要添加的-I选项。命令行上设定的变量将会覆盖掉该变量原有的值。所以不当地在makefile中设定CPPFLAGS将会破坏大多数用户预设地自定义结果。为了避免此问题，你可以使用如下地方式重新定义编译常量:
  COMPILE.c = $(CC) $(CFLAGS) $(INCLUDES) $(CPPFLAGS) $(TARGET_ARCH) -c
  INCLUDES = -I project/include

  或者你可以使用附加形式地赋值动作，我们将会在其他形式地赋值动作一节中讨论此做法。
  
### 支持源代码控制系统的隐含规则
  make的隐含规则还针对两种源代码控制系统RCS和SCCS提供了支持。可惜make心有余而力不足，未能跟上源代码控制系统及现代软件工程日新月异的脚本。我从未发现有人使用make所支持的源代码控制功能，也未曾看到过任何的源代码控制软件使用make这个功能。基于以下几个理由，建议各位不要使用make的这个功能。
  
### 一个简单的help命令
  大型的makefile文件包含了许多工作目标，这让用户很难搞清楚。减少此类问题的方法之一，就是以一个简洁的help命令为默认目标。然而，手动维护帮助文本总是很麻烦。要避免此问题，可直接从make规则库中收集可用的命令。接下来的help工作目标将会把可用的工作目标 显示成一份排序的四个字段的列表:
  待续， 没有看太明白
  
### 特殊工作目标
  特殊工作目标是一个内置的假想工作目标，用来变更make的默认行为。例如，.PHONY这个特殊的工作目标用来声明它的必要条件并不代表一个实际的文件，而且应该被视为尚未更新。.PHONY将会是最常见的特殊工作目标，但是你还会看到其他的特殊工作目标。
  
  特殊工作目标的语法跟一般工作目标的语法没有什么不同，也就是target: prerequisite, 但是target并非文件而是一个假想工作目标。它们实际上比较像是用来修改make内部算法的指令。
  
  目前共有12个特殊工作目标，可分成三类:第一类用来在更新工作目标时修改make的行为；第二类的动作就好像时make的全局标记，用来忽略相应的工作目标；最后是.SUFFIXES这个工作目标，用来指定旧式的后缀规则。
  
  下面列出了最有用的工作目标修饰符:
  * .INTERMEDIATE: 这个特殊工作目标的必要条件被视为中间文件。如果make在更新另一个工作目标期间创建了该文件，则该文件将会在make运行结束时被自动删除。如果在make想要更新该文件之际该文件已经存在了，则该文件不会被删除。当你要自定义规则链时，这会非常有用。举例来说，大多数java工具都可以接受windows形式的文件列表。自定义规则以建立列表并把它们的输出文件视为中间文件，可让make清除这些临时性文件。
  * .SECONDARY: 这个特殊工作目标的必要条件会被视为中间文件，但不会被自动删除。.SECONDARY最常用来标示存储在程序库里的目标文件。按照惯例，这些目标文件一旦被加入档案库后就会被删除。在项目开发期间保存这些目标文件，但仍使用make进行程序库的更新，有时会比较方便。
  * .PRECIOUS: 当make在运行期间被中断时，如果自make启动以来该文件被修改过，make将会删除它正在更新的工作目标文件。因此make不会在编译树中国年留下尚未编译完成的文件。但有些时候你却不希望make这么做，特别时在该文件很大而且编译的代价很高时。如果该文件极为珍贵，你旧该用.PRECIOUS来标示它，这样make才不会在自己被中断时删除该文件。
  * .DELETE_ON_ERROR: 与上面的相反。

### 自动产生依存关系
  当我们将单词计数程序重构为使用头文件时，有个棘手的小问题会找上我们。尽管就此例而言，我们可以轻易地在makefile文件中手动加入目标文件与C头文件的依存关系，但是在正常的程序(而不是玩具程序)里这是一个烦人以及动辄得咎得工作。事实上，在大多数程序中，这几乎是不可能的事，因为大多数的头文件还会包含其他头文件所形成的复杂树状结构。举例来说，在我的系统上，头文件stdio.h会被扩展成包含15个其他的头文件。以手动方式解析这些关系是一个令人绝望的工作。但如果这些文件的重新编译失败，可能会导致数小时的调试，或者更糟的是因此产生出一个具有缺陷的程序。到底该怎么办呢?
  
  计算机擅长于搜索以及模式匹配。让我们使用一个程序来找出这些文件之间的关系，我们甚至可以使用此程序以makefile的语法编写出这些依存关系。正如你猜测，此类程序已经存在，至少对C/C++而言是这样的。在gcc中这是一个选项，许多其他的C/C++编译器也都会读进源文件并写出makefile的依存关系。例如，下面是我为stdio.h寻找依存关系的方式:
```
bogon:test apple$ echo "#include <stdio.h>#" > stdio.c
bogon:test apple$ gcc -M stdio.c 
stdio.c:1:19: warning: extra tokens at end of #include directive [-Wextra-tokens]
#include <stdio.h>#
                  ^
                  //
stdio.o: stdio.c /usr/include/stdio.h /usr/include/sys/cdefs.h \
  /usr/include/sys/_symbol_aliasing.h \
  /usr/include/sys/_posix_availability.h /usr/include/Availability.h \
  /usr/include/AvailabilityInternal.h /usr/include/_types.h \
  /usr/include/sys/_types.h /usr/include/machine/_types.h \
  /usr/include/i386/_types.h /usr/include/sys/_pthread/_pthread_types.h \
  /usr/include/sys/_types/_va_list.h /usr/include/sys/_types/_size_t.h \
  /usr/include/sys/_types/_null.h /usr/include/sys/stdio.h \
  /usr/include/sys/_types/_off_t.h /usr/include/sys/_types/_ssize_t.h \
  /usr/include/secure/_stdio.h /usr/include/secure/_common.h
```
  gcc -M  somefile.c 将somefile.c所有依赖关系打印出来,包括标准类库头。 -MM不包含标准类库头。
  
  不错吧， 不过这里是通过运行gcc获取的。传统上有两种方法可用来将自动产生的依赖关系纳入makefile，第一种也是最古老的方法，九十在makefile结尾加入一行内容: `# Automatically generated dependencies follow - Do Not Edit`
  然后编写一个脚本以便加入这些自动产生的脚本。这么做当然必手动加入要好，但不是很漂亮。 第二种方法就是为make加入一个include指令。如今大多数的make版本都支持include指令，GNU make当然一定可以这么做。
  
  因此，诀窍就是编写一个makefile工作目标，此工作目标的动作就是以-M选项对所有源文件执行gcc,并将结果保存到一个依存文件中(dependency file), 然后重新执行make以便把刚才所产生的依存文件引入makefile, 这样咎可以触发我们所需要的更新(即加入动作)。在GNU make中国年，你可以使用如下的规则来实现此目的:
```
depend: count_words.c lexer.c counter.c
  $(CC) -MM $(CPPFLAGS) $^ > $@

include depend
```
  运行make编译程序之前，你首先应该执行make depend以产生依存关系。这么做虽然不错，但是当人们对源代码加入或移除依存关系时，通常不会重新产生depend文件。这会造成无法重新编译源文件，整个工作又会变得一团乱。
  
  在GNU make中，你可以使用一个很酷的功能以及一个简单的算法来解决此问题。首先介绍这个简单的算法。如果我们每次为每个源文件产生依存关系，将之存入相应的依存文件(一个.d的文件名)，并以该.d文件为工作目标来加入此依存规则，这样，当源文件被改动时，make就会知道需要更新该.d文件(以及目标文件):`counter.o counter.d: src/counter.c include/counter.h include/lexer.h`
  你可以使用如下的模式规则以及命令脚本来产生这项规则:
```
%.d: %.c
  $(CC) -MM $(CPPFLAGS) $< > $@.$$$$; \
  SED 'S,\($*\)\.o[ :]*,\1.o $@ : ,g' < $@.$$$$ > $@; \
  rm -f $@.$$$$
```

  现在介绍这个很酷的功能。make会把include指令所指定的任何文件视为一个需要更新的工作目标。所以，如果我们标明要引入.d文件，则make会在读进makefile文件时自动创建这些.d文件。我们的makefile加入了自动产生依存关系的功能之后会变成下面这样:
```
VPATH = src include
CPPFLAGS = -I include

SOURCES = count_words.c \
  lexer.c               \
  counter.c
  
count_words: counter.o lexer.o -ll
count_words.o: counter.h
counter.o: counter.h lexer.h
lexer.o: lexer.h

include $(subst .c,.d,$(SOURCES))

%.d: %.c
  $(CC) -MM $(CPPFLAGS) $< > $@.$$$$; \
  SED 's,\($*\)\.o[ :]*,\1.o $@ : ,g' < $@.$$$$ > $@; \
  rm -f $@.$$$$
```
