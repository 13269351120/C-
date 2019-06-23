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

