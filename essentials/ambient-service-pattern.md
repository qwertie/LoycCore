---
title: Dependency Injection and the Ambient Service Pattern
layout: article
toc: true
date: 8 Dec 2016
---

Dependency Injection (also called Inversion of Control) is an important technique to decouple classes and larger components of an application from each other. If you write a class `C` using Dependency Injection (DI), then `C` doesn't choose its own dependencies, and some other part of the program can do so on its behalf. This makes it possible to swap out a service that `C` needs for some other service without changing `C` itself, which may allow `C` to be used in new situations that had never been imagined when `C` was written.

Once you get used to it, dependency injection is a straightforward practice to use, especially in lower-level code - algorithms, data structures, database access, and so on. When a class `C` is designed for Dependency Injection (DI), it means that `C` does not directly refer to other classes it depends on.

Explaining DI/IoC isn't the main goal of this article, but I will summarize some common "best practices":

- Prefer to hold references to _interfaces_ (or abstract classes) instead of _concrete classes_. This implies that most classes should implement an interface (e.g. `class C : IC`). Occasionally, the interfaces are placed in a separate, small assembly so that a third party can implement the interface without adding a reference to the large, main assembly.

- If `C` needs a single instance of an object that implements some interface called `IService`, the reference to that service should be provided to each `C` object through a constructor argument to `C`:

        IService _service;
        public C(IService service) { _service = service; }

    This is called **constructor injection**, and it makes `C` independent of whatever specific class implements `IService`.
    
    Note that `C` is _not independent from the interface_, so ideally the interface should be specifically designed so that its underlying implementation can be replaced with something quite different. I've seen a lot of interfaces that are very, very specifically designed for one particular implementation, so that the underlying implementation cannot realistically be changed. This isn't _completely_ useless - at least it's still possible to make [Test Doubles](https://martinfowler.com/bliki/TestDouble.html) - but poor interface design can drastically reduce the reusability of your code.
    
    Occasionally, it's useful that a single application can now create _some_ `C` objects that use a certain implementation of `IService`, and other `C` objects that use a _different_ implementation of `IService`. This benefit usually disappears in apps that use service locators (see below).

- If `C` can use a service _optionally_, or if it makes sense to switch to a different implementation of a service _after_ the `C` object is created, `C` can expose a reference to the service as a property instead of as a constructor argument (or in addition to the constructor argument). This is called property injection:

        IService Service { get; set; }

- **Do not** use service locators (such as the MEF `CompositionContainer` or `Microsoft.Practices.ServiceLocation`). [They are an anti-pattern](https://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/) that [violates encapsulation](https://blog.ploeh.dk/2015/10/26/service-locator-violates-encapsulation/). Global variables are better, because at least your IDE can find references to global variables (in contrast, your IDE has no command for "find all cases where an `IService` was obtained through a service locator" and "find the implementation of `IService` assigned to our service locator"). Service locators hide dependencies, make programs harder to understand, and they tie your hands by limiting your ability to control the creation and distribution of objects. **Note:** even when you use constructor injection with MEF, it tends to cause difficulties. For example, errors are moved from compile-time to run-time, class constructors can no longer accept simple parameters such as strings or integers, nontrivial object creation policies (other than [the two policies MEF supports](CreationPolicy)) are hard to achieve, and developers have to spend valuable time reading about MEF's behavior. My advice is to avoid IoC containers entirely, or at least to avoid using them to manage lower-level classes. Don't let your IoC container affect the design or implementation of your algorithms, data structures, database access or business logic.

- If `C` needs to create objects that provide a certain interface `IThing`, its constructor can require a factory (such as `Func<IThing>`) for making those objects. Alternatively, `IThing` itself might contain a `Clone` or `New` method so that, given _one_ `IThing` object, anyone can create another `IThing` object.

        Func<IThing> _makeThing;
        public C(Func<IThing> thingFactory) { _makeThing = thingFactory; }

Often, in an app designed with DI, all the different components are wired together in a single place, if possible, so that you can look in one place to see how components are connected and dependent on each other.

**Note:** although this article is written for .NET coders, the principles discussed are general and can be applied to any language, from Java to Haskell.

Motivation
----------

Let me be clear. Whenever it is realistic to use constructor injection, you should.

However, best practices like this become cumbersome or unrealistic in some circumstances:

1. The code base is large and was not originally designed to follow modern best practices - in other words, it's a typical code base.
2. Management doesn't foresee that a common service will ever need to be replaced, and you know they would frown upon inserting extra plumbing for constructor injection throughout the codebase - you're on the clock, you have to work efficiently, and they are not going to set aside time to debate best practices. Management may, however, be willing to accept _easier_ forms of DI such as global variables or service locators that have a smaller footprint on the codebase.
3. Numerous classes use many distinct services (more than 4 or 5) and it is relatively cumbersome to store references to the services due to limitations of the programming language (e.g. Java/C++/C# require the developer to declare one constructor parameter and write at least two statements for each field to be initialized, in contrast with TypeScript which can do all three operations at once). In code that tries to minimize per-class responsibilities or follows the Single Responsibility Principle, this shouldn't happen a lot. The main reason a class may need many different services is that its responsibility is _plumbing_: it may not use any services itself, but it orchestrates the lifecycle and/or behavior of multiple other components. Each subcomponent only needs a subset of the services, and some subcomponents are themselve just orchestrating a smaller part of the system. In my experience, all but the lowest levels of an application tend to be rich in plumbing. The more plumbing code that must been written, the stronger the resistance will be to refactoring it, so if you can reduce the amount of such boilerplate, you can more easily shift to better designs.
4. There are could be up to a million (or more) instances of a particular class, so that storing pointers to its common services significantly increases memory usage. A good example of this is the `ToString` method of a [Loyc tree](http://loyc.net/loyc-trees) node or a [Token](http://ecsharp.net/doc/code/structLoyc_1_1Syntax_1_1Lexing_1_1Token.html). The string-conversion strategy will vary depending on the application or context in which a Loyc tree is used, but a compiler or IDE may need hold millions of nodes in memory at once, so storing a reference in each node to _any service whatsoever_ will significantly increase memory usage.
5. Installing the service through constructor injection would require extensive changes to the code, yet the service is only expected to be used temporarily or in special cases, so that the effort of adding constructor-injection plumbing does not seem worthwhile. An example of this would be a profiling-related service that is only used to add "probes" to help optimize a system.
6. (A) Existing third-party users would break if you demanded that they supply a new constructor argument, and you don't want to break them, or (B) you want to encourage usage of your service by third parties, so you do not want to burden users with a requirement to supply dependencies in order to use your service. For instance, perhaps you'd like a component factory to automatically record a time stamp on new instances without forcing the user to figure out how to supply it with a "current time service interface".
7. You are the author of a Standard Library for a programming language, which provides standard services available in all programs. You can avoid exposing functionality through (nonreplaceable) global methods, and you can make sure all your services implement an interface, but you can't force third parties to use constructor injection. However, the Ambient Service Pattern lets you help third parties make their own code more testable if they fail to use constructor injection.

The Ambient Service Pattern
---------------------------

### Introduction ###

The Ambient Service Pattern (or Ambient Context Pattern) involves using a thread-local or context-local variable to keep track of a "default" instance of a particular type of service or factory. The default service or factory can be changed temporarily at any time using a mechanism that will automatically restore the original service instance later.

This pattern should only be used when

1. Normal property/constructor injection is unwanted or inapplicable (see above), and
2. The service has a reasonable default implementation/behavior that is always available. This is important to ensure that your components will work when they are constructed in the ordinary way. If I can construct your component, and my program compiles, your component should just work, not require some other random things to be also initialized first.

Here are some examples of services that may appropriately use the pattern:

- Logging
- Localization
- Profiling (to gather performance statistics),
- A displayer of message boxes
- A dispatcher for delayed operations (message loop, thread pool, or timer)
- A configuration dictionary or tree
- A clock (current time provider)
- A reactivity or notification source (e.g. [Update Controls](http://updatecontrols.net))
- A factory for something very basic, like a frequently-used data structure
- Serializers for XML, JSON, protocol buffers, etc.
- File access

Note that in every single one of the cases above, most existing software _does not_ correctly use constructor injection to obtain these services.

In practice, "proper" dependency injection may be considered such a chore that some people give up on it, at least when it comes to the most frequently-used "pervasive" services. Don't give up - use Ambient Service Pattern instead.

### Details ###

Typically, implementing this pattern involves a thread-local variable (or an otherwise context-local variable) to manage the "default" instance of the service:

~~~csharp
public interface IService {
    // The pattern is the same regardless of which interface it wraps.
}

public class Service
{
    static ThreadLocal<IService> _default = new ThreadLocal<IService>(
        () => new VerySimpleImplementation(); // default value
    );

    public static IService Default => _default.Value;

    public static SavedThreadLocal<IService> SetDefault(IService newValue)
    {
        if (newValue == null)
            throw new ArgumentNullException("newValue");
        return new SavedThreadLocal<IService>(_default, newValue);
    }
}
~~~

`Default` should never return null, so that code which relies on it won't crash.

Your high-level code switches to another service implementation like this:

~~~csharp
  using (Service.SetDefault(/* get an IService object from somewhere */)) {
    /* some code that uses Service.Default */
  }
  /* original service is restored here */
~~~

In C#, `SetDefault` should use a helper struct for saving the old value of the `Default` property, so that the old value can be saved and restored by a `using` statement:

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

`SavedThreadLocal<T>` exists in LoycCore 2.2+, and you can also just copy and paste it into any assembly that needs it (just put it into a unique namespace to avoid name collisions with other copies.)

As its name implies, the `Default` property is just a default instance of a service. There is nothing stopping a particular class from using constructor injection instead (taking an `IService` argument as a constructor argument). But you'll probably want consistency within a given codebase - either all your classes will take an `IService` constructor argument, or they will all use `Service.Default`.

### Example scenario ###

Let's consider an example where this pattern might make sense. In a compiler, the error/warning service might print to Standard Output by default, but certain types of analysis might be "transactional" or "tentative": if an error occurs, the operation is aborted, and no error is printed, although if a warning occurs, it is buffered and printed if the operation succeeds.

A concrete example of this scenario is the C++ rule known as SFINAE. Template substitution may produce an error, but if that error occurs during overload resolution, it is not really an error and no error message should be printed. In a compiler, we can model this rule by switching to a different error service, performing the operation, and then switching back afterward. If our error/warning service is [`IMessageSink`](http://ecsharp.net/doc/code/interfaceLoyc_1_1IMessageSink.html) (you know, a sink: a hole you can pour messages into) which follows the Ambient Service Pattern, then we can temporarily disable the default message sink like this:

~~~csharp
  using (MessageSink.SetDefault(MessageSink.Null)) // discard messages
  {
    PerformOverloadResolution(...);
  }
~~~

Preferably you will not permanently change the default service, but C# allows it:

~~~csharp
  Service.SetDefault(/* get an IService object from somewhere */);
~~~

You _could_ use the Ambient Service Pattern this way, but _should_ you? Probably not. The `PerformOverloadResolution` method may as well just take accept an argument of type _IMessageSink_. This way, the fact that it depends on an _IMessageSink_ service is clear to someone reading your code. Up above I wrote a list of reasons you might use the pattern, and none of those reasons seem to apply in this situation.

On the other hand, if you need to construct a complicated graph of objects involving dozens of classes that all might produce output, and you know that it is appropriate for all of them to share the _default_ `IMessageSink`, this might be a reasonable time to use the Ambient Service Pattern, since

1. it greatly reduces the amount of plumbing code in your app, and 
2. it is easier for someone reading the code to see that these various classes are all sending output to the same place (with constructor injection, you can see that all classes have a reference to `IMessageSink`, but it's non-obvious whether the reference varies.)

### Problems with this pattern in .NET ###

The point of using a thread-local variable is so that temporary changes to the service do not affect other threads, since each thread might be doing unrelated tasks that require different service objects.

Unfortunately, when you create a new thread (or `Task` or `BackgroundWorker`), **the CLR prohibits you from propagating thread-local data from the parent thread to the child thread**. Instead, new threads just get a default value for each thread-local variable. In Loyc.Essentials I actually implemented a whole infrastructure to work around this problem. This involved [`ThreadEx`](http://ecsharp.net/doc/code/classLoyc_1_1Threading_1_1ThreadEx.html), a wrapper around `Thread`, and [`ThreadLocalVariable<T>`](http://ecsharp.net/doc/code/classLoyc_1_1Threading_1_1ThreadLocalVariable.html), an alternative to `ThreadLocal<T>` that works hand-in-hand with `ThreadEx` to propagate values from parent threads to child threads. 

This solution has two major shortcomings, though. Most obviously, most people don't create threads using `Thread` nowadays; they use `Task<T>` or `BackgroundWorker`. The solution "change Thread to ThreadEx" doesn't apply, and there is no way to force `Task<T>` or `BackgroundWorker` to use `ThreadEx` instead of `Thread`. Sometimes `ThreadEx.PropagateVariables` can be used as a workaround, but it takes some effort to use correctly.

Due to this limitation of the BCL, the Ambient Service Pattern won't work as intended in C# programs that often split work among threads. The simplest workaround for this is for the class providing the `Default` service to allow the global "fallback" object to be changed. So the service would offer a `SetDefaultFallback` method in addition to the `SetDefault` method.

Java solves the main problem here, as the JVM lets [thread-local values be inherited](https://docs.oracle.com/javase/7/docs/api/java/lang/InheritableThreadLocal.html) from a parent thread to a child thread, with an optional transformation. [Sun got a patent on this obvious idea](https://patents.google.com/patent/US6820261B1/en), so it's understandable that .NET doesn't support it, but Microsoft really went the extra mile with its lack of support. .NET completely hides the relationship between parent and child (you can't find out a child's parent thread ID or vice versa) and there is no event for thread creation.

The US patent was filed 1999-07-14 and granted a full five years later, but IIUC the [international 20-year term](https://en.wikipedia.org/wiki/Term_of_patent) for patents starts at the filing date, so the patent should have just expired. Are you hearing this, Microsoft? (No, of course you're not.)

A second problem is that .NET tasks can sometimes migrate between threads. The state of an async task may be somehow affiliated with [`ExecutionContext`](https://msdn.microsoft.com/en-us/library/system.threading.executioncontext(v=vs.110).aspx), but I haven't figured out how this works or under what circumstances thread-jumping occurs. Sadly, `ExecutionContext` does not capture thread-local variables and does not support any user-defined data, so if a task jumps between threads there is no way to take thread-local variables along for the ride.

There is another context class called [`CallContext`](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.remoting.messaging.callcontext?redirectedfrom=MSDN&view=netframework-4.8) which is "a specialized collection object similar to a Thread Local Storage for method calls and provides data slots that are unique to each logical thread of execution". But this class seems specifically designed to populate a side-channel during calls across AppDomains (remote procedure calls), and the documentation does not mention `Task` or `ExecutionContext` (or even WCF) so I would assume these slots are not transferred when a `Task` jumps across threads.

### Variations ###

The Ambient Service Pattern can be used for factories, too:

~~~csharp
public class ServiceFactory
{
    static ThreadLocal<Func<IService>> _default = new ThreadLocal<Func<IService>>(
        () => (() => new VerySimpleImplementation());
    );

    public static Func<IService> Default => _default.Value;

    public static SavedThreadLocal<Func<IService>> SetDefault(Func<IService> newValue)
    {
        if (newValue == null)
            throw new ArgumentNullException("newValue");
        return new SavedThreadLocal<Func<IService>>(_default, newValue);
    }
}
~~~

When the pattern is used for configuration information, it can be called by a more general name that I recently discovered, the [Ambient Context Pattern](https://aabs.wordpress.com/2007/12/31/the-ambient-context-design-pattern-in-net/):

~~~csharp
public class ServiceConfig
{
    static ThreadLocal<ServiceConfig> _defaultConfig = 
       new ThreadLocal<ServiceConfig>(() => new ServiceConfig());

    public static ServiceConfig Default => _default.Value;

    public static SavedThreadLocal<ServiceConfig> SetDefault(ServiceConfig newValue)
    {
        if (newValue == null)
            throw new ArgumentNullException("newValue");
        return new SavedThreadLocal<ServiceConfig>(_default, newValue);
    }
    ...
    ...
}
~~~

### Is this an anti-pattern? ###

[Some would argue](https://freecontent.manning.com/the-ambient-context-anti-pattern/) that I am describing an anti-pattern, but in practice this pattern is much better than what most developers are already doing, namely, hard-coding references to _specific implementations_ of services like "get the current time" (`DateTime.Now`), "show a message box _using WinForms_" (`MessageBox.Show(...)`) and "get a _Log4Net_ logger" (`log4net.LogManager.GetLogger(...)`).

As I argued before, constructor injection should be your preferred method of dependency injection, but there are situations when it is not practical and you are very tempted to use a global variable or an inflexible IoC container instead. If, after giving the matter some thought, you still prefer a global variable or some MEF thing over constructor injection, you should seriously consider using Ambient Service Pattern instead.

### Alternative to ASP: bundling ###

One way to lessen the burden of propagating services through the layers of your application is to bundle services together into bigger services, so that fewer arguments are passed around:

~~~csharp
interface IGuiHelpers {
    void MessageBox(string text, string title);
    MessageBoxResult MessageBox(string text, string title, MessageBoxButtons buttons);
    MessageBoxResult MessageBox(string text, string title, MessageBoxIcon icon);
    IProgressDisplayer ShowProgressDialog();
    string StringInputBox(string prompt, string title, string defaultValue);
    int NumericInputBox(string prompt, string title, int min, int max, int defaultValue);
    ...
}
~~~

This style of interface is bad, since it is not apparent which specific services a given component uses, and it will be harder for unit-test code to provide a dummy implementation.

A better approach to bundling services together into bigger services is to first unbundle them:

~~~csharp
public delegate MessageBoxResult MessageBoxFunc(
    string text, string title, 
    MessageBoxButtons buttons, MessageBoxIcon icon);
public delegate IProgressDisplayer ShowProgressDialogFunc();
public delegate string StringInputBoxFunc(string prompt, string title, string defaultValue);
public delegate int NumericInputBoxFunc(
    string prompt, string title,
    int min, int max, int defaultValue);

public struct GuiHelpers {
    public GuiHelpers(...) {...}
    public MessageBoxFunc MessageBox { get; }
    public ShowProgressDialogFunc ShowProgressDialog { get; }
    public StringInputBoxFunc StringInputBox { get; }
    public NumericInputBoxFunc NumericInputBox { get; }
}
~~~

Now if a class just needs one or two of the services, it can accept just those in its constructor, instead of a full bundle.

It's tempting to misuse bundles. For example, you might define a bundle of four services, and wind up writing classes that accept a bundle but only use two or three items from that bundle. Or you might extend the bundle with a new member, but most existing callers won't use the new item. A class demanding services that it doesn't use is a lesser sin than a class that hides its dependencies entirely behind a service locator, but it's misleading, makes one wonder if the class will, in the future, use services it doesn't use today, and creates ambiguity in unit tests (what should the unit tests assign, in the bundle, to service references that are not used by the unit being tested?)

Summary
-------

In summary, when you don't want your app's components to be hard coded with a particular implementation of the services they use, or to use a particular IoC container or library, but you don't want to clutter up your components with an excessive number of constructor parameters either, the Ambient Service Pattern can be used to reduce the plumbing burden for services that

1. May be widely used throughout an application
2. Have a reasonable default implementation that is always available

Ordinary dependency injection (such as constructor injection) should still be used in most cases. The Ambient Service Pattern is meant only for widely used "background" services.

Ambient Service Pattern in Loyc libraries
-----------------------------------------

Several Loyc components use the pattern.

**Loyc.Essentials.dll:**

- Internationalization: `Localize.Localizer` and `Localize.SetLocalizer`
- Internationalization formatting: `Localize.Formatter` and `Localize.SetFormatter`
- Logging (`IMessageSink`): `MessageSink.Default` and `MessageSink.SetDefault`

**Loyc.Syntax.dll:**

- `IParsingService`: `ParsingService.Default` and `ParsingService.SetDefault`
- `ILNodePrinter`: `LNode.Printer` and `LNode.SetPrinter`
- `Token`: `Token.ToStringStrategy` and `Token.SetToStringStrategy`

P.S.
----

There is a project called [`SystemWrapper`](https://systemwrapper.codeplex.com/) that defines interfaces for .NET's static file system methods and for lots of other stuff in the .NET BCL. It doesn't use the Ambient Service Pattern, though.
