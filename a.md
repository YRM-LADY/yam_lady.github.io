### 模块 :
###### 在 C++ 程序中改进模块化是一个显然的需求。从 C 语言中，C++ 继承了 #include 机制，依赖从头文件使用文本形式包含 C++ 源代码，这些头文件中包含了接口的文本定义。一个流行的头文件可以在大型程序的各个单独编译的部分中被 #include 数百次。  
######  基本问题是：  
> * 不够卫生：一个头文件中的代码可能会影响同一翻译单元中包含的另一个 #include 中的代码的含义，因此 #include 并非顺序无关。宏是这里的一个主要问题，尽管不是唯一的问题。
> * 分离编译的不一致性：两个翻译单元中同一实体的声明可能不一致，但并非所有此类错误都被编译器或链接器捕获。
> * 编译次数过多：从源代码文本编译接口比较慢。从源代码文本反复地编译同一份接口非常慢。

###### 在 C++ 的早期，通常 10% 的文本来自头文件，但现在它更可能是 90% 甚至 99%。考虑下面的代码：
```
#include<iostream>
int main()
{
    std::cout << "Hello, World\n";
}
```
######  这段标准代码有 70 个字符，但是在 #include 之后，它会产生 419909 个字符需要编译器来消化。尽管现代 C++ 编译器已有骄人的处理速度，但模块化问题已经迫在眉睫。
### modules 试图解决的痛点
###### 模块是一个用于在翻译单元间分享声明和定义的语言特性。 它们可以在某些地方替代使用头文件。比如你有多个翻译单元, 每一个都调用了 iostream, 就得都处理一遍. 预处理完的源文件就会立刻膨胀. 真的会很慢.有了 modules 以后, 我们可以模块化的处理. 已经编译好的 modules 直接变成编译器的中间表示进行保存, 需要什么就取出什么, 这就非常地快速了.比如你只是用了 cout 的函数, 那么编译器下次就不需要处理几万行, 直接找 cout 相关函数用就行了。

######  我们通常可以认为，一个程序是由一组翻译单元（Translated units）组合而成的。这些翻译单元在没有额外信息的情况下是互相独立的，要将他们联系到一起需要这些翻译单元声明外部名称，编译器和链接器就是通过这些外部名称把独立的翻译单元组合起来的，而模块就可以认为是一个或者一组独立的翻译单元以及一组外部名称的组合体。那么模块名（Module name）就是引用组合体符号，模块单元（Module unit）其实就是组合体的翻译单元，模块接口单元（Module interface unit）很显然就是组合体里的那一组外部名称。

######  正规来说，一个模块由模块单元组成，模块单元分为模块接口单元和模块实现单元（Module implementation unit）。另外一个模块可以有多个模块分区，模块分区也是模块单元，模块分区的目的是方便模块代码的组织。对于每个模块，必须有一个没有分区的模块接口单元，该模块单元称为主模块接口单元。 导入一个模块，实际上导入的就是主模块的接口。
### 关于 modules 的技术细节
###### modules 编译出来会有两个部分:
> 1. 编译器缓存, 基本上代表了全部源代码信息. 编译器已经处理好了源代码, 并且把内部缓存留了下来. 下次读取的时候会更快. 主流三大编译器都有 lazy loading 的功能, 可以只取出需要的部分. 这不就快了?
> 2. 编译出来的 object, 这个 object 只有链接的时候需要了, 不需要特别处理, 也不需要再次编译.
###### 所以啊, 就是有了这个缓存, 才会让它更快. 各个编译器不能通用哦. 而且编译选项不同的时候, 内部的表示不一样, 所以也不能通用.

###### 这个缓存文件:

> + 在 Clang 那里, 叫 BMI, 后缀名是 .pcm.
> + 在 GCC 那里, 叫 CMI, 后缀名是 .gcm.
> + 在 MSVC 那里, 叫 IFC, 后缀名是 .ifc.
###### 对于这个缓存文件, 这三家编译器还使用了 lazy loading 的技术. 也就是说需要哪些定义/声明就加载哪些. 编译器的这种 lazy loading 进一步提升了速度.

###### 如何做到接口隔离?
> 
C++ 标准中新提出了一种 module linkage, 意味着只有同一个 module 内部可见. 之前的 C++ 标准中只有 external linkage (全局可见) 和 internal linkage (同一个翻译单元可见).

为了实现这个 module linkage 功能, GCC 和 Clang 共同使用了一种新的 name mangling 技术. 具体地说, 如果

modules 对外 export 的名称, 按照平时方式进行 name mangling.
对于 modules 内部, 需要 module linkage 的名称, 使用一种全新的 _ZW 开头的 name mangling.
这两个技术一结合, 编译器就能分辨出啥是好的, 啥是坏的了. 编译器编译的时候, 就知道 _ZW 开头的函数只能同一个 module 内互相调用, 就在编译的时候进行隔离了.

这样做有一个巨大利好, 不用改动链接器了. 在链接器的角度看来 module linkage 和 external linkage 是一回事.



为了历史兼容性, 我们希望在 module 文件里面仍然插入一些 external linkage 的函数. 为此我们引入了 global module fragment 的语义, 

### modules 最重要的功能是什么



我原来感觉, 不引入多余的符号会是一个很大的亮点, 但是 modules 需要使用源代码进行分发. 所以这个 modules 做的隔离只是防止意外引入重名的符号罢了, 因为如果人家硬要链接上, 你也拦不住他.

但如果只是防意外引入的话, 以前我们也有 static 函数 (internal linkage) 可以做到, 或者我们可以使用 namespace 进行包装, 效果都是很好的.

现在的我感觉, modules 的 lazy loading 大大加快了编译速度, 这一点是远好于传统方式的. modules 可以用什么就取什么, 对于高频出现的文件 (比如标准库) 会很好. 这样, 你不用为你不使用的函数花编译时间.

所以, 以前的我感觉 modules 带来的隔离是最棒的 (也是最初设计它的原因之一), 但现在的我认为: 只取出代码中我们需要的部分, 大大提升编译速度才是 modules 最大的利好.
