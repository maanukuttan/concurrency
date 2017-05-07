# Managing Threads
## Launching a thread
1. Thread object can be created with `std::thread` taking a `functor/lambda`
```cpp
struct a_task {
    void operator()() const;
};
    
std::thread th1{ a_task() }; //uniform initialization syntax
std::thread th1( ( a_task()) ); //using a functor
std::thread th1( []() { so_some_task(); } ); //lambda
```

> note:  
> `std::thread th1( a_task() )`  creates a function
> declaration [most vexing parse]  of a
> function `th1` return `std::thread` taking a parameter which is a function
> pointer returning `a_task`

2. Once the thread is created one can decide to `join` or `detach`
3. If the thread is not joined or detached before the thread is destroyed, `std::thread` 
dtor calls `std::terminate()`, thus the program terminates
4. make sure that all the objects the the `std::thread` uses are valid till its lifetime, otherwise UB
```cpp
void foo() {
    auto val = 10;
    std::thread t1{ [&val]() {
        for (auto j = 0; j < 100000; ++j)
            do_something(val);
    }};
    t1.detach();
} //may be t1 still running and using val
```
5. `join()`  can be used to wait till the execution of the thread
6. `join()` can be called only once per thread
7. The act of calling `join()` also cleans up any storage associated with the thread, so the 
`std::thread` object is no longer associated with the now-finished  thread; 
it  isn’t  associated  with  any  thread
8. `joinable()` can be used to see whether a thread is can be joined or not
9. detached thread runs in the background, they are often called `daemon threads`

## Thread function arguments
1. By passing additional arguments to thread constructor
2. All arguments are **copied**
```cpp
void foo(const std::string& str);
void hoo() {
    auto chr = "some string...";
    std::thread t1(foo, chr); //bad
} //function can exit before str is created from chr

//create a string from chr before passing chr to new thread
std::thread t1(foo, std::string{ chr });
```
3. If the values needs to passed by reference then use `std::ref`
4. Thread ca be used to call the class member functions
```cpp
class cls {
public:
    void foo();
};

cls obj{};
std::thread t1{ &cls::foo, &obj };
```
## Thread and move
1. `std::thread` is not copy-able class like `std::unique_ptr`
2. ownership can be transferred between threads using `std::move`

## Number of threads
1. `std::thread::hardware_concurrency()` gives number of threads that can run `||y`
2. This is only a hint, it can return 0 too

## Identifying threads
1. `std::thread::id` is the thread identifier, obtained by
    1. `get_id()` on thread object
    2. `std::this_thread::get_id()`
2. The can be copied and compared
3. Two `thread::id` equal means
    1. both are same thread
    2. both are not holding any thread
4. The  only  guarantee given  by  the  standard  is  that  thread IDs  
that  compare  as  equal  should  produce  the same output, 
and those that are not equal should give different output.
```cpp
std::thread::id master_thread;
void some_core_part() {
    if (std::this_thread::get_id() == master_thread)
        do_master_work();
    do_common_work();
}
```

# Shared Data
## Issues with shared data
1. If the shared data is read only, then there is no problem
2. **invariants** statements that are always true about a data structure
3. **race condition** when the outcome depends on the relative ordering
of execution of operations on two or more threads
4. **data race** is a race condition due to the concurrent modification of _a single object_
5. data race leads to UB
6.Race condition can be avoided with
    1. wrap the data with protection mechanism
    2. design the data structure and its invariants in a **lock-free** manner
    3. **Software transactional memory** : the required data write and read are
    stored in a transactional log and then commmit in one step
    If commit is not proceed as the data structure is already modified then
    restart the transaction

## `std::mutex`
1. 

