---
title: c++快速入门
date: 2019-06-11
tags: c++
---
>本文重点关注与c语言不同的地方

## hello,world

```
#include <iostream>
using namespace std;

int main(){
    cout << "Hello,World"<<endl;
}
```
linux下使用g++编译
```
g++ helloworld.cpp
./a.out
Hello,World
```

## 基本语法

### 枚举
```
#include <iostream>
using namespace std;

int main(){
    //enum color {green=5,red,blue} c;//red为6,blue为7
    enum color {green,red,blue} c;//green默认从0开始
    //c = blue;
    c = red;
    //c = green;
    cout << c <<endl;
}
```

### 常量
使用#define或者const定义常量

```
#define LENGTH 10   
#define WIDTH  5

const int  LENGTH = 10;
const int  WIDTH  = 5;
```

### static,extern,thread_local

static
    * 修饰全局变量时,表示可见性仅在本文件内
    * 修饰局部变量时,函数返回后不会销毁变量
    * 修饰类成员时,仅有一个成员副本被所有类对象共享
extern
    * 当有两个或多个文件共享相同的全局变量或者函数

thread_local
    * 声明的变量仅可在它在其上创建的线程上访问

### string 
c++标准库中提供了string类型
```
   string str1 = "Hello";
   string str2 = "World";
```
### 函数
函数参数可以赋默认值

### 引用

```
int&  r = i;
double& s = d;
```

函数参数可以是引用,返回值也可以是引用.当参数是引用时函数中的修改会变更引用对象的值,当返回值是引用时实际返回的是一个指向返回值的隐式指针,因此可以放在左边赋值

### 输入输出流

cout,cin,cerr,clog 
cout<<"hello,world"
cin>>name

## 面向对象

###  构造函数
使用初始化列表来初始化字段
```
Line::Line( double len): length(len)
{
    cout << "Object is being created, length = " << len << endl;
}

等价于

Line::Line( double len)
{
    length = len;
    cout << "Object is being created, length = " << len << endl;
}
```
其中length是类中的一个成员变量

冒号后边也可以是一个父类的构造函数


### 析构函数

```
~Line();
```
### 拷贝构造函数

```
classname (const classname &obj) {
   // 构造函数的主体
}
```
拷贝构造函数在三种情况下会被调用

* 一个对象以值传递方式传入函数
* 一个对象以值传递的方式从函数返回
* 一个对象需要通过另外一个对象进行初始化 

### 继承

```
class derived-class: access-specifier base-class
```

access-specifier可以为public,protected,private,修饰符会对基类中的成员在派生类的可访问性做出不同处理

```
class <派生类名>:<继承方式1><基类名1>,<继承方式2><基类名2>,…
{
<派生类类体>
};
```
多继承

### 重载
同一作用域内,可以重载函数和操作符
函数有相同的名称但是有不同的参数列表,编译器进行重载决策
```
#include <iostream>
using namespace std;
 
class printData
{
   public:
      void print(int i) {
        cout << "整数为: " << i << endl;
      }
 
      void print(double  f) {
        cout << "浮点数为: " << f << endl;
      }
 
      void print(char c[]) {
        cout << "字符串为: " << c << endl;
      }
};
 
int main(void)
{
   printData pd;
 
   // 输出整数
   pd.print(5);
   // 输出浮点数
   pd.print(500.263);
   // 输出字符串
   char c[] = "Hello C++";
   pd.print(c);
 
   return 0;
}
```

赋值运算符重载

```
#include <iostream>
using namespace std;
 
class Distance
{
   private:
      int feet;             // 0 到无穷
      int inches;           // 0 到 12
   public:
      // 所需的构造函数
      Distance(){
         feet = 0;
         inches = 0;
      }
      Distance(int f, int i){
         feet = f;
         inches = i;
      }
      void operator=(const Distance &D )
      { 
         feet = D.feet;
         inches = D.inches;
      }
      // 显示距离的方法
      void displayDistance()
      {
         cout << "F: " << feet <<  " I:" <<  inches << endl;
      }
      
};
int main()
{
   Distance D1(11, 10), D2(5, 11);
 
   cout << "First Distance : "; 
   D1.displayDistance();
   cout << "Second Distance :"; 
   D2.displayDistance();
 
   // 使用赋值运算符
   D1 = D2;
   cout << "First Distance :"; 
   D1.displayDistance();
 
   return 0;
}
```
输出如下:
```
First Distance : F: 11 I:10
Second Distance :F: 5 I:11
First Distance :F: 5 I:11
```

### 纯虚函数

```
class Shape {
   protected:
      int width, height;
   public:
      Shape( int a=0, int b=0)
      {
         width = a;
         height = b;
      }
      // pure virtual function
      virtual int area() = 0;
};
```

= 0 告诉编译器，函数没有主体，上面的虚函数是纯虚函数。

## 其他

### 模板

函数模板:

```
template <class type> ret-type func-name(parameter list)
{
   // 函数的主体
}
```

示例:


```

#include <iostream>
#include <string>
 
using namespace std;
 
template <typename T>
inline T const& Max (T const& a, T const& b) 
{ 
    return a < b ? b:a; 
} 
int main ()
{
 
    int i = 39;
    int j = 20;
    cout << "Max(i, j): " << Max(i, j) << endl; 
 
    double f1 = 13.5; 
    double f2 = 20.7; 
    cout << "Max(f1, f2): " << Max(f1, f2) << endl; 
 
    string s1 = "Hello"; 
    string s2 = "World"; 
    cout << "Max(s1, s2): " << Max(s1, s2) << endl; 
 
   return 0;
}
```

类模板:
```
template <class type> class class-name {
.
.
.
}
```
示例:
```
#include <iostream>
#include <vector>
#include <cstdlib>
#include <string>
#include <stdexcept>
 
using namespace std;
 
template <class T>
class Stack { 
  private: 
    vector<T> elems;     // 元素 
 
  public: 
    void push(T const&);  // 入栈
    void pop();               // 出栈
    T top() const;            // 返回栈顶元素
    bool empty() const{       // 如果为空则返回真。
        return elems.empty(); 
    } 
}; 
 
template <class T>
void Stack<T>::push (T const& elem) 
{ 
    // 追加传入元素的副本
    elems.push_back(elem);    
} 
 
template <class T>
void Stack<T>::pop () 
{ 
    if (elems.empty()) { 
        throw out_of_range("Stack<>::pop(): empty stack"); 
    }
    // 删除最后一个元素
    elems.pop_back();         
} 
 
template <class T>
T Stack<T>::top () const 
{ 
    if (elems.empty()) { 
        throw out_of_range("Stack<>::top(): empty stack"); 
    }
    // 返回最后一个元素的副本 
    return elems.back();      
} 
 
int main() 
{ 
    try { 
        Stack<int>         intStack;  // int 类型的栈 
        Stack<string> stringStack;    // string 类型的栈 
 
        // 操作 int 类型的栈 
        intStack.push(7); 
        cout << intStack.top() <<endl; 
 
        // 操作 string 类型的栈 
        stringStack.push("hello"); 
        cout << stringStack.top() << std::endl; 
        stringStack.pop(); 
        stringStack.pop(); 
    } 
    catch (exception const& ex) { 
        cerr << "Exception: " << ex.what() <<endl; 
        return -1;
    } 
}
```


## 参考链接
* https://www.runoob.com/cplusplus/cpp-tutorial.html
