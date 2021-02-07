# VII.1 Exceptions

## 7.1 Basic idea

В runtime может пойти что-то не так. Как это происходит? Выполняется недопустимая операция, программа вылетает с ошибкой \(exceptions\). Мы же хотим эту операцию отловить \(сделать специальные действия на тот случай, если программа даст ошибку\). Программа при этом не завершится.

```cpp
int f(int x, int y, int* result) {
    // result - код завершения
    return result;
}

int main() {
    f(1,0, result);
    if (result == x) std::cout << "error";
    else {
        continue;
    }
}
```

Такой подход громоздкий и неудобный. Выход - исключения.

```cpp
int f(int x, int y, int* result) {
    if (y == 0)
        throw 1; // можно бросать что угодно (любой тип), но // лучше все-таки определенные объекты
    return x/y;
}

int main() {
    f(1,0);
}
```

> > > Aborted, terminate called after throwing an instance of 'int'

Terminate - аварийное завершение программы. Ошибку никто не поймал. Как поймать?

```cpp
#include <iostring>
int f(int x, int y, int* result) {
    if (y == 0)
        throw 1; // можно бросать что угодно (любой тип), но // лучше все-таки определенные объекты
    return x/y;
}

int main() {
    try {
        f(1,0);
    } catch (int x) {
        std::cout << "ERR\n";
    } catch (float x) {
        std::cout << "FLOATAERR\n";
    }
}
```

> > > ERR Но программа завершилась удачно

```cpp
try {
  f(1,0);  
} catch (...) { // поймать любой тип
    std::cout << "ERR";
}
```

throw нет - просто идем дальше, если есть, попадаем в ту секцию catch, объект которого возвращен.

* Try ловит только то, что было брошено с помощью throw: нельзя поймать UB, битую ссылку, выход за границу массива и т.д. \(если не описать вручную логику ошибки в коде и не вернуть throw\). А вот python может поймать такие штуки.
* Не каждая ошибка - исключение \(деление на ноль - не исключение, а низкоуровневая ошибка - floationg point exception\).
* throw - оператор \(т.е. можно вставлять в выражения\), return - управляющее слово.

## 7.2 Difference between exceptions and runtime errors

Некоторые ошибки генерируют исключения \(dynamic\_cast, new\), которые можно отловить.

```cpp
int main() {
    dynamic_cast<Mother&>(f); // бросает std::bad_cast
    int *p = new int[1000000000000];
    // исключение std::bad_alloc
    std::out_of_range; // обращение к контейнеру с проверкой границы
    std::bad_type_id;
    std::vector<int> v(10);
    try { // безопасность > производительность
        v.at(10000000000000) = 1;
    } catch (...) {
        std::cout << "!";
    }
}
```

delete не бросает исключения \(язык оставляет ответственность Вам\).

## 7.2.1 Std::exception \(check cppreference\)

В случае чего, бросаем эти ошибки

std::runtime\_error есть в библиотеке, называется также, как runtime error \(зачем\). Это все равно не настоящий RE!

std::exception.what\(\) - что случилось

```cpp
int main() {
    std::vector<int> v(10);
    try (
        v.at(1000000000000) = 1;
    } catch (std::exception& ex) {
        std::cout << ex.what();
    }
}
```

Можно создать собственный тип исключения и делать с ним все, что захотим:

```cpp
    try {
        throw std::out_of_range("exception");
    } catch (std::exception& ex) { // ловим любые стандартные исключения
        std::cout << ex.what(); // exception
    }
```

А можно создать наследника std::exception \(хороший кодстайл\).

## 7.3 Rules of catching

* не действует приведение типов \(даже если они приводятся\), за исключением ситуации приведения между родителем и наследником и const/non cost

  ```cpp
  #include <iostream>
  int main() {
    try {
        throw 'a';
    } catch (int x) {
        std::cout << '1';
    } // char не поймаем (запрещено приведение типов)
  }
  ```

* выбор подходящего catch

  ```cpp
  #include <iostream>
  int main() {
    try {
        throw std::out_of_range('aaa');
    } catch (std::exception) {
        std::cout << 1;
    } catch (std::out_of_range) {
        std::cout << 2;
    } catch (...) {
        std::cout << 3;
    }
  }
  ```

  > > > 1

Работает не как разрешение перегрузки. Компилятор выведет первую подходящую инструкцию.

## 7.4 Exceptions and copying

```cpp
struct Noisy {
    Noisy() {
        std::cout << "created";
    }
    Noisy(const Noisy&) {
        std::cout << "copyed";
    }
    ~Noisy() {
        std::cout << "destroyed";
    }
};

void f() {
    Noisy x;
    throw x;
}

int main() {
    try {
        f();
    } catch (const Noisy& x) {
        std::cout << "caught";
    }
}
```

* создали Noisy в f\(\)
* x - локальный объект, он уничтожится, но его нужно бросить
* значит, x будет скопирован, чтобы можно было бросить, локальный x будет унчитожен
* копированный пойман, потом уничтожен

```cpp
int main() {
    try {
        f();
    } catch (const Noisy x) {
        std::cout << "caught";
    }
}
```

тут вообще 3 копирования \(создается новый объект в catch\)

Если в данной ситуации с локальным обхектом запретить конструктор копирования, бросить не получится \(объект должен быть копируемым/перемещаемым\).

throw Noisy\(\) работает без копирования.

Еще пример:

```cpp
void g() {
    try {
        throw std::out_of_range("a");
    } catch (const std::exception& ex) {
        std::cout << "caught";
        throw; // передать exception вверх изнутри catch
        // throw ex; // бросить локальный еще раз, т.е. скопируем как родителя
    }
}

int main() {
    try {
        g();
    } catch (std::out_of_range& oor) {
        std::cout << "caught2"; // поймаем нужное исключение
    }
}
```

## 7.5 Exceptions in constructors and RAII idiom

```cpp
voif f() {
    int* p = new int(5);
    throw 1;
    delete p; // memory leak
}
int main() {
    try {
        f();
    } catch (...) {

    }
}
```

Исключения в конструкторах - плохо:

```cpp
struct S {
    int* p = nulllptr;
    S(): p(new int(5)) {
        throw 1; // объект типо не до конца создан, значит нельзя вызывать деструктор (объект недосоздан)
    }
    ~S() {
        delete p;
    }
}

int main() {
    try {
        S s;
    } catch (...) {
        // memory leak
    }
}
```

Перед тем, как сделать throw, освободить память \(костыль\). Для решения этой проблемы были придуманы умные указатели.

**RAII - Resource Acquisition Is Initialization**

 Допустим, не знаем про умные указатели. Как решить проблему?

```cpp
template <typename T>
struct SmartPtr {
    T* ptr;
public:
    SmartPtr(T* ptr): ptr(ptr) {}
    ~SmartPtr() {
        delete ptr;
    }
}

struct S {
    SmartPtr<int> p = nulllptr;
    S(): p(new int(5)) {
        throw 1; // объект типо не до конца создан, значит нельзя вызывать деструктор (объект недосоздан)
    }
    ~S() {
        delete p;
    }
}

void f() {
    SmartPtr<int> p = new int(5);
    SmartPtr<int> pp = p; // segfault, скопирует все поля
    // два delete по одному месту
    throw 1;
}

int main() {
    try {
        S s;
    } catch (...) {
        // memory leak
    }
}
```

Объект SmartPtr как локальный будет уничтожен при throw. Нельзя копировать и ставить умные указатели на одно место. А можно сделать по-другому \(но об этом потом\).

