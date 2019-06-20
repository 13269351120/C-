1)i++与++i哪个效率更高？
* 对于内建数据类型，除去编译器优化的影响，从生成的汇编角度看，效率差别不大。
* 对于自定义数据类型，前缀式(++i)可以返回对象的引用，而后缀式(i++)必须返回对象的值，所以导致大对象的时候产生了较大的复制开销，引起效率降低。因此在自定义类型，应该尽可能地使用前缀式，它天生『体质』较佳。

2）关键字static
* static全局变量或者是static局部变量，只会被初始化一次，而且只能在当前文件范围内使用，相比于普通全局变量，static全局变量拥有本地化数据和代码范围的优势。
* static函数，如果是类的static函数，相比于类的普通成员函数第一个隐含的参数this区别外，它是属于类本身，在使用范围上也在当前文件中有效。

3）指针和引用
* 指针在运行时可以改变其指向的值，而引用从一而终，也可以简单理解为常量指针，所以必然需要初始化一个初值，在使用引用时，无需像指针一样需要检验其非空，间接的简化编程。
* 引用机制解决的主要问题：支持运算符重载，在运算符重载中，如果是```T operator + (const T *a, const T *b)``` ，那么用户使用的时候，需要使用```&a+&b```形式。会很麻烦，所以需要引入引用机制，这看上去是编程语言简洁性和正确性的妥协。
* 引用虽说是别名，而其本质上是由指针来实现的，所以引用与指针脱不了干系，也一直容易弄混，其本身是存在空间占用的，但是当我们去取引用的地址的时候，编译器会对引用（指针）解引用然后取被引用的值得地址，```int a = 1; int &b = a;``` &b 会被解释&(*b) 即&a。引用是需要有空间的，存放着引用的值得地址，但是引用的地址对程序员是透明的，看上去也没什么用。

4)内存对齐
原因：各个硬件平台对存储空间的处理上有很大的不同。一些平台对某些特定类型的数据只能从某些特定地址开始存取。例如有些平台每次读都是从偶地址开始，如果一个int存放在偶地址开始的地方，那么一个读周期就可以读出；而如果存放在奇地址开始的地方，可能会需要2个读周期，并对两次读出的结果的高低字节进行拼凑才能得到该int型数据，显然在读取效率上会下降很多，这是空间和时间的博弈。
准则：
* 首地址（head）：首地址能够被最宽基本类型成员大小整除。
* 总大小（total）：总大小是最宽基本类型成员的整数倍，不够就在最后加上填充字节。
* 偏移量（offset）：每个成员相对于结构体首地址的偏移量是本身大小的整数倍。
看上去比较难记：head total offset(HTO)，都是整除，涉及到整体的，比如首地址和总大小，就和老大有关（最宽成员），涉及到自身的，个体偏移，就和自身大小有关。

拓展：
实现一个在可各个平台上内存对齐的malloc？
核心思路与技巧：
* `malloc`分配的内存不一定是内存对齐的，所以大致的方向是利用malloc额外分配一些内存（以空间换时间），在分配好的内存中选取内存对齐的地址返回给用户。
* 在释放内存时，由于用户拿到的内存不是`malloc()`分配的首地址，不能让用户直接调用系统自带的`free()`函数，需要配套实现一个`aligned_free()`。
* 使用 (size + alignment) &~ (alignment - 1)这样的位运算可以迅速得到向上对齐的内存位置，这里还需要额外考虑将原先`malloc()`得到的内存地址存放起来，可以简单的将其放置在对齐内存地址的前一块内存中，作为cookie。这样就需要预留给cookie。
综上所述：
* 需要分配的内存大小为：`new_size = size + alignment -1 + sizeof(void*)` 。
* 对齐后返回给用户的地址为：`aligned = (void**)((origin + sizeof(void*) + alignment-1) &~ (alignment-1))` 其中origin为malloc返回的原地址，这里把`aligned`转成(void**)大概是可以直接用-1来取得上一块内存而无需考虑平台。
* 存放原地址的位置：`aligned[-1] = origin` 

可能的实现方案：
```cpp
void *aligned_malloc(int size, int alignment) {
    if (alignment & (alignment-1)) {
        std::cout << "alignmet should be power of 2" << std::endl;
        return NULL;
    }
    int offset = alignment - 1 + sizeof(void*);  // sizeof(void*):for saving real address
    void *origin = NULL;
    void **aligned = NULL;
    if (origin = malloc(size + offset) == NULL) {
        std::cout << "ERROR: malloc failed" << std::endl;
        return NULL;
    }
    aligned = (void**)(((size_t)origin + offset) &~ (alignment-1)); 
    aligned[-1] = origin; // saving real address
    return aligned;
}

void aligned_free(void* aligned) {
    void* origin = (void**)aligned[-1];
    free(origin);
}
```
