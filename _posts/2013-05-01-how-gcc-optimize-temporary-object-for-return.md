---
layout: post
title: "how gcc optimize temporary object for return"
description: ""
category: cpp
tags: [c++, gcc, 编程]
---
{% include JB/setup %}

直接上代码：<br />

    /* FILE: foo.cpp */
    
    #include <iostream>
    
    using namespace std;
    
    struct Object
    {
        Object()
        {   
            fprintf(stderr, "Construct %x\n", this);
        }
        Object(const Object &obj)
        {   
            a = obj.a;
            b = obj.b;
            fprintf(stderr, "Copy Construct %x\n", this);
        }
        Object &operator=(const Object &obj)
        {   
            a = obj.a;
            b = obj.b;
            fprintf(stderr, "operator= %x\n", this);
        }
        virtual ~Object()
        {   
            fprintf(stderr, "Desctruct %x\n", this);
        }
    
        int32_t a;
        int32_t b;
        int32_t c[10];
    };
    
    Object func()
    {
        Object obj;
        obj.a = 1;
        obj.b = 2;
        return obj;
    }   
    
    int main(int argc, const char *argv[])
    {
        Object obj = func();
        std::cout << "a=" << obj.a << " b=" << obj.b << std::endl;
        return 0;
    }


关于这段代码，现在问你一个经典面试题：这个程序编译后输出的结果是什么样的？Object的构造函数、拷贝构造函数、赋值重载函数、析构函数分别调用了多少次？<br />
如果你在学校认真听课了，看了一些面试宝典，那么你会回答说：so easy! 构造函数调用1次，拷贝构造函数调用2次，赋值重载函数调用0次、析构函数调用3次。运行结果如下： <br />

    -bash$ ./foo
    Construct 31ddd1d0
    Copy Construct 31ddd2a0
    Desctruct 31ddd1d0
    Copy Construct 31ddd260
    Desctruct 31ddd2a0
    a=1 b=2
    Desctruct 31ddd260

过程分析：func函数里在栈上生成了一个对象obj，这里调用了一次构造函数。当return一个栈上的对象时，需要将obj拷贝到一个临时对象里，这里调用了一次拷贝构造函数。函数调用返回时，func函数里的obj对象会随着作用域的结束而析构掉，这里调用了一次析构函数。回到main函数里，func函数返回的临时对象赋值给main函数的obj，这里调用了一次拷贝构造函数，然后临时对象使命结束，调用一次析构函数。main函数结束时，main函数里的obj对象析构，调用一次析构函数。 <br />
分析的头头是道，有理有据，好像还真是那么回事，教科书和面试宝典里也这么说的。<br />
那么我们来测试一把：

    -bash$ g++ -O0 -fno-elide-constructors -o foo foo.cpp
    -bash$ ./foo
    Construct 31ddd1d0
    Copy Construct 31ddd2a0
    Desctruct 31ddd1d0
    Copy Construct 31ddd260
    Desctruct 31ddd2a0
    a=1 b=2
    Desctruct 31ddd260

结果还真跟想像的一样。那就真是这样了么？好像哪里有一点点不对劲，哦，g++的参数有点奇怪。-O0这个我知道，是取消编译器的优化选项，大家一般都用-O2的，让编译器做更多的事，不过在测试时为了更好的验证我们的问题，我们先把它关掉。另外一个参数，-fno-elide-constructors，这是个什么参数呢？还真没听说过，不管了，先man下再说：<br />

    -bash$ man g++
           -fno-elide-constructors
               The C++ standard allows an implementation to omit creating a temporary which is only used
               to initialize another object of the same type.  Specifying this option disables that opti-
               mization, and forces G++ to call the copy constructor in all cases.

先不管这个-fno-elide-constructors选项是什么意思，我们把这个选项去掉再试试：<br />

    -bash$ g++ -O0 -o foo foo.cpp
    -bash$ ./foo
    Construct f15aa40
    a=1 b=2
    Desctruct f15aa40

不对吧，结果怎么是这样的呢？天啊，编译器脑抽了吧？！[围观] <br />
我都-O0指定不优化了啊，为什么只构造了一次Object对象呢？[思考] <br />
为什么跟之前的结果不一样呢？哦，肯定是那个叫-fno-elide-constructors的选项搞的鬼。仔细看看man里边的那段话是什么意思去：<br />
    c++标准允许实现版本不创建临时对象：如果这个临时对象只是用来对另外一个相同类型的对象做初始化。指定这个选项将禁止这个优化，强制g++在所有情况下都调用拷贝构造函数

翻译的真别扭，分析上面的实例吧：上面例子里func函数里的obj对象，它其实什么活都没干，只是用来做返回，而且返回值的类型也是Object，那么它的构造函数就可以忽略了。同样func函数的返回值，也只是为了给main函数里的obj对象赋值的，这里的构造函数也可以忽略了。说到底，从头到尾只需要构造一个Object对象：那就是main函数里的obj。main函数把这个obj的地址传给func函数，在func函数里直接给操作这个对象。 <br />

好像明白了，也就是说gcc真不厚道，都不告诉我就做了优化，优化掉了2次对象的拷贝。<br />

好吧，这个故事到这儿好像就差不多了。<br />
再等等，我好像又想起一点东西了，上面的main函数里调用func函数的地方改为：

    const Object &obj = func();

如果用一个const的引用去接收func函数的返回值，这样会不会更好呢？貌似能减少一次对象的拷贝：在加了-fno-elide-constructors选项的情况下，能将避免main函数里obj对象的构造，它直接引用了函数返回值的临时对象。这样做靠谱么？多说无益，测试一把吧：

    -bash$ g++ -O0 -fno-elide-constructors -o foo foo.cpp
    -bash$ ./foo
    Construct 8c319c90
    Copy Construct 8c319d20
    Desctruct 8c319c90
    a=1 b=2
    Desctruct 8c319d20

哇噻，还真是这样！！！少了一次对象拷贝！<br />
这里能不能不用const引用，直接这样用：

    Object &obj = func();

编译下试试。报错了：<br />

    error: invalid initialization of non-const reference of type ‘Object&’ from a temporary of type ‘Object’

哦，编译器告诉我说这个临时对象是不能用非const的引用去引用它的！<br /><br />
那这么用const引用去引用临时对象靠不靠谱呢？func函数返回值的临时对象看起来是在main函数的栈里，在cout语句运行之后才析构的。<br />
那是不是说我就可以用一个`const Object \*`的指针去获取这个临时对象的地址呢？有意思，再修改下main函数：

    const Object *obj = &(func());

编译下试试。Oh, 好像有点问题：<br />

    warning: taking address of temporary

编译器警告我说：你去取一个临时对象的地址，这种行为是不对的！管它呢，就警告而已，忽略，直接运行

    -bash$ ./foo
    Construct 95593a70
    Copy Construct 95593ae0
    Desctruct 95593a70
    Desctruct 95593ae0
    a=1 b=2

好像结果也是对的。不对，好像有点问题：为什么所有的对象都被析构掉了后，才输出了a和b的值呢？这个时候对象都没了，哪儿还有a和b啊！<br />
那a、b的结果是怎么来的呢？哦，对象虽然析构了，但是指向这个对象的指针还在，这块内存也还在的，你非要去访问它，操作系统也拿你没办法。<br />
不过如果这个时候有新的函数调用，生成了新的函数调用栈，或者其它的代码覆盖了原来func函数的栈，原来那块内存的内容可就不定是多少了，那cout出的结果也就不一定还是1和2了。<br />
<br />
到这里好像有点头绪了：函数返回的临时对象，可以用const的引用去引用它，这是gcc做了特殊处理，这个时候临时对象会在引用的变量作用域结束时析构。但是临时对象不能去取地址，因为如果你去取了一个临时对象的地址，这个指针就可以随意传到任何地方去使用，这样就太危险了，所以gcc会警告你说这种行为是不对的，而且不会对这种情况做任何特殊处理，函数调用结束后临时对象马上就析构了。<br />
那么用const的引用去引用函数返回的临时对象，引用的这个变量的作用域只在当前函数范围内，有没有办法把它往外层函数继续传呢？达到跟取地址一样的效果？<br />
我们可以写个func1的函数，在这个函数里直接调func并return func的返回值。再来一段测试代码：

    /* FILE: foo.cpp */
    
    #include <iostream>
    
    using namespace std;
    
    struct Object
    {
        Object()
        {   
            fprintf(stderr, "Construct %x\n", this);
        }
        Object(const Object &obj)
        {   
            a = obj.a;
            b = obj.b;
            fprintf(stderr, "Copy Construct %x\n", this);
        }
        Object &operator=(const Object &obj)
        {   
            a = obj.a;
            b = obj.b;
            fprintf(stderr, "operator= %x\n", this);
        }
        virtual ~Object()
        {   
            fprintf(stderr, "Desctruct %x\n", this);
        }
    
        int32_t a;
        int32_t b;
        int32_t c[10];
    };
    
    Object func()
    {
        Object obj;
        obj.a = 1;
        obj.b = 2;
        return obj;
    }   
    
    const Object &func1()
    {
        const Object &obj = func();
        return obj;
    }
    
    int main(int argc, const char *argv[])
    {
        const Object &obj = func1();
        std::cout << "a=" << obj.a << " b=" << obj.b << std::endl;
        return 0;
    }


这段测试代码跟前面不一样的地方在于多了一个func1函数，func1函数返回了func函数的返回值。<br />

    -bash$ g++ -O0 -fno-elide-constructors -o foo foo.cpp
    -bash$ ./foo
    Construct d8d67210
    Copy Construct d8d67270
    Desctruct d8d67210
    Desctruct d8d67270
    a=1 b=2

这下清楚了，输出的结果很明显是不正确的，在cout前，临时对象已经被析构掉了。<br />
<br />
<br />
由此，我们可以得出结论：<br />
重点来了：<br />

- 书上/面试宝典上说的函数返回对象时的2次对象拷贝，你根本不用去理它，编译器会帮你优化掉的，相信编译器！
- 别想着用引用去接受函数返回值减少对象拷贝，不靠谱！我们测试的最后一个例子就说明了，引用函数返回的对象，是依赖于编译器的实现的。
