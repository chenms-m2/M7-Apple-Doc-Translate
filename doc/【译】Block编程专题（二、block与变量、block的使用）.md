[Blocks Programming Topics 官档传送门](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Blocks/Articles/00_Introduction.html#//apple_ref/doc/uid/TP40007502)

本文翻译自2011-03-08版的官档。
您可以在官档结尾的Document Revision History中查阅版本。
注：【】包含的部分引自原文。

## block和变量
这一章讲述block和变量的交互，也涉及到内存管理。

## 变量类型
在block体中，变量一般有5种处理方式。
你可以使用3种标准的变量类型，就像在方法中使用一样：
- 全局变量（包括局部静态变量）
- 全局函数（从技术角度说这不是变量）
- 作用域中的局部变量和形参
block还支持两种变量类型：
- 函数级别的 __block变量【At function level are __block variables】，block体中可以修改它的值，并且block被复制到堆上时可以维护这个变量。
- const导入【const imports】
最后，在方法实现中，block还可能引用Objective-C（后文简称OC）对象。可参考Object and Block Variables章节。

在block中使用变量有下面5条规则：
1. 全局变量始终可访问，包括作用域中的局部静态变量。
2. 形参是始终可用的（类似函数形参）。
3. 作用域中的局部变量会被捕获为const变量。
block会捕获其block体所在位置的局部变量值（译注：意译）【Their values are taken at the point of the block expression within the program】，嵌套的block会从最近的外层捕获值。
4. 使用__block修饰的局部变量相当于按引用传值，因此可以修改其值。
作用域中任意修改该值的行为都会在block中体现，包括其他block对该变量的赋值。详情请参考[The __block Storage Type](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Blocks/Articles/bxVariables.html#//apple_ref/doc/uid/TP40007502-CH6-SW6)章节。
5. block中声明的局部变量，行为类似于函数中的局部变量。
block执行时会创建这些参数，这些参数也可以被内部的block捕获。【These variables can in turn be used as const or by-reference variables in blocks enclosed within the block】

下面的代码展示了block使用局部变量：
```
int x = 123;
 
void (^printXAndY)(int) = ^(int y) {
 
    printf("%d %d\n", x, y);
};
 
printXAndY(456); // prints: 123 456
```
需要注意的是，在block中给捕获的局部变量赋值会导致error：
```
int x = 123;
 
void (^printXAndY)(int) = ^(int y) {
 
    x = x + y; // error
    printf("%d %d\n", x, y);
};
```
如果你想修改捕获的局部变量的值，你需要用__block修饰符，接下来的一节会讲到。

### __block存储类型
你可以使用__block来修饰局部变量，以此来指示block体中可以修改此变量值。__block和register、auto、static等存储类型修饰符是类似的，和它们是互斥的。

__block变量可以被它声明所在的作用域访问，也可以被捕获它的block（包括复制的block）访问。即使__block被声明的栈帧结束，它也未必会被销毁，如果捕获它的block在栈帧结束后在堆上存货，__block也随之存活。在同一作用域生成的block都可以捕获该作用域的__block变量。

出于优化的考虑，__block变量也是在栈上创建的（和block一样），如果使用Block_copy复制了block（OC中是对block发送copy消息），__block变量也会和block一样被复制到堆上，因此__block变量的地址可能会变化。

关于__block变量有两个限制条件：它们不能是变长数组【variable length arrays】，也不能是包含C99变长数组（C99 variable-length arrays）的结构体。

下面的代码展示了使用__block变量：
```
__block int x = 123; //  x lives in block storage
 
void (^printXAndY)(int) = ^(int y) {
 
    x = x + y;
    printf("%d %d\n", x, y);
};
printXAndY(456); // prints: 579 456
// x is now 579

```

下面的代码展示了__block变量和其他类型变量的交互：
```
extern NSInteger CounterGlobal;
static NSInteger CounterStatic;
 
{
    NSInteger localCounter = 42;
    __block char localCharacter;
 
    void (^aBlock)(void) = ^(void) {
        ++CounterGlobal;
        ++CounterStatic;
        CounterGlobal = localCounter; // localCounter fixed at block creation
        localCharacter = 'a'; // sets localCharacter in enclosing scope
    };
 
    ++localCounter; // unseen by the block
    localCharacter = 'b';
 
    aBlock(); // execute the block
    // localCharacter now 'a'
}

```

### 对象和block变量
block对使用OC对象、C++对象以及block作为变量作了支持。

#### OC对象
block复制时，会对捕获的OC对象持有强引用，如果你在方法中使用block捕获了对象变量：
- 如果你通过引用访问了实例变量，block会强引用self
- 如果你通过值访问了实例变量，block会强引用该变量（译注：看示例代码，感觉不是这意思，是直接捕获成员变量还是捕获局部变量的区别）
下面的代码展示两种情况：
```
dispatch_async(queue, ^{
    // instanceVariable is used by reference, a strong reference is made to self
    doSomethingWithObject(instanceVariable);
});
 
 
id localVariable = instanceVariable;
dispatch_async(queue, ^{
    /*
      localVariable is used by value, a strong reference is made to localVariable
      (and not to self).
    */
    doSomethingWithObject(localVariable);
});
```
如果要重写特定对象变量的行为，你可以考虑用__block变量。

#### C++对象（不了解C++，故原文抄录如下）
In general you can use C++ objects within a block. Within a member function, references to member variables and functions are via an implicitly imported this pointer and thus appear mutable. There are two considerations that apply if a block is copied:
- If you have a __block storage class for what would have been a stack-based C++ object, then the usual copy constructor is used.
- If you use any other C++ stack-based object from within a block, it must have a const copy constructor. The C++ object is then copied using that constructor.

#### block
当你复制一个block时，它所引用的所有block需要的都会被复制（可能整个block树都会被复制）。如果你有一个block变量，并且在这个block中引用了其他的block，其他block会被复制。

## 使用block
### 调用block【Invoking a Block】
如果你声明了一个block变量，你可以像使用函数指针那样使用它，如下面代码所示：
```
int (^oneFrom)(int) = ^(int anInt) {
    return anInt - 1;
};
 
printf("1 from 10 is %d", oneFrom(10));
// Prints "1 from 10 is 9"
 
float (^distanceTraveled)(float, float, float) =
                         ^(float startingSpeed, float acceleration, float time) {
 
    float distance = (startingSpeed * time) + (0.5 * acceleration * time * time);
    return distance;
};
 
float howFar = distanceTraveled(0.0, 9.8, 1.0);
// howFar = 4.9
```
更为常见的是，你将block作为函数或方法的参数传入。多数情况下，你会创建一个“内联”的block。

### 使用block作为函数的参数
你可以把block作为函数的参数，和其他参数没有区别。通常你不需要声明block，而是在需要的位置传入一个内联的block。下面的例子使用了qsort_b函数。qsort_b类似于标准的qsort_r函数，不过以block作为最后一个参数。
```
char *myCharacters[3] = { "TomJohn", "George", "Charles Condomine" };
 
qsort_b(myCharacters, 3, sizeof(char *), ^(const void *l, const void *r) {
    char *left = *(char **)l;
    char *right = *(char **)r;
    return strncmp(left, right, 1);
});
// Block implementation ends at "}"
 
// myCharacters is now { "Charles Condomine", "George", "TomJohn" }
```
注意到block包含在函数的参数列表中。
接下来的例子展示如何在dispatch_apply方法中使用block，dispatch_apply函数声明如下所示：
```
void dispatch_apply(size_t iterations, dispatch_queue_t queue, void (^block)(size_t));
```

函数向queue提交一个block，会被多次调用。它有三个参数，第一个参数指定迭代次数，第二个参数制定执行队列，第三个参数就是要执行的block，它有一个参数，当前迭代的index。
你可以使用dispatch_apply简单地打印index，如下所示：
```
#include <dispatch/dispatch.h>
size_t count = 10;
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
 
dispatch_apply(count, queue, ^(size_t i) {
    printf("%u\n", i);
});
```

### 使用block作为方法的参数
Cocoa提供了很多使用block的方法，你可以像传入其他参数一样传入block参数。下面的示例代码展示了传入block参数：
```
NSArray *array = @[@"A", @"B", @"C", @"A", @"B", @"Z", @"G", @"are", @"Q"];
NSSet *filterSet = [NSSet setWithObjects: @"A", @"Z", @"Q", nil];
 
BOOL (^test)(id obj, NSUInteger idx, BOOL *stop);
 
test = ^(id obj, NSUInteger idx, BOOL *stop) {
 
    if (idx < 5) {
        if ([filterSet containsObject: obj]) {
            return YES;
        }
    }
    return NO;
};
 
NSIndexSet *indexes = [array indexesOfObjectsPassingTest:test];
 
NSLog(@"indexes: %@", indexes);
 
/*
Output:
indexes: <NSIndexSet: 0x10236f0>[number of indexes: 2 (in 2 ranges), indexes: (0 3)]
*/
```
下面的代码展示了使用__block变量的场景：
```
__block BOOL found = NO;
NSSet *aSet = [NSSet setWithObjects: @"Alpha", @"Beta", @"Gamma", @"X", nil];
NSString *string = @"gamma";
 
[aSet enumerateObjectsUsingBlock:^(id obj, BOOL *stop) {
    if ([obj localizedCaseInsensitiveCompare:string] == NSOrderedSame) {
        *stop = YES;
        found = YES;
    }
}];
 
// At this point, found == YES
```
### 复制block
通常，你不需要复制block。只有当你需要延时执行block时（超出block声明时所在的作用域），才需要将block复制到堆上。
你可以使用下面的函数来复制及释放block：
```
Block_copy();
Block_release();
```
为了避免内存泄露，你要保持复制和释放函数的成对调用。

### 避免使用的模式
block表达式（ ^{ ... }）在栈上创建的，生命周期和其被创建时所在的作用域一致，下面示例中的模式是要**避免**的：
```
void dontDoThis() {
    void (^blockArray[3])(void);  // an array of 3 block references
 
    for (int i = 0; i < 3; ++i) {
        blockArray[i] = ^{ printf("hello, %d\n", i); };
        // WRONG: The block literal scope is the "for" loop.
    }
}
 
void dontDoThisEither() {
    void (^block)(void);
 
    int i = random():
    if (i > 1000) {
        block = ^{ printf("got i at: %d\n", i); };
        // WRONG: The block literal scope is the "then" clause.
    }
    // ...
}
```

### 调试
- 你可以设置断点进入block单步调试。你可以在GDB session中使用invoke-block执行block，如下所示：
```
$ invoke-block myBlock 10 20
```
如果你要传入C字符串，需要用引号包含字符串，例如你想向doSomethingWithString传入this string，如下所示：
```
$ invoke-block doSomethingWithString "\"this string\""
```