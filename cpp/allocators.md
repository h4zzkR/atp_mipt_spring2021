## 10.3 Idea of allocator, std::allocator implementation

- standard allocator

```cpp
template <typename T>
struct allocator {
    T* allocate(size_t n) {
        return ::operator new(n * sizeof(T));
    }
    
    void deallocate(T* ptr, size_t n) {
        ::operator delete(ptr);
    }
    
    template <typename... Args>
    void construct(T* ptr, const Args&... args) {
        new (ptr) T(args...);
    }
    
    void destroy(T* ptr) {
        ptr->~T();
    }
};
```

## 10.4 Pool allocators / Stack allocators

Изначально запрошивает много памяти (pool). Хранит стек аллокатора (указатель на первый незанятый байт). Allocate сдвигает указатель, возвращает "кусок" памяти. Используется, когда заведомо знаем, что у нас есть много памяти в запасе и когда знаем, что точно влезем в pool.

![image-20210327183720852](/home/h4zzkr/snap/typora/33/.config/Typora/typora-user-images/image-20210327183720852.png)

deallocate аллокатор игнорирует (ничего не делает). Экономит время крч, дает выигрыш для контейнеров, которые часто выделяют себе память (list, unordered_map, map).

Что, если аллокатор не будет использовать динамическую память?

Аллокатор, который на стеке создает массив огромного размера и ведет себя как pool. Тогда вообще не будем обращаться к new ни разу. Получается квазидинамический массив....

## 10.5 Allocator_traits

На самом деле обязательные части аллокатора - allocate,deallocate, construct и destroy работают +- одинаково, их можно не определять самостоятельно.

См member types на cppreference std::allocator, std::allocator_traits

![image-20210327190155033](/home/h4zzkr/snap/typora/33/.config/Typora/typora-user-images/image-20210327190155033.png)



## 10.5 Vector improved implementation

Отныне в контейнерах нельзя использовать new/delete напрямую

```cpp
template <typename T, typename Alloc = std::allocator<T>>
class Vector {
    T* arr;
    size_t sz;
    size_t cap;
    Alloc alloc;
    
    using AllocTraits = std::allocator_traits<Alloc>
    
public:
    Vector(size_t n, const T& value = T(), const Alloc& alloc = Alloc());
    
    T& operator[](size_t i) {
        return arr[i];
    }
    
    const T& operator[](size_t i) const {
        return arr[i];
    }
    
    void reserve(size_t n) {
        if (n <= cap) return;
        
        // T* newarr = alloc.allocate(n);
        T* newarr = AllocTraits::allocate(alloc, n);
        size_t i = 0;
        try {
            // эта весчь не дружит с аллокатором, надо писать вручную
            // std::uninitialized_copy(arr, arr + sz, newarr);
            for (; i < sz; ++i) {
                AllocTraits::construct(alloc, newarr + i, arr[i]);
            }
        } catch (...) {
            for (size_t j = 0; j < i; ++j) {
                AllocTraits::destroy(alloc, newarr + j);
            }
            AllocTraits::deallocate(newarr, n);
            throw;
        }
        
        for (size_t i = 0; i < sz; ++i) {
            AllocTraits::destroy(alloc, newarr + i);
        }
        
        AllocTraits::deallocate(arr, n);
        arr = newarr;
    }
    
    void push_back(const T& value) {
        if (sz == cap) reserve(2 * cap);
        AllocTraits::construct(alloc, arr + sz, value);
        ++sz;
    }
    
    void pop_back() {
        AllocTraits::destroy(alloc, arr + sz - 1);
        --sz;
    }
    
    Vector<T, Alloc>& operator=(const Vector<T, Alloc>& other) {
        if (this == &another) return *this;
        for (size_t = 0; i < sz; ++i) {
            pop_back()
        }
        AllocTraits::deallocate(arr, cap);
        
        if (AllocTraits::propagate_on_container_copy_assignment::value &&
            alloc != other.alloc) {
            alloc = other.alloc; // ecxeption??
        }
        
        sz = other.sz;
        cap = another.cap;
        
        AllocTraits::allocate(alloc, other.cap);
        for (size_t = 0; i < sz; ++i) {
            push_back(other[i]);
        }
        return *this;
    }
};
```

## 10.6 Rebinding allocators

```cpp
template <typename T, typename Alloc = std::allocator<T>>
class list {
    struct Node {...};
    ...;
    std::allocator_traits<Alloc>::rebind_alloc<Node> alloc;
    
public:
    list(const Alloc& alloc = Alloc()) : alloc(alloc) {}
    	
};
```

Аллокатор от типа Т. Но нужен тип Node, который будем выделять во время расширения.

Изнутри контейнера получаем аллокатор для другого типа.

since cpp20 is rebind_alloc in allocator_traits

## 10.7 Allocator copying and assigning to each other

 Копирование подразумевает то, что на новом аллокаторе можно без проблем освободить то, что было зарезервировано старым (равны в смысле оператора ==)

![image-20210327194445927](/home/h4zzkr/snap/typora/33/.config/Typora/typora-user-images/image-20210327194445927.png)

Можно сделать много аллокаторов, которые указывают на один пул. В таком случае удалять пул надо только тогда, когда все аллокаторы уничтожаются. Это можно сделать через shared_ptr.

shared_ptr<Pool> - принимает в конструкторе указатель (обычный), дальше можем его присваивать, копировать и т.д. Внутри хранит счетчик количества сущностей, указывающих на объект. Разделяемое владение памятью.



## 10.8 Allocator behavior on container copy/assign

![image-20210327195339566](/home/h4zzkr/snap/typora/33/.config/Typora/typora-user-images/image-20210327195339566.png)

Когда копировать, когда нет?

allocator_traits::select_on_container_copy_construction - возвращает объект аллокатора для нового контейнера:

- вернуть копию исходного
- вернуть новый

Присваивание векторов выше в примере кода



check c++ named_requirements: когда пишем всякие stl совместимые вещи, они должны удовлетворять этим требованиям

AllocatorAwareContainer

