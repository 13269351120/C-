## C++11新特性  
### shared_ptr
`shared_ptr`是C++11很重要的更新，其设计非常巧妙，有很多知识可以深挖。  
* enable_shared_from_this的作用以及实现原理  
* shared_ptr weak_ptr 源码剖析  
* ... 知识点遇到了再剖析  
---  
#### `enable_shared_from_this`的作用以及实现原理  
**使用场景**：当类A被shared_ptr管理，且在类A的成员函数里需要把当前对象作为参数传给其他函数的时候，就需要传递一个指向自身的shared_ptr   
**原因剖析**：  
问题一：为什么这里不能直接传this指针呢？  
使用智能指针的初衷就是为了方便内存资源的管理，这包括两个方面：  
1）内存中不存在无用的内容，即保证内存不发生泄漏的情况  
2）内存中如果还有用户可能需要用到某一对象，需要保证其生命，即保证不使用空悬指针  
参考解答：  
首先在使用形式上就有不一致的问题：某些地方使用智能指针，某些地方又使用裸指针。  
其次在功能上也有问题：使用this就是把对象交给了别人使用，而this指针又可能会失效，别人使用一个已经失效的指针就会发生错误。，所以这里需要使用shared_ptr来延长对象的生命期。  
问题二：那么可以使用shared_ptr<T>(this)吗？   
参考解答：  
也不可以，因为用一个裸指针去初始化多个shared_ptr，这些shared_ptr之间并没有任何的联系，他们的引用计数都是1，所以一定会发生内存上的重复释放。引用计数在shared_ptr的复制构造函数和赋值操作时增加。  
从而就引出了 `enable_shared_from_this`类。  
**实现原理**：  
重要的成员函数：shared_from_this()  
重要的成员变量：mutable weak_ptr<T> weak_this_   
以下是boost库中的实现：  
```cpp
shared_ptr<T> shared_from_this()
{
    shared_ptr<T> p( weak_this_ );
    BOOST_ASSERT( p.get() == this );
    return p;
}
```  
这个函数用一个weak_ptr来构造一个shared_ptr对象，这是会增加引用计数的。  
问题一：为什么这里要用weak_ptr来构造一个shared_ptr对象，而不是直接保存一个shared_ptr对象？  
参考解答：
之前说过shared_ptr的一个作用是延长对象的生存期，那么如果保存着一个shared_ptr，意味着对象永远不会被销毁，所以要用weak_ptr。  
问题二：既然weak_ptr中保存的就是对象的弱引用，那是在什么时候赋值的呢？  
参考解答：  
首先类A使用enable_shared_from_this需要继承自该类，那么在生成类A的对象的时候，会调用enable_shared_from_this的构造函数，以及A本身的构造函数。在执行enable_shared_from_this的构造函数的时候，虽然会初始化weak_this_，但是这个时候并没有指向任何对象。  
接着当外部程序把指向类A对象的指针作为初始化参数来初始化一个shared_ptr的时候，`shared_ptr<A> p(new A());`  
```cpp
template<class Y>
explicit shared_ptr( Y * p ): px( p ), pn( p ) 
{
    boost::detail::sp_enable_shared_from_this( this, p, p );
}
```  
它会调用sp_enable_shared_from_this   
```cpp
template< class X, class Y, class T >
inline void sp_enable_shared_from_this( boost::shared_ptr<X> const * ppx,
Y const * py, boost::enable_shared_from_this< T > const * pe )
{
  if( pe != 0 )
  {
      pe->_internal_accept_owner( ppx, const_cast< Y* >( py ) );
  }
}
```
里面又调用了enable_shared_from_this 的 _internal_accept_owner ：  
```cpp
template<class X, class Y> void _internal_accept_owner( shared_ptr<X> const * ppx, Y * py ) const
{
    if( weak_this_.expired() )
    {
        weak_this_ = shared_ptr<T>( *ppx, py );
    }
}
```
在这里对weak_this_进行拷贝赋值，使得weak_ptr_作为类对象shared_ptr的一个观察者。  
从上面说明来看，需要注意的是：shared_from_this()仅仅在weak_this_有效后调用，而weak_this_有效是要在，用裸指针初始化shared_ptr，即调用shared_ptr<T>的构造函数之后才能使用，原因再重复一遍：weak_this_并不是在enable_shared_from_this的构造函数中被赋值，而是在shared_ptr的构造函数中被赋值。  
所以典型的错误使用：  
```cpp
class D:public boost::enable_shared_from_this<D>
{
public:
    D()
    {
        boost::shared_ptr<D> p=shared_from_this(); //在此时，weak_this_还未指向任何对象
    }
};

OR

class D:public boost::enable_shared_from_this<D>
{
public:
    void func()
    {
        boost::shared_ptr<D> p=shared_from_this();
    }
};
void main()
{
    D d; //对象d还没有被shared_ptr接管
    d.func();
}
```
总结为：不要试图对一个没有被shared_ptr接管的对象使用shared_from_this()。  
说到底，就是enable_shared_from_this类里面有一个weak_this_保存了裸指针的信息。  

#### 右值引用 move 完美转发
- 左值 右值 
- 右值引用 移动构造 移动语义
- std::move 异常处理
- RVO
- 引用折叠 完美转发 

> 左值 和 右值  
一般判断或定义：  
左值：等号左边的变量，可以取地址的变量  
右值：将亡值 + 纯右值 ，纯右值比较容易：临时变量，运算表达式1+3，字面值true   
将亡值则是和C++11右值引用相关的：比如move的返回值，T&&函数返回值

> 右值引用  
遇到了一个问题：涉及到内部有指针的类，返回临时对象的情况
```cpp
HasPtrMem GetTemp() {
    return HasPtrMem();
}

int main() {
    HasPtrMem a = GetTemp();
}
```
> 在这种变态的场景里面，在调用GetTemp函数的时候，会调用一次构造函数，在返回的时候会调用拷贝构造函数，在初始化a的时候，又会调用一次拷贝构造函数，一共又会调用3次析构函数，如果我们的资源比较大，这一次次的拷贝和析构带来性能的损失是比较严重的，有没有方法减少内存的拷贝或析构呢？

> 移动语义，所有权转移：这里出现很多临时对象，可以借鉴浅拷贝的思想，仅仅拷贝指针来提高性能。但仅仅拷贝指针又是不够的，因为原来的临时对象在离开作用域的之后就会被析构掉，从而新的对象中的指针变成了空悬指针，一旦解引用，就会发生错误，所以需要在浅拷贝的基础上，将原来的指针赋值为nullptr，这样的话，就将内存的所有权彻底交给了新的对象，而不是像浅拷贝一样简单粗暴的共享。  

> 以上这种做法称为移动构造，移动构造减少一次无意义的内存拷贝，它的参数应该是什么呢？他传入的是一个右值，而右值除了能被const T& (这是一种万能引用)接收外，只能被右值引用所接收了，那么为什么参数不用const T&呢？ 一个是因为 const T& 为参数的构造函数已经存在了，是拷贝构造函数啊，第二个原因是，在移动构造函数中，我们是需要对传入的右值进行指针操作的，是不满足const语义的。所以这里函数的参数是T&&，这是右值引用。   

> 现在知道了，为什么需要移动构造 为什么会有右值引用。   
右值引用有一些特点需要说明。比如我们常见的左值引用，它是一个具名变量的别名，那么根据刚刚的陈述，右值引用就是匿名变量的别名，有了这个别名，该右值就重获新生，它的生命期会被延长到右值引用变量的生命期一样长

> std::move  
std::move 的作用是 将左值转换成 右值，从而可以调用移动构造函数，所以可以这么理解，move一个左值变成右值是为了去调用移动构造来提高性能。这是右值存在的最大的价值之一，还有就是完美转发。  
那么std::move有什么可能的坏处吗？如果我将一个左值转化成了右值之后，还在使用它，那就是搬起石头砸自己的脚了，所以move操作默认是程序员知道在转换后，左值是失效的。 那么如果转化成右值后，我们根本没有定义移动构造函数呢？ 这也没有关系，刚刚说了拷贝构造函数的参数是 const T&，这是一种万能的引用方式，它可以接收右值，所以转换后如果没有定义移动构造函数，就会有一个兜底的方案就是去调用拷贝构造函数。所以在编写高性能的swap函数时，我们可以采取move的优化方案。
```cpp
template <class T>
void swap(T& a, T&b) {
    T&& tmp(std::move(a));
    a = std::move(b);
    b = std::move(tmp);
}
```
这个是没有任何副作用的。

还有一个经常让人困惑的地方：
```cpp
class HugeMem {...}; //这是一个内部会去堆中分配大量内存的类

class Moveable {
    public:
        Moveable():h(1024){}
        ~Moveable() {}
        Moveable(Moveable &&m): h(move(m.h)), m.ptr = nullptr
        {
        }
        HugeMem h;
}
```
> 一个类里面包含了一个大对象，在这个类的移动构造函数中，我也想调用大对象这个类的移动构造函数，这样就需要将m.h转化为右值，这个其实一开始是有一点搞脑子的，为什么这里不能去掉move，直接，h(m.h)呢，m这个东西是右值吗？不是！这点最费解了，从传入的参数来看，它接受右值引用，但它本身却是一个左值，它可以取地址，只是这个左值即将要灰飞烟灭，我们可以安全的将其转换成右值，所以这一点最关键，接受右值引用的是一个左值，需要进行move转换。如果不进行转换，h(m.h)会调用拷贝构造函数，将不会有真正的移动语义了。

> 在使用move的时候，我们发现其最终想去调用的是移动构造函数，而移动构造函数内部是由指针之间的操作来实现的，如果移动构造函数内部抛出异常，那么指针可能就会发生错误，可能会导致空悬指针的出现，所以涉及到move操作，最好使用不抛出异常的移动构造，有一个 move_if_noexcept函数来保障move的安全性，如果类定义了不可抛异常的移动构造函数，move_if_noexcept就会去调用移动构造函数，如果没有noexcept，那就会为了安全起见去调用拷贝构造函数。  

> RVO  
对于RVO是编译器的一些优化策略，它可能做到比移动构造更加优秀的性能，但是并不是对任何情况都有效，所以这个知识点目前只需要知道就可以了，移动语义还是一种稳健的优化策略。  

> 完美转发：  
定义是相对陌生的，因为涉及到函数模板：在函数模板中，完全按照模板的参数的类型，将参数传递给函数模板中调用的另外一个函数，其本质就是使包装函数能够正确高效的执行。如果传入的是左值，那么调用的时候也是左值传入，如果传入的时候是右值，那么在调用的时候也是右值传入。 不会产生额外的开销，就好像转发者不存在一样。  

> 我们看到如果想要高效的转发，就需要避免拷贝，如果一个函数的参数是值传递，那么会产生拷贝的费用，虽然可以做到正确的转发，但是称不上完美无开销。所以我们需要将传值转换为传引用。   
那么应该是什么引用呢？这种引用要可以接受左值引用和右值引用。 我们知道用const T& 常量左值引用是万能的引用，但是这种引用是const类型的，接收能力是无敌了，但是转发函数也需要是const类型的才行，如果不是，那么就不能转发。所以会有常量转发版本和非常量转发版本，导致代码的冗余。  而且如果是非常量左值版本，转发函数如果是右值引用，就会发生问题：右值引用无法接收左值。  

> 引用折叠：  
为了解决以上的问题，C++11引入一条所谓引用折叠的规则，将复杂的未知表达式折叠为已知简单式，其规则为：一旦定义中出现了左值引用，引用折叠总是优先将其折叠成左值引用。
```cpp
template <class T> 
void IamForwarding(T && t) {
    IrunCodeActually(static_cast<T &&>(t));
}
```
如果传入左值引用，
```cpp
template <class T> 
void IamForwarding(T && &t) {
    IrunCodeActually(static_cast<T && &>(t));
}
```
根据引用折叠
```cpp
template <class T> 
void IamForwarding(T & t) {
    IrunCodeActually(static_cast<T &>(t));
}
```
static_cast没有起到作用就可以正确转发。 
如果传入右值引用，
```cpp
template <class T> 
void IamForwarding(T && &&t) {
    IrunCodeActually(static_cast<T && &&>(t));
}
```
根据引用折叠：
```cpp
template <class T> 
void IamForwarding(T && t) {
    IrunCodeActually(static_cast<T &&>(t));
}
```
这里t是一个左值，所以需要将其转换成右值，这个时候static_cast起到作用。  
这里其实就是move操作，但是这里为了区分move的作用，给这种操作又命名为了forward
```cpp
template <class T> 
void IamForwarding(T && t) {
    IrunCodeActually(forward(t));
}
```
这种转发的操作可以完成完美转发，没有对const进行限制，也没有对左值右值有限制。

> 思路总结：  
多余的拷贝构造 =》移动语义 =》 移动构造 =》 临时对象 =》 右值引用 =》 move =》将接收右值引用的左值转换为右值 从而可以调用移动构造函数

> 包裹函数 =》 无损失透传参数 =》 引用传递 =》 否定const T& 万能引用 =》 引入引用折叠 =》 右值引用 =》 左值 右值分别分析 统一处理 static_cast<T&&> =》形成forward 实质就是static_cast<T&&> 
