Hi, all

上次少爷问了个问题，要在c++里使用stl map进行计数，能不能这样写：

    std::map<std::string, int> dict;
    for (int i = 0; i < tokens.size(); ++i)
    {
        const std::string &token = tokens[i];  
        ++(dict[token]);
    }

还是说必须这样写：

    std::map<std::string, int> dict;
    for (int i = 0; i < tokens.size(); ++i)
    {
        const std::string &token = tokens[i];
		std::map<std::string, int>::iterator it = dict.find(token);
		if (it == dict.end())
		    dict[token] = 1;
		else
		    ++(it->second);
    }
    
    
看stl map关于operator[]的描述：

    Allows for easy lookup with the subscript ( @c [] ) operator. Returns data associated with the key specified in subscript.  If the key does not exist, a pair with that key is created using default values, which is then returned.

所以当传给std::map operator[]的key不存在的时候，会插入一个默认值。

      mapped_type& operator[](const key_type& __k)
      {   
          // concept requirements
          __glibcxx_function_requires(_DefaultConstructibleConcept<mapped_type>)
          iterator __i = lower_bound(__k);
          // __i->first is greater than or equivalent to __k.
          if (__i == end() || key_comp()(__k, (*__i).first))
              __i = insert(__i, value_type(__k, mapped_type()));
          return (*__i).second;
      }   

这里的mapped_type就是std::map的第二个模板参数，对应到我们的程序里就是int了。可以看到这里当__k找不到时，会调用`insert(__i, value_type(__k, mapped_type()))`来插入一条记录。这下有意思了，把这里的mapped_type换成int：`insert(__i, value_type(__k, int()))`，这会是什么样的效果呢？接下来我们就来看下int的构造函数int()以及一些其它类型的构造函数产生的效果：

    #include <stdio.h>
    #include <string.h>
    
    #define PRINTD(x) fprintf(stdout, "int    "#x" = 0x%08X = %d\n", (x), (x))
    #define PRINTF(x) fprintf(stdout, "float  "#x" = 0x%08X = %f\n", (*((int *)&(x))), (x))
    #define PRINTS(x) fprintf(stdout, "struct "#x" = 0x%08X = %d\n", (x).m, (x).m)
    
    struct StructUseDefaultConstructor
    {
    public:
        int m;
    };
    
    struct StructWithConstructor
    {
    public:
        StructWithConstructor() : m(2) { }
    public:
        int m;
    };
    
    void func1()
    {
        static const int STACK = 100;
        int a[STACK];
        memset(a, 7, sizeof(int)*STACK);
    }
    
    void func2()
    {
        int a;
        int b = int();
        int c;
        float e;
        float f = float();
        float g;
        StructUseDefaultConstructor i;
        StructUseDefaultConstructor j = StructUseDefaultConstructor();
        StructUseDefaultConstructor k;
        StructWithConstructor l;
        StructWithConstructor m = StructWithConstructor();
        StructWithConstructor n;
        PRINTD(a);
        PRINTD(b);
        PRINTD(c);
        PRINTF(e);
        PRINTF(f);
        PRINTF(g);
        PRINTS(i);
        PRINTS(j);
        PRINTS(k);
        PRINTS(l);
        PRINTS(m);
        PRINTS(n);
    }
    
    int main(int argc, char *argv[])
    {
        func1(); // call func1 to set a block of stack memory with flag
        func2();
        return 0;
    }

再运行这段代码前，你能预料到它的输出结果么？

    int    a = 0x07070707 = 117901063
    int    b = 0x00000000 = 0
    int    c = 0x07070707 = 117901063
    float  e = 0x07070707 = 0.000000
    float  f = 0x00000000 = 0.000000
    float  g = 0x07070707 = 0.000000
    struct i = 0x07070707 = 117901063
    struct j = 0x00000000 = 0
    struct k = 0x07070707 = 117901063
    struct l = 0x00000002 = 2
    struct m = 0x00000002 = 2
    struct n = 0x00000002 = 2

所有带=号的，值都被初始化成0了，而不带=号的除了StructWithConstructor外，都没有初始化，值是我们在func1里设置的每一个byte都是0x07。

结论：

1. int/float等基本类型也是有构造函数的，而且构造函数会对其进行zero-initialization
2. `int num;` 与 `int num = int()`是不一样的，前者num值是随机的，后者num值会被构造函数初始化为0
