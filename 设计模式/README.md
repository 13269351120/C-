## 设计模式  
### 观察者模式 

### 装饰者模式   
**基本概念**：顾名思义装饰者模式就是装饰已有的类用的，向现有的对象添加新的功能，同时又不改变其结构，就好像装饰品一样挂载在上面一样。  
**产生原因**：为了增加类的功能，而且发现增加的功能是另一个维度上的，可以优先考虑装饰模式而非继承，因为继承会引入静态特征，随着扩展的功能的增多，子类数量膨胀。   
**实现形式**：为了产生抽象，原来是有一个基类Shape，提供一个纯虚函数draw()。同时它有若干个派生类，比如Circle，Rectangle类，在这些子类中实现了draw()，可以通过多态去通过`Shape *shape = new Circle(); shape->draw();`调用子类的方法。这个时候需要在形状上增加一个新的功能。一种方法是继承，而另一种方法就是装饰者模式提供的方法。用一个Decorator类去继承Shape，将构造函数传入Shape的对象`Decorator(Shape* shape) {_shape = shape;}`，以便使用多态，可以完成和基类一样的`draw() {shape->draw();}`接口。具体使用的时候，要写一个concret_decorator类去继承Decorator，在这个concret_decorate类中去添加新的功能，比如可以在原来的draw()上增加一些东西`concret_decorator::draw() { shape->draw(); printf("this is new function!");}`。 
具体使用的时候只要：
```cpp
Shape *circle = new Circle();
concret_decorate* d = new Decorator(circle);
d->draw();
```
还有一个很好的例子是奶茶，现在很多奶茶店都有各种各样的奶茶若干款，绿色奶茶，黄色奶茶，白色奶茶等都是继承自奶茶，但是用户同样可以添加比如珍珠，芋圆，青稞等辅料，这些新的功能，是所有奶茶所通用的，不需要通过继承自绿色奶茶等来实现，这个时候只要出现一个装饰类，增加功能后，进行拼装即可。
```cpp
MilkTea *green_tea = new GreenTea();
Decorator *d = new BubbleMilkTea(green_tea);
d->drink();
```


### 单例模式

### 工厂模式  

### 代理模式  
