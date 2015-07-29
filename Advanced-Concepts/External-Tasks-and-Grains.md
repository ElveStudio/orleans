---
layout: page
title: External Tasks and Grains
---
{% include JB/setup %}

By design, any sub-Tasks spawned from grain code (for example, by using `await` or `ContinueWith` or `Task.Factory.StartNew`) will be dispatched on the same per-activation [TPL Task Scheduler](https://msdn.microsoft.com/en-us/library/dd997402(v=vs.110).aspx) as the parent task and therefore inherit the same single-threaded execution model as the rest of grain code. This is the main point behind single threaded execution of [grain turn based concurency](http://dotnet.github.io/orleans/Step-by-step-Tutorials/Concurrency).

In some cases grain code might need to “break out” of the Orleans task scheduling model and “do something special”, such as explicitly pointing a `Task` to a different task scheduler or using the .NET Thread pool. Example of such cases may be when the grain code has to execute a synchronous remote blocking call (such as remote IO). Doing that in the grain context will block the grain as well as one of the Orleans threads and thus should never be made. Instead, the grain code can execute this piece of blocking code on the thread pool thread and join (`await`) the completion of that execution and proceed in the grain context. We expect that escaping from Orleans scheduler will be a very advanced and seldom required usage scenario beyond the “normal” usage patterns.

### Task based APIs:

1) `await`, `Task.Factory.StartNew`, `Task.ContinuewWith`, `Task.WhenAny`, `Task.WhenAll`, `Task.Delay` all respect the current Task Scheduler. That means that using them in the default way, without passing a different TaskScheduler, will cause them to execute in the grain context.

2) Both `Task.Run` and the `endMethod` delegate of `Task.Factory.FromAsync` do NOT respect the current task Scheduler. They both use the `TaskScheduler.Default` scheduler, which is the .NET thread pool task Scheduler. Therefore, they will ALWAYS run on .NET thread pool outside of the single-threaded execution model for Orleans grains, [as detailed here](http://blogs.msdn.com/b/pfxteam/archive/2011/10/24/10229468.aspx). Any continuation of `await` code chained to them will run back under the “current” scheduler at the point the task was created, which is the grain context. 

3) `configureAwait(false)` is an explicit API to escape the current task Scheduler. It will cause the code after an awaited Task to be executed on the `TaskScheduler.Default` scheduler, which is the .NET thread pool and will thus break the single-threaded execution of Orleans grain. You should in general **never ever use `configureAwait(false)` in the grain code.**

4) Methods with signature `async void` should not be used with grains, they are intended for graphical user interface event handlers.

### Example:

Below is a sample code that demonstrates the usage of `TaskScheduler.Current`, `Task.Run` and a special custom scheduler to escape from Orlean grain context and how to get back to it.

``` csharp
   public Task MyGrainMethod()
   {
        // Grab the Orleans task scheduler
        var orleansTs = TaskScheduler.Current; 
        await TaskDelay(10000);
        // Current task scheduler did not change, the code after await is still running in the same task scheduler.
        Assert.AreEqual(orleansTs, TaskScheduler.Current); 
        
        Task t1 = Task.Run( () => 
        { 
             // This code runs on the thread pool scheduler, not on Orleans task scheduler
             Assert.AreNotEqual(orleansTS, TaskScheduler.Current);
             Assert.AreEqual(TaskScheduler.Default, TaskScheduler.Current); 
        } );
        await t1;
        // We are back to Orleans task scheduler, since await was executed in Orleans task scheduler context we are now back to that context.
        Assert.AreEqual(orleansTS, TaskScheduler.Current); 
        
        // Example of using ask.Factory.StartNew with a custom scheduler to escape Orleans scheduler
        Task t2 = Task.Factory.StartNew(() =>
        {
             // This code runs on MyCustomSchedulerThatIWroteMyself scheduler, not on Orleans task scheduler
             Assert.AreNotEqual(orleansTS, TaskScheduler.Current);
             Assert.AreEqual(MyCustomSchedulerThatIWroteMyself, TaskScheduler.Current); 
        },
        CancellationToken.None, TaskCreationOptions.None,
        scheduler: MyCustomSchedulerThatIWroteMyself);
        await t2;
        // We are back to Orleans task scheduler.
        Assert.AreEqual(orleansTS, TaskScheduler.Current); 
   }
```

### Advanced Example - making grain call from a code that runs on thread pool

There is sometimes an even more advanced scenario of a piece of code that needs to “break out” of the Orleans task scheduling model and run on a thread pool (or some other, non-Orleans context) but still needs an ability to make a grain call to another grain. If you will just try to make a grain call not from within Orleans context, you will get an exception telling you that you are "trying to send a message on a silo not from within grain and not from within system target (RuntimeContext is not set to SchedulingContext)". 

Below is a code that demonstrates how a grain call can be made from a piece of code that runs inside on a silo but not in the grain context.

``` csharp
   public Task MyGrainMethod()
   {
        // Grab the Orleans task scheduler
        var orleansTs = TaskScheduler.Current; 
        Task<int> t1 = Task<int>.Run( () => 
        { 
             // This code runs on the thread pool scheduler, not on Orleans task scheduler
             Assert.AreNotEqual(orleansTS, TaskScheduler.Current);
             // do what ever you need to do here. Now lets say you need to make a grain call.
             Task<int>  t2 = Task<int> .Factory.StartNew(() =>
             {
                // This code runs on Orleans task scheduler since we specified scheduler: orleansTs.
                Assert.AreEqual(orleansTS, TaskScheduler.Current);
             },
             CancellationToken.None, TaskCreationOptions.None,
             scheduler: orleansTs);
             int res = await t2;
             // This code runs back on the thread pool scheduler, not on Orleans task scheduler
             Assert.AreNotEqual(orleansTS, TaskScheduler.Current);
             // do more of what you need
             return res;
        } );
        
        int result = await t1;
        // We are back to Orleans task scheduler, since await was executed in Orleans task scheduler context we are now back to that context.
        Assert.AreEqual(orleansTS, TaskScheduler.Current); 
   }
```

## Summary

What are you trying to do?   | How to do it 
------------- | -------------
Run background work on .NET thread-pool threads. No grain code or grain calls allowed.  |  `Task.Run`
Run worker task from grain code with Orleans turn-based concurrency guarantees. | `Task.Factory.StartNew`  
Grain interface call | Method return types = `Task` or `Task<T>` 
Timeouts for executing work items  | `Task.Delay` + `Task.WhenAny` 
Use with `async`/`await` | The normal .NET Task-Async programming model. Supported & recommended  
 `configureAwait(false)` | Do not use. Will escape Orleans's scheduler