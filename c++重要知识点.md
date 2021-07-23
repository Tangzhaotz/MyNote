## c++重要知识点

### day1

### 1、const修饰变量和指针

const修饰指针，常量指针，指针指向可以改，指针指向的值不可以改

```c++
const int *p;
```

const修饰的是常量，指针常量，指针指向不可以改，指针指向的值可以改

```c++
int * const p;
```

 ### 2、不要返回局部变量的地址和引用

因为局部变量的地址存储与内存的栈区，当调用完后由操作系统自动释放，当再次调用时，会出现随机数，不正确。

### 3、堆区

堆区的数据由程序员创建并释放，若程序员不释放，程序结束时由操作系统回收（注意与栈区的区别），在堆区创建的数据可以返回地址或者引用

### 4、注意const修饰的引用

```c++
const int & ref=10;等价于
temp=10;
const int &ref = temp;
```

### 5、函数重载的条件

1、同一个作用域下

2、函数名称相同

3、函数参数类型不同，或者个数不同，或者顺序不同

### 6、引用作为重载的条件

```c++
void func(int &a)
{
    cout << "func(int &a)调用" << endl;
}
void func(connst int &a)
{
    cout << "func(const int &a)调用" << endl;
}

int main()
{
    int a = 10;
    func(a);  //调用无const
    func(10);//调用有const，原因见4
}
```

### 7、构造函数的调用

调用无参构造函数时，不要加括号，加了编译器会认为是一个函数声明

```c++
class Person
{
  public:
    int age;
    Person()
    {
        cout << "无参构造函数" << endl;
    }
    Person(int a)
    {
        age = a;
        cout << "有参构造函数" << endl;
    }
};

void test01()
{
    Person p;  //无参构造函数调用
    //Person p();  调用无参构造不能加括号
}
```

### 8、深拷贝与浅拷贝

浅拷贝会带来重复释放堆区的问题，需要利用深拷贝进行改进

### 9、静态成员

静态成员就是在成员变量和长远函数前加上static关键字，称为静态成员

静态成员分为：

**静态成员变量：**

所有对象共享同一份数据

在编译阶段分配内存

类内声明，类外初始化

**静态成员函数：**

所有对象共享同一个函数

静态成员函数只能访问静态成员变量

```c++
class person
{
   public：
       static int m_A;//静态成员变量
    private:
    static int m_B;  //类内声明
}；
int Person::m_A=10;  //类外初始化
int Person::m_B = 10;

void test01()
{
    //静态成员变量的访问方式
    //1、通过对象
    Person p1;
    p1.m_A=100;
    cout << "p1.m_A=" << p1.m_A << endl;  //100
    
    Person p2;
    p2.m_A=200;
    //共享同一份数据，一个变化，另外一个也变化
    cout << "p1.m_A=" << p1.m_A << endl;  //200
    cout << "p2.m_A=" << p2.m_A << endl;  //200
    
    //2、通过类名
    cout << "m_A=" << Person::m_A << endl;
    //cout << "m_B=" << Person::m_B << endl;//不能访问，因为是私有权限
}

//静态成员函数
class Person
{
  public:
    static int m_A;  //静态成员变量
    int m_b;  //非静态成员变量
    
    static void func()
    {
        cout << "func调用" << endl;
        m_A=100;
        //m_b = 100;  //错误，因为静态成员函数只能访问静态成员变量，不能访问非静态成员变量
    }
};
```

### 10、c++中，类内的成员变量和成员函数分开存储

只有非静态成员变量才属于类的对象上

```c++
class Person
{
  public:
    int m_A;  //非静态成员变量，占对象空间
    static int m_B;  //静态成员变量不占对象空间
    
    void func()  //非静态成员函数不占对象空间
    {
        cout << "m_A" << this->m_A << endl;
    }
    static void sfunc()
    {
        //静态成员函数也不占对象空间
    }
};
```

### 11、c++中this指针

c++中成员变量和成员函数是分开存储的，对于共享的同一份代码，如何知道哪个对象调用自己，需要通过this指针来操作，this指针指向被调用的成员函数所属的对象。

this指针隐含每一个非静态成员函数内的一种指针

this指针不需要定义，直接使用即可

this指针用途：

1、当形参和成员变量同名时，可以用this指针来区分

2、在类的非静态成员函数中返回对象本身，可以用return *this



```c++
class Person
{
public:
    int age;  //成员变量
    
	Person(int age)  //这里的age表示形参，与成员变量同名，可以用this区分
    {
        this->age = age;
    }
    
    Person & PersonADdPerson(Person p)  //成员函数
    {
        this->age += age;
        return *this;
    }
}

void test01()
{
    Person p1(10);
    cout << "p1.age=" << p1.age << endl;  //10
    
    Person p2(10);
    p2.PersonAddPerson(p1).PersonAddPerson(p1).PersonAddPerson(p1);  //40,调用返回对象本身，即返回p1
}
```

### 12、空指针访问成员函数

c++中空指针也可以访问成员函数，但是要注意有没有用到this指针

```c++
class Person
{
  public:
    int mAge;
    void ShowClassName()
    {
        cout << "我是Person类" << endl;
    }
    void ShowPerson()
    {
        if(this==NULL)
        {
            return;
        }
        cout << mAge << endl;
    }
};

void test01()
{
    Person *p = NULL;  //空指针
    p->ShowClassName();  //空指针可以调用成员函数
    //p->ShowPerson(); //成员函数用到this指针，不可以调用
}
```

### 13、const修饰成员函数

成员函数后加const我们称为常函数

常函数内不可以修改成员属性

成员属性声明时加mutable后，常函数中仍然可以修改

常对象：声明对象前加const称为常对象

常对象只能调用常函数（与静态成员函数类似，只能调用静态成员变量）

### 14、友元

友元的目的就是为了让一个函数或者类访问另一个类中的私有成员

### 15、继承

继承中出现子类和父类同名成员函数，需要加作用域

```c++
class Base
{
  public:
    int m_A;
    Base()
    {
        m_A=100;
    }
};

class Son:public Base
{
  Son()
  {
      m_A=200;
  }
};

void test01()
{
    Son s;
    cout << "Son下的m_A=" << s.m_A << endl;//200
    cout << "Base下的m_A=" << s.Base::m_A << endl;//100
}
```

### 16、菱形继承（钻石继承）

两个派生类继承同一个基类，又有某个类同时继承两个派生类，这种继承称为菱形继承或者钻石继承

例如:羊和驼都是继承动物类，羊驼继承羊和驼

存在的问题：对于同一个动物，草泥马从羊和驼继承来两份动物属性，会存在歧义，所以只需要一份动物就行，利用虚继承解决

```c++
class Animal
{
  public:
    int m_Age;
};
//羊类
class Sheep:virtual public Animal
{
    
}
//驼类
class Tuo:virtual public Animal
{
    
}
//羊驼类
class SheepTuo:public Sheep,public Tuo
{
    
}

void test01()
{
    SheepTuo st;
    st.Sheep::m_Age=100;
    st.Tuo::m_Age = 200;
    cout << "st.Sheep::m_Age =" << st.Sheep::m_Age << endl;//200
    cout << "st.Tuo::m_Age =" << st.Tuo::m_Age << endl;  //2020
    cout << "st.m_Age =" << st.m_age << endl; //200
}
```

### 分割线----------------------

