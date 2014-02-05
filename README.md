Java Good Practices
===================
This GitHub repo is meant to gather a maximum of Java good practices. So, feel free to add/correct practices. 

# General good practices
## Maximum number of threads
How many threads can I create and start? ... it is very hard to answer this question ... 
But, there might be some limits imposed by your operating system and hardware configuration.

This number is mainly impacted by the stack size of each thread. This size can be changed with the `-Xss` JVM parameter. If threads do not use a large amount of data, and if you have 2GB of addressable memory, simply divide these 2GB by the `-Xss` value. For Linux, the `ulimit -a` will give some informations.

## Consuming queues in a perfect and ideal world
Lets consider an application offering a simple `ping` service. Anybody is allowed to post a hostname in a queue and the application retreives a hostname from this queue and ping it one time. Assuming the mean ping time equals to 100ms and the application consumes the queue using 10 threads - each thread is in charge to ping only one host - what the theoretical optimal input rate of the queue ? In other terms, how many hostnames can be consumed per second. 

In this case, the answer is quite trivial - 10/0.1. The optimal input rate is 100 hostnames per second. Each thread will handle at most 10 hostnames per second.  

In order to guarantee a constant memory usage - a bounded queue length, the mean input rate must be smaller or equal to the mean output rate. 

[More details](http://en.wikipedia.org/wiki/Queueing_theory)

# Constructors
## Don't pass 'this' out of a constructor
Within a class, the 'this' Java keyword refers to the native object, the current instance of the class. Within a constructor, you can use the this keyword in 3 different ways:
* on the first line, you can call another constructor, using `this(...)`;
* as a qualifier when referencing fields, as in `this.fName`
* as a reference passed to a method of some other object, as in `blah.operation(this);`
You can get into trouble with the last form. The problem is that, inside a constructor, the object is not yet fully constructed. An object is only fully constructed after its constructor completely returns, and not before. But when passed as a parameter to a method of some other object, the `this` reference should always refer to a fully-formed object.

[More details](http://www.javapractices.com/topic/TopicAction.do?Id=252)

## Constructors shouldn't start threads
Constructors shouldn't start threads. According to Java Concurrency in Practice, by Brian Goetz and others (an excellent book), there are two problems with starting a thread in a constructor (or static initializer):
* in a non-final class, it increases the danger of problems with subclasses
* it opens the door for allowing the this reference to be passed to another object before the object is fully constructed
There's nothing wrong with creating a thread object in a constructor or static initializer - just don't start it there.

The alternative to starting a thread in the constructor is to simply start the thread in an ordinary method instead.

[More details](http://www.javapractices.com/topic/TopicAction.do?Id=254)

## Lazy initialization
Lazy initialization is a performance optimization. It's used when data is deemed to be 'expensive' for some reason. For example:
* if the hashCode value for an object might not actually be needed by its caller, always calculating the hashCode for all instances of the object may be felt to be unnecessary.
* since accessing a file system or network is relatively slow, such operations should be put off until they are absolutely required.
Lazy initialization has two objectives:
* delay an expensive operation until it's absolutely necessary
* store the result of that expensive operation, such that you won't need to repeat it again
As usual, the size of any performance gain, if any, is highly dependent on the problem, and in many cases may not be significant. As with any optimization, this technique should be used only if there is a clear and significant benefit.

To avoid a `NullPointerException`, a class must self-encapsulate fields that have lazy initialization. That is, a class cannot refer directly to such fields, but must access them through a method.

The `hashCode` method of an immutable Model Object is a common candidate for lazy initialization.

[More details](http://www.javapractices.com/topic/TopicAction.do?Id=34)

# Threads
## Use finally to unlock
Sometimes `Lock` objects are used to control access to mutable data in a multithreaded environment. When a thread is finished with the data, you must ensure the lock is released.
Brian Goetz in [Java Concurrency in Practice](http://www.amazon.com/exec/obidos/ASIN/0321349601/ref=nosim/javapractices-20) strongly recommends the following technique for ensuring that a lock is released. It uses a `try..finally` style; this ensures that no matter what happens during the time the lock is held, it will eventually be unlocked. Even if an unexpected RuntimeException is thrown, the lock will still be released.
```java
Lock myLock = ...
myLock.lock();
try {
  //interact with mutable data
}
finally{
  myLock.unlock();
}
```

[More details](http://www.javapractices.com/topic/TopicAction.do?Id=253)

## Read-write locks
In the great majority of database applications, the frequency of read operations greatly exceeds the frequency of write operations. This is why databases implement read-write locks for their records, which allow for concurrent reading, but still demand exclusive writing. This can markedly increase performance.
Occasionally, a class may benefit as well from a read-write lock, for exactly the same reasons - reads are much more frequent than writes. As usual, you should measure performance to determine if a read-write lock is really improving performance.

[More details](http://www.javapractices.com/topic/TopicAction.do?Id=118)


## Handle InterruptedException
In thread-related code, you will often need to handle an InterruptedException. There are two common ways of handling it:
* just throw the exception up to the caller (perhaps after doing some clean up)
* call the interrupt method on the current thread
In the second case, the standard style is:
```
try {
  ...thread-related code...
}
catch(InterruptedException ex){
  ...perhaps some error handling here...
  //re-interrupt the current thread
  Thread.currentThread().interrupt();
}
```
### Background
In multi-threaded code, threads can often block; the thread pauses execution until some external condition is met, such as:
* a lock is released
* another thread completes an operation
* some I/O operation completes
* and so on...
Threads can be interrupted. An interrupt asks a thread to stop what it's doing in an orderly fashion. However, the exact response to the interrupt depends on the thread's state, and how the thread is implemented:
```
if the thread is currently blocking 
  stop blocking early
  throw InterruptedException
else thread is doing real work
  thread's interrupted status is set to true
  if thread polls isInterrupted() periodically
    (which is preferred)
    orderly cleanup and stop execution
    throw InterruptedException
  else 
    regular execution continues
```
The main idea here is that an interrupt should not shutdown a thread immediately, in the middle of a computation. Rather, a thread and it's caller communicate with each other to allow for an orderly shutdown.

[More details](http://www.javapractices.com/topic/TopicAction.do?Id=251)

## Prefer modern libraries for concurrency
Version 1.5 of the JDK included a new package called `java.util.concurrent`. It contains modern tools for operations involving threads. You should usually prefer using this package instead of older, more low-level techniques.
The java.util.concurrent package is fairly large. The best guide to its contents is likely the excellent book [Java Concurrency in Practice](http://www.amazon.com/exec/obidos/ASIN/0321349601/ref=nosim/javapractices-20), written by Brian Goetz and other experts. The following is a very brief sketch of the most important elements in `java.util.concurrent`.

The most important idea is the separation of tasks and the policy for their execution. A task (an informal term) comes in two flavours:
* java.lang.Runnable - a task which doesn't return a value (like a void method).
* Callable - a task which can return a value (this is the more flexible form)
For executing tasks, there are 3 important interfaces, which are linked in an inheritance chain. Starting at the top level, they are:
* Executor - a single execute method
* ExecutorService - common workhorse; submit and invoke methods; includes lifecycle methods
* ScheduledExecutorService - executes tasks after a specific time interval, or periodically
The Executors factory class returns implementations of ExecutorService and ScheduledExecutorService.
When you need to be informed of the result of a task after it completes, you will likely use these items:
* Future - see the result of a task, or cancel it
* CompletionService - accepts tasks, and returns their Futures
* ExecutorCompletionService - an implementation of CompletionService
Finally, the following are used to synchronize between threads:
* CountDownLatch
* CyclicBarrier
* FutureTask

Concurrency uses many specialized terms. Some definitions:

* user thread, daemon thread - in Java, the JRE terminates when all user threads terminate; a single user thread will prevent the JRE from terminating. A * daemon thread, on the other hand, will not prevent the JRE from terminating.
* blocking - if a thread pauses execution until a certain condition is met, then it's said to be blocking.
* safety - when nothing unpleasant happens in a multithreaded environment (this is admittedly vague).
* liveness - execution continues, eventually; there may be significant pauses in a thread, but the application never completely hangs.
* deadlock - execution hangs, since a pair of threads each holds (and keeps) a lock that the other thread needs in order to continue.
* race condition - if you are unlucky with the timing of how threads interleave their execution, then bad things will happen.
* re-entrant - if a thread already holds a re-entrant lock, and if it needs to re-acquire the lock, then the re-acquisition always succeeds without blocking. Most locks are re-entrant, including Java's intrinsic locks (see below).
* intrinsic lock - the built-in locks of the Java language. These correspond to uses of the synchronized keyword.
* mutex lock - mutually exclusive locks are held by at most one thread at a time. For example, intrinsic locks are mutexes. Read-write locks are not mutexes, since they allow N readers to access the same data at once.
* thread confinement - data that is only accessed by a single thread is always safe, since it's confined to a single thread.
* semaphore - object pools containing a limited number of resources use a semaphore to keep track of how many of the resources are currently in use.

Here's the lifecycle of a task submitted to an Executor:
* created - the task object has been created, but not yet submitted to an Executor.
* submitted - the task has been submitted to an Executor, but hasn't been started yet. In this state, the task can always be cancelled.
* started - the task has begun. The task may be cancelled, if the task is responsive to interruption.
* completed - the task has finished. Cancelling a completed task has no effect.

[More details](http://www.javapractices.com/topic/TopicAction.do?Id=248)

# Collections 
## `ConcurrentHashMap`
Always use `ConcurrentHashMap` instead of `HashMap` because:
* the `ConcurrentHashMap` is natively thread-safe
* the `ConcurrentHashMap` is faster than the `HashMap` even in mono-threaded environment

[More details](http://www.codercorp.com/blog/java/why-concurrenthashmap-is-better-than-hashtable-and-just-as-good-hashmap.html)

## `String` keys in `HashMap`
Avoid `String` keys in `HashMap`because:
* using a `String` as key does not avoid collision
* the `hashCode()` of a `String` is not final (recomputed each time)

[More details](http://stackoverflow.com/questions/1516549/bad-idea-to-use-string-key-in-hashmap)

# Logging
*NB: Keep in mind that all the logs are stored in files and these files size could rapidly grow!*

## Use Guarded Logging
"Guarded logging" is a pattern that checks to see if a log statement will result in output before it is executed. Since the logger itself makes this check, it may seem redundant to do this for every call. However, as almost every logging call creates String objects, it is critical. The accumulation of these Strings causes memory fragmentation and unnecessary garbage collection. Garbage collection, while beneficial, also produces significant performance loss. To reduce this as much as possible, always guard logging statements with a severity less than SEVERE. Since SEVERE is always expressed by the logger, these do not need a guard. Here is an example:
```java
if(LOG.isDebugEnabled()){
     LOG.debug("This is " + "a best practice " + "example");
}
```

## Use `StringBuilder` and `toString()` or `toPrint()`
Sometimes, when you want to print all the values of an object, it is helpful to use a `StringBuffer` or `StringBuilder`. In this way, you can assemble all of the data into one clean output statement and avoid cluttering up your log file with redundant data (i.e. multiple lines with several time and level stamps for just one object). The snippet below shows an example of how to do this, but it is still not the best approach.
```java
if(LOG.isDebugEnabled()){
    StringBuffer sb = new StringBuffer();
    sb.append("\n           label: " + cat.getLabel());
    sb.append("\n    categoryType: " + cat.getCategoryType());
    sb.append("\n       attribute: " + cat.getAttribute());
    sb.append("\n           value: " + cat.getValue());
    sb.append("\n      Child Cats: " + cat.getCategories().toString());
    sb.append("\n           links: " + cat.getLinks().toString());
    LOG.debug(">> saveCategory(...)-> passing this cat: " + sb.toString());
}
```
While the code above creates a clean logging statement without multiple lines containing redundant data, there is still a problem. Notice that we are printing the values of the object "cat" (of the class Category). What would be better is to move most of this code into a `toString()` method on the Category class itself. In this way, all developers can simply call the `toString()` method to get a print of all the object's current data. Sometimes, however, you might find that you'd rather use a `toString()` method for application functionality instead of for logging or because the `toString()` method is already being used in another way. In this case, you could create a method called `toPrint` instead. Following is an example:
```java
/**
* @return a String representation of this object on multiple lines
*         (primarily to be used for debugging and logging)
*/
public String toPrint() {
    StringBuilder sb = new StringBuilder();
 
    if (categories != null) {
        sb.append("\n categories.size: " + categories.size());
    } else {
        sb.append("\n      categories: null");
    }
    if (links != null) {
        sb.append("\n      links.size: " + links.size());
    } else {
        sb.append("\n           links: null");
    }
    sb.append("\n         ownerID: " + this.ownerID);
    sb.append("\n          userID: " + this.userID);
    sb.append("\n       attribute: " + this.attribute);
    sb.append("\n           value: " + this.value);
    sb.append("\n          locale: " + this.locale);
    sb.append("\n    categoryType: " + this.categoryType);
 
    return sb.toString();
}
```
Notice also in the code snippet above that we have taken care to align all of the colons separating the key from value. This makes the `toPrint()` method's output much easier to read. Also notice that we've added a newline character to the beginning of each property statement that appends to the `StringBuffer`.

# Overriding Object Methods
## Never rely on `finalize`
You cannot rely on finalize to always reclaim resources. Instead, if an object has resources which need to be recovered, it must define a method for that very purpose - for example a close method, or a dispose method. The caller is then required to explicitly call that method when finished with the object, to ensure any related resources are recovered.
With JDK 7+, the try-with-resources feature can be used to reclaim resources for any object that implements AutoCloseable.

The `System.runFinalizersOnExit(boolean)` method is deprecated. 

[More details](http://www.javapractices.com/topic/TopicAction.do?Id=24)


# Exceptions
## Be specific in throws clause
In the throws clause of a method header, be as specific as possible. Do not group together related exceptions in a generic exception class - that would represent a loss of possibly important information.
An alternative is the exception translation practice, in which a low level exception is first translated into a higher level exception before being thrown out of the method.

[More details](http://www.javapractices.com/topic/TopicAction.do?Id=27)

## Checked versus unchecked exceptions
### Unchecked exceptions:
* represent defects in the program (bugs) - often invalid arguments passed to a non-private method. To quote from The Java Programming Language, by Gosling, Arnold, and Holmes: "Unchecked runtime exceptions represent conditions that, generally speaking, reflect errors in your program's logic and cannot be reasonably recovered from at run time."
* are subclasses of RuntimeException, and are usually implemented using IllegalArgumentException, NullPointerException, or IllegalStateException
* a method is not obliged to establish a policy for the unchecked exceptions thrown by its implementation (and they almost always do not do so)

### Checked exceptions:
* represent invalid conditions in areas outside the immediate control of the program (invalid user input, database problems, network outages, absent files)
* are subclasses of Exception
* a method is obliged to establish a policy for all checked exceptions thrown by its implementation (either pass the checked exception further up the stack, or handle it somehow)
It's somewhat confusing, but note as well that RuntimeException (unchecked) is itself a subclass of `Exception` (checked).

[More details](http://www.javapractices.com/topic/TopicAction.do?Id=129)

## `Finally` and `catch`
The `finally` block is used to ensure resources are recovered regardless of any problems that may occur.
There are several variations for using the `finally` block, according to how exceptions are handled.

If you're using JDK 7+, then most uses of the finally block can be eliminated, simply by using a try-with-resources statement. If a resource doesn't implement AutoCloseable, then a finally block will still be needed, even in JDK 7+. Two examples would be:
* resources related to threads
* resources related to a Graphics object
JDK < 7, and resources that aren't `AutoCloseable`
The following examples are appropriate for older versions of the JDK. They are also appropriate in JDK 7+, but only for resources that don't implement `AutoCloseable`.

[More details](http://www.javapractices.com/topic/TopicAction.do?Id=25)













# Good practices sources
* http://www.javapractices.com/
* http://docs.oracle.com/
* http://stackoverflow.com/
* My own experience


  
