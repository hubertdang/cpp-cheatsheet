## Basics

### Implicit type conversion

- Assigning mismatched numerical types (e.g., `int` to `float`, or `double` to `float`, etc.) is fine (applies to `return` as well)

    ```cpp
    // COMPILES
    int fn(float i) {
        return i; // fn(2.55) would return 2
    }
    ```

- Passing a different type to a function is fine as long as the compiler can definitively deduce the function you're trying to call (i.e., no ambiguity)

    ```cpp
    // COMPILES BECAUSE THERE IS ONLY 1 fn()
    int fn(int i) {
        return i;
    }

    int main(int argc, char* argv[]) {
        cout << fn(1) << fn(2.0) << endl; // outputs 12
        return 0;
    }
    ```

    ```cpp
    // FAILS COMPILATION BECAUSE fn(2.0) PASSES A DOUBLE, WHICH WORKS FOR BOTH
    float fn(float i) {
        return i;
    }

    int fn(int i) {
        return i;
    }

    int main(int argc, char* argv[]) {
        cout << fn(1) << " " << fn(2.0) << endl;
        return 0;
    }
    ```

### Namespaces

- *Unnamed* (or *anonymous*) namespaces can be used to achieve the same thing as a `static` global in C (i.e., scope accessible only by the current file)

    ```cpp
    namespace {
        int p = 4; // Access just by using p
    }
    ```

- Namespaces don't all have to be defined in one place

    ```cpp
    // n1::i and n1::t are both valid
    namespace n1 {
        int i = 1;
    }

    namespace n1 {
        int t = 2;
    }
    ```

### Immutability

* `const` variables are **read-only** after initialization (unless you use `mutable`, `const_cast`, or override directly to memory) and are resolved at compile time whenever possible

    ```cpp
    const int a = 1; // Evaluated at compile time
    const int b = a + 1; // Evaluated at compile time because a was evaluated as 1
    *(int *)&a = 2; // This will work
    a = 3;          // This will not
    ```

* A `const` reference (e.g., `const int& a`) protects the variable the reference points to (which must be `const` itself), not the reference itself, as references already can't be changed to point to something else

    ```cpp
    void fn(const int& x) {
        std::cout << x << std::endl; // Can only read x
    }

    int main() {
        const int a = 5;
        int b = 2;
        fn(a); // Ok
        fn(b); // Fails
    }
    ```

* `const` member functions guarantee that they will not modify any member variables of the `class` or `struct`

    ```cpp
    // Assume age is a member variable
    int getAge() const {
        age = 10; // Fails compilation 
        return age;
    }
    ```

* `constexpr` expressions (e.g., variable assignment) *must* be resolved at compile time
* `constexpr` functions will be resolved at compile time *if needed*, otherwise, it is up to the compiler

    ```cpp
    constexpr int fn(int a) {
        return a * 2;
    }

    int x = fn(1); // Compiler might optimize to execute at compile time, we don't know
    constexpr int y = fn(1); // Guaranteed to execute at compile time because y needs it to
    ```

* `consteval` functions (not applicable to expressions) *must* be resolved at compile time, similarly to `constexpr` expressions
* If a function is neither `constexpr` or `consteval`, it will execute at runtime

## User-Defined Types

### Abstract classes

* An abstract class is a class with at least one pure virtual function

    ```cpp
    class Employee {
    public:
        virtual void calculatePay() = 0; // "virtual" and "= 0" makes this pure virtual
    };
    ```

* If all members of a class are pure virtual functions, the class is called a "pure interface"
* Abstract classes cannot be instantiated, so you can use a pointer or reference of the abstract type to point to a concrete child object

### Inheritance

* To mark a class as a subclass of another, do this:

    ```cpp
    class IntVector: public IntCollection {...}; // IntVector is a subclass of IntCollection
    ```

* *Upcasting* is when you observe a subclass as its parent class. This type of cast can be implicit

    ```cpp
    // By pointer (safe)
    IntVector* iv = new IntVector;
    IntCollection* ic = iv;
    ```

    ```cpp
    // By reference (safe)
    IntVector iv;
    IntCollection& ic = iv;
    ```

    ```cpp
    // By value (object slicing)
    IntVector iv;
    IntCollection& ic = iv;
    ```

* *Downcasting* is when you recover data and functionality of an object that was previously upcasted. This kind of cast must be explicit and generally should be avoided because it often breaks things

    ```cpp
    IntVector& iv = (IntVector&) ic; // Assume ic is an IntCollection& type
    ```

### Essential operations

* Constructors are used when a new object is created, different constructors are used depending on the context

    ```cpp
    class C {
    public:
        C();                // Default constructor
        C(const C& other);  // Copy constructor when creating a new object from an existing one
        C(C&& other);       // Move constructor when creating an object by transferring ownership of
                            // resources from a temporary object (rvalue)
    };
    ```

* The *copy* operation defaults to member-wise copy, which isn't appropriate for variables holding resource handlers
* **Copy assignment operator** is used when both objects already exist, and you're just overwriting one with the other

    ```cpp
    C& operator=(const C& other) {...} // a = b;
    ```

* Constructors can use direct initialization of member variables that don't require any special processing

    ```cpp
    class C {
        int a = 0;
        int b = 0;
    public:
        C(int a, int b) : a(a), b(b) {}
    };
    ```

## Error Handling

### Exiting a program
* Functions registered with `atexit()` are executed during normal program termination, which happens immediately after `main()` returns or when `exit()` is explicitely called
* Functions registered with `set_terminate()` are executed when an exception is thrown but not caught, the exception handling mechanism fails, or std::terminate() is called manually

### `try`-`catch` blocks
* Implicit type conversion does not happen in `catch` blocks (e.g., you can't throw an `int` and catch a `float`)
* Exceptions should be caught as `const` references to prevent object slicing, avoid unnecessary copying, and preserve polymorphic behavior

    ```cpp
    try {
        fn(); // throws a std::exception or a derivation of it
    } catch (const std::exception& e) {
        cerr << e.what() << endl;
    }
    ```

### Exception safety

* Basic exception safety guarantees that invariants are preserved and no resources are leaked

    ```cpp
    class int_vector {
        size_t size = 0;
        int *elems = nullptr;
    public:
        int_vector() = default;
        void resize(size_t new_size) {
            delete[] e;
            size = new_size; // Broken invariant starts
            try {
                elems = new int[new_size];
            } catch (const std::exception&) {
                // Set invariant to valid state
                elems = nullptr;
                size = 0;
                throw;
            }
        }
    }
    ```

    ```cpp
    class reversable_vector {
        vector<int> _original;
        vector<int> _reversed; 
    public:
        void add(int a) {
            _original.push_back(a); // Outside try block because push_back has strong guarantee
            try {
                _reversed.insert(_reversed.begin(), a);
            } catch (...) {
                _original.clear();
                _reversed.clear();
                throw;
            }
        }
    }
    ```

* Strong exception safety guarantees basic exception safety and no state change

    ```cpp
    class int_vector {
        size_t size = 0;
        int *elems = nullptr;
    public:
        int_vector() = default;
        void resize(size_t new_size) {
            int *temp = new int[new_size];
            size = new_size; // Broken invariant starts
            std::swap(elems, temp);
            delete[] temp;
        }
    }
    ```

    ```cpp
    class reversable_vector {
        vector<int> _original;
        vector<int> _reversed; 
    public:
        void add(int a) {
            vector<int> temp = _reversed;
            temp.insert(temp.begin(), a); // So far, object state is unchanged
            _original.push_back(a); // If this fails, object state will be left unchanged
            std::swap(_reversed, temp); 
        }
    }
    ```

* No exception (`noexcept`) guarantees there is no way an exception exits the scope of the function. This does not mean no exceptions can be thrown within the function, they just need to be caught. Letting an exception leave a `noexcept` function or `main` triggers `std::terminate`

    ```cpp
    void swap() noexcept {
        std::swap(_original, _reversed);
    }
    ```

## Generic Programming

### Template types

* `auto` function parameters are template types behind the scenes, however templates allow you to enforce multiple variables to be the same type

    ```cpp
    template <typename T>
    T add(T a, T b) {...} // both must be the same type
    ```

    ```cpp
    auto add(auto a, auto b) {...} // a and b can be different types
    ```

* Constrain template types by replacing the `typename` keyword with the name of a **concept** or using a `requires` clause

    ```cpp
    template<integral T> // Only integral types (e.g., int, long, etc., just no floating point)
    T fn(T a) {...}
    ```

    ```cpp
    template<typename T>
    requires integral<T> // Only integral types (e.g., int, long, etc., just no floating point)
    T fn(T a) {...}
    ```

* You can combine concepts together

    ```cpp
    template<integral T>
    requires integral<T> || floating_point<T> // Only numbers
    T fn(T a) {...}
    ```

* You can define your own concepts

    ```cpp
    template <typename T>
    concept Number = integral<T> || floating_point<T>; // Only numbers

    template <Number T>
    T fn(T a) {...}
    ```

    ```cpp
    template <typename T>
    concept HasValueMemberAndClearMethod = requires(T t) {
        t.value;    // Must have a public member named "value"
        t.clear();  // Must have a public method named "clear"
    }

    template <HasValueMemberAndClearMethod>
    T fn(T a) {...}
    ```

    ```cpp
    template <typename T>
    concept Addable = requires(T a, T b) {
        { a + b } -> std::same_as<T>; // a + b must compile AND return type T
    }

    template <Addable>
    T fn(T a) {...}
    ```

* You can discard code branches at compile-time in templates based on type traits
    ```cpp
    if constexpr (std::is_pointer_v<T>) {...}
    else if constexpr (std::is_integral_v<T>) {...}
    else if constexpr (std::is_floating_point_v<T>) {...}
    ...
    ```

### Template parameters

* If your requirements call for a compile-time parameter, you want a template parameter (e.g., "implement a function that receives an integral value and *an int exponent at compile*")

    ```cpp
    template<integral T, int exponent>
    T integral_power(T base) {...} // e.g., integral_power<int, 3>(4); is 4^3
    ```

* Template parameters are treated as l-values at runtime, so they are *read-only*. Think of template parameters as `#define` constants 

### Functors

* A *functor* is an object that acts like a function when asked to
* Overload the `operator()` to allow an object to be called and look like a function

    ```cpp
    class equal_to_val {
        const int val;
    public:
        equal_to_val(int v): val(v) {}
        bool operator()(const int& to_test) {
            return val == to_test;
        }
    };

    int main() {
        equal_to_val is_equal_to_five(5);   // Initialize val to 5
        bool result1 = is_equal_to_five(6); // false
        bool result2 = is_equal_to_five(5); // true 
    }
    ```