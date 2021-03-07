---
description: Standart containers
---

# 9 Containers

### 9.1 Overview of containers

insert - вставить в контейнер по итератору

deque - двусторонняя очередья

list - двусвязный список \(linked list in Java\) - вставка за O\(1\) по итератору

**Sequence containers**

                            **indexing\[\]        push\_back/front      insert        erase       find        iterator\_category**

* vector            O\(1\)                       O\(1\)\*                   O\(n\)           O\(n\)        N            random\_access
* deque            O\(1\)                       O\(1\)\*                   O\(n\)           O\(n\)        N            random\_access
* list/fwrd          N                          O\(1\)                     O\(1\)           O\(1\)        N           bidirect/forward

**Ассоциативные контейнеры**

* set/map      O\(logn\)                      N                    O\(logn\)    O\(logn\)   O\(logn\)         bidirectional
* un-d set/map  O\(1\)                       N                      O\(1\)            O\(1\)       O\(1\)                forward

![](../.gitbook/assets/image%20%2813%29.png)

### 9.2 Vector implementation

```cpp
#include <iostream>

template <typename T>
class Vector {
    T* arr;
    size_t zs;
    size_t cap;
    
public:
    Vector(size_t n, const T& value = T());
    ...
    T& operator[](size_t i) {
        return arr[i];
    }
    const T& operator[](size_t i) const {
        return arr[i];
    }
    T& at(size_t i) {
        if (i >= sz) throw std::out_of_range("....");
        return arr[i];
    }
    const T& at(size_t i) const {
        if (i >= sz) throw std::out_of_range("....");
        return arr[i];
    }
    ...
    size_t size() const { return sz; }
    void resize(size_t n, const T& value = T()) {
        if (n < cap) reserve(cap);
        if (n > sz) {
        ... // add
        } else if (n < sz) {
        ... // remove
        }
    }
    size_t capacity() const { return cp; }
    void shrink_to_fit(); // change capacity to size
    void reserve(size_t n) { // ВЫДЕЛИТЬ память хотя бы для n элементов
        if (n <= cp) return;
        // код на двойку), не надо так писать (UB)
        // T* newarr = new T[n];
        // for (size_t i = 0; i < sz. ++i) {
        //   newarr[i] = arr[i]; // Segfault maybe
        // }
        // delete[] arr; // Segfaul maybe
        // arr = newarr;

        // При reserve должна выделяться память, 
        // а тут идет заполнение оюъектов значениями по умолчанию
        // создание объектов по умолчанию мб тяжелым (захват ресурса)
        
        // Может не быть конструктора по умолчанию (CE)
         
         
        // #### V2 (на тройку)
        // T* newarr = reinterpret_cast<T*>(new int8_t[n * sizeof(T)]);
        // for (size_t i = 0; i < sz. ++i) {
        //   newarr[i] = arr[i]; // Segfault maybe
        // }
        // выделили столько байт, сколько нужно для хранения n штук T
        // логика правильная, но будет segfault: под массивом память,
        // а не объект типа T, нельзя присваивать (в copy constructor
        // будет удаление ничего)
        // Здесь нужно вызвать конструктор T по данному адресу от данного объекта
    
        T* newarr = reinterpret_cast<T*>(new int8_t[n * sizeof(T)]);
        for (size_t i = 0; i < sz. ++i) {
           new(newarr + i) T(arr[i]); // вызов конструктора копирования по адресу, exception problem
        }
        // delete[] arr; // опять UB
        // В большинстве случаев в arr не все объекты типа T
        
        for (size_t i = 0; i < sz; ++i) {
            (arr + i)->~T;
        }
        delete[] reinterpret_cast<int8_t*>(arr);
        arr = newarr;

        // #### V3 (на хор6)
        
        // memcpy может привести к UB!

        // Проблемы: неэффективное копирование (хз че делать, но с cpp11 вроде можно по-умному - spoiler - std::move)
        // небезопасность относительно исключений
        // если конструктор объекта бросит исключение, то вектор не достроится (будет memory leak)
        // контейнеры должны быть строго безопасны относительно исключений (если беда, то контейнер делает ctrl-z):

        T* newarr = reinterpret_cast<T*>(new int8_t[n * sizeof(T)]);
        size_t i = 0;
        try {
            for (; i < sz. ++i) {
            // placement new
            new(newarr + i) T(arr[i]); // вызов конструктора копирования по адресу, exception problem
            }
        } catch (...) {
            for (size_t j = 0; j < i; ++j) {
                new(arr + i)->~T();
            }
            delete[] reinterpret_cast<int8_t*>(arr);
            throw; // передать исключение дальше
        }

        // ^ есть стандартная функция
        try {
            std::uninitialized_copy(arr, arr + sz, newarr); // логика проверки копирования, бросает исключения и удаляет то, что не получилось
        } catch (...) {
            delete[] reinterpret_cast<int8_t*>(arr);
            throw; // передать исключение дальше
        }

        // TODO аллокаторы, std::move
    }
    
    void push_back(const T& value) {
        if (sz == capacity) reserve(2*cp);
        new(arr + sz) T(value); // arr[sz] = value - UB
        ++sz;
    }
    
    void pop_back() {
        (arr+sz-1)->~T();
        --sz;
    }
};

// std::vector не перевыделяет память после pop_back по умолчанию
// v.clear() не гарантирут очистку памяти (capacity), делает size = 0
```

### 9.3 Vector&lt;bool&gt;

```cpp
#include <iostream>
#include <vector>

template <>
class Vector<bool> {
    // хранит bool как один бит, а не байт
    int8_t* arr;
    size_t sz, cap;

    struct BitReference() {
        int8_t* cell;
        uint8_t num;

        BitReference& operator=(bool b) {
            if (b) {
                *cell |= (1u << num);
            } else {
                *cell &= ~(1u << num);
            }
            return *this;
        }
    };

public:
    BitReference operator[](size_t t) {
        return BitReference{arr + i / 8, i % 8}; 
    }

    operator bool() const {
        return *cell & (1u << num);
    }
};


template <typename U>
void f(const &U) = delete;


int main() {
    std::vector<bool> vb(10, false);
    vb[5] = true;
    f(vb[5]); // U = std:_Bit_reference
}

```

### 9.4 Deque architecture

![](../.gitbook/assets/image%20%2814%29.png)

![&#x43F;&#x435;&#x440;&#x435;&#x441;&#x442;&#x430;&#x43D;&#x43E;&#x432;&#x43A;&#x430; &#x443;&#x43A;&#x430;&#x437;&#x430;&#x442;&#x435;&#x43B;&#x435;&#x439;](../.gitbook/assets/image%20%2812%29.png)

