---
title: Prepare all your apps for localization
tagline: No matter how lazy you are
layout: article
toc: false
date: 11 Jan 2017
---

Loyc.Essentials has a `Localize` class, which is a global hook into which a 
string-mapping localizer can be installed. If you're using Loyc.Essentials
anyway, you should use it. It makes your program ready for translation with 
virtually no effort.

The idea is to convince programmers to support localization by making it 
dead-easy to do. By default it is not connected to any translator (it just 
passes strings through), so people who are only writing a program for a 
one-language market can easily make their code "multiligual-ready" without 
doing any extra work.

All you do is call the `.Localized()` extension method, which is actually
shorter than writing the traditional `string.Format()`. (also: `using Loyc;`)

**Note**: Admittedly, you can't use it with C# 6 string interpolation like `$"this {...}"`
because C# seems to be hard-coded to translate string interpolation into calls 
to `string.Format`, whose behavior cannot be customized.

The translation system itself is separate from `Localize`, and connected
to `Localized()` by a delegate, so that multiple translation systems are 
possible. This class should be suitable for use in any.NET program, and
some programs using this utility will want to use different localizers.

Use it like this:

    string result = "Hello, {0}".Localized(userName);

Or, for increased clarity, use named placeholders:

    string result = "Hello, {person's name}".Localized("person's name", userName);

Whatever localizer is installed will look up the text in its database and
return a translation. If no translation to the end user's language is
available, an appropriate default translation should be returned: either the
original text, or a translation to some default language, e.g. English.

The localizer will need an external table of translations, conceptually like 
this:

| Key name      | Language | Translated text |
|---------------|----------|-----------------|
| "Hello, {0}"  | "es"     | "Hola, {0}"     |
| "Hello, {0}"  | "fr"     | "Bonjour, {0}"  |
| "Load"        | "es"     | "Cargar"        |
| "Load"        | "fr"     | "Charge"        |
| "Save"        | "es"     | "Guardar"       |
| "Save"        | "fr"     | "Enregistrer"   |

Many developers use a resx file to store translations. This is supported as explained below.

Localizing Longer Strings
-------------------------

For longer messages, it is preferable to use a short name to represent the
message so that, when the _English_ text is edited, the translation tables for _other_ languages do not have to change. To do this, use the `Symbol` method:

~~~csharp
// The translation table will be searched for "ConfirmQuitWithoutSaving"
string result = Localize.Symbol("ConfirmQuitWithoutSaving",
    "Are you sure you want to quit without saving '{filename}'?", "filename", fileName);

// Enhanced C# syntax with symbol literal
string result = Localize.Symbol(@@ConfirmQuitWithoutSaving,
    "Are you sure you want to quit without saving '{filename}'?", "filename", fileName);
~~~

This is most useful for long strings or paragraphs of text, but I expect
that some projects, as a policy, will use symbols for all localizable text.

Again, you can call this method without setting up any translation table.
However, the actual message is allowed to be `null`. In that case, if no 
translator has been set up or no translation is available, `Localize.Symbol`
returns the symbol itself (the first argument) as a last resort.

If the variable argument list is not empty, [`Localize.Formatter`](http://ecsharp.net/doc/code/classLoyc_1_1Localize.html#abf0bc6e4f5f4b69f3185d8ae11f3a43b) is called to build the completed string from the format string. It's possible to do formatting separately - for example:

    Console.WriteLine("{0} is {0:X} in hexadecimal".Localized(), N);

In this example, `WriteLine` itself does the formatting, instead of `Localized`.

As demonstrated above, `Localize`'s default formatter, `StringExt.FormatCore`, 
has an extra feature that the standard formatter doesn't: named arguments. 
Here is an example:

~~~csharp
...
string verb = (IsFileLoaded ? "parse" : "load").Localized();
MessageBox.Show(
    "Not enough memory to {load/parse} '{filename}'.".Localized(
      "load/parse", verb, "filename", FileName));
~~~

As you can see, named arguments are mentioned in the format string by
specifying an argument name such as `{filename}` instead of a number
like `{0}`. The variable argument list contains the same name followed 
by its value, e.g. `"filename", FileName`. This feature gives you, the
developer, the opportunity to tell the person writing translations what 
the purpose of a particular argument is.

The translator must not change any of the arguments: the word `{filename}`
is not to be translated.

At run-time, the format string with named arguments is converted to a
"normal" format string with numbered arguments. The above example would
become "Could not {1} the file: {3}" and then be passed to `string.Format`.

Design rationale
----------------

Many developers don't want to spend time writing internationalization or
localization code, and are tempted to write code that is only for one
language. It's no wonder, because it's a pain in the neck compared to 
hard-coding strings. Microsoft suggests that code carry around a 
[`ResourceManager`](https://msdn.microsoft.com/en-us/library/system.resources.resourcemanager(v=vs.110).aspx) object and directly request strings from it:

    private ResourceManager rm;
   
    rm = new ResourceManager("RootNamespace.Resources", this.GetType().Assembly);
   
    Console.Writeline(rm.GetString("StringIdentifier"));

This approach has drawbacks:

  * It may be cumbersome to pass around a `ResourceManager` instance between all
    classes that might contain localizable strings; a global facility is
    much more convenient.
  * The programmer has to put all translations in the resource file;
    consequently, writing the code is bothersome because the programmer has
    to switch to the resource file and add the string to it. Someone reading
    the code, in turn, can't tell what the string says and has to load up
    the resource file to find out.
  * It is not easy to change the localization manager; for instance, what if
    someone wants to store translations in an.ini, .xml or.les file rather 
    than inside the assembly? What if the user wants to centralize all
    translations for a set of assemblies, rather than having separate
    resources in each assembly? 
  * GetString returns `null` if the requested identifier was not found,
    potentially leading to blank output or a `NullReferenceException`.

Microsoft does address the first of these drawbacks by providing a code 
generator built into Visual Studio that gives you a global property for
each string; see [here](http://stackoverflow.com/questions/1142802/how-to-use-localization-in-c-sharp).

Even so, you may find that this class provides a more convenient approach
because your native-language strings are written right in your code, and
because you are guaranteed to get a string at runtime (not null) if the 
desired language is not available.

Combining with ResourceManager
------------------------------

This class supports ResourceManager via the `UseResourceManager` 
helper method. For example, after calling `Localize.UseResourceManager(resourceManager)`,
if you write

        "Save As...".Localized()

Then `resourceManager.GetString("Save As...")` is called to get the
translated string, or the original string if no translation was found 
(and yes, in your resx file you *can* use spaces and punctuation in the left-hand side).
You can even add a "name calculator" to encode your resx file's naming 
convention, e.g. by removing spaces and punctuation (for details, look at
the [`UseResourceManager`](http://ecsharp.net/doc/code/classLoyc_1_1Localize.html) method.)

It is common in .NET programs to have one "main" resx file, e.g. Resources.resx,
that contains default strings, along other files with non-English translations 
(e.g. Resources.es.resx for Spanish). When using `Localized()` you might use a 
slightly different approach: you still have a Resources.resx file, but you 
leave the string table empty (you can still use it for other resources, such as 
icons). This causes Visual Studio to generate a `Resources` class with a 
`ResourceManager` property so that you need can easily get the instance of
`ResourceManager` that you need.

- When your program starts, call `Localize.UseResourceManager(Resources.ResourceManager)`.
- Use the `Localized()` extension method to get translations of short strings.
- For long strings, use `Localize.Symbol("ShortAlias", "Long string", params...)`. The first argument is the string passed to `ResourceManager.GetString()`

GNU gettext
-----------

I haven't tried this, but C/C++ open source software often uses GNU gettext to do translations. Reportedly, [it's possible to use gettext with C# too](https://manas.tech/blog/2009/10/01/using-gnu-gettext-for-i18n-in-c-and-asp-net.html).
