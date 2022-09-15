# 1. C/C++内存分布
## 1.1 回顾
首先要知道C/C++程序内存区域的划分：<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/28510338/1663118815847-def7fc73-9151-4b76-b8fe-7956643f2cc9.png)<br />图片来源：[https://manybutfinite.com/post/anatomy-of-a-program-in-memory/](https://manybutfinite.com/post/anatomy-of-a-program-in-memory/)<br />【注意】

- 栈：从高地址往低地址增长，存放的是非静态局部变量、函数参数以及返回值等具有**临时性**的值；
- 内存映射段：装载共享动态内存库，用户可使用系统接口创建共享内存，用作进程间通信，是高效的I/O映射方式。这一部分后续会学习；
- 堆：从低地址往高地址增长，用于程序运行时动态内存的分配，说白了：动态内存的维护都是要用指针保存地址的，堆就是保存指针的地方，因此堆的容量很小，比如VS编译器给堆分配的大小是1M，一些LInux中能达到8M；
- 数据段：存放着全局数据和静态数据，这部分和栈对应，是存放着具有**常性**的值；
- 代码段：可执行的代码、只读常量。
> 关于堆和栈的地址增长方向不同的原因，众说纷纭，读者可自行查阅资料，我给出我认为比较合理的：
> - 堆维护动态内存，在物理世界中存放东西由下到上是符合我们认知的；而栈只需要入栈和出栈操作，就像我们从一摞书里抽出一本书一样，无需从具体的一边操作。

这是我们在先前学习C/C++时已经了解过的，本节内容第一部分是复习，二是学习C++中新的内存开辟方式。
## 1.2 练习
```cpp
int globalVar = 1;
static int staticGlobalVar = 1;

void Test()
{
    static int staticVar = 1;
    int localVar = 1;
    int num1[10] = { 1, 2, 3, 4 };
    char char2[] = "abcd";
    const char* pChar3 = "abcd";
        
    int* ptr1 = (int*)malloc(sizeof(int) * 4); 
    int* ptr2 = (int*)calloc(4, sizeof(int)); 
    int* ptr3 = (int*)realloc(ptr2, sizeof(int) * 4);
    free(ptr3);
    free(ptr1);
}
```
【选择题】<br />选项: A.栈 B.堆 C.数据段(静态区) D.代码段(常量区) 
```cpp
globalVar在____? staticGlobalVar在____? 
staticVar在____? localVar在____?
num1 在____?
char2在____?*char2在____？
pChar3在____?*pChar3在____?
ptr1在____?*ptr1在____?
```
<br />【填空题】
```cpp
//32位平台
sizeof(num1) = ____;
sizeof(char2) = ____;      strlen(char2) = ____;
sizeof(pChar3) = ____;     strlen(pChar3) = ____;
sizeof(ptr1) = ____;
```
【答案】

1. CCCAA AAADAB
1. 40 5 4 4 4 4
# 2. C++内存管理
C++兼容C，所以C语言中的malloc、realloc等内存管理函数都可以在C++中使用，但在处理更复杂的场景时，原来的内存管理方式就显得捉襟见肘，C++提出了新的内存管理方式：定义新的**操作符new和delete**进行动态内存管理。<br />先说结论：new和delete对于内置类型和malloc等C语言内存管理函数功能上没什么区别，只是用法简化了。而它们更大的作用是处理自定义类型对象的内存管理。
## 2.1 对于内置类型
下面通过三个例子了解new和delete的使用方法：
```cpp
// 动态申请一个int类型的空间
int* ptr1 = new int;

// 动态申请一个int类型的空间并初始化为10
int* ptr2 = new int(10);

// 动态申请10个int类型的空间
int* ptr3 = new int[10];

delete ptr1;
delete ptr2;
delete[] ptr3;
```
【注意】

- ptr1指向的是一个动态内存分配的、未初始化的无名对象（有的编译器可能会初始化为0）；
- ptr2指向的是一个值为10，类型是int的对象；
- ptr3指向的是一个大小为10个int，也就是40（32位）个字节、未初始化的对象；
- delete：如果有申请指定大小的空间，需要使用[]。

对于内置类型，new和delete的使用比使用malloc函数和free一样，只是形式上不同。
## 2.2 对于自定义类型
对于自定义类型，new和delete最大的不同就是它们会分别调用对象的构造函数和析构函数。
```cpp
class A
{
public:
    A()
    {
        cout << "析构函数" << this <<endl;
    }
    ~A()
    {
        cout << "构造函数" << this << endl;
    }


private:
    int _a;
};
int main()
{
    A *ptr = new A;
    delete ptr;
    return 0;
}
```
结果：
> 析构函数0x60000086c010
> 构造函数0x60000086c010

new和delete就是为了自定义类型对象而准备的，new会调用构造函数构造，delete会调用析构函数清理，new出来的地址存放在堆区。<br />【注意】<br />在C语言中，我们常常会检查malloc是否开辟成功，以返回值是否为NULL为判断依据。new也一样，只不过它有新的机制：异常机制。<br />异常机制是语言内置的，不需要用户自己另外根据返回值判断是否成功，避免了忘记检查的情况。
# 3. operator new、operator delete函数
## 3.1 原理
new和operator new就像老板和员工的区别，前者是运算符，后者是函数，当用户要new一个对象出来时，new会调用operator new函数；delete也一样。<br />值得注意的是，它们都是全局函数，而且它们的底层都是用malloc和free函数来开辟和释放空间，这就是上文中提到对于内置类型它们只是使用方法不同的原因。
```cpp
/*
operator new:该函数实际通过malloc来申请空间，
当malloc申请空间成功时直接返回;
申请空间失败，尝试执行空间不足应对措施，
如果改应对措施用户设置了，则继续申请，否则抛异常。
*/
void *__CRTDECL operator new(size_t size) _THROW1(_STD bad_alloc)
{
 // try to allocate size bytes
 void *p;
 while ((p = malloc(size)) == 0)
      if (_callnewh(size) == 0)
     {
         // report no memory
// 如果申请内存失败了，这里会抛出bad_alloc 类型异常
         static const std::bad_alloc nomem;
         _RAISE(nomem);
}
return (p);
}
```
```cpp
/*
operator delete: 该函数最终是通过free来释放空间的
*/
void operator delete(void *pUserData)
{
    _CrtMemBlockHeader * pHead;
    RTCCALLBACK(_RTC_Free_hook, (pUserData, 0));
    if (pUserData == NULL)
    _mlock(_HEAP_LOCK); /* block other threads */
    return;
    __TRY
    /* get a pointer to memory block header */
    pHead = pHdr(pUserData); _ASSERTE(_BLOCK_TYPE_IS_VALID(pHead->nBlockUse));
    /* verify block type */
    __FINALLY
    _free_dbg( pUserData, pHead->nBlockUse );  
        __END_TRY_FINALLY
    _munlock(_HEAP_LOCK); /* release other threads */ 
    return;
}
```
```cpp
/*
free:
*/
#define   free(p)
_free_dbg(p, _NORMAL_BLOCK)
```
	通过查看源代码，可以了解两个全局函数的底层实现。
# 4. new和delete的实现原理
## 4.1 内置类型
对于内置类型，new和malloc、delete和free不同之处：new和delete申请和释放的是单个元素的空间，new[]和delete[]申请的是连续的空间，new申请失败会抛异常。
## 4.2 自定义类型

1. new：
   - 调用operator new函数申请空间；
   - 在申请的空间上执行构造函数，完成对象的构造。
2. delete：
   - 在空间上执行析构函数，完成对象中资源的清理；
   - 调用operator delete函数释放对象的内存空间。
3. new T[N]：
   - 调用operator new[]函数，然后在这个函数中调用operator new函数，对连续的空间为多个对象进行N次申请；
   - 在申请的空间上执行N次构造函数。
4. delete[]：
   - 在释放的对象空间上执行N次构造函数，完成N个对象中资源的清理；
   - 调用operator delete[]函数释放空间，在这个函数中再调用operator delete函数释放空间。
# 5. 总结
# 5.1 malloc/free和new/delete的异同
### 5.1.1 共同点
它们都是在堆上开辟或释放的内存空间，而且都需要用户手动操作。
### 5.1.2 不同点

- malloc和free是函数，new和delete是操作符；
- malloc申请的空间不能初始化，而new可以；
- malloc申请空间时需要手动使用sizeof计算空间，而new不用，只需要跟上对象的类型即可，个数也只需要用`[]`添加；
- malloc的返回值是void*，接收指针需要强转，new不需要，因为已经指定了对象的类型；
- malloc失败必须通过用户自己检查返回值的值，而new失败不需要（不过要捕获异常）；
- 对于内置类型和自定义类型的区别。
## 5.2 内存泄漏
### 5.2.1 内存泄漏是什么
这里的内存是通常所说的运行内存。<br />简单地说，内存被程序正常使用是通常情况，但是如果内存被分配给了某个程序，却拿不回来，这一块内存相对于之前的内存就是“泄漏”了。<br />如果一个程序有内存泄漏的问题，那么它停止后会将内存自动返还给系统，但是像操作系统、后台服务，基本上一旦运行起来就不太可能关闭，那么内存泄漏的问题就会一直存在，最后会导致机器响应迟钝，最终卡死。
### 5.2.2 如何规避
和C语言中的malloc和free一样，一旦有new，就必须有delete，它们是成对使用的。 
