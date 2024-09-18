**最近工作中遇到一个访问无效指针类似0xf,看起来非常诡异的指针，而且打印是还会出现乱码，甚至直接crash。深入debug多半天才找出根本原因。
问题的本质简化后基本如下：**

```cpp []
tony@LAPTOP-5N7QA0JM:~/exapmle$ cat use_temp_obj_after_release_crash.cpp
#include <bits/stdc++.h>
using namespace std;
class GlobalClass {
    public:
    string type = "globalType";
};

GlobalClass gc;

class UserClass {
    public:
    string* user = nullptr;
};

UserClass uc;

int foo(void) {
    auto print = [&](GlobalClass localC) {
        uc.user = &localC.type;
        cout << localC.type << endl;
        cout << *uc.user << endl;
    };

    print(gc);
    cout << *uc.user << endl;
    return 0;
}

int main(void) {
    foo();
    cout << "Print some unrelated strings" << endl;
    cout << *uc.user << endl;
    return 0;
}
```

**在linux环境下编译并执行，会出现乱码**
```sh
tony@LAPTOP-5N7QA0JM:~/exapmle$ g++ use_temp_obj_after_release_crash.cpp -o bad.exe
tony@LAPTOP-5N7QA0JM:~/exapmle$ ./bad.exe
globalType
globalType
globalType
Print some unrelated strings
 �����������U�������¥���ץ��������������"����/����P����?����Y����{�������������������Ĭ���̬���쬎������5����b��������������Я���ܯ���!Е��3����d@p��U8
             �

```
**根本原因分析**

 问题的根本原因就在于23行的print lambda函数在传递对象时，会调用C++的默认copy构造函数，copy创建一份local的 GlobalClass类的对象，该对象只在lambda函数内生存，当出了该范围之后，就会被删除释放。
 如果按照bad的下发，uc.user指向的是临时对象的type，在lambda内部是仍然有效，所以正常打印，出了print范围打印就无效了（本测试中仍然正常打印，猜测有可能localC推迟到了 foo最后的右大括号才释放），无论如何，
 当我们代码出了foo之后，再打印就是乱码了，因为localC已经不存在了。我们的uc.user还指向被释放对象的无效指针，这是打印的内容就是随机的了。根据前面执行的内存中剩余的东西而定。

 明白了问题的根源之后，修复也就非常简单了，只需要传递原来全局对象变量的指针，这样uc.user指向全局变量的type，直到程序结尾都是有效的。


**正确代码:**
```cpp []
tony@LAPTOP-5N7QA0JM:~/exapmle$ cat use_temp_obj_after_release_crash_fixed.cpp
#include <bits/stdc++.h>
using namespace std;
class GlobalClass {
    public:
    string type = "globalType";
};

GlobalClass gc;

class UserClass {
    public:
    string* user = nullptr;
};

UserClass uc;

int foo(void) {
    auto print = [&](GlobalClass* localC) {
        uc.user = &localC->type;
        cout << localC->type << endl;
        cout << *uc.user << endl;
    };

    print(&gc);
    cout << *uc.user << endl;
    return 0;
}

int main(void) {
    foo();
    cout << "Print some unrelated strings" << endl;
    cout << *uc.user << endl;
    return 0;
}
```

**正确的执行结果**
```
tony@LAPTOP-5N7QA0JM:~/exapmle$ g++ use_temp_obj_after_release_crash_fixed.cpp -o good.exe
tony@LAPTOP-5N7QA0JM:~/exapmle$ ./good.exe
globalType
globalType
globalType
Print some unrelated strings
globalType
```

**吃一堑长一智，下回不要轻易传递对象copy咯，也记住访问已经释放的无效指针可能出现乱码，甚至导致程序崩溃。**
