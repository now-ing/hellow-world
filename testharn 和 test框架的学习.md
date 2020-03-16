

## 1.  参考https://blog.csdn.net/yang_chen_shi_wo/article/details/46341553 

匿名命名空间
当定义一个命名空间时，可以忽略这个命名空间的名称：

    namespce {
    
        char c;
    
        int i;
    
        double d;
    
    }
    
     编译器在内部会为这个命名空间生成一个唯一的名字，而且还会为这个匿名的命名空间生成一条using指令。所以上面的代码在效果上等同于：
    
    namespace __UNIQUE_NAME_ {
    
        char c;
    
        int i;
    
        double d;
    
    }
    
    using namespace __UNIQUE_NAME_;

 


     在匿名命名空间中声明的名称也将被编译器转换，与编译器为这个匿名命名空间生成的唯一内部名称(即这里的__UNIQUE_NAME_)绑定在一起。还有一点很重要，就是这些名称具有internal链接属性，这和声明为static的全局名称的链接属性是相同的，即名称的作用域被限制在当前文件中，无法通过在另外的文件中使用extern声明来进行链接。如果不提倡使用全局static声明一个名称拥有internal链接属性，则匿名命名空间可以作为一种更好的达到相同效果的方法。

注意:命名空间都是具有external连接属性的,只是匿名的命名空间产生的__UNIQUE_NAME__在别的文件中无法得到,这个唯一的名字是不可见的.

C++ 新的标准中提倡使用匿名命名空间,而不推荐使用static,因为static用在不同的地方,涵义不同,容易造成混淆.另外,static不能修饰class

 

因为using namespace __UNIQUE_NAME_;声明

所以在此文件中这些匿名命名空间中的所有东西都是可见的。

————————————————
版权声明：本文为CSDN博主「yang_chen_shi_wo」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/yang_chen_shi_wo/article/details/46341553











``` C++
::Tester("arena_test.cc",59).Is(c,"hahahaha"); 

::Tester("arena_test.cc", 59).Is(c,"hahaha"); 

::Tester("arena_test.cc", 59).Is(c,"haha");


```






这种调用方式仅仅是申请了一个无名的临时的类实例。有存储空间但是没有名字。多个重复的相同的无名变量有不同的内存空间。上面这三个是在不同的内存空间中，通过debug进行分析，先是执行初始化函数，然后才执行is函数。也就是初始化了三个匿名变量。



————————————————
版权声明：本文为CSDN博主「yang_chen_shi_wo」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/yang_chen_shi_wo/article/details/46341553