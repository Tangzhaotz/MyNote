## C++面试高频

### 1、智能指针

#### 1、c++使用智能指针的原因

c++中没有内存回收机制，不像java一样有垃圾回收机制，所以在堆区开辟的内存需要程序员自己释放，，虽然程序员自己管理堆区的内存能够提高程序的效率，但是整体来说，堆内存的管理很麻烦，使用普通的指针容易造成内存泄漏（忘记释放）、二次释放、程序发生异常等内存泄漏的问题，所以提出使用智能指针来管理堆区的内存。

```c++
/*
@author:tangzhao
date:2021.8.13
content:samrt_point
*/

#include<iostream>
using namespace std;

int main()
{
    int *p1 = new int(1);  //在堆区开辟一块新的区域，存放数值1，并使得指针p1指向这块区域
    int *p2 =p1;
    int *p3 =p1;  //另外两根指针也是指向这块区域
    cout << "*p1=" << *p1 << endl;  //1,解引用，取出指针所指向的内存区域的值
    cout << "*p2=" << *p2 << endl;  //1
    cout << "*p3=" << *p3 << endl;  //1

    delete p1;  //释放指针p1，它所指向的内存也会被释放
    cout << "*p2=" << *p2 << endl;  
    cout << "*p3=" << *p3 << endl; 

    return 0;
}


```

![image-20210813051905965](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813051905965.png)

![image-20210813141909596](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813141909596.png)

被释放后变为下面的状态：

![image-20210813142134338](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813142134338.png)

如上图所示被释放后，p2和p3就是处于一个悬空的状态，这样会造成内存泄漏，严重的时候会造成异常，所以我们需要设计智能指针。

产生上述的错误的原因在于：p1指针的“愚蠢和无知”，因为它在释放的时候，不知道还有没有其他指针指向自己指向的这块区域，所以就把这块内存给释放了。

##### 智能指针的使用情况：

1. 程序不知道自己需要使用多少对象
2. 程序不知道所需对象的准确类型
3. 程序需要在多个对象间共享数据



#### 2、智能指针的几种情况及原理实现

##### 2.1、shared_ptr的使用

c++11之后，智能指针包含在头文件<memory>中，shared_ptr是一种可以有多个指针指向相同的对象，shared_ptr使用引用计数器，每一个shared_ptr的拷贝都指向相同的内存，每使用它一次，内部的引用计数器就加1，每析构一次，内部的引用计数器就减1，当引用计数器的数值为0的时候，就能够自动删除所指向的堆内存，shared_ptr内部的引用计数是线程安全的，但是在对象的读取的时候需要加锁。

**特点：**

（1）、初始化：智能指针是一个模板类，可以指定类型，类似于vector等标准的容器，也可以使用make_shared初始化，不能将指针直接赋值给一个智能指针，因为智能指针实际还是一个类，普通的指针是指针，例如：

```c++
shared_ptr<int>p1=new int(4);  //错误，因为，左边是智能指针，是一个类，右边是指针
```

（2）、拷贝和赋值：拷贝使得引用计数加一，赋值使得原对象的引用计数减一

（3）、get函数获取原始指针

（4）、注意：不要使用一个原始指针初始化多个shared_ptr，否则会造成二次释放同一块内存

（5）、注意避免循环引用，shared_ptr的一个最大的陷阱是循环引用，循环，循环引用会导致堆内存无法正确释放，导致内存泄漏

```c++
/*
@author:tangzhao
date:2021.8.12
content:shared_ptr
*/

#include<iostream>
#include<memory>
using namespace std;

int main()
{
    int a =10;

    shared_ptr<int>ptra = make_shared<int>(a);  //创建一根智能指针指向原始的对象
    shared_ptr<int>ptra2(ptra);  //拷贝，会使得引用计数加一
    cout << ptra.use_count() << endl;  //2

    int b =20;
    int *pb = &a;
    //shared_ptr<int>pb = p;  //错误，因为左边是类，右边是普通的指针

    shared_ptr<int>ptrb = make_shared<int>(b);
    ptra2 = ptrb;  //赋值，会使得p1的引用计数减一，ptrb的引用计数加一
    pb = ptrb.get();  //获取原始指针

    cout << ptra.use_count() << endl;  //1
    cout << ptrb.use_count() << endl;  //2

    return 0;

}

结果：
    [Running] cd "/root/code/cplusplus/" && g++ shared.cpp -o shared && "/root/code/cplusplus/"shared
2
1
2
```

![image-20210813145432537](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813145432537.png)

注意：普通指针是不会计入引用计数的

当实现这一句代码的时候变为下面的状态：

![image-20210813151411688](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813151411688.png)

![image-20210813151330294](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813151330294.png)

##### 2.2、unique_ptr的使用

unique_ptr“唯一”拥有其所指对象，同一时刻只能有一个unique_ptr指向给定对象（通过禁止拷贝语义、只有移动语义来实现）。相比与原始指针unique_ptr用于其RAII的特性，使得在出现异常的情况下，动态资源能得到释放。unique_ptr指针本身的生命周期：从unique_ptr指针创建时开始，直到离开作用域。离开作用域时，若其指向对象，则将其所指对象销毁(默认使用delete操作符，用户可指定其他操作)。unique_ptr指针与其所指对象的关系：在智能指针生命周期内，可以改变智能指针所指对象，如创建智能指针时通过构造函数指定、通过reset方法重新指定、通过release方法释放所有权、通过移动语义转移所有权。

```c++
#include<iostream>
#include<memory>
using namespace std;

int main()
{
    {
    unique_ptr<int>uptr(new int(10));
   // unique_ptr<int>uptr2=uptr;  //错误，不能赋值
    //unique_ptr<int>uptr2(uptr);  //不能拷贝
    cout<<*uptr << endl;  //10

    unique_ptr<int>uptr2 = move(uptr);
    //cout << *uptr << endl;  出错
    cout<<*uptr2 << endl;  //10
    uptr2.release();  //释放所有权
    }
    //这里开始就不再是uptr的作用域了，内存会被释放掉
}
```

![image-20210813152244707](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813152244707.png)

转移权限之后

![image-20210813152339618](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813152339618.png)

##### 2.3、weak_ptr的使用

weak_ptr是为了配合shared_ptr而引入的一种智能指针，因为它不具有普通指针的行为，没有重载operator*和->,它的最大作用在于协助shared_ptr工作，像旁观者那样观测资源的使用情况，weak_ptr可以从一个shared_ptr或者另一个weak_ptr对象构造，获得资源的观测权。但weak_ptr没有共享资源，它的构造不会引起指针引用计数的增加。使用weak_ptr的成员函数use_count()可以观测资源的引用计数，另一个成员函数expired()的功能等价于use_count()==0,但更快，表示被观测的资源(也就是shared_ptr的管理的资源)已经不复存在。weak_ptr可以使用一个非常重要的成员函数lock()从被观测的shared_ptr获得一个可用的shared_ptr对象， 从而操作资源。但当expired()==true的时候，lock()函数将返回一个存储空指针的shared_ptr。

![image-20210813154040109](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813154040109.png)

##### 2.4、智能指针的循环引用

对于普通的指针，当两个对象之间相互指向的时候，如果我们释放一个指针，另外一个不会被释放掉，考虑下面的例子：假设每一位父亲有一个儿子，那么这个儿子也会有一个父亲，通过以下代码来实现原始指针的做法：

```c++
#include<iostream>
#include<memory>
using namespace std;


class father;
class son;
class father
{
private:
    son * myson;  //成员变量是一个对象，表示每位家长有一个孩子
public:
    void setson(son * s)  
    {
        this->myson =s;  //构造函数赋值
    }

    void dosomething()
    {
        cout << "father have a son"<<endl;
    }

    ~father()  //析构函数
    {
        delete myson;  //删除对象
    }
};

class son
{
    private:
    father * myfather;
    public:
    void setfather(father * f) 
    {
        this->myfather = f;
    }

    void dosomething()
    {
        cout << "son have a father" << endl;
    }
    ~son()
    {
        delete myfather;
    }

};


int main()
{
    {

    father *p = new father;  //创建一个父亲对象，并用指针p指向它
    son * s = new son;
    
    p->setson(s);
    s->setfather(p);

    delete s;  //只会将c删除
    }
    
    return 0;
}
```

![image-20210813161808635](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813161808635.png)

如上图所示，释放s所指的对象，那么p和myson指针依然会存在，会造成内存泄漏问题

**使用智能指针**

```c++
#include <iostream>
#include <memory>
using namespace std;

class son;
class father;

class father {
private:
    shared_ptr<son> sonPtr;  //成员设置为智能指针
public:
    void setson(std::shared_ptr<son> s) {
        this->sonPtr = s;
    }

    void doSomething() {
        if (this->sonPtr.use_count()) {

        }
    }

    ~father() {
    }
};

class son {
private:
    std::shared_ptr<father> fatherPtr;
public:
    void setfather(std::shared_ptr<father> f) {
        this->fatherPtr = f;
    }
    void doSomething() {
        if (this->fatherPtr.use_count()) {

        }
    }
    ~son() {
    }
};

int main() {
    std::weak_ptr<father> wpp;  //设置weak_ptr指针监视对象
    std::weak_ptr<son> wpc;  //设置weak_ptr指针监视对象
    {
        std::shared_ptr<father> p(new father);
        std::shared_ptr<son> c(new son);
        p->setson(c);
        c->setfather(p);
        wpp = p;
        wpc = c;
        std::cout << p.use_count() << std::endl; // 2
        std::cout << c.use_count() << std::endl; // 2
    }
    //作用域范围之外，p和c就被释放了
    std::cout << wpp.use_count() << std::endl;  // 1
    std::cout << wpc.use_count() << std::endl;  // 1
    return 0;
}
```

开始的时候创建两个指针和两个对象分别指向自己的对象，如下图所示：

![image-20210813163110622](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813163110622.png)

当他们分别调用下列函数的时候，会增加自己的引用计数器的值

![image-20210813163242660](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813163242660.png)

![image-20210813163250638](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813163250638.png)

当离开作用域之后，p和c被释放，但是引用计数分别都减为1，对象不能被释放

![image-20210813163518493](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813163518493.png)

上面的图中，因为会相互的循环调用，所以一个都释放不了，类似于操作系统中的死锁，这样还是发生内存的泄漏。

解决办法：

修改上述代码，按红色的圈修改：

![image-20210813164235003](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813164235003.png)

此时，对象之间的引用关系如下图所示：

![image-20210813164438283](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813164438283.png)

当离开作用域之后，p和c被释放，那么son对象的引用计数器就变为0，那么son就会被释放，从而成员指针也不会存在，由于p已经被释放，此时father对象的引用计数器为1，而随着son被释放，那么father的引用计数器也变为0，所以两个都被释放。代码如下：

```c++
#include <iostream>
#include <memory>
using namespace std;

class son;
class father;

class father {
private:
    //shared_ptr<son> sonPtr;  //成员设置为智能指针
    weak_ptr<son>sonPtr;  //设置为弱引用
public:
    void setson(std::shared_ptr<son> s) {
        this->sonPtr = s;
    }

    void doSomething() {
        if (this->sonPtr.lock()) {  //这里新建一个指针

        }
    }

    ~father() {
    }
};

class son {
private:
    std::shared_ptr<father> fatherPtr;
public:
    void setfather(std::shared_ptr<father> f) {
        this->fatherPtr = f;
    }
    void doSomething() {
        if (this->fatherPtr.use_count()) {

        }
    }
    ~son() {
    }
};

int main() {
    std::weak_ptr<father> wpp;  //设置weak_ptr指针监视对象
    std::weak_ptr<son> wpc;  //设置weak_ptr指针监视对象
    {
        std::shared_ptr<father> p(new father);
        std::shared_ptr<son> c(new son);
        p->setson(c);
        c->setfather(p);
        wpp = p;
        wpc = c;
        std::cout << p.use_count() << std::endl; // 2
        std::cout << c.use_count() << std::endl; // 1
    }
    //作用域范围之外，p和c就被释放了
    std::cout << wpp.use_count() << std::endl;  // 0，这里最后两者都被释放了，所以引用计数为0
    std::cout << wpc.use_count() << std::endl;  // 0
    return 0;
}
```

![image-20210813164953643](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210813164953643.png)

### 2、c++多态、虚函数以及多重继承

多态是c++面向对象的三大特性之一，主要是为了实现继承的时候，能够实现不同的对象调用实现不同的功能。c++中的多态主要分为两类：

1、静态多态：主要是函数重载、运算符重载，在编译阶段就能确定函数的地址

2、动态多态：派生类和虚函数实现，在运行阶段确定函数的地址

#### 1、静态多态

主要看函数重载的问题

```c++
/*
@author:tangzhao
date:2021.8.14
*/
#include<iostream>
using namespace std;

void func(int a,int b)
{
    cout << a +b << endl;
}

void func(double c,double d)
{
    cout << c + d <<endl;
}

int main()
{
    int a =10;
    int b =10;

    double c = 3.14;
    double d = 2.1;

    func(a,b);  //调用的是整数的计算
    func(c,d);  //调用的是小数的计算

    return 0;
}
```

上面就是函数的重载，在编译阶段，就能确定具体的func（）调用的是哪一个，比较简单。

#### 2、动态多态

举例子：人就是一个类，共同的特点就是都会说话，但是不同的国家说话的语言不一样，每个地方的人都有自己的方言或者母语，如下面的代码所示

```c++
/*
@author:tangzhao
date:2021.8.14
*/
#include<iostream>
using namespace std;

class people   //人类
{
private:
    string name;
public:
    void speak()  //人类的属性就是可以说话
    {
        cout << "人类会说话" <<endl;
    }

};

class Chinese:public people  //中国人，继承了人类的特点
{
private:
    string chinesename;
    
};

class America:public people  //美国人，也是继承了人类的特点
{
    private:
        string Amername;
};

int main()
{
    people p1;
    p1.speak();//人类会说话

    Chinese c1;
    c1.speak();  //人类会说话

    America a1;
    a1.speak();  //人类会说话

    return 0;
}
```

上述就是继承中的特点，虽然中国和美国人都会说话，但是中国人讲的是普通话，美国人讲的是英语，不一样，所以需要用多态来实现，当我们调用中国人的时候，显示他说的是普通话，美国人调用的时候，显示说英语。

##### 2.1、多态的实现

多态是靠虚函数实现的，C++中的虚函数的作用主要是实现了多态的机制。关于多态，简而言之就是用父类型别的指针指向其子类的实例，然后通过父类的指针调用实际子类的成员函数。这种技术可以让父类的指针有“多种形态”，这是一种泛型技术。所谓泛型技术，说白了就是试图使用不变的代码来实现可变的算法。比如：模板技术，RTTI技术，虚函数技术，要么是试图做到在编译时决议，要么试图做到运行时决议。

C++的虚函数(Virtual Function)是通过一张虚函数表(Virtual Table)来实现的。简称为V-Table。在这个表中，主要是一个类的虚函数的地址表，这张表解决了继承、覆盖(override)的问题，保证其能真实的反应实际的函数。这样，在有虚函数的类的实例中这张表被分配在了这个实例的内存中，所以当我们用父类的指针操作一个子类的时候，这张虚函数表就显得尤为重要了，他就像一个地图一样，指明了实际所应该调用的函数。
虚函数表中只存有一个虚函数的指针地址，不存放普通函数或是构造函数的指针地址。只要有虚函数，C++类都会存在这样的一张虚函数表，不管是普通虚函数  亦或 是 纯虚函数，亦或是 派生类中隐式声明的这些虚函数都会 生成这张虚函数表。

虚函数表的创建时间：在一个类构造的时候，创建这张虚函数表，而这个虚函数表是供整个类所共有的。虚函数表存储在对象最开始的位置。

虚函数表其实就是函数指针的地址。函数调用的时候，通过函数指针所指向的函数来调用函数。

```c++
/*
@author:tangzhao
date:2021.8.14
*/
#include<iostream>
using namespace std;

class people
{
private:
    string name;
public:
    virtual void speak()
    {
        cout << "人类会说话" <<endl;
    }

};

class Chinese:public people
{
private:
    string chinesename;
    public:
    void speak(string n)
    {
        cout << n << "说中文" << endl;
    }
    
};

class America:public people
{
    private:
        string Amername;
    public:
    void speak(string n)
    {
        cout << n << "说英语" << endl;
    }
};

int main()
{
    people p1;
    p1.speak();

    Chinese c1;
    c1.speak("小明");

    America a1;
    a1.speak("tom");

    return 0;
}
```

运行结果：

![image-20210814145629317](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814145629317.png)

##### 2.2、虚函数的几种类型

**1、无继承情况**

```c++
#include <iostream>  
  
using namespace std;  
  
class Base  
{  
public:  
    Base(){cout<<"Base construct"<<endl;}  
    virtual void f() {cout<<"Base::f()"<<endl;}  
    virtual void g() {cout<<"Base::g()"<<endl;}  
    virtual void h() {cout<<"Base::h()"<<endl;}  
    virtual ~Base(){}  
private:
    int a;
    string b;
    char c;
};  
  
int main()  
{  
    typedef void (*Fun)();  //定义一个函数指针类型变量类型 Fun  
    Base *b = new Base();  
    //虚函数表存储在对象最开始的位置  
    //将对象的首地址输出  
    cout<<"首地址："<<*(int*)(&b)<<endl;  
  
    Fun funf = (Fun)(*(int*)*(int*)b);  
    Fun fung = (Fun)(*((int*)*(int*)b+1));//地址内的值 即为函数指针的地址，将函数指针的地址存储在了虚函数表中了  
    Fun funh = (Fun)(*((int *)*(int *)b+2));  
  
    funf();  
    fung();  
    funh();  
  
    cout<<(Fun)(*((int*)*(int*)b+4))<<endl; //最后一个位置为0 表明虚函数表结束 +4是因为定义了一个 虚析构函数  
  
    delete b;  
    return 0;
}
```

![image-20210814151809494](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814151809494.png)

如上图所示，虚函数指针指向的是一个虚函数表，每个虚函数表的元素存储的是虚函数的地址，通过虚函数指针就能够调用不同的虚函数实现。

**2、继承，无虚函数覆盖的情形**

```c++
#include <iostream>  
using namespace std;  
  
class Base {  
public:  
    virtual void f() { cout << "Base::f()" << endl; }  
    virtual void g() { cout << "Base::g()" << endl; }  
    virtual void h() { cout << "Base::h()" << endl; }  
    private:
    int a;
    string b;
    char c;
};  
  
class Derive: public Base {  
    virtual void f1() { cout << "Derive::f1()" << endl; }  
    virtual void g1() { cout << "Derive::g1()" << endl; }  
    virtual void h1() { cout << "Derive::h1()" << endl; }  
};  
  
int main()  
{  
    typedef void (*Fun)();  
  
    Base *b = new Derive;  
    cout << *(int*)b << endl;  
    Fun funf = (Fun)(*(int*)*(int*)b);  
    Fun fung = (Fun)(*((int*)*(int*)b + 1));  
    Fun funh = (Fun)(*((int*)*(int*)b + 2));  
    Fun funf1 = (Fun)(*((int*)*(int*)b + 3));  
    Fun fung1 = (Fun)(*((int*)*(int*)b + 4));  
    Fun funh1 = (Fun)(*((int*)*(int*)b + 5));  
  
  
    funf(); // Base::f()  
    fung(); // Base::g()  
    funh(); // Base::h()  
    funf1(); // Derive::f1()  
    fung1(); // Derive::g1()  
    funh1(); // Derive::h1()  
  
    cout << (Fun)(*((int*)*(int*)b + 6));  
    return 0;
}
```

在内存中的结构如下图所示：

![](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814153407986.png)

如上图所示，子类除了继承了父类的虚函数以外，还有自己的虚函数，而且虚函数表中的顺序也是有先后顺序的。父类的虚函数在子类的虚函数前面

**3、继承，虚函数覆盖的情形**

```c++
#include <iostream>  
using namespace std;  
  
class Base {  
public:  
    virtual void f() { cout << "Base::f()" << endl; }  
    virtual void g() { cout << "Base::g()" << endl; }  
    virtual void h() { cout << "Base::h()" << endl; }  
};  
  
class Derive: public Base {  
    virtual void f() { cout << "Derive::f()" << endl; }    //重写父类的虚函数
    virtual void g1() { cout << "Derive::g1()" << endl; }  
    virtual void h1() { cout << "Derive::h1()" << endl; }  
};  
  
int main()  
{  
    typedef void (*Fun)();  
  
    Base *b = new Derive;  
    cout << *(int*)b << endl;  
    Fun funf = (Fun)(*(int*)*(int*)b);  
    Fun fung = (Fun)(*((int*)*(int*)b + 1));  
    Fun funh = (Fun)(*((int*)*(int*)b + 2));  
    Fun fung1 = (Fun)(*((int*)*(int*)b + 3));  
    Fun funh1 = (Fun)(*((int*)*(int*)b + 4));  
  
  
    funf(); // Derive::f()  
    fung(); // Base::g()  
    funh(); // Base::h()  
    fung1(); // Derive::g1()  
    funh1(); // Derive::h1()  
  
    cout << (Fun)(*((int*)*(int*)b + 5));  
    return 0;  
}
```

当子类重写父类的虚函数的时候，父类的虚函数就会被子类的实例覆盖，覆盖的 f() 函数被放到虚函数表中原来父类虚函数的位置。没有被覆盖的函数依旧。可通过获取成员函数指针来调用成员函数（即时是private类型的成员函数），这就出现一定的安全问题。

![image-20210814154944276](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814154944276.png)

**4、多重继承（无虚函数覆盖）**

```c++
#include <iostream>  
using namespace std;  
  
class Base1 {  
public:  
    virtual void f() { cout << "Base1::f()" << endl; }  
    virtual void g() { cout << "Base1::g()" << endl; }  
    virtual void h() { cout << "Base1::h()" << endl; }  
};  
  
class Base2 {  
public:  
    virtual void f() { cout << "Base2::f()" << endl; }  
    virtual void g() { cout << "Base2::g()" << endl; }  
    virtual void h() { cout << "Base2::h()" << endl; }  
};  
  
  
class Base3 {  
public:  
    virtual void f() { cout << "Base3::f()" << endl; }  
    virtual void g() { cout << "Base3::g()" << endl; }  
    virtual void h() { cout << "Base3::h()" << endl; }  
};  
  
  
class Derive: public Base1,public Base2, public Base3 {  
    virtual void g1() { cout << "Derive::f()" << endl; }  
    virtual void h1() { cout << "Derive::g1()" << endl; }  
};  
  
int main()  
{  
    typedef void (*Fun)();  
  
    Derive d;  
    Base1 *b1 = &d;  
    Base2 *b2 = &d;  
    Base3 *b3 = &d;  
  
  
    b1->f(); //Derive::f()  
    b2->f(); //Derive::f()  
    b3->f(); //Derive::f()  
    b1->g(); //Base1::g()  
    b2->g(); //Base2::g()  
    b3->g(); //Base3::g()  
  
  
    Fun b1fun = (Fun)(*(int*)*(int*)b1);  
    Fun b2fun = (Fun)(*(int*)*((int*)b1+1));  
    Fun b3fun = (Fun)(*(int*)*((int*)b1+2));  
  
    b1fun(); // Derive::f()  
    b2fun(); // Derive::f()  
    b3fun(); // Derive::f()  
  
    return 0;  
}
```

![](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814155641928.png)

每个父类都有自己的虚函数表。

子类的成员函数被放到了第一个父类的表中。（所谓的第一个父类是按照声明的顺序来确定的）

这样做就是为了解决不同的父类类型的指针指向同一个子类实例，而能够调用到实际的函数。

**5、多重继承（有虚函数覆盖）**

```c++
#include <iostream>  
using namespace std;  
  
class Base1 {  
public:  
    virtual void f() { cout << "Base1::f()" << endl; }  
    virtual void g() { cout << "Base1::g()" << endl; }  
    virtual void h() { cout << "Base1::h()" << endl; }  
};  
  
class Base2 {  
public:  
    virtual void f() { cout << "Base2::f()" << endl; }  
    virtual void g() { cout << "Base2::g()" << endl; }  
    virtual void h() { cout << "Base2::h()" << endl; }  
};  
  
  
class Base3 {  
public:  
    virtual void f() { cout << "Base3::f()" << endl; }  
    virtual void g() { cout << "Base3::g()" << endl; }  
    virtual void h() { cout << "Base3::h()" << endl; }  
};  
  
  
class Derive: public Base1,public Base2, public Base3 {  
    virtual void f() { cout << "Derive::f()" << endl; }    //重载了父类的虚函数
    virtual void g1() { cout << "Derive::g1()" << endl; }  
};  
  
int main()  
{  
    typedef void (*Fun)();  
  
    Derive d;  
    Base1 *b1 = &d;  
    Base2 *b2 = &d;  
    Base3 *b3 = &d;  
  
  
    b1->f(); //Derive::f()  
    b2->f(); //Derive::f()  
    b3->f(); //Derive::f()  
    b1->g(); //Base1::g()  
    b2->g(); //Base2::g()  
    b3->g(); //Base3::g()  
  
  
    Fun b1fun = (Fun)(*(int*)*(int*)b1);  
    Fun b2fun = (Fun)(*(int*)*((int*)b1+1));  
    Fun b3fun = (Fun)(*(int*)*((int*)b1+2));  
  
    b1fun(); // Derive::f()  
    b2fun(); // Derive::f()  
    b3fun(); // Derive::f()  
  
    return 0;  
}
```

![image-20210814160730173](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814160730173.png)

**通过父类型的指针访问子类自己的虚函数**

我们知道，子类没有重载父类的虚函数是一件毫无意义的事情。因为多态也是要基于函数重载的。虽然在上面的图中我们可以看到Base1的虚表中有Derive的虚函数，但我们根本不可能使用下面的语句来调用子类的自有虚函数：

```c++
Base1 *b1 = new Derive();

b1->f1(); //编译出错
```

##### 代码验证：

```c++
/*
@author:tangzhao
date:2021.8.14
*/
#include<iostream>
using namespace std;

class Animal
{
    private:
    string name;
    public:
    virtual void speak()
    {
        cout << "动物在说话" << endl;
    }
};

class cat:public Animal
{
    public:
    void speak()
    {
        cout << "会喵喵叫" <<endl;
    }

    void eat()
    {
        cout <<"小猫会吃东西" << endl;
    }
};

class Dog:public Animal
{
    public:
    void speak()
    {
        cout <<"汪汪叫"<<endl;
    }
};

int main()
{
    Animal a;
    a.speak();

    cat c;
    c.eat();
    c.speak();

    Animal * p = new cat();  //父类指针指向子类的对象
    p->speak();
    //p->eat();  报错

    Animal *q = new Dog();
    q->speak();

    /*
    cat * c1 = new Animal();  //子类的指针指向父类的对象，行不通
    c1->speak();
    c1->eat();
    */

    delete p;
    delete q;

    return 0;
}
```

运行结果：

![image-20210814165625292](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814165625292.png)

由上面的代码可以看出，当父类指针或者引用指向子类的对象的时候，虚函数的调用会根据实际的子类的对象的重写的函数来执行。

#### 3、析构函数和构造函数设置为虚函数

**构造函数是不能设置为虚函数的，因为虚函数是基于对象的，构造函数是用来产生对象及对对象进行初始化的，若构造函数设置为虚函数的话，那么需要对象来调用，而对象又需要构造函数来创建，那么就会陷入矛盾的状态。**

##### 3、虚析构函数

```c++
/*
@author:tangzhao
date:2021.8.14
*/
#include<iostream>
using namespace std;


class Animal
{
public:
    Animal()
    {
        cout << "Animal的构造函数调用"<<endl;
    }

    virtual void speak()=0;  //纯虚函数
    ~Animal()
    {
        cout << "Animal的析构函数调用" << endl;
    }
};

class Cat:public Animal
{
public:
    Cat(string name)
    {
        cout << "Cat构造函数调用" << endl;
        m_name = new string(name);  //在堆区开辟新的空间存储
    }
    virtual void speak()
    {
        cout << *m_name << "小猫在说话" <<endl;
    }

    ~Cat()
    {
        cout << "Cat析构函数调用" <<endl;
        if(this->m_name !=NULL)
        {
            delete m_name;
            m_name = NULL;
        }
    }
public:
    string *m_name;
};


void test01()
{
    Animal *animal = new Cat("tom");
    animal->speak();

    delete animal;
}


int main()
{
    test01();

    return 0;
}

```

运行结果

![image-20210814182543016](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814182543016.png)

由上图可知，我们因为在代码中子类对象在堆区开辟内存存储name属性，但是因为父类指针指向子类对象，在释放的过程中，并没有去释放子类在堆区开辟的空间，造成内存泄漏。

![image-20210814182726910](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814182726910.png)

这一部分的代码并没有执行。

**解决办法：将父类的析构函数设置为虚析构**

![image-20210814182906912](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814182906912.png)

修改这一行即可。运行结果：

![image-20210814182941757](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814182941757.png)

这样才能使得子类在堆区的内存被释放掉。

也可以将父类析构函数设置为纯虚析构，但是在类外需要实现，否则会报错。

### 3、c++中的static关键字

#### 3.1、内存的分区模型

![image-20210814193256775](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814193256775.png)

对于堆区的数据，一般是程序员申请的空间，用完之后需要自己释放，而栈区的数据包含的是局部变量（函数内定义的）、参数等，生命结束后由系统自动释放。而静态区的数据要等整个程序运行结束以后才释放。

#### 3.2、static

static 是C++中很常用的修饰符，它被用来控制变量的存储方式和可见性。

**引入static的原因：**

函数内部定义的变量，在程序执行到它的定义处时，编译器为它在栈上分配空间，大家知道，函数在栈上分配的空间在此函数执行结束时会释放掉，这样就产生了一 个问题: 假如想将函数中此变量的值保存至下一次调用时，如何实现？ 最轻易想到的方法是定义一个全局的变量，但定义为一个全局变量有许多缺点，最明显的缺点是破坏了此变量的访问范围（使得在此函数中定义的变量，不仅仅受此 函数控制，所有的函数都共享这个全局变量）。

**什么时候需要static：**

需要一个数据对象为整个类而非某个对象服务,同时又力求不破坏类的封装性,即要求此成员隐藏在类的内部，对外不可见。

C++的static有两种用法：面向过程程序设计中的static和面向对象程序设计中的static。前者应用于普通变量和函数，不涉及类；后者主要说明static在类中的作用。

###### 1、面向过程设计的static

**1.1、静态全局变量**

![image-20210814194737563](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814194737563.png)

静态全局变量的特点：

1、该变量在全局数据区分配内存

2、未经初始化的静态全局变量会被程序自动初始化为0（自动变量的值是随机的，除非它被显式初始化）

3、静态全局变量在声明它的整个文件都是可见的，而在文件之外是不可见的；　静态全局变量不能被其它文件所用； 其它文件中可以定义相同名字的变量，不会发生冲突

![image-20210814195228043](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814195228043.png)

如上图所示，另外一个文件引用静态变量，报错

如果将静态变量改为全局变量的话，如下图：

![image-20210814195451609](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814195451609.png)

这样的话就可以在另外一个文件使用

1.2、静态局部变量

![image-20210814202405809](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814202405809.png)

解决办法：设置为静态变量的话，可以实现变量的永远存储

![image-20210814202653179](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814202653179.png)

因为第一次调用fun（）函数后，最后实现了++，所以a的值变为11，然后下一次fun（）调用就输出11，然后再++，第三次调用之后变为12.

静态局部变量的特点：

1、该变量在全局数据区分配内存；

2、静态局部变量在程序执行到该对象的声明处时被首次初始化，即以后的函数调用不再进行初始化；

3、静态局部变量一般在声明处初始化，如果没有显式初始化，会被程序自动初始化为0； 

4、它始终驻留在全局数据区，直到程序运行结束。但其作用域为局部作用域，当定义它的函数或语句块结束时，其作用域随之结束；

**1.3、静态函数**

在函数的返回类型前加上static关键字,函数即被定义为静态函数。静态函数与普通函数不同，它只能在声明它的文件当中可见，不能被其它文件使用。

![image-20210814203325907](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814203325907.png)

定义静态函数的好处： 
• 静态函数不能被其它文件所用； 
• 其它文件中可以定义相同名字的函数，不会发生冲突；

###### 2、面向对象的static（类中的static关键字）

**2.1、静态数据成员**

在类内数据成员的声明前加上关键字static，该数据成员就是类内的静态数据成员。

静态数据成员的特点：

1、所有对象共享同一份数据

2、在编译阶段分配内存

3、类内生明，类外初始化

```c++
#include<iostream>
using namespace std;

class Person
{
private:
    static int age;  //静态成员变量，私有变量，类内生明
public:
    static int height;  //静态成员变量，公共变量，类内声明
};

//类外初始化
int Person::height =180;
int Person::age =20;

void test01()
{
    Person p1;
    cout << "p1的身高为：" << p1.height <<endl;

    Person p2;
    cout << "p2的身高为：" << p2.height <<endl;

    //修改
    p1.height = 200;
    cout << "-------------------" <<endl;

    cout << "p1的身高为：" << p1.height <<endl;
    cout << "p2的身高为：" << p2.height <<endl;
}

int main()
{
    test01();

    return 0;
}
```

![image-20210814204927719](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814204927719.png)

1、对于非静态数据成员，每个类对象都有自己的拷贝。而静态数据成员被当作是类的成员。无论这个类的对象被定义了多少个，静态数据成员在程序中也只有一份拷贝，由该类型的所有对象共享访问。也就是说，静态数据成员是该类的所有对象所共有的。对该类的多个对象来说，静态数据成员只分配一次内存，供所有对象共用。所以，静态数据成员的值对每个对象都是一样的，它的值可以更新； 

2、静态数据成员存储在全局数据区。静态数据成员定义时要分配空间，所以不能在类声明中定义

3、静态数据成员和普通数据成员一样遵从public,protected,private访问规则；

4、因为静态数据成员在全局数据区分配内存，属于本类的所有对象共享，所以，它不属于特定的类对象，在没有产生类对象时其作用域就可见，即在没有产生类的实例时，我们就可以操作它；**静态成员类出现他就出现**

5、同全局变量相比，使用静态数据成员有两个优势：  静态数据成员没有进入程序的全局名字空间，因此不存在与程序中其它全局名字冲突的可能性； 

可以实现信息隐藏。静态数据成员可以是private成员，而全局变量不能；

**2.2、静态成员函数**

与静态数据成员一样，我们也可以创建一个静态成员函数，它为类的全部服务而不是为某一个类的具体对象服务。静态成员函数与静态数据成员一样，都是类的内部实现，属于类定义的一部分。普通的成员函数一般都隐含了一个this指针，this指针指向类的对象本身，因为普通成员函数总是具体的属于某个类的具体对象的。通常情况下，this是缺省的。如函数fn()实际上是this->fn()。但是与普通函数相比，静态成员函数由于不是与任何的对象相联系，因此它不具有this指针。从这个意义上讲，它无法访问属于类对象的非静态数据成员，也无法访问非静态成员函数，它只能调用其余的静态成员函数。

```c++
#include<iostream>
using namespace std;

class Person
{
public:
    static void func()
    {
        cout << "func()调用" << endl;
        m_A =100;
        //m_B =200;  //错误，静态成员函数只能调用静态成员变量
        cout << "m_A =" << m_A<<endl;
    }

    static int m_A;  //静态变量
    int m_B;  //普通变量
private:
    static void func2()
    {
        cout << "func2()的调用" << endl;
    }
};

int Person::m_A=10;

void test01()
{
    Person p1;
    p1.func();
}

int mian()
{
    test01();
    return 0;
}
```

 1、出现在类体外的函数定义不能指定关键字static；

2、静态成员之间可以相互访问，包括静态成员函数访问静态数据成员和访问静态成员函数

3、非静态成员函数可以任意地访问静态成员函数和静态数据成员； 

4、静态成员函数不能访问非静态成员函数和非静态数据成员；

5、由于没有this指针的额外开销，因此静态成员函数与类的全局函数相比速度上会有少许的增长； 

### 4、const相关

1、const对变量的限制：被const修饰的变量的值不能再被改变

```c++
int a =10;
a=20;  //正确

const int b = 30;
b =40; //错误
```

2、初始化和const

const对象必须被初始化

```c++
const int i =40; //正确
const int j;  //错误，需要进行初始化
```

如果利用一个对象去初始化另外一个对象，则它们是不是const都无关紧要

```c++
int i 42;
const int c =i;  //正确
int j =c;  //正确
```

3、默认状态下，const对象仅在文件内有效

默认情况下，const对象被设定为仅在文件内有效，当多个文件出现同名的const时，其实等同于定义了多个const对象，如果确实需要在文件中共享const对象，那么在const前添加extern关键字即可；

4、const的引用

对常量的引用不能被用作修改它所绑定的对象

```c++
const int ci =1024;  //常量
const int &r1 = ci;  //对常量的引用
r1=42;  //错误，r1是对常量的引用
int &r2 = ci;  //错误，试图让一个非常量引用指向一个常量对象
```

const引用与初始化

允许为一个非常量的引用绑定非常量的对象、字面值甚至是一个表达式

```c++
int i =42;
const int &r1 = i;  //将一个常量引用绑定到普通的对象上
const int &r2 =42;  //将常量引用绑定到字面值
const int &r3 = r1 *2;  //将常量引用绑定到表达式
int &r4 = r1*2;  //错误，将非常量引用指向一个常量对象
```

综上所述：也就是说常量引用是大哥，它想绑定什么都行，非常量的引用就不行了。

5、对const的引用可能引用一个非const的对象

也就是说const引用的对象可能const不能修改，但是可以通过其他途径来修改

```c++
int i =42;
int &r1 = i;  //非常量的引用
const int &r2 = i;  //常量的引用，不允许修改值
r1=0;  //正确，此时i的值可以被r1改变，但是r2也指向了i，所以虽然r2不能直接修改i的值，但是可以通过其他方式修改
```

6、指针和const

可以令指针指向常量或者非常量

6.1、指向常量的指针

类似于指向常量的引用，指向常量的指针，不允许改变所指对象的值

```c++
const double pi =3.14;  //pi是一个常量，他的值不能被改变
double *ptr = &pi;  //错误，ptr是一个普通的指针
const double *cptr =&pi;
*cptr = 42;  //错误，不能修改对象的值
```

6.2、常量指针

指针的指向不可以修改，但是指向的对象的值可以被修改

*在const之前表示的就是一个常量指针

```c++
int a =0;
int * const cur = &q; //cur将会一直指向a，指向不能改变
const double pi = 3.14;
const double * const ptr = &pi;  //两者都不能再改变
```

7、顶层const与底层const

顶层const指的是指针本身是一个常量，即指针的指向不能被改变

底层const指的是指针所指的对象是一个常量，即指针所指的对象的值不能被改变

```c++
区别方法：*在const左边，表示的是指针是常量；

*在const的右边的话，表示的是指针所指的对象是常量
```

```c++
int i =42;
int *const p1 = &i;  //顶层const，p1是一个指针，指针不能被改变
const int * p2 = &i;  //底层const，p2所指的对象的值不能被改变
```

8、constexpr和常量表达式

常量表达式是指值不会改变而且在编译过程中就能得到计算结果的表达式。

判断方法：一个对象（表达式）是不是常量表达式，由它的数据类型和初始值共同决定。

```c++
const int a = 20;  //是一个常量表达式const int b = a+1; //是一个常量表达式int c = 30;  //不是一个常量表达式，因为不是const修饰const int d = get_size();  //不是一个常量表达式，因为右边不确定
```

在一个复杂的系统中，很难验证一个表达式是不是常量表达式，C++ 11新标准规定，允许将变量声明为constexpr类型以便由编译器来验证变量的值是否是一个常量表达式。若不是，则编译报错。同时，声明为constexpr的变量一定是一个常量，而且必须用常量表达式初始化。

```c++
constexpr int mf =20;  //是一个常量表达式
constexpr int sz = size();  //只有当size是一个constexpr函数时，才是正确的生明语句
int i = 10;
constexpr int t = i;                //错误，i不是常量
```

字面值类型：对声明constexpr时用到的类型被称为字面值类型。算术类型、引用、指针、枚举和一些特殊的类都属于字面值类型，而IO库、string类型则不属于字面值类型，也就不能被定义成constexpr。

引用、指针与constexpr

1、尽管指针和引用都能被定义成constexpr，但它们的初始值却受到严格限制。一个constexpr指针的初始值必须是nullptr或者0，或者是存储于某个固定地址中的对象。

2、函数体内定义的变量一般来说并非存在固定地址中（除了static等修饰，且const也不行，static之前不能加限定词volatile，否则出错），相反，定义于所有函数体之外的对象其地址固定不变，能用来初始化constexpr指针，同时能用constexpr引用绑定到这样的变量上。
3、对于指针而言，constexpr仅对指针有效，与指针所指的对象无关。

```c++
const int *p = nullptr;            //正确，p是一个指向整型常量的指针

constexpr int *q = nullptr;        //正确，但q是一个指向整数的常量指针
```

4、constexpr指针既可以指向常量也可以指向一个非常量。

```c++
constexpr int *np = nullptr;        //正确，np是一个指向整数的常量指针，其值为空

int j = 0;

constexpr int i = 42;
```

### 5、树

#### 5.1、树的基本概念

###### **1、什么是树？**

树是一种抽象的数据类型，是一种存储方式，但是目前计算机内部用的很少，大部分还是栈。

特点：

1、每个结点有一个或者多个子节点

2、分层的结构，上一层是父结点、

3、存在唯一一个没有父节点的结点，那就是根结点

具体就是下面这样（注意与我们生活中的树区分，按照生活中的树来说，计算机的树是一棵倒立的树）

![image-20210814214422720](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814214422720.png)

节点深度：对任意节点x，x节点的深度表示为根节点到x节点的路径长度。所以根节点深度为0，第二层节点深度为1，以此类推
节点高度：对任意节点x，叶子节点到x节点的路径长度就是节点x的高度
树的深度：一棵树中节点的最大深度就是树的深度，也称为高度
父节点：若一个节点含有子节点，则这个节点称为其子节点的父节点
子节点：一个节点含有的子树的根节点称为该节点的子节点
节点的层次：从根节点开始，根节点为第一层，根的子节点为第二层，以此类推
兄弟节点：拥有共同父节点的节点互称为兄弟节点
度：节点的子树数目就是节点的度
叶子节点：度为零的节点就是叶子节点
祖先：对任意节点x，从根节点到节点x的所有节点都是x的祖先（节点x也是自己的祖先）
后代：对任意节点x，从节点x到叶子节点的所有节点都是x的后代（节点x也是自己的后代）
森林：m颗互不相交的树构成的集合就是森林
![image-20210814214759176](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814214759176.png)

###### 2、树的分类

树分为有序树和无序树，无序树就是任意结点没有什么关系，这样在计算机里面没有实际的意义，所以一般考虑的是有序树。

###### 3、二叉树

二叉树就是一个父节点最多只有两个子节点的树，也是用的最多的树的结构，子节点也被称为左子树和右子树

性质：

1、第i层最多有2^(i-1)个结点

2、深度为k的二叉树至多有2^k - 1个结点

3、一棵二叉树,叶节点为N0,度为2的节点数为N2,则N0=N2+1;

4、n个结点的完全二叉树深度必为log2(n+1)

5、对于完全二叉树,编号为i的节点,左孩子编号必为2i,右孩子编号2i+1

**3.1、完全二叉树**

设二叉树的深度为k,除第k层外,其他各层节点树的结点数都达到了最大值（即都是2）,k层所有的节点都连续集中在最左边，而且按层遍历的时候顺序是有序的

![image-20210814221139357](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814221139357.png)

特点：

- 叶子节点只出现在最下层或者次下层
- 最下层的叶子节点几种在左部
- 倒数第二层如果存在叶子节点,一定在右部连续位置
- 如果节点的度为1,该节点只有左孩子没有右孩子
- 同样节点数目的二叉树,完全二叉树深度最小

**3.2、满二叉树**

![image-20210814221308044](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814221308044.png)

- 从外形上来看，满二叉树一个三角形
- 满二叉树的节点总数为2^k-1，且节点数一定为奇数个。满二叉树的第k层节点的个数是2^(k-1)

**3.3、平衡二叉树**(AVL树)

![image-20210814221852734](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814221852734.png)

平衡二叉树的左子树和右子树的深度之差的绝对值不超过1.

**3.4、二叉查找树**

二叉查找树的左孩子小于当前结点的值,右孩子大于当前节点的值.图解:

![image-20210814222227372](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210814222227372.png)

特点：

1. 若任意结点的左子树不空,则左子树上所有结点的值均小于它的根结点的值;
2. 若任意结点的右子树不空,则右子树上所有的结点的值均大于它的根结点的值
3. 任意结点左右子树分别为二叉查找树
4. 对二叉查找树进行中序遍历即可得到有序的数列;

**3.5、哈夫曼树**

哈夫曼大叔说，对于一棵树，从一个结点到另外一个结点的路径上的分支数目称为路径的长度。树的路径长度就是从树根到每一结点的路径长度之和。如果考虑到带权的结点，结点的带权的路径长度就为从该结点到树根之间的路径长度与结点上权的乘积。树的带权路径长度为树中所有叶子结点的带权路径长度之和。

![image-20210816132634798](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210816132634798.png)

如上图所示，A到D的路径的长度为2，到C的路径的长度为1。

如果带权值的话，可以计算带权路径的最小长度

![image-20210816132816679](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210816132816679.png)

根节点到叶子结点的带权路径的长度计算为：

A->D:3x1+8x2=19

A->E:3x1+5x2=13；

概念:给定n个权值作为n个叶子结点，构造一棵二叉树，若带权路径长度达到最小.最优二叉树
应用:数据的压缩,编码长度的优化
路径:从一个结点往下可以达到的孩子或孙子结点之间的通路
路径长度:通路中节点的个数,包含目标节点,不包含原始节点;
节点的带权路劲长度:从根节点到该节点的路径长度与该节点权的乘积;
霍夫曼编码:编码规则是从根节点出发，向左标记为0，向右标记为1。

**3.6、B树**





### 6、虚拟内存的概念

没有虚拟内存的时候会出现的问题：

1、内存是有限的，多个进程要执行的时候，要等待，并且需要频繁的装入内存，很影响效率。

2、由于指令都是直接访问物理内存的，那么这个进程就能修改其他进程的数据，甚至会修改内核地址的数据，会不安全。

内存是用于存放数据的硬件，程序执行之前需要把数据加载到内存才能被CPU处理

![image-20210816151231633](C:\Users\tangzhao\AppData\Roaming\Typora\typora-user-images\image-20210816151231633.png)

内存中的空间是按地址编号的，由外存的数据加载到内存，需要进行地址的寻址，这需要用到逻辑地址和物理地址；

逻辑地址：就是相对的地址

物理地址：实际的地址

装入的方式：

1、绝对装入：装入之前就指定好装入到内存的哪个位置（只适合单道程序）

2、静态重定位：专门有一个装入程序来计算装入的地址，需要提前分配足够的空间

3、动态重定位，通过重定位寄存器计算在内存中的地址

链接的方式：

1、静态链接

2、装入时动态链接

3、运行时动态链接

覆盖技术

交换技术

分页管理

局部性原理

虚拟内存：请求调页和页面置换



### 7、get和post的区别

1、安全性：get请求的数据是URL地址明文发送的，不安全，post的请求数据不会出现在地址栏，较为安全

2、数据量限制：get请求有数据量的限制，而post没有

3、等幂性：get请求数据不会修改服务器的状态，如读取静态文件（图片、html文件等），相当于获取数据；

post一般会改变服务器的状态，比如添加数据库文件和信息等。

4、get产生一个TCP数据包，POST产生两个TCP数据包

对于get方式，浏览器会将http header和data一并发送出去，服务器响应200（返回数据）

对于post方式，浏览器先发送header，服务器响应100，浏览器再发送data，服务器响应200（返回数据）

Get是向服务器发索取数据的一种请求，而Post是向服务器提交数据的一种请求，

Get是获取信息，而不是修改信息，类似数据库查询功能一样，数据不会被修改
Get请求的参数会跟在url后进行传递，请求的数据会附在URL之后，以?分割URL和传输数据，参数之间以&相连,％XX中的XX为该符号以16进制表示的ASCII，如果数据是英文字母/数字，原样发送，如果是空格，转换为+，如果是中文/其他字符，则直接把字符串用BASE64加密。
Get传输的数据有大小限制，因为GET是通过URL提交数据，那么GET可提交的数据量就跟URL的长度有直接关系了，不同的浏览器对URL的长度的限制是不同的。
GET请求的数据会被浏览器缓存起来，用户名和密码将明文出现在URL上，其他人可以查到历史浏览记录，数据不太安全。在服务器端，用Request.QueryString来获取Get方式提交来的数据

Post请求则作为http消息的实际内容发送给web服务器，数据放置在HTML Header内提交，Post没有限制提交的数据。Post比Get安全，当数据是中文或者不敏感的数据，则用get，因为使用get，参数会显示在地址，对于敏感数据和不是中文字符的数据，则用post
POST表示可能修改变服务器上的资源的请求，在服务器端，用Post方式提交的数据只能用Request.Form来获取

### 8、C++什么类不能被继承

C++什么类不能被继承，首先思考派生类继承基类将会发生什么默认操作？派生类在调用自身的构造函数之前需要先调用基类的构造函数。那么我们就让这个不想被别人继承的类的构造函数无法被其派生类构造。现在主要有三种方式阻止类的构造函数被调用，一是，将自身的构造函数与析构函数放在private作用域内；二是，将自身作为一个已存在类的友元类。这两种方式都能阻止派生类的继承（因为自身无法构造函数），第三种，使用C++11新特性final。
**一、将自身的构造函数与析构函数放在private作用域**

当本文声明一个对象时，编译器将调用构造函数（如果有），而这个调用将通常是public的，假如构造函数是private呢，由于在class外部不允许访问private成员，所以将会导致编译出错。如何解决这个问题呢？
我们可以使用class的static公有成员，因为它独立于class对象之外，不必产生对象也可以使用它们。假如在类的某个static函数中创建了该class的对象，并以引用或者指针的形式将其返回（这里不以对象返回，主要是构造函数是私有的，外部不能创建临时对象），使用new在堆上创建对象，这样可以手动的创建和销毁对象，而如果在栈上创建对象，则会在生命周期结束的时候释放掉对象出现不可预估的错误。

### **9、输入网址后,会经历哪几个步骤**

https://blog.csdn.net/weixin_42269817/article/details/107165713

