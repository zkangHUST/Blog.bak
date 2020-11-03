# Makefile傻瓜教程
Makefile是组织代码编译的一种简单办法。本教程甚至还没有触及到make工具的表面，但是作为入门指南，它可以帮助你快速又轻松地为中小型项目创建自己的Makefile。
## 1. 一个简单的例子
让我们从一个简单例子开始，首先我们需要准备三个文件。这三个文件可以分别代表主程序，函数实现和函数声明。
```C
//hellomake.c
#include<hellomake.h>
int main()
{
    // call a func in another file.
    myPrintHelloMake();
    return 0;
}

//hellofunc.c
#include<stdio.h>
#include<hellomake.h>

void myPrintHelloMake()
{
    printf("Hello, makefiles!\n");
    return;
}

//hellomake.h
void myPrintHelloMake();
```
有了这三个文件，我们可以用下面的指令来编译
```C
gcc -o hellomake hellomake.c hellofunc.c -I.
```
这条指令编译两个c文件，并且生成可执行文件hellomake。`-I.`参数告诉工gcc从当前目录寻找头文件hellomake.h。如果没有makefile的话，我们测试/修改/调试代码的时候一般在terminal上用上下键切换编译指令，这样就不用每编译一次都敲一次编译指令了。

但是不幸的是，这种方法又两个缺点：1， 如果你不小心丢掉了编译指令（比如不小心关掉了terminal）或者换了一个电脑，你就得从新敲一遍编译指令；2，就算你只修改了某一个c文件，你也必须把所有的源文件全部重新编译一次，这个是非常耗时间的，也是完全没有必要的。这两个问题都可以通过写一个makefile来解决。

## 2. Makefile version1
最简单的Makefile文件长这样：
```MakeFile
hellomake: hellomake.c hellofun.c
    gcc -o hellomake hellomake,c hellofun.c -I.
```
把这两行写入到一个名为Makefile或makefile的文件中，然后在terminal输入`make`，系统就会按照makefile文件中的规则编译你的代码。注意，`make`命令不带参数的话，会默认执行makefile文件中的第一条规则。此外，通过将命令依赖的文件列表放在`:`之后的第一行，make知道如果其中的任何文件发生更改，则需要执行hellomake规则。这样就马上解决了第一个问题，不需要在terminal中上下翻找最后一个编译命令了。但是这样写，make仍然不能有效的只编译最新的更改。

有一点需要特别注意的是，在gcc命令之前有一个tab字符。事实上，在任何指令之前，都必须加一个tab字符，这是make的要求，不加tab， make会不开心的：）

## 3. Makefile version2
我们更进一步，让makefile更加高效一点。
```Makefile
CC=gcc
CFLAG=-I.
hellomake:hellomake.o hellofunc.o
    $(CC) -o hellomake hellomake.o hellofunc.o
```
现在，我们在makefile中定义了一些常量CC和CFLAG，这些常量告诉make如何编译hellomake.c和hellofunc.c文件。CC宏定义了要使用哪个编译器，CFLAG是一些编译标志。通过将目标文件hellomake.o和hellofunc.o放在依赖列表中，make就知道首先需要编译c文件得到目标文件，然后链接得到可执行文件hellomake。对于大多数小型项目来说，这个形式的makefile已经足够了。但是这个版本的makefile还忽略了一点：对头文件的依赖。如果你修改了hellomake.h文件，然后重新执行make，这时候即使需要重新编译c文件，make也不会重新编译的。为了解决这个问题，我们需要告诉make，c文件依赖哪些h文件。

## 4. Makefile version3
```makefile
CC=gcc
CFLAG=-I.
DEPS = hellomake.h

%.o:%.c $(DEPS)
    $(CC) -c -o $@ $<  $(CFLAGS)
hellomake: hellomake.o hellofunc.o
    $(CC) -o hellomake hellomake.o hellofunc.o
```
在这个版本中，我们新加了一个常量DEPS，这是c文件依赖的头文件集合。然后，我们又定义了一个适用于所有*.o文件的规则，这个规则说明.o文件依赖于同名的.c文件和DEPS中包含的头文件，为了产生.o文件，make需要使用CC常量定义的编译器编译.c文件。`-c`标志表示产生目标文件, `-o $@`表示将输出文件命名为`:`左边的文件名， `$<`表示依赖列表中的第一个项。

最后使用特殊的宏`$@ $^`做最后一次简化，让编译规则更加通用。`$@ $^`分别表示`:`左边和右边。在下面的例子中，所有的头文件都应该作为DEPS宏的一部分，所有的目标文件*.o都应该作为OBJ宏的一部分。

## 5. makefile version4
```makefile
CC=gcc
CFLAGS=-I.
DEPS = hellomake.h
OBJ = hellomake.o hellofunc.o

%.o:%.c $(DEPS)
    $(CC) -c -o $@ $<  $(CFLAGS)
hellomake: $(OBJ)
    $(CC) -o $@ $^ $(CFLAGS)
```
如果我们想把头文件，源文件和其他库文件分别放在不同的文件夹，那么makefile该怎么写呢？
另外，我们可以隐藏那些烦人的中间文件（目标文件）吗？当然可以！下面的makefile定义了include文件夹，lib文件夹路径，并且把目标文件放到src文件夹的子文件夹obj里面。同时，还定义了一个宏，用于包含任何你想要包含的库，不如math库`-lm`。这个makefile文件应该位于src目录，注意，这个makefile 还包含了一个规则用于清理source和obj文件夹，只需要输入`make clean`即可。`.PHONY`规则可以让make不去改动任何名为clean的文件（如果有的话）。

## 6. makefile version5
```makefile
IDIR = ../include
CC=gcc
CFLAGS=-I$(IDIR)

ODIR=obj
LDIR=../lib

LIBS=-lm

_DEPS=hellomake.h
DEPS=$(pathsubst %,$(IDIR)/%,$(_DEPS))

_OBJ= hellomake.o hellofunc.o
OBJ = $(pathsubst %,$(ODIR)/%,$(_OBJ))

$(ODIR)/%.o:%.c $(DEPS)
    $(CC -c -o $@ $< $(CFLAGS
hellomake:$(OBJS)
    $(CC) -o $@ $^ $(CFLAGS) $(LIBS)
.PHONY:clean
clean:
    rm -f $(ODIR)/*.o *~ core $(INCDIR)/*~
```
现在，我们有了非常不错的makefile，你可以对其进行简单的修改来管理中小型软件项目了。你也可以将多个规则写到一个makefile里面，甚至可以在一个规则中调用其他规则。有关make和makefile的更多资料，敬请参考《GNU make手册》。

## 参考资料
[1]. [A Simple Makefile Tutorial](https://www.cs.colby.edu/maxwell/courses/tutorials/maketutor/)