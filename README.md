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
4. make sure that all the objects `std::thread` uses are valid till its lifetime, otherwise UB
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
1. making all the code that excess the data **mutually exclusive**
2. lock the mutex before accessing the data, unlock once its done with the data
3. While one thread lock the mutex all other threads wait till the mutex got unlocked.
4. Its not recommeted to call the individual funcitons `lock()` and `unlock()` directly
5. Use RAII class `std::lock_guard`

```cpp
std::mutex some_mutex;

void add_to_colection(int val) {
    std::lock_guard<std::mutex> guard{ some_mutex };
    vec.push_back(val);
}

bool is_in_collection(int val) {
    std::lock_guard<std::mutex> guard{ some_mutex };
    return std::find(std::begin(vec), std::end(vec), val) != std::end(vec);
}
```
6. Any code that has access to the pointer or reference of shared data can now
access/modify the protected data without locking the mutex
7. **deadlock** threads cannot proceed as each is waiting for the other to release its mutex
8. Always lock two mutexes in the same order to avoid deadlock [this may not be always true]

## `std::lock`
1. `std::lock` can lock two or more mutexes at once without the risk of deadlock
2. `std::adopt_lock` indicate the `lock_guard` that the mutexes are already locked
and they should just adopt the ownership of the existing lock on mutex in the ctor
```cpp
{
    std::lock(mu1, mu2); //calling thread locks the mutex
    std::lock_guard<std::mutex> l1{ mu1, std::adopt_lock };
    std::lock_guard<std::mutex> l2{ mu2, std::adopt_lock };
    do_somithing(shared_data);
}
```
3. `std::lock` provides **all or nothing** approach, which means if any mutex throws exception while
locking all the mutexes will be unlocked [either get all lock or nothing]
4. `std::lock` helps to avoid deaslock when aquire 2/more locks together; It cannot help when
acquired separately

## Avoiding deadlock
1. avoid nested locks
2. avoid call user-supplied code while holding a lock
3. Acquire locks in a fixed order, if you cant acquire as a single operation using `std::lock`
4. Use a lock hierarchy [take light]

## `std::unique_lock`
1. more flexible as it doesn't always own a mutex
2. `std::adopt_lock` lock object manage the lock on mutex
    1. It assumes that the calling thread already owns the losk
    2. wrapper should adopt the ownership of the mutex and release it when
    goes out of scope
3. `std::defer_lock` mutex should remain unlocked on construction 
    1. It assumes that the calling thread is going to call lock later
    2. wrapper going to release the lock when it goes out of scope
4. lock can be acquired later by calling `lock()` on `std::unique_lock` obj.
5. slower than `std::lock_guard`, need more space too
```cpp
std::lock(m1, m2); // calling thread locks the mutex  
std::lock_guard<std::mutex> lock1(m1, std::adopt_lock);    
std::lock_guard<std::mutex> lock2(m2, std::adopt_lock);
// access shared data protected by the m1 and m2

std::unique_lock<std::mutex> lock1(m1, std::defer_lock);    
std::unique_lock<std::mutex> lock2(m2, std::defer_lock);    
std::lock(lock1, lock2);
// access shared data protected by the m1 and m2
```
6. `std::lock_guard` with `std::adopt_lock` strategy assumes the mutex is already acquired
7. `std::unique_lock` with `std::defer_lock` strategy assumes the mutex is not acquired on construction, 
rather than explicitly going to be locked
8. compiler will catch the error if you forget to define one of the `unique_lock` statements
9. if you forget one of the `lock_guard` statements, compiler will not show any error, but there will be deadlock
10. Locking with appropriate grained granularity
    1. fine grained granularity [small amount is protected with lock]
    2. coarse grained granularity [large amount of date is protected by lock]
The idea is not to block the other threads with unnecessary time consuming tasks, which may reduce the improvements
gained by multithreading
Here `std::unique_lock` can be really handy
```cpp
void get_process_data() {
    std::unique_lock<std::mutex> lk{ mu };
    auto data = get_next_data(); //needs to be thread safe
    lk.unlock();
    // each thread function on different chunk of data. 
    // So can be all thread runs simultanously 
    auto result = process(data); 
    lk.lock();
    write_result(result); //write needs to be synchronized so lock()
}
```
11. In general lock needs to be held for period of time as minimum as possible

## `std::call_once` 
1. Lazy initialization is common in single-threaded code 
```cpp
std::shared_ptr<some_resource> res_ptr;

void foo() {
    if (!res_ptr) res_ptr.reset(new some_resource{});
    res_ptr->do_something();
}
```
2. double-checked locking pattern is bad
```cpp
void ub_code() {
    if (!res_ptr) {
        std::lock_guard<std::mutex> lk{ res_ptr };
        if (!res_ptr) res_ptr.reset( new some_resource{} ); //write
    }
    res_ptr->do_something(); //read
}
```
3. Here the write is not synchronized with the read
4. There is no guarantee that @ `do_something` ptr may not be fully initialized
5. `std::once_flag` and `std::call_once`
```cpp
std::once_flag of;

void foo() {
    std::call_once(of, [](){
        res_ptr.reset( new some_resource{} );
    });
    res_ptr->do_something();
}
```

## static variable
1. static variables all are thread safe

## `std::recursive_mutex`
1. UB -> if a thread try to lock a mutex which its already locked
2. You can have multiple lock on a single instance of same thread
3. Another thread can access only if all locks are released by owning thread
4. If 1 thread calls lock 3 times another thread can access only if the thread 1 call unlock 3 times

# Synchronizing concurrent operations

