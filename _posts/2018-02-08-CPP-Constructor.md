---
layout:     post
title:      Cpp Defalult Constructor
subtitle:    "\"C++对象模型之默认构造函数\""
date:       2018-02-08
author:     Wangsigui
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - C++
---


# C++ 默认构造函数
　　对于C++的默认构造函数，以前一直有这样两个误解。其一，当一个类没有显示定义的默认构造函数时，编译器会自动合成一个默认构造函数；其二，编译器合成的默认构造函数会自动初始化类中的成员变量。这两个理解都是错的。
	　　那么到底哪些情况下编译器会自动为类合成默认构造函数呢？
>default constructors……在**需要**的时候被编译器产生出来
>　　　　　　　　　　　　　　　　　　　　——《深入探索Ｃ++对象模型》

我们可以先看下面的代码
```c++
　　class Foo{
	public:
		int val;
		Foo *pnext;　
　　};
　　void foo_bar(){
	　　Foo bar;    //声明了一个对象，
				　 //程序希望能够将bar的成员变量初始化为0
	　　if(bar.val || bar.pnext ){
		　　//do something
	　　}
	　　//...do something
　　}
```
 　　上面的程序正确的语意应该是希望能够在对象被声明的同时完成初始化的工作。那么在这种情况编译器会不会为它合成一个这样的default constructor呢？一种比较权威的答案如下
 >对于 class X，如果没有任何user declared constructor，那么会有一个 default constructor 被隐式地声明出来······一个被隐式声明出来的 default constructor 将是一个trivial(浅薄无能的，没啥用的)constructor······

　　具体哪些情况下会合成这种 trivial constructor 不在本文讨论范围之内，读者只需简单记住这种构造函数不会做任何事情，即不会为你初始化成员变量（*但是g++编译器好像做过优化，在没有显示声明任何构造函数的情况下，声明一个对象实例也会初始化它的成员变量。这一点我没有仔细验证过，所以我也不确定，如果有知道的读者还望告知*）。我们接下来会介绍哪些情况下编译器会合成non trivial constructor。

## **1.带有Default Constructor的 Member Class Object**
``` c++
class Foo{
public:
	Foo(){}
	Foo(int){}
	
};

class Bar{
public:
	Foo foo;              //含有一个Foo对象
	char* str;
};
```

　　由于 *Bar::foo* 必须要被初始化，这是编译器的责任，所以合成的constructor可能会像下面这样

```c++
inline
Bar::Bar(){
	foo.Foo::Foo(); //为方便起见省略了this指针
}
```
　　如果程序员显示地提供了一个可以初始化 *Bar:: str* 的default constructor呢？像下面这样

```
//程序员提供的constructor
Bar::Bar(){
	str = 0;
}
```
　　这个时候程序需求被满足了，但是编译器的需求还未被满足，但是由于已经显示地定义了一个default constructor 了，所以编译器无法再合成第二个了，编译器会采取什么样的行动呢？

>如果 calss *A* 内含一个或一个以上的member class objects，那么class *A* 的每一个constructor 必须调用每一个 member classes 的 default constructor。

　　编译器会扩张已有的constructors，在其中安插一些代码，使得user code被执行之前，先调用必要的 default constructors。延续上一个例子，扩张的constructor可能会像下面这样
```
//扩张的 default constructor
//c++伪码
Bar::Bar(){
    foo.Foo::Foo();   //附加的compiler code
    str = 0;          //user code
}
```
如果有多个class member objects 都要求 constructor 初始化操作，C++语言要求以 "member objects 在class 中的声明顺序" 来调用各个 constructors（注意：**不一定是 default constructors**）。看看下面这个例子：
```
class Dopey
{
public:
    Dopey(){}
    
};

class Sneezy
{
public:
    Sneezy(){}
    Sneezy(int){}
    
};

class Bashful
{
public:
    Bashful(){}
    
};

class Snow_White
{
private:
    int mumble;
public:
    Dopey dopey;
    Sneezy sneezy;
    Bashful bashful;
    //...
};

//程序员提供的default constructor
Snow_White::Snow_White():sneezy(1024){
    mumble = 2048;
}
```
这种情况下，它将被扩张为
```
Snow_White::Snow_White():sneezy(1024)
{
    //插入member class objects 
    //按照声明顺序调用其constructors
    dopey.Dopey::Dopey();
    sneezy.Sneezy::Sneezy(1024);     //调用的并非 default constructor
    bashful.Bashful::Bashful();
    
    //user code
    mumble = 2048;
}
```


## **2."带有Default Constructor" 的 Base Class** \
### 一、子类中不含有任何constructor
　　类似的道理，如果一个没有任何 constructor 的 class 派生自一个"带有default constructor" 的 base class，那么这个 derived class 的 default constructor 会被视为 nontrivial，并因此会被编译器合成出来。
1.如果子类派生自多个父类
```
class A{
public:
    A(){}

};

class B{
public:
    B(){}

};

class C: public A, public B{
};
```
同样的，C++也会以继承中基类的声明顺序依次调用它们的default constructor。合成的default constructor可能如下所示
```
C::C(){
    //按照声明顺序调用
    A::A();
    B::B();
}
```

2.如果子类派生自一个继承链
```
class A{
public:
    A(){}

};

//B继承A
class B: public A{
public:
    B(){}
//...
};

//C继承B
class C: public B{
public:
    C(){}
//...
};

//D继承C
class D: public C{
};
```
这种情况其实很简单，因为每一个类都会先向上调用它父类的constructor，所以合成的default constructor可能是这样
```
D:D(){
    A::A();
    B::B();
    C::C();
}
```
3.如果子类也含有member class objects
这种情况下，member class objects 也会按照声明顺序被调用，不过它们会在基类的constructors调用之后才会被调用。看一个稍微复杂一点的例子
```
//以下代码并不完全，一些输出部分被省略了
class Member{
public:
    Member(){}

};

class Other{
public:
    Other(){}

};

class Base{
public:
    Base(){}
//...
};

class Derived: public Base{
public:
    Derived(){}
//...
};

class Mix: public Other, public Derived{
public:
    Member member;
};
```
这个例子中，class *Mix* 继承了 class *Other* 和 class *Derived*, 其中 *Mix* 和 *Devrived*， *Base* 构成了一条派生链，并且 *Mix* 含有 一个对象成员 class *Member*，乍看上去好像构造函数的调用会很复杂，其实我们综合3点来看，也就很简单了。首先按照基类的声明顺序来调用constructors；如果和基类构成了一条派生链，那么就从上往下顺序调用；最后调用member class objects 的 constructors。所以合成的default constructor 可能如下：
```
Mix::Mix{
    //Other第一个被声明
    Other::Other();
    //派生链
    Base::Base();
    Derived::Derived();
    //member class objects
    member.Member::Member();
}
```
结果如下图

![image](C:\Users\Administrator\Pictures\order.PNG)

### 二、子类含有显式的construtor（**没有强调是否为default constructor**）
如果子类中已经存在显示定义的constructors(不管其中有没有default constructor)，编译器会扩充每一个constructor来完成它的责任，**但是编译器不会合成新的default constructor，这也就是说，在这种情况下如果你需要default constructor，程序员得自己明确地声明定义出来**。我们延续上一个例子
```
class Member{
public:
    Member(){
		cout << "Member" << endl; 
	}
	
	Member(int i){
		cout << "Member:" << i << endl;
	}

};

class Other{
public:
    Other(){
		cout << "Other" << endl;
	}

};

class Base{
public:
    Base(){
		cout << "Base" << endl;
	}
};

class Derived: public Base{
public:
    Derived(){
		cout << "Derived" << endl;
	}

};

class Mix: public Other, public Derived{
private:
	int mumble;
public:
    Member member;
    //程序员提供的default constructor 
    Mix():member(1){
    	mumble = 2;
	}
};

```
扩充后的constructor应该像下面这样
```
Mix::Mix():member(1){
    Other::Other();
    Base::Base();
    Derived::Deriver();
    member.Member::Member(1);
    
    mumble = 2;
}
```
具体原因就不再解释了，如果前面都能理解的话，这段伪码应该很容易理解。结果如下图：

![这里写图片描述](http://img.blog.csdn.net/20160821112456240)

##**3."带有一个virtual function"的class**

这一大类里面包含两种情况。
1.class 声明（或继承）一个virtual functions

2.class 派生自一个继承串链，其中有一个或更多的virtual base classes

不管哪一种情况，由于缺乏user声明的 constructors， 编译器会详细记录合成一个default constructor 的必要信息。以下面这个程序片段为例。

```
class Widget{
public:
    virtual void flip() = 0;
};

//注意，原本在书中这里的参数列表是 void flip(const Widget& widget);
//但是这样g++编译无法通过，原因是因为一个常量对象无法调用一个非常量的成员函数
//简单的解决办法就是将这里的const去掉
//关于const的用法，以后有机会我也会用一篇博文记录下来
void flip(Widget& widget){
    widget.flip();                //动态绑定
}

class Bell: public Widget{
public:
    void flip(){
        cout << "Bell" << endl;
    }
};


class Whistle: public Widget{
public:
    void flip(){
        cout << "Whistle" << endl;
    }
};
void foo(){
    Bell b;
    Whistle w;
    flip(b);                      //将会输出"Bell"
    flip(w);                      //将会输出"Whistle"
}
```

在编译期间下面两个行为会被扩张

1.一个virtual function table（vtbl,虚函数表）会被编译器产生出来，内放class 的 virtual functions 的地址

2.在每一个 class object 中，一个额外的 pointer member(vptr,虚指针)会被编译器合成出来，用来指向vtbl。

此外，*widget.flip()* 的虚拟调用操作会被重新改写，以使用widget的 vptr  和 vtbl 中的 flip() 条目；

```
(*widget.vptr[1])(&widget)                     //widget.flip()

```
vptr 和 固定索引 1 就是编译器完成的工作，*&widget* 代表要交个"被调用的某个*flip()*" 的 **this** 指针。为什么索引是1呢？
虚表中内容的简单示意如下

|index| contents|
|-----|---------|
|0|type_info for class|
|1|virtual func 1|
|2|virtual func 2|
|...|...|

可以看到，虚函数表的开头存放的是类型信息，之后才开始存放虚函数的地址，所以虚函数的在虚函数表中的索引是从1开始的。虚函数表和虚指针的初始化是这类情况下编译器完成的主要工作。

同理，如果类中已有constructors，那么编译器会扩充这些constructors来完成上述对 vbtl 和 vptr 的工作。

## **4."带有一个Virtual Base Class" 的 Class** (虚继承)

Virtual base class 的实现法在不同的编译器之间有极大的差异。然而，每一中是实现法都有一个共同点

>必须使 virtual base class 在其每一个 derived class object 中的位置，能够于执行期准备妥当。

书中的做法是在 derived class object 中安插一个指针，看一下书中的例子

```c++
class X { public: int i; };
class A : public virtual X { public: int j;   };
class B : public virtual X { public: double d };
class C : public A, public B { public: int k; };
void foo(const A* pa){
    pa->i = 1024;                   //实际A的类型是不确定的
}

int main(){
    foo(new A);                    //可以是A的一个指针
    foo(new C);                    //也可以是A的一个子类的指针
}

```

如上面这个例子所示，由于在编译期间函数 *foo()* 的参数列表中的类型实际是不确定的，所以编译器无法固定住 " 经由pa而存取的*X::i* " 的实际偏移位置，所以编译器必须改变 “执行存取操作”的那些代码，让 *X::i* 可以延迟到执行期才决定下来。一种可能的策略如下：

```
void foo(const A* pa){
    pa->_vbcX->i = 1024;
}
```

其中_vbcX是编译器合成的指针，指向 virtual base class X。

指针_vbcX(各个编译器实现各不相同，不一定是这样的一个指针，但是编译器一定会合成些什么东西) 是在 class object 构造期间被完成的。如果 class 中有定义好的constructor，编译器会扩充每一个 constructor，安插那些 "允许每一个virtual base class 执行期存取操作" 的代码。 如果 class 没有定义任何constructor ， 编译器会为它合成一个 default constructor，所做的事情相同。

## **5.总结**

在上述4种情况下，编译器会为没有声明任何constructor 的class 合成 nontrivial default constructor。这种 default constructor 只会满足编译器的需求，除此之外的任何工作都不应该被这样的default constructor完成。如果程序需要另外的初始化操作，例如把一个指针赋值为0，这个工作应该由程序员自己完成。要消除开篇介绍的两个误解：

1.任何class 没有如果没有定义 default constructor，就会被合成出来

2.编译器合成出来的default constructor 会显式设定 "class 内每一个data member 的默认值" (初始化操作)

没有一个是真的。

## **6.参考文献**

[《深入探索C++对象模型》](https://book.douban.com/subject/1091086/)
