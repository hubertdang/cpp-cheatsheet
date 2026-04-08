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

### Partial specialization

* Only classes and variables can be partially specialized, function templates cannot

* When a type matches multiple specializations, the compiler always chooses the **most specialized** match

    ```cpp
    template <typename T> struct Foo;            // Primary (matches everything)
    template <typename T> struct Foo<T*>;        // Spec 1 (matches any pointer)
    template <typename T> struct Foo<const T*>;  // Spec 2 (matches only const pointers)

    Foo<int> a;        // Uses Primary
    Foo<int*> b;       // Uses Spec 1
    Foo<const int*> c; // Uses Spec 2 (Matches Spec 1 & 2, but 2 is more specialized)
    ```

* If the compiler finds multiple matches but neither is strictly "more specialized" than the other, it stops and throws an **ambiguous template instantiation** error

    ```cpp
    template <typename T, typename U> struct Bar;          // Primary
    template <typename T>             struct Bar<T, T>;    // Spec 1 (Both match)
    template <typename T, typename U> struct Bar<T*, U>;   // Spec 2 (First is pointer)

    Bar<int, double> a;  // Uses Primary
    Bar<int, int> b;     // Uses Spec 1
    Bar<int*, double> c; // Uses Spec 2

    // ERROR: Ambiguous!
    Bar<int*, int*> d;   
    // Matches Spec 1 (both are int*) 
    // Matches Spec 2 (first is int*)
    // Neither rule is strictly a subset of the other.
    ```

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

## Standard Template Library (STL) Algorithms

### Iterators

| Category | Movement | Capabilities |
| :--- | :--- | :--- |
| **Input** | Forward only | Read once (Single-pass) |
| **Output** | Forward only | Write once (Single-pass) |
| **Forward** | Forward only | Read/Write (Multi-pass) |
| **Bidirectional**| Forward / Backward | Read/Write, `++`, `--` |
| **Random Access**| Jump anywhere (O(1)) | Read/Write, `+`, `-`, `[]`, `<` |
| **Contiguous** (C++17) | Jump anywhere (O(1)) | Guaranteed adjacent in memory |

### Algorithms

* `any_of` returns `true` if **at least one** element satisfies the specified predicate

    ```cpp
    bool has_even = std::any_of(v.begin(), v.end(), [](int n) {
        return n % 2 == 0;
    });
    ```
* `all_of` returns `true` if **every** element satisfies the specified predicate

    ```cpp
    bool all_even = std::all_of(v.begin(), v.end(), [](int n) {
        return n % 2 == 0;
    });
    ```

* `none_of` returns `true` if **no** element satisfies the specified predicate

    ```cpp
    bool none_even = std::none_of(v.begin(), v.end(), [](int n) {
        return n % 2 == 0;
    });
    ```

* `find` returns an iterator to the **first** element that satisfies the specified value, `end()` iterator otherwise 

    ```cpp
    auto it = std::find(v.begin(), v.end(), 10);
    if (it != v.end()) {
        // Found the first 10
    }
    ```


* `find_if` returns an iterator to the **first** element that satisfies the specified predicate, the iterator to the end of the range otherwise 

    ```cpp
    auto it = std::find(v.begin(), v.end(), [](int n) {
        return n % 2 == 0;
    });
    if (it != v.end()) {
        // Found the first even number
    }
    ```

* `mismatch` returns a `std::pair` containing iterators to the first mismatched elements in two ranges of elements. If the ranges are identical, it returns iterators to the end of the ranges

    ```cpp
    auto [it1, it2] = std::mismatch(v1.begin(), v1.end(), v2.begin(), v2.end());
    ```

* `transform` applies a function to a range of elements and stores the result in a destination range. Binary `transform` assumes that the second iterator is *at least* the size of the first

    ```cpp
    // Unary transform
    std::vector<int> v = {1, 2, 3, 4};
    std::vector<int> result;

    // Use result.begin() insetad of back_inserter if memory is pre-allocated
    std::transform(v.begin(), v.end(), std::back_inserter(result), [](int n) {
        return n * n;
    });
    ```

    ```cpp
    // Binary transform
    std::vector<int> a = {1, 2, 3, 4};
    std::vector<int> b = {1, 2, 3, 4}; // b.size() >= a.size() must be true
    std::vector<int> result;

    std::transform(a.begin(), a.end(), b.begin(), std::back_inserter(result), [](int x, int y) {
        return x - y;
    });
    ```

* `reduce`, by default, sums the elements in a range, **in no particular order**. You can override this behaviour, however the operation you provide must be associative and commutative (doesn't matter what order everything is done in, e.g., addition, multiplication)

    ```cpp
    std::vector<int> v = {1, 2, 3, 4};

    // Default returns 10
    int sum = std::reduce(v.begin(), v.end());

    // With initial value of 5 returns 15
    int total = std::reduce(v.begin(), v.end(), 5);

    // Provide associative operation (multiplication) with initial value 1
    int product = std::reduce(v.begin(), v.end(), 1, [](int a, int b) {
        return a * b;
    });

    // Parallel execution 
    int par_sum = std::reduce(std::execution::par, v.begin(), v.end());
    ```

* `transform_reduce` does a `transform` and then a `reduce` in one call

    ```cpp
    // Unary transform
    std::vector<int> v1 = {1, 2, 3};

    int sum_squares = std::transform_reduce(v1.begin(), v1.end(), 0,
                                            [](int a, int b) {
                                                return a + b; // reduce operation
                                            },
                                            [](int a) {
                                                return a * a; // transform operation
                                            });
    ```

    ```cpp
    // Binary transform
    std::vector<int> v1 = {1, 2, 3};
    std::vector<int> v2 = {4, 5, 6};

    int dot_product = std::transform_reduce(v1.begin(), v1.end(), v2.begin(), 0,
                                            [](int a, int b) {
                                                return a + b; // reduce operation
                                            },
                                            [](int a, int b) {
                                                return a * b; // transform operation
                                            });
    ```

* `partial_sum` takes an input range and outputs a new range of the same size, where each element is the sum of all preceding elements up to that point. Note that the output of the first element is always itself, e.g., `output[0]` equals `input[0]`, or `input[0] + initial_value` if an initial value is provided. Modern C++ should use `inclusive_scan` instead

    ```cpp
    std::vector<int> v = {1, 2, 3, 4};
    std::vector<int> result;

    std::partial_sum(v.begin(), v.end(), std::back_inserter(result));
    // result = {(1), (1 + 2), (1 + 2 + 3), (1 + 2 + 3 + 4)} = {1, 3, 6, 10}
    ```

    ```cpp
    std::vector<int> v = {2, 2, 2, 2};
    std::vector<int> powers;

    // Override default behaviour (addition)
    std::partial_sum(v.begin(), v.end(), std::back_inserter(result), [](int a, int b) {
        return a * b;
    });
    // result = {(2), (2 * 2), (2 * 2 * 2), (2 * 2 * 2 * 2)} = {2, 4, 8, 16}
    ```

* `exclusive_scan` is just like `partial_sum`, except it excludes the current element from its own sum, and therefore requires an initial value

    ```cpp
    std::vector<int> v = {1, 2, 3, 4};
    std::vector<int> result;

    std::exclusive_scan(v.begin(), v.end(), std::back_inserter(result), 10);
    // result = {(10), (10 + 1), (10 + 1 + 2), (10 + 1 + 2 + 3)} = {10, 11, 13, 16}
    ```

* `inclusive_scan` is the parallel-friendly version of `partial_sum`. Initial value is optional

    ```cpp
    std::vector<int> v = {1, 2, 3, 4};
    std::vector<int> result;

    std::inclusive_scan(v.begin(), v.end(), std::back_inserter(result));
    // result = {(1), (1 + 2), (1 + 2 + 3), (1 + 2 + 3 + 4)} = {1, 3, 6, 10}

    // Parallel execution
    std::inclusive_scan(std::execution::par, v.begin(), v.end(), v.begin());
    ```

* `adjacent_difference` takes an input range and outputs a new range of the same size, where each element is the difference between the current element and the one immediately before it (e.g., `y_i = x_i - x_(i-1)`)

    ```cpp
    std::vector<int> v = {1, 2, 3, 4};
    std::vector<int> result;

    std::adjacent_difference(v.begin(), v.end(), std::back_inserter(result));
    // result = {(1), (2 - 1), (3 - 2), (4 - 3)} = {1, 1, 1, 1}
    ```