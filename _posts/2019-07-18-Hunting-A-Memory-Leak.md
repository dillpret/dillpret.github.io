---
layout: post
title: Hunting a Memory Leak
published: true
---
So the task was simple. I was sent an email telling me that our little service had eaten nearly 3GB of memory on the server. We had a leak, and it was my job to fix it. Being the sweet summer child that I am, this was my first encounter with a memory leak. This is what I found, and what I learned.

> Context: I was working in dotNet Framework, using Visual Studio 2019 as my IDE. We were using Castle Windsor for dependancy injection.

## Looking for clues

Firstly, I finally realised the value of a good memory profiler. I used *dotMemory* from JetBrains at the recommendation of a mentor. I also tried the memory profiler built into Visual Studio. It was more convenient, but didn't give me the same level of insight. A memory profiler allowed me to take snapshots of the program as it was running, and then view the differences between two snapshots in terms of allocated objects and so on. This allowed me to spot that there were waaaay too many instances of certain objects, giving me a place to start looking for problems.

> Side note: The usefulness of good namespace organisation shone through when faced with too much information. The profiling tool allowed for organising objects by namespace and this made the otherwise incomprehensible wall of object names fairly easy to filter and browse.

## The Disposable pattern

The trail lead me to my next big learning curve: the Disposable pattern. 
> The docs really helped me out here and I recommend reading them if you run into this.   
https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose

The pattern involves managing the deallocation of resources when there is a combination of managed and unmanaged resources. The variation used in our project involved overriding the Finalize method of an object and supressing normal garbage collection. 

Now normally, if your object uses a `Disposable` (ie. anything implementing `IDisposable`), you have two options. The first is to implement `IDisposable` in your object as well and call `Dispose()` on the dependancies when your object is itself disposed. This was the first problem I found. I had written code in a project using this pattern without properly understanding it, and so I had neglected to properly implement it everywhere. Going and finding everything I'd missed *after* the fact was a pain.

The second option, when dealing with Disposables, is to wrap your usage of the dependancy in a `using` statement, which calls `Dispose()` automatically on completion. For most of our solution we did the former, because it allows for these disposable dependancies to be injected and then disposed properly with the object itself. The exception was Factories...

Our factory classes were responsible for abstracting away the complexity of providing an instance of some object, and they did this by getting the relevant object straight from Castle Windsor, our dependancy injection library. We were generally using these objects immediately after getting them from the factories and then never again, so we simply put a `using` statement around them and called it a day.

## Castle Windsor

This brings me the main cause of the issue. **Castle Windsor keeps the reference to the object resolved by the factory and doesn't call Dispose at the end of a `using` statement as you would assume.** You have to remember to manually call `Release()` on every reference you get by calling `Resolve()`, especially if the dependancy has been configured to have a Transient lifestyle.

> Transient lifestyle means that every time an instance of the object is requested, a new one is created. Contrast this with a Singleton, for example, where there is only one instance of the object shared over the whole program.

I therefore fixed the leak by adding a method to the factories which would call `Release` on a given reference. The pattern for using them then became:
``` Csharp
var foo = factory.GetFoo();
foo.DoWork();
factory.ReleaseFoo(foo);
```

## TL;DR - Gotchas and Learnings

* Understanding how to use a memory profiling tool is crucial for finding a memory leak.
* If you see a pattern being used in your code base (the `Disposable` pattern, for instance), take the time to understand it *before* you start adding your own code.
* Disposables resolved by Castle Windsor don't respect the `using` statement.
