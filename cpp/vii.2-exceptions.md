# VII.2 Exceptions



### 7.6 Exceptions specifications and noexcept keyword

```cpp
int f(int x, int y) noexcept { // эта функция не кидает исключений (ну типа)
    if (y == 0)
           throw 1;
    return x / y;
}

int f(int x) {
    return f(1, x)a
}
int main() {
    f(1,0); // будет RE, компилятор это не отслеживает
    // но warning будет
}
```

```cpp
struct S {
    int f(int x, int y) {
        if (y == 0)
            throw 1;
        return x / y;
    }
};

struct F {
    int f(int x, int y) noexcept {
        return x * y;
    }
};


// условный noexcept
template <typename T>
int f(int x, const T& y) noexcept(noexcept(x.f(1,0))){
    return x.f(1, 0);
    // noexcept1 - specifier, noexcept2 - operator, return 
    // False, если вызываем функции с: throw, new, dynamic_cast, noexcept function call
} // f мб noexcept и не noexcept

int main() {
    std::cout << noexcept(1 / 0); // не вычисляется, noexcept проверяет
    // содержит ли выражение noexcept вызовы
}
```

noexcept - обещание, что метод не завершается плохо. Писать или не писать noexcept - ваш выбор.

### 7.7 Exceptions in destructors

```cpp
struct Dangerous {
    int x = 0;
    Dangerous(int x): x(x) {}
    
    ~Dangerous() {
        if (x == 0)
            throw 1;
    }
};

void g() {
    Dangerous dang(0);
    std::cout << dang.x;
}


void f() {
    Dangerous dang(0);
    std::cout << dang.x;
    g();
 // exception1 on delete dang from f
 // second exception2 on delete dang from g
 // бросить исключение в момент, когда летит исключение
}

int main() {
    try {
        f();
    } catch (...) {
        std::cout << "caught\n";
    }
}
```

&gt;&gt;&gt; terminate

Язык не поддерживает несколько исключений одновременно, когда во время обработки одной ошибки вылетает другая.

При выходе из деструктора будут вызываться другие деструкторы, которые вызовут еще исключения и программа упадет с ошибкой.

Исключения в деструкторах гораздо хуже исключений в конструкторах. Никогда так не нужно делать.

_since cpp11 all destructors are noexcept_

Если **вы хотите взломать кому-то ... эээ... пк,** и хочется бросить исключение из деструктора, нужно:

```cpp
struct Dangerous {
    int x = 0;
    Dangerous(int x): x(x) {}
    
    ~Dangerous() noexcept(false) {
        if (x == 0)
            throw 1;
    }
};
```

### 7.8 Exceptions and safety guarantee

throw / assert &lt;тут что-нибудь написать&gt;

Проблемы из-за исключений: усложняют написание библиотек:

* нужно помнить про noexcept
* пример с вектором:

```cpp
struct S {
    int x = 0;
    s(int x): x(x) {}
    
    S(const S& x): x(s.x) { // copy constructor
        if (x == 8)
            throw 1;
    }
};
    
int main() {
    std::vector<S> v;
    for (int i = 0; i < 100; ++i) {
        v.push_back(S(i));
    }
}
```

Что делать вектору при расширении памяти? На 8-м шаге он начнет перекладывать объекты в новое место, если кто-то вызовет исключение - объекты переложить не получится.

![](../.gitbook/assets/image%20%283%29.png)

&lt; todo &gt;

Вектор, если будет исключение, вернется к старому состоянию.

push\_back - строгая гарантия безопастности \(если деструкторы не бросают исключения\)

Basic exceptions guarantee and Strong exceptions gaurantee

базовая - гарантия того, что объект останется в валидном состоянии, даже если будет исключение \(не будет UB и полудостроенных состояний\), сильная - объект останется в исходном состоянии





