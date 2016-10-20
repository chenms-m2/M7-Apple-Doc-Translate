[Blocks Programming Topics 官档传送门](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Blocks/Articles/00_Introduction.html#//apple_ref/doc/uid/TP40007502)

本文翻译自2011-03-08版的官档。
您可以在官档结尾的Document Revision History中查阅版本。
注：【】包含的部分引自原文。

## 简介
block【Block objects】是C级别的语法，runtime层面的功能；它很像标准C函数，不过除了可执行代码外，它还可以捕获栈或堆上的变量【variable bindings to automatic (stack) or managed (heap) memory】。

你可以使用block作为API的参数，也可以作为变量【optionally stored】，block可以被多线程访问。block一个典型应用场景是作为回调，因为block不仅有回调时要执行的代码，还捕获了回调时可能用到的变量。

支持block的GCC和Clang被集成进了OS X v10.6 Xcode developer tools，你可以在OS X v10.6及以上、iOS4及以上的版本使用block。block运行时是开源的，你可以在[LLVM’s compiler-rt subproject repository](http://llvm.org/svn/llvm-project/compiler-rt/trunk/)查看。block也被作为[](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1370.pdf)提交给了C标准工作组【C standards working group】。由于Objective-C（后文简称OC）与C++都是从C衍生的，因此block被设计为支持这三种语言（也包含OC++），它的语法就可以体现出这一目标。

从本文档中你可以学到如何在C、C++以及OC中使用block。

### 本文档的组织
本文档包含以下章节：
- Getting Started with Blocks 提供了一个快速而实用的block简介。
- Conceptual Overview 提供了block中概念的简介。
- Declaring and Creating Blocks 向你展示了如何声明block变量，以及如何实现block。
- Blocks and Variables 描述了block和变量的交互，也定义了__block这个存储类型修饰符。
- Using Blocks 展示了各种block使用场景。

## 开始使用block
下面的章节通过一个实际的例子来开始block之旅。

### 声明和使用block
- ^符号用来声明block变量，也用来标识出block表达式【a block literal】的开始，block的实现被{}符号环绕，如下面的例子所示（和C一样，语句的结束使用;符号）：
```
int multiplier = 7;
int (^myBlock)(int) = ^(int num) {
    return num * multiplier;
};
```
这个示例如下图所示：

![](image/blocks.jpg)

注意到block可以使用它被声明时所在域中的变量。
如果你定义了一个block变量，你使用它的方式类似使用函数变量，如下：
```
int multiplier = 7;
int (^myBlock)(int) = ^(int num) {
    return num * multiplier;
};
 
printf("%d", myBlock(3));
// prints "21"

```
### 直接使用block
大多数情况下，你不需要声明block变量，通常时在需要block参数时，直接写block表达式传给它，下面的示例中的qsort_b函数，类似于标准的qsort_r函数，只不过最后一个参数使用了block：
```
char *myCharacters[3] = { "TomJohn", "George", "Charles Condomine" };
 
qsort_b(myCharacters, 3, sizeof(char *), ^(const void *l, const void *r) {
    char *left = *(char **)l;
    char *right = *(char **)r;
    return strncmp(left, right, 1);
});
 
// myCharacters is now { "Charles Condomine", "George", "TomJohn" }

```

### Cocoa中的block
Cocoa框架中有些方法使用block作为参数，典型的场景是集合类的遍历操作，或者作为回调。下面的示例展示了如何使用block作为NSArray的sortedArrayUsingComparator:方法的参数，示例中block是NSComparator类型的局部变量：

```
NSArray *stringsArray = @[ @"string 1",
                           @"String 21",
                           @"string 12",
                           @"String 11",
                           @"String 02" ];
 
static NSStringCompareOptions comparisonOptions = NSCaseInsensitiveSearch | NSNumericSearch |
        NSWidthInsensitiveSearch | NSForcedOrderingSearch;
NSLocale *currentLocale = [NSLocale currentLocale];
 
NSComparator finderSortBlock = ^(id string1, id string2) {
 
    NSRange string1Range = NSMakeRange(0, [string1 length]);
    return [string1 compare:string2 options:comparisonOptions range:string1Range locale:currentLocale];
};
 
NSArray *finderSortArray = [stringsArray sortedArrayUsingComparator:finderSortBlock];
NSLog(@"finderSortArray: %@", finderSortArray);
 
/*
Output:
finderSortArray: (
    "string 1",
    "String 02",
    "String 11",
    "string 12",
    "String 21"
)
*/

```

### __block变量
block的强大功能之一是可以修改捕获的变量。你可以使用__block标记变量为可被block修改的。以上一节的代码为例，你可以在比较字符串的过程中使用一个block变量记录相同字符串的个数，如下所示：
```
NSArray *stringsArray = @[ @"string 1",
                          @"String 21", // <-
                          @"string 12",
                          @"String 11",
                          @"Strîng 21", // <-
                          @"Striñg 21", // <-
                          @"String 02" ];
 
NSLocale *currentLocale = [NSLocale currentLocale];
__block NSUInteger orderedSameCount = 0;
 
NSArray *diacriticInsensitiveSortArray = [stringsArray sortedArrayUsingComparator:^(id string1, id string2) {
 
    NSRange string1Range = NSMakeRange(0, [string1 length]);
    NSComparisonResult comparisonResult = [string1 compare:string2 options:NSDiacriticInsensitiveSearch range:string1Range locale:currentLocale];
 
    if (comparisonResult == NSOrderedSame) {
        orderedSameCount++;
    }
    return comparisonResult;
}];
 
NSLog(@"diacriticInsensitiveSortArray: %@", diacriticInsensitiveSortArray);
NSLog(@"orderedSameCount: %d", orderedSameCount);
 
/*
Output:
 
diacriticInsensitiveSortArray: (
    "String 02",
    "string 1",
    "String 11",
    "string 12",
    "String 21",
    "Str\U00eeng 21",
    "Stri\U00f1g 21"
)
orderedSameCount: 2
*/
```
更详细的信息请参考[Blocks and Variables](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Blocks/Articles/bxVariables.html#//apple_ref/doc/uid/TP40007502-CH6-SW1)一章。

## 概念综述
block为C及C衍生语言（C++、OC）提供了一种特殊的函数体表达式【an ad hoc function body as an expression】，在其他语言中，类似的功能有时被叫做闭包，我们提到其他语言中闭包的时候，一般也用block一词，除非某些语境下需要明确区分。

### block的函数性
block是匿名内联的代码集合：
- 有类似函数的参数列表
- 有隐式的或显式声明的返回值类型
- 能捕获其被定义域中的变量
- 能修改捕获的变量
- 能共享其被定义域中其他block对变量的修改
- 在其被定义域（如方法栈帧）结束后，依然可以使用或修改捕获的变量

你可以复制一个block甚至传给其他线程延迟执行（或者在本线程runloop的后续处理周期），编译器和runtime可以保证被捕获的变量在还有block引用时始终存在。尽管block是C和C++的实现，但也可以视为OC的对象。

### 使用场景
blcok通常是小规模的、自包含的代码片段，它比较常用的场景是封装一些工作，常用于并发执行、集合迭代、回调等。
block比传统函数更适合作为回调的理由如下：
1. block允许你在调用方法的地方写回调代码，因此系统框架API经常使用block作为参数
2. block可以访问局部变量
使用回调函数，你需要额外保存相关的变量信息，而使用block你可以在block体中直接使用局部变量。


## 声明和创建block
### 声明block引用
block型变量持有block的引用。它的定义方式类似于函数指针，唯一的不同是使用^代替*，block可以使用C的类型，下面示例中block都是合法的：
```
void (^blockReturningVoidWithVoidArgument)(void);
int (^blockReturningIntWithIntAndCharArguments)(int, char);
void (^arrayOfTenBlocksReturningVoidWithIntArgument[10])(int);
```
block也支持可变参数(...表示的参数)，没有参数的话，参数列表必须显式的指定为void。
block被设计为完全类型安全的，为达到此目的，它向编译器提供了可供验证block使用、参数传值、返回值赋值的所有元信息。【locks are designed to be fully type safe by giving the compiler a full set of metadata to use to validate use of blocks, parameters passed to blocks, and assignment of the return value】你可以将block引用转型为任意类型的引用，反之亦反。但是你不能用*来给block引用解引用，因为编译期间无法获得block的尺寸。

你可以使用typedef给block创建类型，在代码多处使用block的场景中，使用typedef是优秀的实践：
```
typedef float (^MyBlockType)(float, float);
 
MyBlockType myFirstBlock = // ... ;
MyBlockType mySecondBlock = // ... ;

```

### 创建block
使用^作为block表达式的开始，随后是()中的参数列表，以及{}中的block体。下面的示例代码定义一个简单block，并将其赋值给了之前声明oneFrom变量：
```
float (^oneFrom)(float);
 
oneFrom = ^(float aFloat) {
    float result = aFloat - 1.0;
    return result;
};
```
如果你没有显示的声明block的返回值类型，系统会从block内容中推测，如果返回值是推测的并且参数列表为空，你可以省略block参数列表中的void，如果block中有多个return语句，必须保证它门返回的类型是一致的，必要的时候要添加转型。

### 全局block
在文件级别，你也可以使用全局block
```
#import <stdio.h>
 
int GlobalInt = 0;
int (^getGlobalInt)(void) = ^{ return GlobalInt; };
```


