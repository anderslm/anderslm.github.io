---
layout: post
title:  "C# error handling using a result type"
date:   2021-09-20 07:48:00 +0200
categories: C#
---
# A problem
Consider a function definition of a function that takes a string and calculates then length
of the string:
```c#
int LengthOf(string s);
```
This tells us that the function takes a single parameter of type `string` and returns 
an `int`. Is that all though? What happens if we pass `null` as an argument?
If the function is implemented like this:
```c#
int LengthOf(string s) => s?.Length ?? 0;
```
Then it will return `0` when given `null` as an argument.

What happens if the function is implemented like this instead?
```c#
int LengthOf(string s) => s.Length;
```
If we pass `null` into `LengthOf` then the function will throw a `NullReferenceException`.

We are so used to this behavior that we tend to not think about it. But shouldn't a 
function definition show the expected input and the expected output?

There is no way to tell from the function definition that `LengthOf` can throw an
exception, and what's worse is that you would have to read the implementation or
learn from experience whether or not this is the case.

This is a problem and it should be fixed.

# Result

In order to address this problem we have to come up with a return type for `LengthOf`
that is able to tell us that it can fail. We can do this be implementing what is known
as a result type.

A result type is a type that represents either a successful value or an error. We need
some mechanisms that allows us to safely use a success value, handle an error and wrapping
a function in a result.

The basic implementation should therefore have four different functions:
* **Map**: Apply a function to the success value if the result is in a success state
* **Bind**: Apply a function, that returns another result, to the success value if the result is in a success state
* **Match**: Apply a function to either the success value or the error value and return the result of the functions
* **Try**: Call a function catching any errors and create either a successful or a failed result.

This is how this could be implemented:
```c#
public struct Result<T>
{
    public T Success { get; init; }

    public Exception Error { get; init; }

    public bool IsOk => Success != null && Error == null;

    public static Result<T> Ok(T a) => new() { Success = a };

    public static Result<T> Fail(Exception e) => new() { Error = e };

    public static Result<T> Try(Func<T> f)
    {
        try
        {
            return Ok(f());
        }
        catch (Exception e)
        {
            return Fail(e);
        }
    }
    
    public Result<U> Map<U>(Func<T, U> f)
        => IsOk ? Result<U>.Ok(f(Success)) : Result<U>.Fail(Error);

    public Result<U> Bind<U>(Func<T, Result<U>> f)
        => IsOk ? f(Success) : Result<U>.Fail(Error);

    public U Match<U>(Func<T, U> onSuccess, Func<Exception, U> onError)
        => IsOk ? onSuccess(Success) : onError(Error);
}
```
# Use it for `LenghtOf`
Now that we have the result type we can rewrite our definition of `LenghtOf` to:
```c#
Result<int> LengthOf(string s);
```
From this definition it is clear that `LengthOf` might not succeed and it forces 
the developer to handle that fact.

It could look like this:
```c#
public static Result<int> LengthOf(string s)
    => Result<int>.Try(() => s.Length);

public static void Main(string[] args)
{
    var lengthString =
        LengthOf(args.FirstOrDefault())
            .Match(l => $"Length is {l}", e => $"Couldn't calculate length. Error was: {e.Message}");

    Console.WriteLine(lengthString);
}
```

# Programming with results
The power of this only becomes apparent when you use the result for something a 
little more complicated than the example above. Imagine that instead of counting the
length of a string, we should divide the length of the string with itself minus 2,
double it and add 10. 

So given the string "Hello, world" with a length of 12 we should divide it with 10,
multiply it with two and add 10. The result is then 12.4.
That means we risc dividing by zero which can fail.

```c#

public static Result<int> LengthOf(string s)
    => Result<int>.Try(() => s.Length);

public static Result<decimal> Divide(int i1, int i2)
    => Result<decimal>.Try(() => (decimal)i1 / i2);
    
public static decimal Double(decimal i) => i * 2;

public static decimal Add10(decimal i) => i + 10;

public static void Main(string[] args)
{
    var lengthString =
        LengthOf(args.FirstOrDefault())
            .Bind(i => Divide(i, i - 2))
            .Map(Double)
            .Map(Add10)
            .Match(l => $"Result is {l}", e => $"Couldn't calculate result. Error was: {e.Message}");

    Console.WriteLine(lengthString);
}   
```
This is kind of what LINQ is for lists, just for operations which may fail. We've accomplished
to build a pipeline of transformations while hiding the fact that some of the operations
might fail.