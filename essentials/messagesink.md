---
title: Message sinks!
tagline: A very small logging library
layout: article
toc: true
date: 12 Jan 2017
---

At one time, log4net was a very popular logging library for C#/.NET. Several years ago I decided to have a look at it - admittedly, without ever actually using it before - with the goal of producing a much smaller logging library within [Loyc.Essentials](http://core.loyc.net/essentials/) that provided the most commonly-used features of log4net.

That didn't work out. It turned out that log4net was not merely a _large_ library (larger than all of Loyc.Essentials), but it was also very complicated, a spaghetti of interwoven interfaces and dependencies. It was difficult to follow how it worked internally, and I wasn't sure where the useful "core" was that would retain compatibility with the most commonly-used features.

Instead I just designed a small logging library without regard for compatibility with log4net - something that would be useful _not only_ for logging but also for other situations where messages are produced, such as error messages in my compilers.

I call it the "message sink" - a drain you dump messages into. The word "sink" is consistent with the naming convention of other "sources" and "sinks" in Loyc.Essentials, e.g. `ICollectionSink<in T>` is a subset of `ICollection<T>` that lets you adding or removing items, but not get items out, while `IListSource<out T>` is like `IList<T>` but you can only get items, not add or remove items.

The interface
-------------

`IMessageSink` is a simple interface designed to make it easy to define your own implementations, so that you can easily extend it with more functionality as you need it - while balancing the need for good performance. 

	// Alias for IMessageSink<object>
	public interface IMessageSink : IMessageSink<object>
	{
	}
	public interface IMessageSink<in TContext>
	{
		/// <summary>Returns true if messages of the specified type will actually be 
		/// printed, or false if Write(type, ...) has no effect.</summary>
		bool IsEnabled(Severity type);
		
		/// <summary>Writes a message to a log or other target.</summary>
		void Write(Severity level, TContext context, string format);
		void Write(Severity level, TContext context, string format, object arg0, object arg1 = null);
		void Write(Severity level, TContext context, string format, params object[] args);
	}

Most people will just use `IMessageSink`, but you can also customize the meaning of the `context` parameter. Notice the `in` in `IMessageSink<in TContext>`: this means that any `IMessageSink<object>` is implicitly convertible to `IMessageSink<C>` for any class C. Because of this, you can, for example, use `ConsoleMessageSink` as if it were `IMessageSink<C>` even though it only implements `IMessageSink`.

The first method, `IsEnabled(c)`, lets you find out before printing a message whether that message could actually be printed (if it returns false, messages in category `c` are being filtered out.) This lets you avoid doing work to construct a message that will be discarded.

A message has four parts:

1. `Severity type`: an enum that indicates what kind of message this is (how "serious" or how "common") on a numeric scale. Commonly used values include `Severity.Error`, `Severity.Warning`, and `Severity.Debug`. 
2. `TContext context`: an object that represents the "context" or "location" that the message relates to. Typically `TContext = object`, so this parameter could be anything, and the exact meaning of the context can vary from application to application.
3. `string format`: a message to be logged, with optional argument placeholders like `{0}` and `{1}`.
4. format arguments: objects or strings to insert into the format string.

Now, `log4net` lets you write an object _or_ a string, but to me it made more sense to write an object _and_ a string, where the object provides some sort of "context" for the message. In addition to this interface, many log4net-like extension methods are provided that you can use as shortcuts (e.g. `Warn("It's cold outside!")`, which uses `level: Severity.Warning` and `context: null`.)

Only a single `Write()` method is _truly_ needed, but it is expected to be fairly common that a message sink will drop some or all messages without printing them, e.g. if a message sink is used for logging, verbose messages might be "off" by default. It would be wasteful to actually format a message if the message will not actually be printed, and it would even be wasteful to create an array of objects to hold the arguments if they are just going to be discarded. With that in mind, since most formatting requests only need a couple of arguments, there is an overload of `Write()` that accepts up to two arguments without the need to package them into an `params` array.

So there's three `Write`s:

- `Write(Severity, TContext, string)` for strings that can (and should) be written without performing substitution.
- `Write(Severity, TContext, string, object, object)` for messages with one or two arguments. If there is only one argument, the second defaults to `null`. Many other libraries have separate overloads for one, two and three arguments; `IMessageSink` only has one fixed-length overload because it is designed to be easy to _implement_ the interface, not just easy to consume it.
- `Write(Severity, TContext, string, params object[])` for cases with more than two arguments.

Message sinks _may_ perform localization using [`Localize.Localized()`](http://core.loyc.net/essentials/localize.html).

Basic sinks
-----------

The following "basic" sinks are built into Loyc.Essentials (the ones with `.Value` are singletons - you usually don't need to create new instances):

- `ConsoleMessageSink.Value`: writes messages to the console, with different colors for different levels (e.g. Red = error, Yellow = warning, Cyan = debug).
- `TraceMessageSink.Value`: calls `System.Diagnostics.Trace.WriteLine`. 
- `NullMessageSink.Value` (a.k.a. `MessageSink.Null`): discards all messages. However, there is a `Count` property that increases by one with each message received, and an `ErrorCount` of errors.
- `new MessageHolder()`: saves all messages in its `List` property.

`ConsoleMessageSink` and `TraceMessageSink` produce similar output; for example

    ConsoleMessageSink.Value.Write(Severity.Error, "Foo.csv", "Syntax error")

comes out as

    Error: Foo.csv: Syntax error

By default, `ConsoleMessageSink` (but not `TraceMessageSink`) leaves out the severity for lower-level messages (anything below `Warning`), so the text color alone indicates the `Severity`.

**Note**: Message sinks convert the context object to a string by calling `MessageSink.ContextToString`, see below.

Wrapper sinks
-------------

Some sink types are wrapper objects that modify an "inner" or "target" sink:

- `new SeverityMessageFilter(IMessageSink target, Severity minSeverity)`: filters out messages whose severity is below a minimum.
- `new MessageFilter(IMessageSink target, Func<Severity, bool> filter)`: lets you controls filtering for each `Severity` separately.
- `new MessageFilter(IMessageSink target, Func<Severity, object, string, bool> filter)`: lets you filter out messages based on the context and/or format string, as well as the Severity. When someone calls `MessageFilter.IsEnabled(Severity level)`, `MessageFilter` in turn calls `filter(level, null, null)`. The filter method can be changed after construction.
- `new MessageMulticaster(params IMessageSink[] targets)`: broadcasts a message to a list of sinks. The list can be edited after `MessageMulticaster` is constructed. `IsEnabled(level)` returns true if any of the targets return true for that level.
- `new MessageSinkWithContext(IMessageSink target, object context, string messagePrefix = null)`: sets the `context` parameter to the specified object if it is null when `Write()` is called. Also, if a message prefix is provided, it is concatenated with the format string before being passed to `target`. (optimization: if you specified a prefix and `target.IsEnabled` returns false, the method returns without doing anything.)

Comparison with log4net
-----------------------

log4net is typically configured via XML files. It would be nice if a volunteer would step up to add a similar feature to Loyc Core, but I don't personally need XML-based configuration, and in the interest of keeping Loyc.Essentials small, that feature would probably end up in Loyc.Utilities.dll (also on NuGet)

In log4net there is a convention of defining a static field in each of your classes to provide logging:

    private static readonly log4net.ILog log = log4net.LogManager.GetLogger
        (System.Reflection.MethodBase.GetCurrentMethod().DeclaringType);

You can do something similar with message sinks:

    private static readonly IMessageSink log = MessageSink.WithContext
        (System.Reflection.MethodBase.GetCurrentMethod().DeclaringType);

If you want to do Type-specific filtering (e.g. filtering out Debug messages in cerrtain types but not others), Loyc.Essentials doesn't currently support that directly; you'd need to write some custom code.

You'll also need `using Loyc` so that the extension methods are available.

This will send messages to the default message sink (`MessageSink.Default`) using the `Type` of the current class as the _default_ context parameter (when the context given to `Write` is null).

`IMessageSink` has a series of extension methods like these, which lets you use it like log4net:

	public static bool IsErrorEnabled<C>(this IMessageSink<C> sink)
	{
		return sink.IsEnabled(Severity.Error);
	}
	public static void Error(this IMessageSink<object> sink, string format)
	{
		sink.Write(Severity.Error, null, format);
	}
	public static void ErrorFormat(this IMessageSink<object> sink, string format, params object[] args)
	{
		sink.Write(Severity.Error, null, format, args);
	}
	public static void Error<C>(this IMessageSink<C> sink, C context, string format)
	{
		sink.Write(Severity.Error, context, format);
	}
	/* ...more Error methods... */
	
	public static bool IsWarnEnabled<C>(this IMessageSink<C> sink)
	{
		return sink.IsEnabled(Severity.Warning);
	}
	public static void Warn(this IMessageSink<object> sink, string format)
	{
		sink.Write(Severity.Warning, null, format);
	}
	public static void WarnFormat(this IMessageSink<object> sink, string format, params object[] args)
	{
		sink.Write(Severity.Warning, null, format, args);
	}
	public static void Warning<C>(this IMessageSink<C> sink, C context, string format)
	{
		sink.Write(Severity.Warning, context, format);
	}

	/* ...more Warn methods... */

The names `Warn` and `WarnFormat` come directly from log4net. 

**Note**: the methods called `ErrorFormat` (and `WarnFormat`, etc.), which do not take a context parameter, actually _cannot be called_ `Error` instead. If the method had been called `Error` rather than `ErrorFormat` then if you call

    messageSink.Error("", "");

the call would be ambiguous between `IMessageSink<C>.Error(C context, string format)` and `IMessageSink.Error(string format, object arg0)`. So be careful: **the word `Format` is needed to tell the compiler that there is no context parameter!** There is no way, unfortunately, to tell the compiler that the context will never be a string.

For warnings, specifically, `log4net` calls them `Warn` but I call them `Warning`. So I decided that _when providing a context parameter_, the method would be called `Warning`. This ensures that when you call `Warn` but you actually intended to call `WarnFormat`, you'll get a compiler error instead of calling the wrong method.

Customizing behavior
--------------------

Of course, you can always implement your own `IMessageSink` to get custom behavior. You can also quickly create a message sink without implementing the entire `IMessageSink` interface, by calling `MessageSink.FromDelegate`:

    var sink = MessageSink.FromDelgate(
        (level, context, fmt, args) => {}, 
        level => /* return true if level is enabled */);

You can set the default message sink by calling `MessageSink.SetDefault()`. This method returns a `using`-compatible structure so that if you don't want to change it permanently, you can change it temporarily. For example:

    // block all messages temporarily
    using (MessageSink.SetDefault(MessageSink.Null)) {
        DoSomething();
    }
    // old message sink is restored here

This is a case of the [Ambient Service Pattern](http://core.loyc.net/essentials/ambient-service-pattern.html).

Message sinks that need to convert the `context` to a string should do so by calling `MessageSink.ContextToString(context)`. This method's default behavior is to check if the object implements the `IHasLocation` interface:

    public interface IHasLocation // in namespace Loyc
    {
        object Location { get; }
    }

If it does, the `Location` property is called and the returned location is converted to a string; otherwise, `ToString()` is called on the context itself.

This is useful, for example, in my compilers like [Enhanced C#](http://ecsharp.net). The context is the _syntax tree_ that has the error, but the _location_ object represents a location in a source file where that syntax tree is located, so that error messages end up having a location like "Foo.ecs(123,21)".

If this behavior is not what you need, you can override it by calling `MessageSink.SetContextToString(context => ...)`.

Details
-------

Sometimes you'd like to add more details about a message that was just written. You can do that with a message sink with code like this:

    MessageSink.Write(Severity.Error, location, "Expected closing brace here");
    MessageSink.Write(Severity.ErrorDetail, openlocation, "(Opening brace was here)");

Each normal value of `Severity` is an even number, with an associated `Detail` severity that is one less. For example, `Severity.Warning` is 60 and `Severity.WarningDetail` is 59.

When using `SeverityMessageFilter`, you should prefer to use a `Detail` level as the `MinSeverity`:

    s = new SeverityMessageFilter(c.Sink, Severity.NoteDetail);

In anticipation that users would accidentally write `Severity.Note` when they mean `Severity.NoteDetail`, or `Severity.Warning` when they mean `Severity.WarningDetail`, `SeverityMessageFilter` has a third parameter that defaults to true:

    new SeverityMessageFilter(c.Sink, Severity.Warning, includeDetails: true);

This third parameter simply decrements the second parameter if the second parameter is an even number, so that details are included unless you specifically set that parameter to false. However, no such trickery affects the `SeverityMessageFilter.MinSeverity` property.

Other stuff
-----------

Messages that you store in a `MessageHolder` have type `LogMessage`. Its basic properties are `Severity`, `Context`, `Format` and `Args`, and there is also a `Formatted` property that localizes and formats the format string (not including the `Severity` and `Context`). The `ToString()` method, however, combines all four elements in one message by calling `MessageSink.FormatMessage()`.

The `LogException` exception takes four arguments just like a message sink:

    new LogException(severity, context, format, args)

The `severity` argument is not required; the default is `Severity.Error`.

`LogException` has a `Msg` property of type `LogMessage` where it stores this information.

Get it
------

In Visual Studio, you can use NuGet to install Loyc.Essentials which includes everything described here. The source code is [here](https://github.com/qwertie/ecsharp/tree/master/Core/Loyc.Essentials/MessageSinks). Thanks for reading!
