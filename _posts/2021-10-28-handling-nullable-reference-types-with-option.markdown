---
layout: post
title:  "Handling nullable reference types with an option type in C#"
date:   2021-10-29 21:20:00 +0200
categories: C#
---
# Billion-dollar mistake
Back in the sixties Tony Hoare invented the null reference exception when working on ALGOL. Now, over 50 years later, we're still
paying the price of this decision. Tony himself has called it 'my billion-dollar mistake'. 

Many languages has decided against `null` and use different strategies to avoid it. In this post I will demonstrate how to
avoid `null` in C# using the feature called 'nullable reference types' and a custom type called `Option`.

# The problem with `null`
But what is the problem with `null` anyways? You might ask.

Consider this code:
```c#
class Program
{
    public static void Main(string[] args)
    {
        Console.WriteLine(Say.Hello(args.FirstOrDefault()));
    }
}

public static class Say
{
    public static string Hello(string name)
    {
        return $"Hello, {name.Trim()}";
    }
}
```
Executing this program with no arguments will make it throw a `NullReferenceException`. The classic way to avoid is to check 
for null in various ways. 

Like checking for null and return early:
```c#
public static string Hello(string name)
{
    if (name == null) return string.Empty;

    return $"Hello, {name.Trim()}";
}
```
Defaulting to a value in case of null:
```c#
public static void Main(string[] args)
{
    Console.WriteLine(Say.Hello(args.FirstOrDefault() ?? string.Empty));
}
```
Or not calling the function in the first place:
```c#
public static void Main(string[] args)
{
    var name = args.FirstOrDefault();

    if (name != null)
        Console.WriteLine(Say.Hello(name));
}
```
But all these different strategies to avoid a `NullReferenceException` are prune to errors, as the developer must remember
to implement a check for `null` in some way. Many developers either forget to do it, is too lazy to do it or doesn't care enough
and the result is a program where a `NullReferenceException` just waits to happen at runtime.

# Nullable reference types
With the release of C# 8.0 we've got a new feature called 'nullable reference types' that helps the developer to remember to
check for null. You can enable the feature either by writing `#nullable enable` directly in the source code, ex. at the top of
the file, or by putting `<Nullable>enable</Nullable>` in a `<PropertyGroup>` in the `csproj` file.

When 'nullable reference types' are enabled it means that the compiler will warn you if you set reference types to null.
A reference type (object, string, ...) is any type that isn't a value type (struct, int, bool, char, double, ...).

Now the compiler will warn us:
```c#
public static void Main(string[] args)
{
    #nullable enable
    string name = args.FirstOrDefault();

    Console.WriteLine(Say.Hello(name));
}
```
On the line where `name` is initialized we get this warning:
`[CS8600] Converting null literal or possible null value to non-nullable type.`

We can make the warning go away by using any of the three strategies that I listen earlier. But this doesn't solve the problem.
The developer can still ignore the warning, suppress it or a third-party library might initialize a value to `null` without you
knowing about it.

I'm arguing for a different solution where we use the type system to ensure that we don't accidentally rely on a value that
turns out to be `null`.
# Option type
Instead of relying on the developer to be careful enough to avoid `null`, we can utilize the type system instead. If we create
what is known as an option or maybe type. Such a type can be in two different states. One where there is a value and one where
there isn't a value. Lets call those states for 'Just' and 'None'.

In order to ensure that the type itself can't be `null` we can use a `struct` to implement the type. `struct` is a value type 
and can't therefore be null.

We also need a way to apply an optional value to a function in case it's in the 'Some' state. We'll call that
function 'Map'.

```c#
#nullable enable
public struct Option<T>
{
    public static Option<T> Some(T t) => new()
    {
        Value = t,
        IsNone = t is null,
    };

    public static Option<T> None => new()
    {
        Value = default,
        IsNone = true,
    };

    public T? Value { get; init; }
    
    public bool IsNone { get; init; }
    
    public Option<U> Map<U>(Func<T, U> f)
        => IsNone ? Option<U>.None : Option<U>.Some(f(Value!));
}
```
Now when we need to handle things that are optional or values that might be there or might not, we can just
use the `Option<T>` type.

Like this:
```c#
#nullable enable
public static void Main(string[] args)
{
    int LenghtOf(string s) => s.Length;

    var optionalString1 = Option<string>.Some("Hello, world!");
    var optionalString2 = Option<string>.None;
    
    var maybeLength1 = optionalString1.Map(LengthOf);
    var maybeLength2 = optionalString1.Map(LengthOf);
}
```
Now both `maybeLength` variables is an `Option<int>` and only one of them contains a value. Notice that the `LengthOf` function
didn't need to check for null or declare `s` as nullable - it's handled by the `Option<T>` type.