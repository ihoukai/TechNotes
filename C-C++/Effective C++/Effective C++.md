
## 一. 让自己习惯C++
### 1. 视C++为一个语言联邦
### 2. 尽量以const、enum、inline替换#define
#### enum hack
~~~cpp
class GamePlayer {
private:
    static const int NumTurn = 5;       // 常量声明式
    int scores[NumTurn];                // 使用该常量
}
~~~
然而你所看到的是NumTurns的声明式而非定义式。通常c++要求你对你所使用的任何东西提供一个定义式，但如果它是个class专属常量又是static且为整数类型(例如ints,chars,bools),则需特殊处理。只要不取他们的地址，你可以声明并使用它们而无需提供定义式。但如果你取某个class专属常量的地址，或纵使你不取其他地址而你的编译器却（不正确地）坚持要看到一个定义式，你就必须另外提供定义式如下：
~~~cpp
const int GamePlayer::NumTurns; // NumTurns的定义；
~~~
把这个式子放进一个实现文件而非头文件。由于class常量已在声明时获得初值，以此定义时不可以再设初值。

旧式编译器也许不支持上述语法，它们不允许static成员在其声明式上获得初值。此外所谓的“in-class初值设定”也只允许对整数常量进行。如果你的编译器不支持上述语法，你可以将初值放在定义式：
~~~cpp
class CostEstimate{
private:
    static const double FudgeFactor; // static class 常量声明位于头文件内
};
const double CostEstimate::FudgeFactor = 1.35; // static class常量定义位于实现文件内
~~~
这几乎是你在任何时候唯一需要做的事。唯一的例外是当你在class编译期间需要一个class常量值，例如上述的GamePlayer::scores的数组声明式中（是的，编译器必须坚持在编译期间知道数组的大小）。这时候万一你的编译器（错误地）不允许“static整数类型class常量”完成“in-class初值设定”，可改用所谓的“the enum hack”补偿做法。
~~~cpp
class GamePlayer{
private:
    enum { NumTurns = 5 };
    int scores[NumTurns];
}   
~~~
> - 对于单纯常量，最好以const对象或enums替换#defines。
> - 对于形似函数的宏，最好改用inline函数替换#defines。 

### 3. 尽可能使用const

#### 指针与const

~~~
char greeting[] = "Hello";
char *p = greeting;                 // non-const pointer,non-const data
const char *p = greeting;           // non-const pointer,const data
char * const p = greeting;          // const pointer,non-const data
cosnt char * const p = greeting;    // const pointer,const data
~~~

#### const成员函数
将const实施于成员函数的目的，是为了确认该成员函数可作用于const对象身上。

许多人漠视一件事实：两个成员函数如果只是常量性不同，可以被重载。
~~~cpp
class CTextBlock {
public:
    ...
    std::size_t length() const;
private: 
    char* pText;
    std::size_t textLength;
    bool lengthIsValid;
};

std::size_t CTextBlock::length() const
{
    if(!lengthIsValid) {
        textLength = std::strlen(pText);    // 错误！在const成员函数内不能赋值给textLength和lengthIsValid。
        lengthIsValid = true;
    }
    return textLength;
}
~~~
解决办法很简单： 利用c++的一个与const相关的摆动场: mutable(可变的)。mutable释放掉non-static成员变量的bitwise constness约束。

#### 在const和non-const成员函数中避免重复
const_cast 与 static_cast
> - 当const和non-const成员函数有着实质等价的实现时，令non-const版本调用const版本可避免代码重复。

### 4. 确定对象初始化
通常如果你使用c part of c++ 而且初始化可能招致运行期成本，那么久不保证发生初始化。一旦进入non-c parts of c++，规则有些变化。这就很好的解释了为什么array(来自c part of c++)不保证其内容被初始化，而vector(来自STL part of c++)却有此保证。

#### 使用成员初始化列表
~~~cpp
ABEntry::ABEntry(const std::string& name, const std::String& address, const std::list<PhoneNumber>& phones):
theName(name),
theAddress(address),
thePhones(phones),
numTimesConsulted(0)
{}
~~~
使用成员初始化列表比在构造函数内赋值的效率高。在构造函数内赋值的版本 首先调用default构造函数为theName, theAddress和thePhones设初值，然后立刻再对它们赋予新值。default构造函数的一切作为因此浪费了。
> 重要的是别混淆了赋值和初始化

#### 初始化顺序
c++有着十分固定的“成员初始化次序”。次序总是相同：base classes更早于其derived classes被初始化，而class的成员变量总是以其声明次序被初始化。即使它们在成员初值列中以不同的次序出现，也不会有任何影响。
##### 不同编译单元内定义之non-local static对象
所谓static对象，其寿命从被构造出来直到程序结束为止，因此stack和heap-based对象都被排除。这种对象包括global对象、定义于namespace作用域内的对象、在classe内、在函数内、以及在file作用域内被声明为static的对象。**函数内的static对象称为local static对象（因为它们对函数而言是local），其他static对象称为non-local static对象**。程序结束时static对象会被自动销毁，也就是它们的析构函数会在main()结束时被自动调用。

所谓编译单元是指产出单一目标文件的那些源码。基本上它是单一源码文件加上其所含入的头文件(#inclde files).

现在，我们关心的问题涉及至少两个源码文件，每一个内含至少一个non-local static对象。真正的问题是：如果某编译单元内的某个non-local static对象的初始化动作使用了另一个编译单元内的某个non-local static对象，它所用到的这个对象可能尚未被初始化，因为**c++对“定义于不同编译单元内的non-local static对象”的初始化次序并无明确定义**。

幸运的是一个小小的设计便可完全消除这个问题。唯一需要做的是：将每个non-local static 对象搬到自己的专属函数内.这些函数返回一个reference指向它所含的对象。然后用户调用这些函数，而不是直接指涉这些对象。non-local static 对象被local static对象替换了。这是Singleton模式的一个常见实现手法。

**c++保证，函数内的local static对象会在“该函数被调用期间”“首次遇上该对象之定义式”时被初始化**。所以如果你以“函数调用”替换“直接访问non-local static对象”，你就获得了保证，保证你所获得的那个reference将指向一个经历初始化的对象。更棒的是，如果你从未调用non-local static对象的“仿真函数”，就绝不会引发构造和析构成本。
~~~cpp
class FileSystem{...};
FileSystem& tfs()
{
    static FileSystem fs;
    return fs;
}
~~~
> - 为内置型对象进行手工初始化，因为c++不保证初始化它们。
> - 构造函数最好使用成员初值列，而不要在构造函数本体内使用赋值操作。初值列列出的成员变量，其排列次序应该和它们在class中的声明次序相同。
> - 为免除“跨编译单元之初始化次序”问题，请以local static对象替换non-local static对象。

## 二. 构造、析构、赋值
### 5. 了解C++默默编写并调用那些函数
如果你写一个空类，编译器就会为它声明一个copy构造函数、一个copy assignment操作符和一个析构函数。如果你没有声明任何构造函数，编译器也会为你声明一个default构造函数。所有这些函数都是public且inline。

因此，如果你写下：
~~~cpp
class Empty{};
~~~
这就好像你写下这样的代码：
~~~cpp
class Empty{
public:
    Empty(){...}                            // default构造函数
    Empty(const Empty& rhs){...}            // copy构造函数
    ~Empty(){...}                           // 析构函数
    Empty& operator=(const Empty& rhs){...} // copy assignment操作符
}
~~~
唯有当这些函数被需要（被调用），它们才会被编译器创建出来。**注意，编译器产出的析构函数是个non-virtual， 除非这个class的base class自身声明有virtual析构函数**

### 6. 若不想使用编译器自动生成的函数，就该明确拒绝
所有编译器产出的函数都是public。为阻止这些函数被创建出来，你得自行声明它们，你可以将copy构造函数或copy assignment操作符声明为private。

一般而言这个做法并不绝对安全，因为member函数和friend函数还是可以调用你的private函数。除非你够聪明，不去定义它们，那么如果某些人不慎调用任何一个，会获得一个连接错误。**“将成员函数声明为private而且故意不实现它们”这一伎俩是如此为大家接受，因而被用在c++ iostream程序库中阻止copying行为**。

将连接期错误移至编译器是可能的，只要将copy构造函数和copy assignment操作符声明为private就可以办到，但不是在HomeForSale自身，而是一个专门为了阻止copying动作而设计的base class内。
~~~cpp
class UnCopyable{
protected:
    UnCopyable(){}
    ~UnCopyable(){}
private:
    UnCopyable(const UnCopyable&);
    UnCopyable& operator=(const UnCopyable&);
}

class HomeForSale: private UnCopyable {
    // class 不再声明copy构造函数 或copy assign操作符
};
~~~
只要任何人---甚至是member函数或friend函数---尝试拷贝HomeForSale对象，编译器便试着生成一个copy构造函数和一个copy assignment操作符，这些函数的“编译器生成版”会尝试调用其base class的对应兄弟，那些调用会被编译器拒绝，因为其base class的拷贝函数是private。（也可以使用Boost提供的版本noncopyable）
> 为驳回编译器自动提供的机能，可将相应的成员函数声明为private 并且不予实现。使用像Uncopyable这样的base class也是一种做法。

### 7. 为多态基类声明virtual析构函数
c++明白指出，当derived class对象经由一个base class指针被删除，而该base class带着一个non-virtual析构函数，其结构未有定义------实际执行时通常发生的是对象的derived成分没被销毁。

**任何class只要带有virtual函数都几乎确定应该也有一个virtual析构函数；如果class不含virtual函数，通常表示它并不意图被用做一个base class。当class不企图被当做base class，令其析构函数为virtual往往是个馊主意**

欲实现出virtual函数，对象必须携带某些信息，主要用来在运行期决定哪一个virtual函数该被调用。这份信息通常是由一个所谓vptr指针指出。vptr指向一个由函数指针构成的数组，称为vtbl；每一个带有virtual函数的class都有一个相应的vtbl。当对象调用某一virtual函数，实际被调用的函数取决于该对象的vptr所指的那个vtbl---编译器在其中寻找适当的函数指针。

因此，**无端地将所有classes的析构函数声明为virtual，就像从未声明它们为virtual一样，都是错误的**。

有时候令class带一个pure virtual析构函数，可能颇为便利。pure virtual函数导致abstract classes------也就是不能被实体化的class。

**析构函数的运作方式是，最深层派生的那个class其析构函数最先被调用，然后是其每一个base class的析构函数被调用**。

“给base classes一个virtual析构函数”，这个规则只适用于polymorphic（带多态性质的）base classes身上。**这种base classes的设计目的是为了用来“通过base class接口处理derived class对象”**。

**并非所有base classes的设计目的都是为了多态用途**。例如标准string和STL容器都不被设计作为base classes使用，更别提多态了。某些classes的设计目的是作为base classes使用，但不是为了多态用途。这样的classes如条款6的UnCopyable和标准程序库的input_iterator_tag,它们并非被设计用来“经由base class接口处置derived class对象”，因此它们不需要virtual析构函数。

> - polymorphic(带多态性质的)base classes应该声明一个virtual析构函数。如果class 带有任何virtual函数，它就应该拥有一个virtual析构函数。
> - Classes的设计目的如果不是作为base classes使用，或不是为了具备多态性，就不该声明virtual析构函数。

### 8. 别让异常逃离析构函数
> - 析构函数绝对不要吐出异常。如果一个被析构函数调用的函数可能抛出异常，析构函数应该捕捉任何异常，然后吞下他们（不传播）或结束程序。
> - 如果客户需要对某个操作函数运行期间抛出的异常做出反应，那么class应该提供一个普通函数（而非在析构函数中）执行该操作。

### 9. 绝不在构造和析构过程中调用virtual函数
~~~cpp
class Transaction{
public:
    Transaction();
    virtual void logTransaction() const = 0;
};

Transaction::Transaction()
{
    ...
    logTransaction();
}

class BuyTransaction: public Transaction{
public:
    virtual void logTransaction() const;
    ...
};

class SellTransaction: public Transaction{
public:
    virtual void logTransaction() const;
    ...
};

~~~
现在，当以下这行被执行，会发生什么事：
~~~cpp
BuyTransaction b;
~~~
无疑地会有一个BuyTransaction构造函数被调用，但首先Transaction构造函数一定会更早被调用；derived class 对象内的base class成分会在derived class自身成分被构造之前先构造妥当。这时候被调用的logTransaction是Transaction内的版本，不是BuyTransaction内的版本------即使目前即将建立的对象类型是BuyTransaction。**base class构造期间virtual函数绝对不会下降到derived classes阶层。取而代之的是，对象的作为就像隶属base类型一样。非正式的说法或许比较传神：在base class构造期间，virtual函数不是virtual函数**。

**在derived class对象的base class构造期间，对象的类型是base class而不是derived class**。不只virtual函数会被编译器解析至base class，若使用运行期类型信息（例如dynamic_cast和typeid），也会把对象视为base class类型。

相同道理也适用于析构函数。一旦derived class析构函数开始执行，对象内的derived class成员变量便呈现未定义值，所以c++ 视它们仿佛不再存在。**进入base class析构函数后对象就成为一个base class对象，而c++的任何部分包括virtual函数、dynamic_casts等等也就那么看待它**。
> 在构造和析构期间不要调用virtual函数，因为这类调用从不下降至derived class。

### 10. 令operator= 返回一个reference to *this
### 11. 在operator= 处理“自我赋值”
在operator= 函数内手工排列语句的一个替代方案是，使用所谓的copy and swap技术。这个技术和“异常安全性”有密切关系。它是一个常见而够好的operator= 编写办法:
~~~cpp
class Widget{
...
void swap(Widget& rhs)
...
};
Widget& Widget::operator=(const Widget& rhs)
{
    Widget temp(rhs);   // 为rhs数据制作一份复件
    swap(temp);         // 将*this数据和上述复件的数据交换
    return *this;
}
~~~
> - 确保当对象自我赋值时operator= 有良好行为。其中技术包括比较“来源对象”和“目标对象”的地址、精心周到的语句顺序、以及copy-and-swap。
> - 确定任何函数如果操作一个以上的对象，而其中多个对象是同一个对象时，其行为仍然正确。

### 12. 复制对象时勿忘其每一个成分
你应该让derived class的copying函数调用相应的base class函数：
~~~cpp
PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
:Customer(rhs),
priority(rhs.priority)
{
}

PriorityCustomer&
PriorityCustomer::operator=(const PriorityCustomer& rhs)
{
    Customer::operator=(rhs);   // 对base class成分进行赋值动作
    priority = rhs.priority;
    return *this;
}
~~~

## 三. 资源管理
### 13. 以对象管理资源
> - 为防止资源泄露，请使用RAII对象，它们在构造函数中获得资源并在析构函数中释放资源。
> - 两个常被使用的RAII classes分别是trl::shared_ptr和auto_ptr。前者通常是较佳选择，因为其copy行为比较直观。

### 14. 在资源管理类中小心coping行为
trl::shared_ptr允许指定所谓的“删除器”，那是一个函数或函数对象，当引用次数为0时便被调用。
~~~cpp
class Lock{
public:
    explicit Lock(Mutex* pm):mutexPtr(pm, unlock){
        lock(mutexPtr.get());
    }
private:
    std::trl::shared_ptr<Mutex> mutexPtr;
}
~~~
请注意，本例的Lock class不再声明析构函数。class析构函数（无论是编译器生成的，或用户自定的）会自动调用其non-static成员变量的析构函数。而mutexPtr的析构函数会在互斥器引用次数为0时自动调用trl::shared_ptr的删除器。
> - 复制RAII对象必须一并复制它所管理的资源，所以资源的copying行为决定RAII对象的copying行为。
> - 普通而常见的RAII class copying行为是：抑制copying、施行引用计数法。

### 15. 在资源管理类中提供对原始资源的访问
### 16. 成对使用new和delete时要采取相同形式
当你使用new,有两件事发生。第一，内存被分配出来。第二，针对此内存会有一个（或更多）构造函数被调用。当你使用delete，也有两件事发生：针对此内存会有一个（或更多）析构函数被调用，然后内存才被释放。
~~~cpp
typedef std::string AddressLines[4];
std::string* pal = new AddressLines;
delete pal;         // 行为未有定义
delete [] pal;      // 很好
~~~
**为了避免诸如此类的错误，最好不要对数组形式做typedefs动作**。

### 17. 以独立语句将newed对象置入智能指针
假设我们有个函数用来揭示处理程序的优先权，另一个函数用来在某动态分配所得的Widget上进行某些带有优先权的处理：
~~~cpp
int priority();
void processWidget(std::trl::shared_ptr<Widget> pw, int priority);
~~~
现在考虑调用processWidget:
~~~cpp
processWidget(std::trl::shared_ptr<Widget>(new Widget), priority());
~~~
令人惊讶的是，虽然我们在此使用“对象管理式资源”，上述调用却可能泄露资源。
在调用processWidget之前，编译器必须创建代码，做以下三件事：
- 调用priority
- 执行“new Widget”
- 调用trl::shared_ptr构造函数

**c++编译器以什么样的次序完成这些事情呢？弹性很大。这和其他语言如java和c#不同，那两种语言总是以特定次序完成函数参数的核算**。可以确定的是“new Widget”一定执行于trl::shared_ptr构造函数被调用之前，但对priority的调用则可以排在第一或第二或第三执行。如果编译器选择以第二顺位执行它，最终获得这样的操作序列：
1. 执行“new Widget”
2. 调用priority
3. 调用trl::shared_ptr构造函数

万一priority调用导致异常，则“new Widget”返回的指针将会遗失，造成资源泄露。

避免这类问题的办法很简单：使用分离语句。如下：
~~~cpp
std::trl::shared_ptr<Widget> pw(new Widget);
processWidget(pw, priority());
~~~

## 四. 设计与声名
### 18. 让接口容易被正确使用，不易被误用
trl::shared_ptr有一个特别好的性质是：它会自动使用它的“每个指针专属的删除器”，因而消除另一个潜在的客户错误：所谓的“cross-DLL problem”。这个问题发生于“对象在动态连接程序库DLL中被new创建，却在另一个DLL内被delete销毁”。在许多平台上，这一类“跨DLL之new/delete成对运用”会导致运行期错误。
### 19. 设计class犹如设计type
### 20. 使用pass-by-reference-to-const替换pass-by-value
以引用的方式传递参数优点：
- 效率高，没有任何构造函数或析构函数被调用，没有新对象被创建
- 可以避免slicing对象切割问题。当一个derived class对象以by value方式传递并被视为一个base class对象，base class的copy构造函数会被调用，而“造成此对象的行为像个derived class对象”的那些特化性质全被切割掉了，仅仅留下一个base class对象。

> - 尽量以pass-by-reference-to-const替换pass-by-value。前者通常比较高效，并可避免切割问题
> - 以上规则并不适用于内置类型，以及STL的迭代器和函数对象。对它们而言，pass-by-value往往比较适当。

### 21. 必须返回对象时，别妄想返回其reference
~~~cpp
const Rational& operator*(const Rational& lhs,
                          const Rational& rhs)
{
    Rational result(lhs.n * rhs.n, lhs.d * rhs.d);             //  警告！糟糕的代码！
    return result;
}
~~~
这个函数返回一个reference指向result，但result是个local对象，而local对象在函数退出前被销毁了。

一个“必须返回新对象”的函数的正确写法：
~~~cpp
inline const Rational operator*(const Rational& lhs,
                          const Rational& rhs)
{
    return Rational result(lhs.n * rhs.n, lhs.d * rhs.d);
}
~~~
当然，你需要承受operator* 返回值的构造成本和析构成本，然而长远来看那只是为了获得正确行为而付出的一个小小代价。但万一账单很恐怖，你承受不起，别忘了c++和所有编程语言一样，允许编译器实现者施行最优化。因此某些情况下operator* 返回值的构造和析构可被安全地消除。
> - 绝不要返回pointer或reference指向一个local stack对象，或返回reference指向一个heap-allocated对象，或返回pointer或reference指向一个local static对象而有可能同事需要多个这样的对象。

### 22. 将成员变量声明为private
> - 切记将成员变量声明为private。这可赋予客户访问数据的一致性、可细微划分访问控制、允诺约束条件获得保证，并提供class作者以充分的实现弹性。
> - protected并不比public更具封装性。

### 23. 宁以non-member、non-friend替换member函数
让我们从封装开始讨论。如果某些东西被封装，它就不再可见。愈多东西被封装，愈少人可以看到它。而愈少人看到它，我们就有愈大的弹性去变化它，因为我们的改变仅仅直接影响看到改变的那些人事物。因此，愈多东西被封装，我们改变那些东西的能力也就愈大。它使我们能够改变事物而只影响有限客户。

现在考虑对象内的数据。愈少代码可以看到数据（也就是访问它），愈多的数据可被封装，而我们也就愈能自由地改变对象数据。愈多函数可访问它，数据的封装性就愈低。

如果要你在一个menber函数和一个non-member，non-friend函数之间做抉择，而且两者提供相同机能，那么，导致较大封装性的是non-member、non-friend函数，因为它并不增加“能够访问class内之private成分”的函数数量。

在c++，比较自然的做法是让clearBrowser成为一个non-member函数并且位于WebBrowser所在的同一个namespace内：
~~~cpp
namespace WebBrowserStuff{
    class WebBrowser {...};
    void clearBrowser(WebBrowser& wb);
    ...
}
~~~
要知道，**namespace和classes不同，前者可跨越多个源码文件而后者不能**。

一个像WebBrowser这样的class可能拥有大量便利函数，某些与书签有关，某些与打印有关，还有一些与cookie的管理有关......通常大多数客户只对其中某些感兴趣。没道理一个只对书签相关便利函数感兴趣的客户却与例如一个cookie相关便利函数发生编译相依关系。分离它们的最直接做法就是将书签相关便利函数声明于一个头文件，将cookie相关便利函数声明于另一个头文件，再将打印相关便利函数声明于第三个头文件，以此类推：
~~~cpp
// 头文件"webbrowser.h"---这个头文件针对class WebBrowser自身及WebBrowser核心机能
namespace WebBrowserStuff{
class WebBrowser{...};
...     // 核心技能，例如几乎所有客户都需要的non-member函数
}

// 头文件“webbrowserbookmarks.h”
namespace WebBrowserStuff{
...         // 与书签相关的便利函数
}

// 头文件“webbrowsercookies.h”
namespace WebBrowserStuff{
...         // 与cookie相关的便利函数
}
~~~
注意，这正是c++标准程序库的组织方式。

### 24. 若所有参数皆需类型转换，请为此采用non-member函数
~~~cpp
class Rational{
    ...
};
const Rational operator* (const Rational& lhs,
                          const Rational& rhs)
{
    return Rational(lhs.numerator() * rhs.numerator(),
                    lhs.denominator() * rhs.denominator());
}

Rational oneFourth(1, 4);
Rational result;
result = oneFourth * 2;             // 没问题
result = 2 * oneFourth;             // 万岁，通过编译
~~~

### 25. 考虑写出一个不抛异常的swap函数
~~~cpp
namespace std{
    template<typename T>        // std::swap的典型实现
    void swap(T& a, T& b)
    {
        T temp(a);
        a = b;
        b = temp;
    }
}
~~~

std::swap特化版本
~~~cpp
class Widget{
public:
    ...
    void swap(Widget& other)
    {
        using std::swap;
        swap(pImpl, other.pImpl);
    }
    ...
private:
    WidgetImpl* pImpl;
};

namespace std{
    template<>
    void swap<Widget>(Widget& a, Widget& b)
    {
        a.swap(b);
    }
}
~~~

假设Widget和WidgetImpl都是class templates，如果这么写：
~~~cpp
namespace std{
    template<typename T>
    void swap(Widget<T>& a, Widget<T>& b)
    {
        a.swap(b);
    }
}
~~~
一般而言，重载function templates没有问题，但**std是个特殊的命名空间，其管理规则也比较特殊。客户可以全特化std内的templates，但不可以添加新的templates到std里头**。解决方法如下：
~~~cpp
namespace WidgetStuff{
    ...
    template<typename T>
    class Widget {...};
    ...
    
    template<typename T>
    void swap(Widget<T>& a, Widget<T>& b)
    {
        a.swap(b);
    }
}
~~~
现在，任何地点的任何代码如果打算置换两个Widget对象，因而调用swap，c++的名称查找法则会找到WidgetStuff内的Widget专属版本。

假设你正在写一个function template，其内需要置换两个对象值：
~~~cpp
template<typename T>
void doSomething(T& obj1, T& obj2)
{
    ...
    swap(obj1, obj2);
    ...
}
~~~
应该调用哪个swap？是std既有的那个一般化版本？还是某个可能存在的特化版本？抑或是一个可能存在的T专属版本而且可能栖身于某个命名空间内？你希望的应该是调用T专属版本，并在该版本不存在的情况下调用std内的一般化版本。下面是你希望发生的事：
~~~cpp
template<typename T>
void doSomething(T& obj1, T& obj2)
{
    using std::swap;        // 令std::swap在此函数内可用
    ...
    swap(obj1, obj2);       // 为T型对象调用最佳swap版本
    //std::swap(obj1, obj2);  // Important!!!这是错误的swap调用方式
    ...
}
~~~
一旦编译器看到对swap的调用，它们便查找适当的swap并调用之。**C++名称查找法则确保将找到global作用域或T所在之命名空间内的任何T专属的swap**。如果T是Widget并位于命名空间WidgetStuff内，编译器会使用“实参取决之查找规则”找出WidgetStuff内的swap。如果没有T专属之swap存在，编译器就使用std内的swap，这得感谢using声明式让std::swap在函数内曝光。然而即便如此编译器还是比较喜欢std::swap的T专属特化版，而非一般化的那个template，所以如果你已针对T将std::swap特化，特化版会被编译器挑中。
> - 当std::swap对你的类型效率不高时，提供一个swap成员函数，并确定这个函数不抛出异常。
> - 如果你提供一个member swap，也该提供一个non-member swap用来调用前者。对于classes（而非templates），也请特化std::swap。
> - 调用swap时应针对std::swap使用using声明式，然后调用swap并且不带任何“命名空间修饰”
> - 为“用户定义类型”进行std templates全特化是好的，但千万不要尝试在std内加入某些对std而言全新的东西。

## 五. 实现
### 26. 尽可能延后变量定义式的出现时间
### 27. 尽量少做转型动作

#### 对象地址
~~~cpp
class Base {...};
class Derived: public Base{ ... };
Derived d;
Base* pb = &d;          // 隐喻地将Derived* 转换为 Base*
~~~
这里我们不过是建立一个base class指针指向一个derived class对象，但有时候上述的两个指针值并不相同。这种情况下会有个偏移量(offset)在运行期被施行于Derived* 指针身上，用于取得正确的Base* 指针值。

上述例子表明，**单一对象（例如一个类型为Derived的对象）可能拥有一个以上的地址**（例如“以Base* 指向它”时的地址和“以Derived*指向它”时的地址）。C、Java、C#不可能发生这种事！但c++ 可能！实际上一旦使用多重继承，这事几乎一直发生着。即使在单一继承中也可能发生。**这意外着你通常应该避免做出“对象在c++中如何布局”的假设。当然更不该以此假设为基础执行任何转型动作。对象的布局方式和它们的地址计算方式随编译器的不同而不同。**

#### dynamic_case
在探究dynamic_case设计意涵之前，值得注意的是，**dynamic_cast的许多实现版本执行速度相当慢**。例如至少有一个很普通的实现版本基于“class名称之字符串比较”，如果你在四层深的单继承体系内的某个对象身上执行dynamic_cast,刚才说的那个实现版本所提供的每一个dynamic_cast可能会耗用多大四次的strcmp调用，用以比较class名称。

> - 如果可以，尽量避免转型，特别是在注重效率的代码中避免dynamic_casts。如果有个设计需要转型动作，试着发展无需转型的替代设计。
> - 如果转型是必要的，试着将它隐藏于某个函数背后。客户随后可以调用该函数，而不需将转型放进他们自己的代码内。
> - 宁可使用c++ -style转型，不要使用旧式转型。前者很容易辨识出来，而且也比较有着分门别类的职称。

### 28. 避免返回handles指向对象内部成分
### 29. 为“异常安全”而努力是值得的
有个一般性规则是这么说的：较少的码就是较好的码，因为出错机会比较少，而且一旦有所改变，被误解的机会也比较少。

有个一般化的设计策略很典型地会导致强烈保证，这个策略被称为**copy and swap**。原则很简单：为你打算修改的对象作出一份副本，然后在那副本身上做一切必要修改。若有任何修改动作抛出异常，原对象仍保持未改变状态。待所有改变都成功后，再将修改过的那个副本和原对象在一个不抛出异常的操作中置换。**copy-and-swap策略是对对象状态做出“全有或全无”改变的一个很好办法**，但一般而言它并不保证整个函数有强烈的异常安全性。
> - 异常安全函数即使发生异常也不会泄露资源或允许任何数据结构败坏。这样的函数区分为三种可能得保证：基本型、强烈型、不抛异常型。
> - “强烈保证”往往能够以copy-and-swap实现出来，但“强烈保证”并非对所有函数都可实现或具备现实意义。
> - 函数提供的“异常安全保证”通常最高只等于其所调用之各个函数的“异常安全保证”中的最弱者。

### 30. 透彻了解inlining的里里外外

inline函数背后的整体观念是，将“对此函数的每一个调用”都以函数本体替换之，这样做可能增加你的目标码大小。在一台内存有限的机器上，过度热衷inlining会造成程序体积太大。即使拥有虚内存，inline造成的代码膨胀依会导致额外的换页行为，降低指令高速缓存装置的击中率，以及伴随这些而来的效率损失。

inline只是对编译器的一个申请，不是强制命令。这项申请可以隐喻提出，也可以明确提出。隐喻方式是将函数定义于class定义式内。


大部分编译器拒绝将太过复杂的函数inlining，而所有对virtual函数的调用也会使inlining落空。因为virtual意外“等待，直到运行期才确定调用哪个函数”，而inline意外“执行前，先将调用动作替换为被调用函数的本体”。


构造函数和析构函数往往是inlining的糟糕候选人。考虑以下Derived class构造函数：
~~~cpp
class Base {
public:
...
private:
    std::string bm1, bm2;
};

class Derived: public Base {
public:
    Derived(){}
    ...
private:
    std::string dm1, dm2, dm3;
};
~~~
编译器为稍早说的那个表面上看起来为空的Derived构造函数所产生的代码，相当于以下所列：
~~~cpp
Derived::Derived()
{
    Base::Base();
    try{ dm1.std::string::string();}
    catch(...){
        Base::~Base();
        throw;
    }
    
    try{ dm2.std::string::string();}
    catch(...) {
        dm1.std::string::~string();
        Base::~Base();
        throw;
    }
    
    try{ dm3.std::string::string(); }
    catch(...) {
        dm2.std::string::string();
        dm1.std::string::string();
        Base::~Base();
        throw;
    }
}
~~~

程序库设计者必须评估“将函数声明为inline”的冲击：inline函数无法随着程序库的升级而升级。如果f是程序库内的一个inline函数，客户将“f函数本体”编进其程序中，一旦程序库设计者决定改变f，所有用到f的客户端程序都必须重新编译。
> - 将大多数inlining限制在小型、被频繁调用的函数身上。这可使日后的调试过程和二进制升级更容易，也可以使潜在的代码膨胀问题最小化，使程序的速度提升机会最大化。
> - 不要只因为function templates出现在头文件，就将它们声明为inline

### 31. 将文件间的编译依存关系降至最低

编译器必须在编译期间知道对象的大小。考虑这个：
~~~cpp
int main()
{
    int x;              // 定义一个int
    Person p(params);   // 定义一个Person
    ...
}
~~~
当编译器看到x的定义式，它知道必须分配多少内存才够持有一个int。没问题，每个编译器都知道一个int有多大。当编译器看到p的定义式，它也知道必须分配足够空间以放置一个Person，但它如何知道一个Person对象有多大呢？编译器获得这项信息的唯一办法就是询问class定义式。然而如果class定义式可以合法地不列出实现细目，编译器如何知道分配多少空间？

针对Person我们可以这样做：把Person分割为两个classes，一个只提供接口，另一个负责实现该接口。如果负责实现的那个所谓的implemention class取名为PersonImpl，Person将定义如下：
~~~cpp
#include <string>               // 标准库组件不该被前置声明
#include <memory>

class PersonImpl;               // Person实现类的前置声明
class Date;                     // Person接口用到的classes的前置声明
class Address;

class Person{
public:
    Person(const std::string& name, const Date& birthday,
            const Address& addr);
    std::string name() const;
    std::string birthDate() const;
    std::string address() const;
private:
    std::trl::shared_ptr<PersonImpl> pImpl; // 指针指向实现物
}
~~~
这里Person只内含一个指针成员，指向其实现类，这般设计常被称为pimpl idiom。

这样的设计之下，Person的客户就完全与Dates Addresss以及Persons的实现细目分离了。那些classes的任何实现修改都不需要Person客户端重新编译。

这个分离的关键在于以“声明的依存性”替换“定义的依存性”，那正是编译依存性最小化的本质: 现实中让头文件尽可能自我满足，万一做不到，则让它与其他文件内的声明式（而非定义式）相依。其他每一件事都源自这个简单的设计策略：
- **如果使用object references 或 object pointers可以完成任务，就不要使用objects**
- **尽量以class声明式替换class定义式**
~~~cpp
class Date;                     // class 声明式
Date today();   
void clearAppointments(Date d);
~~~
声明today函数和clearAppointments函数而无需定义Date，这种能力可能会令你惊讶，但它并不是真的那么神奇。一旦任何人调用那些函数，调用之前Date定义式一定得先曝光才行。**如果能够将“提供class定义式”的义务从“函数声明所在”之头文件转移到“内含函数调用”之客户文件，便可将“并非真正必要之类型定义”与客户端之间的编译依存性去除掉**。

- **为声明式和定义式提供不同的头文件**
如：
~~~cpp
#include "datefwd.h"            // 这个头文件内声明(但未定义)class Date
Date today();
void clearAppointments(Date d);
~~~
只含声明式的那个头文件名为"datefwd.h"，命名方法取法c++标准程序库头文件<iosfwd>.

另一个制作Handle class的办法是，令Person成为一种特殊的abstract base class，称为Interface class。
一个针对Person而写的Interface class或许看起来像这样:
~~~cpp
class Person{
public:
    virtual ~Person();
    virtual std::string name() const = 0;
    virtual std::string birthDate() const = 0;
    virtual std::string address() const = 0;
    ...
};
~~~
Interface class的客户必须有办法为这种class创建新对象。
~~~cpp
class Person{
public:
    ...
    static std::trl::shared_ptr<Person>
    create(const std::string& name,
            const Date& birthday,
            const Address& addr);
    ...
}
~~~

Handle classes和Interface classes解除了接口和实现之间的耦合关系，从而降低文件间的编译依存性

## 六. 继承与面向对象设计
### 32. 确定你的public继承塑模出is-a关系
### 33. 避免遮掩继承而来的名称
~~~cpp
class Base{
private:
    int x;
public:
    virtual void mf1() = 0;
    virtual void mf1(int);
    virtual void mf2();
    void mf3();
    void mf3(double);
};

class Derived: public Base{
public:
    virtual void mf1();
    void mf3();
    void mf4();
};

Derived d;
int x;
...
d.mf1();        // 没问题
d.mf1(x);       // 错误！因为Derived::mf1遮掩了Base::mf1
d.mf2();        // 没问题
d.mf3();        // 没问题
d.mf3(x);       // 错误！因为Derived::mf3遮掩了Base::mf3
~~~
这段代码带来的行为会让每一位第一次面对它的c++程序员大吃一惊。以作用域基础的“名称遮掩规则”并没有改变，因此base class内所有名为mf1和mf3的函数都被Derived class内的mf1和mf3函数遮掩掉了。从名称查找观点来看，Base::mf1和Base::mf3不再被Derived继承！

**即使base classes和derived classes内的函数有不同的参数类型也适用，而且不论函数是virtual或non-virtual一体适用**。

这些行为背后的基本理由是为了防止你在程序库或应用框架内建立新的derived class时附带地从疏远的base classes继承重载函数。不幸的是你通常会想继承重载函数。

你可以使用using声明式达成目标：
~~~cpp
class Base{
private:
    int x;
public:
    virtual void mf1() = 0;
    virtual void mf1(int);
    virtual void mf2();
    void mf3();
    void mf3(double);
};

class Derived: public Base{
public:
    using Base::mf1;            // 让Base class内名为mf1和mf3的所有东西在Derived作用域内都可以见
    using Base::mf3;
    virtual void mf1();
    void mf3();
    void mf4();
};

Derived d;
int x;
...
d.mf1();        // 没问题
d.mf1(x);       // 现在没有问题了，调用Base::mf1
d.mf2();        // 没问题
d.mf3();        // 没问题
d.mf3(x);       // 现在没问题了，调用Base::mf3 错误！因为Derived::mf3遮掩了Base::mf3
~~~

> - derived classes内的名称会遮掩base classes内的名称。在public继承下从来没有人希望如此
> - 为了让遮掩的名称再见天日，可使用using声明式

### 34. 区分接口继承和实现继承
### 35. 考虑virtual函数以外的其他选择

- 使用non-virtual interface手法，那是Template Method设计模式的一种特殊形式。它以public non-virtual成员函数包裹较低访问性（private或protected）的virtual函数。
- 将virtual函数替换为“函数指针成员变量”，这是strategy设计模式的一种分解表现形式
- 以trl::function成员变量替换virtual函数，因而允许使用任何可调用物搭配一个兼容于需求的签名式。
- 将继承体系内的virtual函数替换为另一个继承体系内的virtual函数。这是strategy设计模式的传统实现手法

> - virtual函数的替代方案包括NVI手法及strategy设计模式的多种形式。
> - 将机能从成员函数移到class外部函数，带来的一个缺点是，非成员函数无法访问class的non-public成员。
> - trl::function 对象的行为就像一般函数指针。这样的对象可接纳“与给定之目标签名式兼容”的所有可调用物。

### 36. 绝不重新定义继承而来的non-virtual函数
### 37. 绝不重新定义继承而来的缺省参数值
让我们一开始就将讨论简化。你只能继承两种函数：virtual和non-virtual函数。然而重新定义一个继承而来的non-virtual函数永远是错误的，所以我们可以安全地将本条款的讨论局限于“继承一个带有缺省参数值的virtual函数”。

这种情况下，本条款成立的理由就非常直接而明确了：**virtual函数系动态绑定，而缺省参数值却是静态绑定**。

### 38. 通过复合塑模出has-a或"根据某物实现出"

复合意味has-a（有一个）或is-implemented-in-terms-of(根据某物实现出)。那是因为你正打算在你的软件中处理两个不同的领域。程序中的对象其实相当于你所塑造的世界中的某些事物，例如人、汽车、一张张视频画面等等。这样的对象属于应用域部分。其他对象则纯粹是实现细节上的人工制品，像是缓冲区、互斥器、查找树等等。这些对象相当于你的软件的实现域。**当复合发生于应用域内的对象之间，表现出has-a的关系；当它发生于实现域内则是表现is-implemented-in-terms-of的关系**。

### 39. 明智而审慎地使用private继承
private继承意外implemented-in-terms-of（根据某物实现出）；尽可能使用复合，必要时才使用private继承

### 40. 明智而审慎地使用多重继承

> - 多重继承比单一继承复杂。它可能导致新的歧义性，以及对virtual继承的需要
> - virtual继承会增加大小、速度、初始化（及赋值）复杂度等等成本。如果virtual base classes不带任何数据，将是最具实用价值的情况
> - 多重继承的确有正当用途。其中一个情节涉及“public 继承某个Interface class”和“private 继承某个协助实现的class”的两相结合。

## 七. 模板与泛型编程
### 41. 了解隐式接口和编译器多态
Templates及泛型编程的世界，与面向对象有根本上的不同。在此世界中显式接口和运行期多态仍然存在，但重要性降低。反倒是银式接口和编译器多态移到前头了。
~~~cpp
template<typename T>
void doProcessing(T& w)
{
    if (w.size() > 10 && w != someNastyWidget) {
        T temp(w);
        temp.normalize();
        temp.swap(w);
    }
}
~~~
1. w必须支持哪一种接口，系由template中执行于w身上的操作来决定。本例看来w的类型T好像必须支持size, normalize和swap成员函数、copy构造函数、不等比较。这一组表达式便是T必须支持的一组隐式接口。
2. 凡涉及w的任何函数调用，例如operator>和operator!=,有可能造成template具现化，使这些调用得以成功。这样的具现行为发生在编译期。“以不同的template参数具现化function templates”会导致调用不同的函数，这便是所谓的编译器多态。

纵使你从未用过templates，应该不陌生“运行期多态”和“编译器多态”之间的差异，因为它类似于“哪一个重载函数该被调用”（发生在编译器）和“哪一个virtual函数该被绑定”（发生在运行期）之间的差异。

> - classes和templates都支持接口和多态
> - 对classes而言接口是显式的，以函数签名为中心，多态则是通过virtual函数发生于运行期
> - 对template参数而言，接口是隐式的，奠基于有效表达式。多态则是通过template具现化和函数重载解析发生于编译期。

### 42. 了解typename的双重意义
> - 使用关键字typename标识嵌套从属类型名称；但不得在base class lists或member initialization内以它作为base class修饰符

### 43. 学习处理模板化基类内的名称
### 44. 将于参数无关的代码抽离templates
### 45. 运用成员函数模板接受所有兼容类型
### 46. 需要类型转换时请为模板定义非成员函数
### 47. 请使用traits classes表现类型信息
### 48. 认识template元编程

## 八. 定制new和delete
### 49. 了解new-handler的行为
当operator new抛出异常以反映一个未获得满足的内存需求之前，它会先调用一个客户指定的错误处理函数，一个所谓的new-handler。为了指定这个“用以处理内存不足”的函数，客户必须调用set_new_handler,那是声明于<new>的一个标准程序库函数:
~~~cpp
namespace std{
    typedef void (*new_handler)();
    new_handler set_new_handler(new_handler p) throw();
}
~~~
当operator new无法满足内存申请时，它会不断调用new-handler函数，直到找到足够内存。一个良好的new-handler函数必须做以下事情:
- 让更多内存可被使用
- 安装另一个new-handler
- 卸除new-handler
- 抛出bad_alloc
- 不返回

~~~cpp
class NewHandlerHolder{
public:
    explicit NewHandlerHolder(std::new_handler nh):handler(nh){}
    ~NewHandlerHolder(){
        std::set_new_handler(handler);
    }
private:
    std::new_handler handler;
    NewHandlerHolder(const NewHandlerHolder&);     // 阻止copying
    NewHandlerHolder& operator=(const NewHandlerHolder&);
}

void* Widget::operator new(std::size_t size) throw(std::bad_alloc)
{
    NewHandlerHolder h(std::set_new_handler(currentHandler));
    return ::operator new(size);
}
~~~

#### 设定class专属之new-handler
~~~cpp
template<typename T>
class NewHandlerSupport {
public:
    static std::new_handler set_new_handler(std::new_handler p) throw();
    static void* operator new(std::size_t size) throw(std::bad_alloc);
    ...
private:
    static std::new_handler currentHandler;
};

template<typename T>
std::new_handler
NewHandlerSupport<T>::set_new_handler(std::new_handler p)throw()
{
    std::new_handler oldHandler = currentHandler;
    currentHandler = p;
    return oldHandler;
}

template<typename T>
void* NewHandlerSupport<T>::operator new(std::size_t size)throw(std::bad_alloc)
{
    NewHandlerSupport h(std::set_new_handler(currentHandler));
    return ::operator new(size);
}

// 以下将每一个currentHandler初始化为null
template<typename T>
std::new_handler NewHandlerSupport<T>::currentHandler = 0;
~~~
这个设计的base class部分让derived classes继承它们所需的set_new_handler和operator new,而template部分则确保每一个derived class获得一个实体互异的currentHandler成员变量。

有了这个class template，为Widget添加set_new_handler支持能力就轻而易举。
~~~cpp
class Widget: public NewHandlerSupport<Widget>{
    ...
};
~~~
NewHandlerSupport template从未使用其类型参数T。实际上T的确不需被使用。我们只是希望，继承自NewHandlerSupport的每一个class，拥有实体互异的NewHandlerSupport复件（更明确地说是其static成员变量currentHandler）。**参数类型T只是用来区分不同的derived class。 Template机制会自动为每一个T生成一份currentHandler**。

### 50. 了解new和delete的合理替换时机
### 51. 编写new和delete时需固守常规
~~~cpp
void* operator new(std::size_t size)throw(std::bad_alloc)
{
    using namespace std;
    if (size == 0) {    // c++规定，即使客户要求0 bytes,operator new 也得返回一个合法指针
        size = 1;
    }
    while(true) {
        // 尝试分配size bytes
        if (分配成功)
        return (一个指针，指向分配得来的内存)

        // 分配失败，找出目前的new-handling函数
        new_handler globalHandler = set_new_handler(0);
        set_new_handler(globalHandler);
        
        if (globalHandler) (*globalHandler)();
        else throw std::bad_alloc();
    }
}
~~~

许多人没有意识到operator new成员函数会被derived classes继承。这会导致某些有趣的复杂度。

~~~cpp
class Base {
public:
    static void* operator new(std::size_t size)throw(std::bad_alloc);
    ...
};

class Derived: public Base //假设Derived未声明operator new
{...};

Derived *p = new Derived;
~~~
如果Base class专属的operator new并非被设计用来对付上述情况，处理此形势的最佳做法是将“内存申请量错误”的调用行为改采用标准operator new，像这样:
~~~cpp
void * Base::operator new(std::size_t size)throw(std::bad_alloc)
{
    if (size != sizeof(Base))           // 如果大小错误，令标准的operator new处理
        return ::operator new(size);
}
~~~

> - operator new应该内含一个无穷循环，并在其中尝试分配内存，如果它无法满足内存需求，就该调用new-handler。它也应该有能力处理0 bytes申请。class专属版本则还应该处理“比正确大小更大的（错误）申请”
> - operator delete应该在收到null指针时不做任何事。class专属版本则还应该处理“比正确大小更大的（错误）申请”

### 52. 写了placement new也要写placement delete

如果operator new接受的参数除了一定会有的那个size_t之外还有其他，这便是所谓的placement new。如下：
~~~cpp
void* operator new(std::size_t, void* pMemory) throw();
~~~

这个版本的new已被纳入c++标准程序库，这个new的用途之一是负责在vector的未使用空间上创建对象。

顺带一提，**由于成员函数的名称会掩盖其外围作用域中的相同名称，你必须小心避免让class专属的news掩盖客户期望的其他news**。假设你有一个base class，其中声明唯一一个placement operator new,客户端会发现他们无法使用正常形式的new:
~~~cpp
class Base{
public:
    ...
    static void* operator new(std::size_t size,
                    std::ostream& logStream) throw(std::bad_alloc); // 这个new会遮掩正常的global形式
    ...
}；

Base *pb = new Base;      // 错误！正常形式的operator new被遮掩
Base *pb = new (std::cerr) Base; // 正确，调用Base的Placement new
~~~

同样道理，derived classes中的operator news会遮盖global版本和继承而得的operator new版本：
~~~cpp
class Derived: public Base {
public:
    ...
    static void* operator new (std::size_t size) throw (std::bad_alloc);
    ...
};

Derived* pd = new (std::clog) Derived;  // 错误！ Base的Placement new被遮盖
Derived* pd = new Derived; // 正确
~~~

缺省情况下c++在global作用域内提供以下形式的operator new:
~~~cpp
void* operator new(std::size_t) throw (std::bad_alloc);
void* operator new(std::size_t, void*) throw ();
void* operator new(std::size_t, const std::nothrow_t&) throw ();
~~~
如果你在class内声明任何operator news，它会遮掩上述这些标准形式。除非你的意思就是要阻止class的客户使用这些形式，否则请确保他们在你所生成的任何定制型operator new之外还可用。

完成以上所言的一个简单做法是，建立一个base class，内含所有正常形式的new 和 delete：
~~~cpp
class StandardNewDeleteForms {
public:
    // normal new/delete
    static void* operator new(std::size_t size) throw(std::bad_alloc)
    { return ::operator new(size);}
    static void operator delete(void* pMemory) throw()
    { ::operator delete(pMemory);}
    
    // placement new/delete
    static void* operator new(std::size_t size, void* ptr) throw()
    { return ::operator new(size, ptr);}
    static void operator delete(void* pMemory, void* ptr) throw()
    {return ::operator delete(pMemory, ptr);}
    
    // nothrow new/delete
    static void* operator new(std::size_t size, const std::nothrow_t & nt) throw()
    { return ::operator new(size, nt);}
    static void operator delete(void* pMemory, const std::nothrow_t &) throw()
    {return ::operator delete(pMemory);}
}
~~~

凡是想以自定形式扩充标准形式的客户，可利用继承机制及using声明式取得标准形式:
~~~cpp
class Widget: public StandardNewDeleteForms {
public:
    using StandardNewDeleteForms::operator new;
    using StandardNewDeleteForms::operator delete;
    
    static void* operator new(std::size_t size,
                            std::ostream& logStream) throw(std::bad_alloc);
    static void operator delete(void* pMemory,
                            std::ostream& logStream)
}
~~~
> - 当你写一个placement operator new,请确定也写出了对应的placement operator delete。如果没有这样做，你的程序可能会发生隐微而时断时续内存泄露
> - 当你声明placement new 和 placement delete,请确定不要无意识地遮掩了它们的正常版本。

## 九. 杂项讨论
### 53. 不要轻忽编译器的警告
### 54. 熟悉包括TR1在内的标准程序库
在编译器附带TR1实现品的那一刻到来之前，如果你喜欢以Boost的TR1-like程序库作为一时权益，或许你会愿意以一个命名空间上的小伎俩让自己将来好过些。
~~~cpp
namespace std{
    namespace tr1 = ::boost;
}
~~~
TR1自身只是一份规范。为获得TR1提供的好处，你需要一份实物。一个好的实物来源是Boost
### 55. 让自己熟悉Boost