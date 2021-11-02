---
layout: post
title:  "A better way to write services in C#"
date:   2021-11-02 14:15:00 +0200
categories: C#
---
By 'service' I mean some kind of mechanism which does I/O. Typically
a client to a Http Api, database access, file access or something similar. A 'service' is typically injected
into your App by an IoC container based on the `interface` it implements.

If we put together the ideas from my last three posts:
* [Functional tasks](https://anderslm.github.io/c%23/2021/09/19/functional-tasks.html)
* [C# error handling using a result type](https://anderslm.github.io/c%23/2021/09/20/csharp-error-handling-using-result-type.html)
* [Handling nullable reference types with an option type in C#](https://anderslm.github.io/c%23/2021/10/29/handling-nullable-reference-types-with-option.html)

It enables us to write extremely expressive interfaces and services.

# User database service

Consider a service that is accessing a database of users. The service should be able to retrieve all users and to
find a single user from an id.

Both queries to the database can be executed asynchronously. Both of them might fail, eg. if the database connection
couldn't be establish. And when looking for a user with a given id, it might be that no user is found.

We have three different concepts here, which can be solved by using types:
* Code that are executed asynchronously: `System.Threading.Task`
* Things that might fail: `Result` from my previous post
* Things that might return a value: `Option` from my previous post

## The interface

By using the `Result` and `Option` type we can express, with a high level of clarity, what is expected from
`UserService`. To keep things super simple, a `User` just has a `UserId` which is just a string:
```c#
public record UserId(string Id);

public record User(UserId Id);

public interface UserService
{
    Task<Result<List<User>>> RetrieveAllUsers();

    Task<Result<Option<User>>> FindUser(UserId id);
}
```
These methods can be read like this:
* `RetrieveAllUsers`: An asynchronous call that might fail which returns a list of users.
* `FindUser`: An asynchronous call that might fail which might return a user when given an id.

## The implementation

As an example lets implement a `UserService` which uses Entity Framework Core.

In addition to the `Option` and `Result` type we need a new function that can execute an asynchronous function and return a `Result`, and
a way to map an `Option` that's wrapped in a `Result` inside a `Task`:
```c#    
public static class Extensions
{
    public static async Task<Result<T>> Try<T>(this Task<T> t)
    {
        try
        {
            return Result<T>.Ok(await t);
        }
        catch (Exception e)
        {
            return Result<T>.Fail(e);
        }
    }

    public static async Task<Result<Option<U>>> Map<T, U>(this Task<Result<Option<T>>> t, Func<T, U> f)
        => (await t).Map(o => o.Map(f));
}
    
public class UserDataContext : DbContext
{
    public DbSet<UserDto> Users { get; set; }
}

public class UserDto
{
    public string Id { get; set; }
}

public class EntityFrameworkUserService : UserService
{
    private UserDataContext dataContext;

    public EntityFrameworkUserService(UserDataContext dataContext) => this.dataContext = dataContext;
    
    public Task<Result<List<User>>> RetrieveAllUsers()
        => dataContext
            .Users
            .ToListAsync()
            .Try();
            
    public Task<Result<Option<User>>> FindUser(UserId id)
        => dataContext
            .Users
            .FirstOrDefaultAsync(u => u.Id == id.Id)
            .Try()
            .Map(Option<UserDto>.Some)
            .Map(a => new User(new UserId(a.Id)));
} 
```

I, for one, thinks that this is a nice way to program.