# CPP Concurrency

This repository are notes regarding cpp concurrency programming.

## Synchronous and Asynchronous

Asynchronous programs can continue with its line of execution without the need to wait for the parallel task to complete. The following figure illustrates this difference.

![img](assets/img/c2-2-a2a.png)

## Processes and Threads

A _process_ (also called a task) is a computer program at runtime. It is comprised of the runtime environment provided by the operating system (OS), as well as of the embedded binary code of the program during execution. A process is controlled by the OS through certain actions with which it sets the process into one of several carefully defined states:

![img2](assets/img/c2-2-a2b.png)


- **Ready** : After its creation, a process enters the ready state and is loaded into main memory. The process now is ready to run and is waiting for CPU time to be executed. Processes that are ready for execution by the CPU are stored in a queue managed by the OS.
- **Running** : The operating system has selected the process for execution and the instructions within the process are executed on one or more of the available CPU cores.
- **Blocked** : A process that is blocked is one that is waiting for an event (such as a system resource becoming available) or the completion of an I/O operation.
- **Terminated** : When a process completes its execution or when it is being explicitly killed, it changes to the "terminated" state. The underlying program is no longer executing, but the process remains in the process table as a "zombie process". When it is finally removed from the process table, its lifetime ends.
- **Ready** suspended : A process that was initially in ready state but has been swapped out of main memory and placed onto external storage is said to be in suspend ready state. The process will transition back to ready state whenever it is moved to main memory again.
- **Blocked** suspended : A process that is blocked may also be swapped out of main memory. It may be swapped back in again under the same conditions as a "ready suspended" process. In such a case, the process will move to the blocked state, and may still be waiting for a resource to become available.

Processes are managed by the *scheduler* of the OS. The scheduler can either let a process run until it ends or blocks (non-interrupting scheduler), or it can ensure that the currently running process is interrupted after a short period of time. The scheduler can switch back and forth between different active processes (interrupting scheduler), alternately assigning them CPU time. The latter is the typical scheduling strategy of any modern operating system.

Since the administration of processes is computationally taxing, operating systems support a more resource-friendly way of realizing concurrent operations: the threads.

A *thread* represents a concurrent execution unit within a process. In contrast to full-blown processes as described above, threads are characterized as light-weight processes (LWP). These are significantly easier to create and destroy: In many systems the creation of a thread is up to 100 times faster than the creation of a process. This is especially advantageous in situations, when the need for concurrent operations changes dynamically.

![img3](assets/img/c2-2-a2c.png)

Threads exist within processes and share their resources. As illustrated by the figure above, a process can contain several threads or - if no parallel processing is provided for in the program flow - only a single thread.

A major difference between a process and a thread is that each process has its own address space, while a thread does not require a new address space to be created. All the threads in a process can access its shared memory. Threads also share other OS dependent resources such as processors, files, and network connections. As a result, the management overhead for threads is typically less than for processes. Threads, however, are not protected against each other and must carefully synchronize when accessing the shared process resources to avoid conflicts.

Similar to processes, threads exist in different states, which are illustrated in the figure below:

![img4](assets/img/c2-2-a2d.png)

- **New** : A thread is in this state once it has been created. Until it is actually running, it will not take any CPU resources.
- **Runnable** : In this state, a thread might actually be running or it might be ready to run at any instant of time. It is the responsibility of the thread scheduler to assign CPU time to the thread.
- **Blocked** : A thread might be in this state, when it is waiting for I/O operations to complete. When blocked, a thread cannot continue its execution any further until it is moved to the runnable state again. It will not consume any CPU time in this state. The thread scheduler is responsible for reactivating the thread.

## Initializing with class
```cpp
#include <iostream>
#include <threads>

class Vehicle
{
  private:
    int _id;
  public:
    Vehicle(int id)
    {
        _id = id;
    }
    void operator () ()
    {
      std::cout << "Vehicle object #" << _id << " has been created\n";
    }
};

int main(int argc, char ** argv)
{
  std::thread t1((Vehicle(1)));              // Add an extra pair of parantheses
  std::thread t2 = std::thread(Vehicle(2));  // Use copy initialization
  std::thread t3 {Vehicle(3)};               // Use uniform initialization with braces

  std::cout << "Finished work in main\n";

  if(t1.joinable())
    t1.join();
  if(t2.joinable())
    t2.join();
  if(t3.joinable())
    t3.join();

  return 0;
}
```

## Promises and Futures

Sending ends of thread are called `Promises` and the receiving end of the thread are called `Future`.

**From parent thread to worker threads**

- Function Collaborator (class)
- Variadic Template
- Member Function
- Using lambda

**Lambda Function**

Syntax of lambda function.

```cpp
#include <iostream>

int main(int argc, char ** argv)
{
    int id = 5;
    void f1 = [&id] () mutable -> void { std::cout << "I have captured:" << id << std::endl};
    f1();
    return 0;
}
```

Symbol | Explanation
--- | ---
[] | Capturing variable
[id] | use the variable name to capture that variable by value
[=] | use = to capture all variables by value
[&id] | use & with variable name for capturing that variable by reference
[&] | use only & to capture all variables by reference
[&, id] | Capture all by reference, except id by value
[&id, id2] | capture id by reference and id2 by value

Symbol | Explanation
--- | ---
() | Accepting variable, just like a function

Symbol | Explanation
--- | ---
-> void | this is option, only needed if you want to specify the return type
mutable -> | lambda function generally do not change the value taken from [], however, you make set it as mutable so that the value capture by [] can be changed

**From worker thread to parent thread**

- Using promises and futures

**Example 1**

Usage of promises and futures. However, the drawback of future and promises are they are of one time use only.

```cpp
#include <iostream>
#include <thread>
#include <future>

void modifyMessage(std::promise<std::string> && prms, std::string message)
{
    std::this_thread::sleep_for(std::chrono::milliseconds(4000)); // simulate work
    std::string modifiedMessage = message + " has been modified"; 
    prms.set_value(modifiedMessage);
}

int main(int argc, char ** argv)
{
    // define message
    std::string messageToThread = "My Message";

    // create promise and future
    std::promise<std::string> prms;
    std::future<std::string> ftr = prms.get_future();

    // start thread and pass promise as argument
    std::thread t(modifyMessage, std::move(prms), messageToThread);

    // print original message to console
    std::cout << "Original message from main(): " << messageToThread << std::endl;

    // retrieve modified message via future and print to console
    std::string messageFromThread = ftr.get();
    std::cout << "Modified message from thread(): " << messageFromThread << std::endl;

    // thread barrier
    if(t.joinable())
      t.join();

    return 0;
}
```

**Example 2**

Usage of promises and future with exception and error catching.

```cpp
#include <iostream>
#include <thread>
#include <future>
#include <cmath>
#include <memory>

void divideByNumber(std::promise<double> &&prms, double num, double denom)
{
    std::this_thread::sleep_for(std::chrono::milliseconds(500)); // simulate work
    try
    {
        if (denom == 0)
            throw std::runtime_error("Exception from thread: Division by zero!");
        else
            prms.set_value(num / denom);
    }
    catch (...)
    {
        prms.set_exception(std::current_exception());
    }
}

int main(int argc, char ** argv)
{
    // create promise and future
    std::promise<double> prms;
    std::future<double> ftr = prms.get_future();

    // start thread and pass promise as argument
    double num = 42.0, denom = 0.0;
    std::thread t(divideByNumber, std::move(prms), num, denom);

    // retrieve result within try-catch-block
    try
    {
        double result = ftr.get();
        std::cout << "Result = " << result << std::endl;
    }
    catch (std::runtime_error e)
    {
        std::cout << e.what() << std::endl;
    }

    // thread barrier
    if(t.joinable())
      t.join();

    return 0;
}
```

- Using task (async task are designed to replace simple threading application, for application where mutex and locks are use, please use std thread to do so.)

**Example 1**

Creating a std::async task. You can see that now the function pass into the thread can have a return type which is very similar to a normal function that we would define.

```cpp
#include <iostream>
#include <thread>
#include <future>
#include <cmath>
#include <memory>


double divideByNumber(double num, double denom)
{
  std::this_thread::sleep_for(std::chrono::milliseconds(500));
  if(denom == 0)
    throw std::runtime_error("Exception from thread: Division by zero");
  return num / denom;
}

int main(int argc, char ** argv)
{
  double num = 88.0, denom = 2.0;
  std::future<double> ftr = std::async(divideByNumber, num, denom);
  try
  {
    double result = ftr.get();
    std::cout << "Result = " << restul << std::endl;
  }
  catch(std::runtime_error e)
  {
    std::cout << e.what() << std::endl;
  }
  return 0;
}
```

**Example 2**

Create a task by choosing whether to use `std::launch::async` or `std::launch::deferred`.

`std::launch::async` will create a thread for that function.

`std::launch::deferred` will only be creating a thread when it is called to do so.

```cpp
#include <iostream>
#include <thread>
#include <future>
#include <cmath>
#include <memory>

double divideByNumber(double num, double denom)
{
    std::cout << "Worker thread id = " << std::this_thread::get_id() << std::endl;
    std::this_thread::sleep_for(std::chrono::milliseconds(500));

    if(denom == 0)
    {
        throw std::runtime_error("Exception from thread#: Division by zero!");
    }
    return num / denom;
}

int main(int argc, char ** argv)
{
    std::cout << "Main thread id = " << std::this_thread::get_id() << std::endl;
    double num = 42.0, denom = 2.0;
    // Choose whether you want to create with async or deferred
    std::future<double> ftr = std::async(std::launch::deferred, divideByNumber, num, denom);

    try
    {
        double result = ftr.get();
        std::cout << "Result = " << result << std::endl;
    }
    catch(std::runtime_error e)
    {
        std::cout << e.what() << std::endl;
    }
    return 0;
}
```

**Example 3**

Let the computer decide whether to use `std::launch::async` or `std::launch::deferred`.

```cpp
#include <iostream>
#include <thread>
#include <future>
#include <cmath>
#include <vector>
#include <chrono>

void workerFunction(int n)
{
    // print system id of worker thread
    std::cout << "Worker thread id = " << std::this_thread::get_id() << std::endl;

    // perform work
    for (int i = 0; i < n; ++i)
    {
        sqrt(12345.6789);
    }
}

int main()
{
    // print system id of worker thread
    std::cout << "Main thread id = " << std::this_thread::get_id() << std::endl;

    // start time measurement
    std::chrono::high_resolution_clock::time_point t1 = std::chrono::high_resolution_clock::now();
    
    // launch various tasks
    std::vector<std::future<void>> futures;
    int nLoops = 10, nThreads = 5;
    for (int i = 0; i < nThreads; ++i)
    {
        futures.emplace_back(std::async(std::launch::async | std::launch::deferred, workerFunction, nLoops));
    }

    // wait for tasks to complete
    for (const std::future<void> &ftr : futures)
        ftr.wait();

    // stop time measurement and print execution time
    std::chrono::high_resolution_clock::time_point t2 = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::microseconds>( t2 - t1 ).count();
    std::cout << "Execution finished after " << duration <<" microseconds" << std::endl;
    
    return 0;
}
```

## Data Racing

Data racing happens when one thread is writing to a resource while another thread is reading from the resource at the same time.


## CMakeLists.txt

Simple program:
```cpp
#include <iostream>
#include <thread>


void threadFunction()
{
    std::this_thread::sleep_for(std::chrono::milliseconds(100)); // Sleep as a simulation of workd
    std::cout << "Finished work in thread\n";
}

int main(int argc, char ** argv)
{
    // Create thread
    std::thread t(threadFunction);

    // Do something in this thread
    std::this_thread::sleep_for(std::chron::milliseconds(50));
    std::cout << "Finished work in main\n";

    // Wait for thread to finish, do check if it is joinable first
    if(t.joinable())
    {
        t.join();
    }

    return 0;
}
```

The CMakeLists.txt will be as follow:
```cmake
cmake_minimum_required(VERSION 3.17)

# Use the CMakeLists.txt's parent directory name as the project id and name
get_filename_component(PROJECT_ID ${CMAKE_CURRENT_SOURCE_DIR} NAME)
string(REPLACE " " "_" PROJECT_ID ${PROJECT_ID})
project(${PROJECT_ID})

find_package(Threads)

add_executable(test main.cpp)

target_link_libraries(test ${CMAKE_THREAD_LIBS_INIT})
```

# Reference
- 5 ways to create a thread [link](https://www.youtube.com/watch?v=hCvc9y39RDw&list=PLk6CEY9XxSIAeK-EAh3hB4fgNvYkYmghp&index=2)