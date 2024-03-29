---
title: Async/Await For The Rest Of Us
tags:
  - async
  - async/await
  - await
  - csharp
  - C#
  - csharp async
  - C# async
categories:
  - "Async/Await"
  - "C#"
date: 2018-08-28 02:26:21
---

What's the deal with `async` and `await` in C#? Why should a .NET developer in 2019 **need** to know what it is and how to use it?

<!--more-->

Perhaps you've used `async/await` but find yourself having to go back and figure it out again?

I've had to figure out the hard way that, for example, using `ConfigureAwait(false).GetResult()` doesn't magically make your async method "work" synchronously.

But this isn't an in-depth look at the internals of `async/await` and how to get fancy with `async/await`.

This is _"Async/await for the rest of us"_. Who are _the rest of us_?

We:

- Might have never used `async/await`
- Are not Microsoft gurus
- Are not C# gurus
- Admittedly forget how to use `async/await` properly at times.
- Might have to go back to Google and figure out why a method can return a `Task` but not use the `async` keyword.
- Might wonder why a method that's **not** marked as `async` can be awaited by it's caller

This - I hope - is an article for "the rest of us" that's to the point and practical.

_P.S. If you do want to dig into this topic more the best starting point is [I'd suggest starting with this article.](https://blog.stephencleary.com/2012/02/async-and-await.html)_

# Why Async?

What's the benefit of using `async/await`?

## For Web Developers

If you build web apps using .NET technologies - the answer is simple: **Scalability**.

When you make I/O calls - database queries, file reading, reading from HTTP requests, etc. - the thread that is handling the current HTTP request is **just waiting**.

That's it. **It's just waiting for a result to come back from the operating system.**

Performing a database query, for example, ultimately asks the operating system to connect to the database, send a message and get a message in return. But that is the OS making these requests - **not your app.**

> Using `async/await` allows your .NET web apps to be able to handle **more HTTP requests while waiting for IO to complete.**

## For Desktop/App Developers

But desktop apps don't handle HTTP requests...

Well, desktop apps **do** handle user input (keyboard, mouse, etc.), animations, etc. And there's only **one UI thread** to do that.

### User Input Is User Input

If we consider that HTTP requests in a web app are _just user input_, and desktop (or app) keyboard and mouse events are _just user input_ - then it's actually worse for desktop/app developers! They only get **one thread** to handle user input!

The I/O issues and fixes still apply.

However, the issue of CPU intensive tasks is another concern. In a nutshell, these types of tasks should not be done on the UI thread.

The types of tasks would include:

- Processing a large number of items in a loop
- Computing aggregations for reporting

If your app does this (on the main/UI thread), then there's nothing to handle user input and UI stuff like animations, etc.

This leads to freezing and laggy apps.

The solution is to offload CPU intensive tasks to a background task/thread. This starts to get into queuing up new threads/tasks, how to use `ConfigureAwait(false)` to keep asynchronous branches of your code on a non-UI context, etc. All things beyond the scope of our article.

# The Async Keyword

Let's start looking at the `async/await` keywords and how they are to be used.

There's confusion over the async keyword. Why? Because it looks like it **makes** your method asynchronous. But, it doesn't.

That's confusing. The `async` keyword **doesn't make my method asynchronous**? Yep.

## What Does it Do Then?

All the `async` keyword does is **enable the await keyword**. That's it. That's all. It does nothing else.

So, just think of the `async` keyword as the `enableAwait` keyword.

# The Await Keyword

The `await` keyword is where the magic happens. It basically says (to the reader of the code):

> I (the thread) will make sure if something asynchronous happens under here, that I'll go do something else (like handle HTTP requests). Some thread in the future will come back here once the asynchronous stuff is done.

Generally, the most common usage of `await` is when you are doing I/O - like getting results from a database query or getting contents from a file.

When you `await` a method that does I/O, it's **not your app** that does the I/O - it's ultimately the operating system. So your thread is just sitting there...waiting...

`await` will tell the current thread to just go away and do something useful. Let the operating system and the .NET framework get another thread later - whenever it needs one.

Consider this as a visual guide:

```c#
var result1 = await SomeAsyncIO1(); // OS is doing I/O while thread will go do something else.
// A thread gets the results.

var result2 = await SomeAsyncIO2(result1); // Thread goes to do something else.
// One comes back...

await SomeAsyncIO3(result2); // Goes away again...
// Comes back to finish the method.
```

You might ask yourself at this point:

> If the `async` keyword doesn't make a method asynchronous then what does?

## What Makes A Method Asynchronous Then?

Well - it's not the `async` keyword as we learned. Go figure.

Any method that returns an "awaitable" - `Task` or `Task<T>` can be awaited using the `await` keyword.

There are actually other awaitables types. And, an "awaitable" method doesn't strictly have to be an asynchronous method. But ignore that - this is meant to be for "the rest of us."

For the purpose of this article, we'll assume that an "asynchronous method" is a method that returns `Task` or `Task<T>`.

### When Does This Happen?

When will we ever need to return a `Task` from a method? It's usually when doing I/O. Most I/O libraries or built-in .NET APIs will have an "Async" version of a method.

For example, the `SqlConnection` class has an `Open` method that will begin the connection. But, it also has an `OpenAsync` method. It also has an `ExecuteNonQueryAsync` method.

```c#
public async Task<int> IssueSqlCommandAsync() {
    using(var con = new SqlConnection(_connectionString))
    {
        // Some code to create an sql command "sqlCommand" would be here...
        await con.OpenAsync();
        return await sqlCommand.ExecuteNonQueryAsync();
    }
}
```

What makes the `OpenAsync` and `ExecuteNonQueryAsync` methods asynchronous is **not** the `async` keyword, but it is that they **return a `Task` or `Task<T>`**.

# Async All The Way Down

It is possible to do something like this (notice the lack of `async` and `await`):

```c#
public Task<int> GetSomeData() {
    return DoSomethingAsync();
}
```

And then `await` that method:

```c#
// Inside some other method....
await GetSomeData();
```

`GetSomeData` doesn't await the call to `DoSomethingAsync` - it just returns the `Task`. Remember that `await` doesn't care if a method is using the `async` keyword - **it just requires that the method return a `Task`**.

It is possible to do this - create a method that calls an asynchronous method but doesn't await.

## It's A Best Practice

However, this is considered a bad practice. Why?

Since this article is supposed to be to the point and practical:

**Using async/await "all the way down" simply captures exceptions in asynchronous methods better.**

If you mark every method that returns a `Task` with the `async` keyword - which in turn enables the `await` keyword - it handles exceptions better and makes them understandable when looking at the exception's message and stack trace.

# Conclusion

To summarize briefly:

- The `async` keyword doesn't make methods asynchronous - it simply enables the `await` keyword.

- If a method returns a `Task` or `Task<T>` then it can be used by the `await` keyword to manage the asynchronous details of our code.

- Doing I/O always results in blocking threads. This means your web apps can't process as many HTTP requests in parallel and freezing and laggy apps.

- Using `async/await` helps us create code that will allow our threads to stop blocking and do useful work while performing I/O.

- This leads to web apps that can handle more requests per second and apps that are more responsive for their users.

I hope this is an understandable introduction to `async/await`. It's not an easy topic - and as always - gaining experience by using this feature will, over time, help us to understand what's going on.

There's so much more to be said and so many more concepts surrounding `async/await`. Some include:

- What is a `SynchronizationContext`? When should I be aware of this?
- What about .NET Core vs .NET Framework - is there a difference I should be aware of?
- Why can I mark a method as `async void`? What does this do? Should I do this?
- How do I offload CPU intensive work to a background thread/task?
- Is it possible to do work on a background thread and return to the UI thread at the very end?
- How do I call a method marked with the `async` keyword from synchronous code? What happens when I do this?

# Keep In Touch

Don't forget to connect with me on [twitter](https://twitter.com/jamesmh_dev) or [LinkedIn](https://www.linkedin.com/in/jamesmhickey/)!

<div style="padding:0   20px; border-radius:6px; background-color: #efefef; margin-bottom:50px; margin-top:20px">
    <h1 class="margin-bottom:0"> Navigating Your Software Development Career
</h1>
An e-mail newsletter where I'll answer subscriber questions and offer advice around topics like:

✔ What are the general stages of a software developer?
✔ How do I know which stage I'm at? How do I get to the next stage?
✔ What is a tech leader and how do I become one?


<div class="text-center">
    <a href="http://eepurl.com/gdIV5X">
        <button class="btn btn-sign-up" style="margin-top:0;margin-bottom:0">Join The Community!</button>
    </a>
</div>
</div>
