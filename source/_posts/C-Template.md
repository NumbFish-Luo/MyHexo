---
title: 用C语言实现模板类的一些个人思路
date: 2021-10-6 11:00:00
toc: true
tags:
- C
categories:
- C
banner_img: /img/C.jpg
banner_img_set: /img/C.jpg
---

## 0. FAQ

- **Q: 为什么要用模板？**
- A: 为了解决函数重载问题。例如，在C++中，我们要比较两个int型变量的哪个大，并返回其中较大的值，可能会写这样的函数：

```cpp
int Max(int a, int b) { return a > b ? a : b; }
int main() {
    printf("%d\n", Max(123, 456));
    return 0;
}
```

但是，如果我们还要比较更多的类型，例如char型，double型之类的，我们都需要重新写一个类似的实现，这很不方便：

```cpp
char Max(char a, char b) { return a > b ? a : b; }
double Max(double a, double b) { return a > b ? a : b; }
int main() {
    printf("%c, %lf\n", Max('a', 'z'), Max(1.1, 2.2));
    return 0;
}
```

这个时候我们可以考虑使用模板来简化代码：

```cpp
template <typename T>
T Max(T a, T b) { return a > b ? a : b; }
int main() {
    printf("%d, %c, %lf\n", Max(123, 456), Max('a', 'z'), Max(1.1, 2.2)); 
    return 0;
}
```

- **Q: C能重载？**
- A: 不能，C只能像这样子：

```c
int Max_int(int a, int b) { return a > b ? a : b; }
char Max_char(char a, char b) { return a > b ? a : b; }
double Max_double(double a, double b) { return a > b ? a : b; }
int main() {
    printf("%d, %c, %lf\n", Max_int(123, 456), Max_char('a', 'z'), Max_double(1.1, 2.2));
    return 0;
}
```

- **Q: 道理我都懂，有C++了，为什么还要用C实现模板？**
- A: 虽然一般情况下都能直接写C++来代替C程序啦，不过确实存在无法使用C++的场合。很不幸，我（曾经的）工作所用的语言只能是C，于是，之前用惯了C++的我决定探索一下如何用C实现模板- -+
- **Q: 这篇文章我好像在哪里见过？**
- A: 这篇文章原本是上传到了CSDN上，现在拿来放到自己的个人网站。

## 1. 目前个人写法风格

### 1.0 模板函数

首先，用前面的Max函数为例，先看一下我目前的写法风格吧:-P

**需要注意的是：为防止宏嵌套出错，例如`_Max(Pair(int, char))`会被拓展成`Max$_Pair(int, char)_$`这样的不合法变量名（Pair的定义见1.1模板类示例），我们都需要多写一个类似于`#define Max(T) _Max(T)`这样的操作**

```c
#define _Max(T) Max$_##T##_$
#define Max(T) _Max(T)

#define _Max_IMPL(T) T Max(T)(T a, T b) { return a > b ? a : b; }
#define Max_IMPL(T) _Max_IMPL(T)

Max_IMPL(int);
Max_IMPL(char);
Max_IMPL(double);

int main() {
    printf("%d, %c, %lf\n", Max(int)(123, 456), Max(char)('a', 'z'), Max(double)(1.1, 2.2));
    return 0;
}
```

下面解释一下上面的代码到底做了什么：

- **用宏来模仿C++的写法 —— `#define _Max(T) Max$_##T##_$`**
  这里用了`Max(T)`宏来模仿C++的`Max<T>`写法。

  例如：`Max(int)`，宏拓展后实际上等于`Max$_int_$`（`$`符号确实是合法变量名），这里用`$_`表示C++中模板的尖括号左侧`<`，用`_$`表示尖括号右侧`>`

- **模板实现 —— `#define _Max_IMPL(T) T Max(T a, T b) { return a > b ? a : b; }`**
  相当于C++的`template <typename T> T Max(T a, T b) { return a > b ? a : b; }`

- **实例化模板 —— `Max_IMPL(int);`**
  由于C不是C++，所以需要使用者自己去实例化对应的模板类型，此处宏拓展出来的代码为`int Max(int)(int a, int b) { return a > b ? a : b; }`，然后`Max(int)`再拓展为`Max$_int_$`，于是最终结果为：
  **`int Max$_int_$(int a, int b) { return a > b ? a : b; }`**

- **使用 —— `Max(int)(123, 456)`**
  相当于C++的`Max<int>(123, 456)`

像这样，我们就实现了一个`T Max<T>(T, T)`模板函数了！在此基础上，我们可以更进一步：如果能做一个**检测代码**中Max使用了哪些**模板类型**，然后在**编译前**自动添加实现Max_IMPL(…)完成“实例化”的程序，就可以完成类似于C++的模板类型推断功能了- -+

### 1.1 模板类

模板类同理，我们以简单的Pair为例（保存两个类型数据的结构）

```c
#define _Pair(T1, T2) Pair$_##T1##_$$_##T2##_$
#define Pair(T1, T2) _Pair(T1, T2)

#define _Pair_IMPL(T1, T2) typedef struct { T1 t1; T2 t2; } Pair(T1, T2)
#define Pair_IMPL(T1, T2) _Pair_DEF(T1, T2)

Pair_IMPL(int, char);
Pair_IMPL(int, double);

int main() {
    Pair(int, char) p_ic;
    Pair(int, double) p_id;
    p_ic.t1 = 123;
    p_ic.t2 = 'a';
    p_id.t1 = 456;
    p_id.t2 = 1.1;
    return 0;
}
```

- **`#define _Pair(T1, T2) Pair$_##T1##_$$_##T2##_$`**
  用`Pair(T1, T2)`模仿C++中`Pair<T1, T2>`的写法
- **`#define _Pair_IMPL(T1, T2) typedef struct Pair(T1, T2) { T1 t1; T2 t2; } Pair(T1, T2)`**
  相当于`template<typename T1, typename T2> struct Pair{ T1 t1; T2 t2; }`
- **`Pair_IMPL(int, char);`**
  实例化`Pair<int, char>`，此处宏拓展的最终结果为：
  `typedef struct { int t1; char t2; } Pair$_int_$$_char_$`
- **`Pair(int, char) p_ic;`**
  相当于`Pair<int, char> p_ic;`

像这样，我们就完成了简单的`Pair<T1, T2>`模板类了

---

参考资料：C语言宏高级用法 https://www.cnblogs.com/alantu2018/p/8465911.html