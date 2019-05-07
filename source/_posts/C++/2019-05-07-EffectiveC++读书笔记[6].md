---
title: Effective C++读书笔记<六>
categories: C++
tags: [C++,读书笔记,Effective C++]
date: 2019-05-07
---

## 《Effective C++》读书笔记<六>

#### Item 32：明确你的public继承塑模出is-a关系

如果令class D以public方式继承class B，相当于告诉编译器，每个类型为D的对象同时也是一个类型为B的对象。你的意思是B比D表现出更为一般的概念，而D比B表现出更特殊化的概念。<b>凡是B对象可以派上用场的地方，D对象一样可以排上用场</b>



#### Item 33：避免遮掩继承而来的名称

如果你在使用public继承，但是却不继承base的函数，便是对<b>is-a</b>关系的违反。

倘若你在子类D中定义了从B中继承而来的同名函数，那么从<b>名称查找</b>的观念来看，B中的函数便不再被继承。你在D中定义的函数遮掩了继承的函数。

```C++
class B{
public:  
    virtual void f1() = 0;
    virtual void f1(int);
    virtual void f2();
    void f3();
    void f3(int);
};

class D:public B{
public:
    virtual void f1();//override f1(), but hide f1(int). 
    void f3();//hide f3().
    void f4();
};
```

你可以使用using声明解决问题

```c++
class D:public B{
public:
    using B::f1;
    using B::f3;
    virtual void f1(); 
    void f3();
    void f4();
};

D d;
d.f1(); // D::f1.
d.f1(1); // B::f1.
d.f2(); // B::f2.
d.f3(); // D::f3.
d.f3(1); // B::f3.
```

值得注意的是，<b>using的意思与字面有区别，不是说接下来的调用都是使用这个using指明的范围，而是说让using指明的名字在当前作用域可见。</b>

有时候不想继承基类所有的函数，而是说只想继承一部分。当然这对于public继承是不可能的，但是对于private继承有时候可能会有这种需求。这个时候使用using会暴露父类所有该名函数，我们需要不同的技术，叫做<b>转交函数</b>。

```c++
class Base{
public:
    virtual void f1() = 0;
    virtual void f1(int);
};
class Derived: private Base{
public:
    virtual void f1(){//转交函数，遮掩父类所有f1函数
        Base::f1();//暗自成为inline.
    }
};

D d;
d.f1();// 调用D::f1()->B::f1().
d.f1(1);// 错误.
```



#### Item 34：区分接口继承和实现继承

身为class设计者，有时你会希望derived classes只继承成员函数的接口；有时你又会希望derived class同时继承函数的接口和实现，但又能override它们所继承的实现；有时你希望derived classes同时继承函数的接口和实现，并且不允许覆写任何东西。

- 成员函数总是会被继承。public继承意味着is-a的关系，父类所有的函数在子类上都能施行。
- pure函数是为了让derived classes只继承函数接口。(我们可以为纯虚函数提供定义，但是意义不大)
- 声明impure virtual函数是为了让derived classes继承该函数的接口和缺省实现。它表示每个子类都必须支持这样一个函数，如果不想写，可使用缺省版本。
- 声明non-virtual函数目的是为了令derived classes继承一份接口和一份将强制性实现。



#### Item 35：考虑virtual函数以外的其他选择

假设我们现在写一款游戏，不同角色攻击时会释放不同的技能，可能我们会想到使用virtual函数，让特殊角色都继承于GameCharacter，GameCharacter提供了一份缺省实现，特殊角色可针对自己的情况改写攻击函数。

当然这也是我们最常规的办法，除此之外，也存在着许多其他的方式供我们选择。

- <b>Non-Virtual Interface的手法实现Template Method模式</b>

```c++
class GameCharacter{
public:
    void attack(){
        beforeAttack();
        doAttack();
        afterAttack();
    }
private:
    virtual void doAttack(){
        ...
    }
};
```

这就是所谓的NVI手法，attack是作为doAttack的外覆器。

NVI的优势在于我们可以在进行实际操作前后做些处理，正如我们beforeAttack()，afterAttack()写的那样。

- <b>基于Function Pointers实现的Strategy模式</b>

我们可以让不同角色保存一个函数指针，该函数指针执行特殊攻击操作。但是这样存在一个问题，就是函数指针指向的函数可能需要访问对象的私有元素，这样可能就需要采用friend关键字来为函数特殊访问权限。

- <b>基于std::function实现的Strategy模式</b>

与上面的Function Pointers相似，只不过<i>std::function</i>具有更好的封装，可以保存成员函数。

- <b>传统的Strategy模式</b>

让不同操作封装在不同类里，并形成继承链，在不同角色中保存有这些操作的对象，并在角色的攻击函数中调用操作对象的接口。

这里只是介绍传统的Strategy模式，在这个例子里面意义不大。



#### Item 36：绝不重新定义继承而来的non-virtual函数

因为这个时候对于一个D对象，通过B指针访问该函数和通过D指针访问该函数的表现不再相同。也就是说用到D指针的地方不能用B指针替代，也就是违背了public继承is-a的关系。

很显然，父类的non-virtual函数体现了某种<b>不变性</b>，一旦子类改变定义，便是对is-a关系的违反。如果希望子类对某些函数表现出特异性，这时就需要virtual关键字，virtual函数通过虚函数表的机制，向子类提供了一种保证：你可以大可以重新定义我，我将仍然维护is-a关系。因为D对象不论是通过D指针还是通过B指针访问，表现都是相同的。



#### Item 37：绝不重新定义继承而来的缺省参数值

virtual函数是动态绑定的，而缺省参数值确实静态绑定的。

```C++
class A{
public:
    virtual void f(int i = 0);
}
class B:public A{
public:
	void f(int i = 1);
}
A* a = new B();
a->f();//我们可能期待参数为1，但实际上是0
```

在上面的例子里面，我们重新定义了继承而来的缺省参数值，但通过指针或引用来访问时，由于缺省参数值是静态绑定的，a的静态类型是A，所以我们绑定了缺省参数值0，在运行时才调用到B::f()，这就很容易造成误解，所以最好的做法是不要重新定义继承而来的缺省参数值。

之所以让绑定缺省参数在编译器进行，是为了降低运行期的开销。



#### Item 38：通过复合塑模出has-a或is-implemented-in-terms-of

复合有两个意义。复合意味<i>has-a</i>或<i>is-implemented-in-terms-of</i>。程序中的对象其实相当于你所塑造的世界中的某些事物，例如人、汽车、视频画面等等。这样的对象属于<b>应用域部分</b>。其他对象则纯粹是实现细节上的人工制品，像是缓冲区、互斥器、查找树等等。这些对象属于<b>实现域部分</b>。复合发生于应用域内对象之间，表现出<i>has-a</i>关系，当发生于实现域内则是表现<i>is-implemented-in-terms-of</i>关系。



#### Item 39：明智而谨慎地使用private继承

```c++
class Person{};
class Student:private Person{};
void eat(const Person&);
void study(const Student&);

Person p;
Student s;
eat(p);//正确
eat(s);//错误，没有is-a关系
```

private继承意味着implemented-in-terms-of。如果让D以private形式继承B，用意是采用B中已经备妥的某些特性，不是因为B和D有任何观念上的关系。

private继承与复合有点相似，我们在两者间的取舍可总结为：尽可能使用复合，必要时才使用private继承。private继承主要用于<B>"一个意欲成为derived class者像访问一个意欲成为base class者的protected成分，或为了重新定义一或多个virtual函数"。</B>

```c++
/*private继承*/
class Timer{
public:
    explicit Timer(int tickFrequency);
    virtual void onTick() const;
};

class Widget: private Timer{
private:
    virtual void onTick() const;
}//Widget可以调用onTick().

/*普通复合*/
class Widget{
private:
    class WidgetTimer:public Timer{
    public:
        virtual void onTick() const;
    };//定义于Widget内部,让其他类无法继承WidgetTimer。
    WidgetTimer timer;
};
```

另外在一种激进情况涉及空间最优化，可能促使你选择"private继承"而不是"继承加复合"。

```c++
class Empty{};
class HoldAnInt{
private:
    int x;
    Empty e;
};//sizeof(HoldAnInt) > sizeof(int)

//独立对象大小不为零。
//sizeoff(Empty) == 1.编译器默默安插一个char到空对象里，虽然char不算大，但在复合关系里，可能存在对齐问题，导致额外的padding开销。

```

我们提到了独立对象不为零，如果该对象是作为另一对象的附带情况就会不一样了。

```c++
class HoldAnInt:private Empty{
private:
    int x;
};
//这个时候sizeof(HoldAnInt) == sizeof(int)
```

这就是所谓的<b>EBO优化(empty base optimization)</b>



#### Item 40：明智而谨慎的选择多继承

- 多重继承比单一继承复杂。可能导致歧义性，以及对virtual继承的需要。
- virtual继承会增加大小、速度、初始化复杂度等等成本。如果virtual base classes不带任何数据，将是最有使用价值的情况。
- 多重继承有正当用途。其中一个情节涉及"public继承某个Interface class"和“private继承某个协助实现的class“两相组合



