---
layout: post
title:  "Functional tasks"
date:   2021-09-19 21:40:00 +0200
categories: C#
---
Working with asynchronous computations in C# one often resorts to using `async` and `await`
in order for the code to look synchronous.

Like this:
```c#
public static async Task<string> DoSomethingAsynchronous()
{
    var result = await GetFromRemoteApi();
    return result.SomethingOfInterest;
}
```
And sometimes you have to use the asynchronous result in another asynchronous call.

Like this:
```c#
public static async Task<string> DoSomethingAsynchronous()
{
    var someResult = await GetFromRemoteApi();
    var someOtherResult = await GetFromAnotherApi(someResult.Something);
    return someOtherResult.SomethingOfInterest;
}
```

I find it a bit tedious to have to come up with these intermediate variable names, `someResult`
and `someOtherResult`, and also having to put in `await` all over just because it is an asynchronous
call.

In a small example like this it's not too bad, but we have all seen far worse in real life.

What if we could handle `Task` just like we handle lists with LINQ? LINQ enables us to create a
nice looking pipeline of transformations over an `IEnumerable`:
```c#
public static IEnumerable<string> GetRoleNamesOfAllActiveUsers()
    => this.Users
        .Where(u => u.IsActive)
        .SelectMany(u => u.Roles)
        .Select(r => r.Name)
        .Distinct();
```
LINQ helps us to avoid writing loops ourselves and I think we should have something similar for 
a `Task` - something that helps us to avoid writing `await`.

I propose to create an initial set of functions to do that:
* `Map`: Dig into a task and extract something from it - like `Select`
* `Bind`: Create a new task from the current task and use the new task - like `SelectMany`

```c#
public static class TaskExtensions
{
    public static async Task<U> Map<T, U>(this Task<T> t, Func<T, U> f)
        => f(await t);
        
    public static async Task<U> Bind<T, U>(this Task<T> t, Func<T, Task<U>> f)
        => await f(await t);
}
```
Remember the example from before?
```c#
public static async Task<string> DoSomethingAsynchronous()
{
    var someResult = await GetFromRemoteApi();
    var someOtherResult = await GetFromAnotherApi(someResult.Something);
    return someOtherResult.SomethingOfInterest;
}
```
Using `TaskExtensions` we can now write this instead:
```c#
public static Task<string> DoSomethingAsynchronous()
    => GetFromRemoteApi()
        .Bind(a => GetFromAnotherApi(a.Something))
        .Map(a => a.SomethingOfInterest);
}
```

There are many more things we can do with `Task`, like sequencing a list of tasks, handling errors,
provide a default value on failure, execute a list of tasks in parallel and probably more that I
didn't think of right now.

But that is for another post.