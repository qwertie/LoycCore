---
title: Dependency Injection and the Ambient Service Pattern
layout: article
toc: true
date: 8 Dec 2016
---

Dependency Injection (also called Inversion of Control) is an important technique to decouple classes and larger components of an application from each other. If you write a class `C` using Dependency Injection (DI), then `C` doesn't choose its own dependencies, and some other part of the program can do so on its behalf. This makes it possible to swap out a service that `C` needs for some other service without changing `C` itself, which may allow `C` to be used in new situations that had never been imagined when `C` was written.

Once you get used to it, dependency injection is a straightforward practice to use as you write your lower-level code. When a class `C` is designed for Dependency Injection (DI), it means that `C` itself avoids directly referring to other classes it depends on - and avoids requiring other classes to directly refer to it.

Explaining DI/IoC is not the main goal of this article (if it's new to you, [Google around](https://www.google.com.ph/search?q=dependency+injection+C%23)). But I will summarize some common "best practices":

- Prefer to hold references to _interfaces_ (or abstract classes) instead of _concrete classes_. This implies that most classes should implement an interface (e.g. `class C : IC`). Occasionally, the interfaces are placed in a separate, small assembly so that a third party can implement the interface without adding a reference to the large, main assembly.

- If `C` needs a single instance of an object that implements some interface called `IService`, the reference to that service should be provided to each `C` object through a constructor argument to `C`:

        IService _service;
        public C(IService service) { _service = service; }

    This is called constructor injection, and it makes `C` independent of whatever class implements `IService`. Occasionally, it's also useful that a single application can now create _some_ `C` objects that use a certain implementation of `IService`, and other `C` objects that use a _different_ implementation of `IService`.

- If `C` can _optionally_ use a service, or if it makes sense to switch to a different implementation of a service _after_ the `C` object is created, `C` can expose a reference to the service as a property instead of as a constructor argument (or in addition to the constructor argument). This is called property injection:

        IService Service { get; set; }

- If `C` needs to create objects that implement a certain interface `IThing`, its constructor can require a factory (such as `Func<IThing>`) for making those objects. Depending on the situation, `IThing` itself might contain a `Clone` or `New` method so that, given _one_ `IThing` object, anyone can create another `IThing` object.

        Func<IThing> _makeThing;
        public C(Func<IThing> thingFactorty) { _makeThing = thingFactorty; }

Often, in an app designed with DI, all the different components are wired together in a single place, if possible, so that you can look in one place to see how components are connected and dependent on each other. If calling all those constructors starts to become a chore, [dependency injection libraries](http://nugetmusthaves.com/Category/IoC) can help you wire your components together. I recommend looking for a small, lightweight library.

However, these basic forms of dependency injection become cumbersome when many classes need many different services at once. If a class uses 10 distinct services, it is annoying to write that big constructor and to provide all needed services as constructor parameters. Sometimes, when several services are somehow related to each other, you could bundle them together into bigger services:

~~~csharp
interface ICar {
    IEngine       Engine       { get; }
    ITransmission Transmission { get; } 
    IWheels       Wheels       { get; } 
}
~~~

But bundling doesn't make sense in every situation.

Pervasive Services
------------------

In this article I would like to focus on services that are "pervasive": services that are used in many different places and would have to be passed to a hundred different constructors if you want to use conventional constructor DI. Examples of pervasive services are localization (to provide French and Spanish translations), logging (almost any component might want to write a diagnostic message to a log), profiling (to gather performance statistics), or a configuration dictionary or tree (so end-users or admins can configure multiple components through a command-line, xml file or other source). In a compiler, a service for error/warning messages might be used in many different places and thus is "pervasive".

And consider file access: some classes will want to read and write "files", but if the component is used in a web browser or security sandbox, it may not have access to the file system and it would be nice to have away to swap in a new "I/O" service that redirects file system calls to something that makes more sense in the secure environment. Even if a process does have access to the file system, the program as a whole (as opposed to the specfic component `C`, which is blissfully unaware of its environment) may wish to override the location where `C` reads and writes its data. Personally, I've always wanted to write a "file system mutation simulator" for testing purposes, which would allow components to _think_ they have changed the file system, but it would actually just _simulate_ those changes, saving all changes to RAM rather than disk. The changes could then be reviewed afterward, and either committed or discarded.

Managing dependencies on so many different services may become such a chore that some people give up on DI, at least when it comes to the most frequently-used "pervasive" services. But that makes it harder to adapt components like `C` to a new environment.

One solution to this problem is to hard-code, inside each component like `C`, a reference to a specific DI framework (or "IoC Container") so it can look up services that were not passed to the constructor. This practice is often taken to its limit, with the result that `C` has no constructor arguments at all. When components rely on a DI framework directly and exclusively, we call it the "service locator pattern", but [it is an antipattern](http://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/) that should be avoided because it's hard for people using `C` to understand `C`'s dependencies.

Another problem with using a "service locator" is that there are many different DI frameworks; taking a dependency on one _particular_ DI framework defeats the original goal of decoupling components from the services they use. Developers don't want to include four different DI frameworks in their project just because their components use four different DI frameworks. It wouldn't be so bad if there were some de-facto standard DI framework. Sadly, there is not. However, you should be aware that .NET [has had a built-in simple service locator since .NET 1.1](http://blog.differentpla.net/blog/2011/12/20/did-you-know-that-net-already-had-an-ioc-container) that implements  `IServiceProvider` and `IServiceContainer`. Also, Loyc.Essentials has a `ServiceProvider` class with tiny extension methods that make these interfaces generic.

The Ambient Service Pattern
---------------------------

So, how can we avoid the burden of passing around lots of pervasive service references? 

I can't name a good alternative for _all_ situations, but the _Ambient Service Pattern_ is one approach to providing services that can be replaced or swapped out, without a need to refer to any separate DI framework.

In general, implementing this pattern involves a thread-local variable to manage the "default" instance of the service:

~~~csharp
public interface IService { ... }

public class Service
{
    static ThreadLocal<IService> _default = new ThreadLocal<IService>();

    public static IService Default
    {
        get { return _default.Value; }
    }
    public static SavedThreadLocal<IService> SetDefault(IService newValue)
    {
        return new SavedThreadLocal<IService>(_default, newValue);
    }
}
~~~

It would be the responsibility of higher-level code (e.g. `Main()`) to call `SetDefault` at the beginning of the program so that `Default` doesn't return null.

Let's consider an example where this might make sense. In a compiler, the error/warning service might print to the console by default, or to an output window, but certain types of analysis might be "transactional" or "tentative": if an error occurs, the operation is aborted, and no error is printed, although if a warning occurs, it is buffered and printed if the operation succeeds.

A concrete example of this scenario is the C++ rule known as SFINAE. Template substitution may produce an error, but if that error occurs during overload resolution, it is not really an error and no error message should be printed. In a compiler, we can model this rule by switching to a different error service, performing the operation, and then switching back afterward. If our error/warning service is called [`IMessageSink`](http://ecsharp.net/doc/code/interfaceLoyc_1_1IMessageSink.html) (you know, a sink: a hole you can pour messages into) and it uses the Ambient Service Pattern, then we can temporarily disable the default message sink like this:

~~~csharp
  using (MessageSink.SetDefault(MessageSink.Null /* discard messages */)
  {
    PerformOverloadResolution(...);
  }
~~~

`SetDefault` relies on a helper struct for saving the old value of the `Default` property, so that the old value can be saved and restored by a `using` statement:

~~~csharp
public struct SavedThreadLocal<T> : IDisposable
{
    T _oldValue;
    ThreadLocal<T> _variable;

    public SavedThreadLocal(ThreadLocal<T> variable, T newValue)
    {
        _variable = variable;
        _oldValue = variable.Value;
        variable.Value = newValue;
    }
    public void Dispose()
    {
        _variable.Value = _oldValue;
    }

    public T OldValue { get { return _oldValue; } }
    public T Value { get { return _variable.Value; } }
}
~~~

Your high-level code can temporarily swap out a service like this:

~~~
  using (Service.SetDefault(/* get an IService object from somewhere */)) {
    /* some code that relies on the changed service */
  }
  /* original service is restored here */
~~~

Or it can permanently change the default service like this:

~~~
  Service.SetDefault(/* get an IService object from somewhere */);
  /* some code that relies on the changed service */
~~~

The `SavedThreadLocal` structure exists in LoycCore 2.2, and you can also just copy and paste it into any assembly that needs it (just put it into a unique namespace to avoid name collisions with other copies.)

As its name implies, the `Default` property is just a default instance of a service. There is nothing stopping a particular class from using constructor injection instead (taking an `IService` argument as a constructor argument). But you'll probably want consistency - either all your classes will take an `IService` constructor argument, or they will all use `Service.Default`.

The Ambient Service Pattern can be used for factories, too:

~~~
public class ServiceFactory
{
    static ThreadLocal<Func<IService>> _default = new ThreadLocal<Func<IService>>();

    public static Func<IService> Default
    {
        get { return _default.Value; }
    }
    public static SavedThreadLocal<Func<IService>> SetDefault(Func<IService> newValue)
    {
        return new SavedThreadLocal<Func<IService>>(_default, newValue);
    }
}
~~~

### Beware the .NET Framework ###

Normally, the Ambient Service Pattern is implemented with thread-local variables since each thread might be doing unrelated tasks that require different service objects.

Unfortunately, when you create a new thread (or `Task` or `BackgroundWorker`), **the CLR prohibits you from propagating thread-local data from the parent thread to the child thread**. In Loyc.Essentials I actually implemented a whole infrastructure to work around this problem. This involved [`ThreadEx`](http://ecsharp.net/doc/code/classLoyc_1_1Threading_1_1ThreadEx.html), a wrapper around `Thread`, and [`ThreadLocalVariable<T>`](http://ecsharp.net/doc/code/classLoyc_1_1Threading_1_1ThreadLocalVariable.html), an alternative to `ThreadLocal<T>` that works hand-in-hand with `ThreadEx` to propagate values from parent threads to child threads. This approach had a couple of problems, though:

1. Nowadays most people don't create threads using `Thread`; they use `Task<T>` or `BackgroundWorker`. There is no way to force `Task<T>` and `BackgroundWorker` to use `ThreadEx` instead of `Thread`. Sometimes `ThreadEx.PropagateVariables` can be used as a workaround, but it takes some effort to use correctly.
2. Theoretically, the state of an async task is somehow affiliated with [`ExecutionContext`](https://msdn.microsoft.com/en-us/library/system.threading.executioncontext(v=vs.110).aspx), not with threads, and tasks can (theoretically, but I don't know when or why) migrate between threads. Sadly, `ExecutionContext` does not capture thread-local variables and does not support any user-defined data, so if a task jumps between threads there is no way to take thread-local variables along for the ride.

Due to this serious limitation of the CLR/BCL, the Ambient Service Pattern works poorly in C# programs that juggle a lot of threads. Please [vote for Microsoft to fix this issue](https://visualstudio.uservoice.com/forums/121579-visual-studio-ide/suggestions/17356351-thread-local-variable-inheritance-or-executioncont).

### Summary ###

In summary, the Ambient Service Pattern is a useful way to with services that are widely used throughout an application, when

- You don't want your app's components to be hard coded with a particular implementation of the services they use
- You don't want your app's components to be hard coded to use a particular IoC container
- You don't want to clutter up your components with an excessive number of constructor parameters, and you judge it better to hard-code certain services rather than suffer from the extra effort you would need to thread those services through all the constructors in your program. Don't do that; use Ambient Service Pattern instead. **Note**: if .NET's lack of thread-local variable propagation will be a problem, consider using ordinary global variables instead. If you're using Loyc.Essentials, you can use `Holder<T>` and `SavedValue<T>` in place of `ThreadLocal<T>` and `SavedThreadLocal<T>`.

The Ambient Service Pattern is meant only for widely used background services. Ordinary dependency injection (such as constructor injection) should still be used in most cases.

### Ambient Service Pattern in Loyc libraries ###

Several Loyc components use the pattern.

**Loyc.Essentials.dll:**

- Internationalization: `Localize.Localizer` and `Localize.SetLocalizer`
- Internationalization formatting: `Localize.Formatter` and `Localize.SetFormatter`
- Logging (`IMessageSink`): `MessageSink.Default` and `MessageSink.SetDefault`

**Loyc.Syntax.dll:**

- `IParsingService`: `ParsingService.Default` and `ParsingService.SetDefault`
- `ILNodePrinter`: `LNode.Printer` and `LNode.SetPrinter`
- `Token`: `Token.ToStringStrategy` and `Token.SetToStringStrategy`

### P.S. ###

There is a project called [`SystemWrapper`](https://systemwrapper.codeplex.com/) that defines interfaces for .NET's static file system methods and for lots of other stuff in the .NET BCL. It doesn't use the Ambient Service Pattern, though.
