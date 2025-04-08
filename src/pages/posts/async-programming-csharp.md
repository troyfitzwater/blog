---
layout: '../../layouts/BlogPostLayout.astro'
title: 'Asynchronous Programming in C#'
pubDate: 2025-04-07
tags: ["csharp", "engineering"]
---

## What is Synchronous Code?

In programming, code is often run synchronously: 1 step happens after another until the program completes. For example, a calculator program would accept 2 inputs for a `Sum` function, add those inputs, and return a response

```csharp
public int Sum(int num1, int num2)
{
	return num1 + num2;
}
```

In some scenarios, we may not be able to immediately continue in a program without waiting for something to complete, or there is a long-running task that other code in our program may not need to wait on to execute.

## Why do we need Asynchronous Code?

Imagine you're in a restaurant. If the restaurant operated synchronously, the server would take your order, go to the kitchen, wait until it's prepared, serve you your food, and then move on to the next table. As you can tell, this would be pretty inefficient.

Instead, servers work asynchronously - they take multiple tables at once, submit their orders to the kitchen, and handle other tasks while the food is being prepared.

The same is true for computers and programming. Imagine if every time a website needed to retrieve resources from a server it froze the UI, this would be pretty frustrating!

### Understanding Threads and Processes

To grasp how asynchronous programming works, we need to understand two fundamental concepts:

- **Process**: Think of this as the restaurant itself. It's the overarching container that provides all the resources our program needs to run—memory, files, etc.
- **Thread**: If the process is the restaurant, a thread is like a server - the worker executes our code. By default, C# programs start with a single thread, often called the "main thread,” and will spawn additional threads as necessary.

The concept of a thread is pretty low-level and often abstracted away in modern programming languages.

## What does Asynchronous Code look like in C#?

### Async, Await, and Task

 

There are three keywords we need to remember for working with asynchronous code in C#:

- `Task<TResult>` : represents work to be done, similar to a `Promise` in JavaScript and TypeScript. The `<TResult>` is a generic that can take on any type provided to it. So `Task<string>` represents work to be done that will return a type of `string`.
- `async` marks a method asynchronous, meaning that asynchronous code can appear in the method’s code block. While you can include this for methods that are not asynchronous, it’s best only to use it when working with asynchronous code, so other developers can better understand what you’re writing.
- `await` : pauses the execution of the program and waits for the return of code to the right of it.

Asynchronous methods should be, by convention, suffixed with `Async` , for example, `GetDataAsync()` or `UpdateUserAsync(userData)` .

## Examples

### 1. async/await simplifies asynchronous code

The combination of async and await can make asynchronous programming feel and look like writing synchronous code. For example:

```csharp
var returnedString = await FetchDataAsync("http://example.org");
Console.WriteLine(returnedString);
return;

async Task<string> FetchDataAsync(string requestUrl)
{
    using var httpClient = new HttpClient();
    return await httpClient.GetStringAsync(requestUrl);
}
```

In the above example, even though code is asynchronous and blocking the further execution of the program via `await` it is still structured like synchronous code, where a value is retrieved and returned.

### 2. Performing other tasks during asynchronous operations

This example shows that while an asynchronous operation, the `FetchDataAsync` method, is occurring, other tasks can be performed in the code; in this example that takes the form of counting seconds waiting for the response from the URL.

```csharp
using System.Diagnostics;

// An example site that purposely takes 30 seconds to load
await ProcessDataAsync("https://www.page-house.com/test/delay30seconds.php");
return;

async Task ProcessDataAsync(string requestUrl)
{
    Console.WriteLine("Fetching data...");
    Console.WriteLine("Meanwhile, the program is still responsive!");
    
    var stopwatch = new Stopwatch();
    stopwatch.Start();
    
    var dataTask = FetchDataAsync(requestUrl);

    // Tasks have a variety of helper methods, like IsCompleted,
    // which returns whether the Task's unit of work has completed
    while (!dataTask.IsCompleted)
    {
        Console.WriteLine($"Seconds waiting: {stopwatch.ElapsedMilliseconds / 1000}");
        await Task.Delay(1000);
    }
    
    stopwatch.Stop();

    var data = await dataTask;
    Console.WriteLine($"Data received: {data}");
}

async Task<string> FetchDataAsync(string requestUrl)
{
    using var httpClient = new HttpClient();
    return await httpClient.GetStringAsync(requestUrl);
}
```

### 3. Wait for multiple asynchronous operations to complete

If we have multiple different asynchronous operations happening, we can wait for all of them to complete before continuing. This may be useful in cases where data from multiple sources, like different files or different sites, need to be loaded before continuing in a program.

Using something similar to the `FetchDataAsync` method from the previous example, we can make file read requests to multiple files and wait for both of them to finish before continuing. Even though the small file will likely finish first, the method `ProcessMultipleFiles` is still waiting for the large file to be parsed before it can continue, and both tasks can happen at the same time.

```csharp
await ProcessMultipleFiles(
    "someLargeFile.json",
    "someSmallFile.json");
return;

async Task ProcessMultipleFiles(string smallFile, string largeFile)
{
    Console.WriteLine("Starting to process multiple files...");
    var stopwatch = new Stopwatch();
    stopwatch.Start();
    
    var smallFileTask = ParseFileAsync(smallFile);
    var largeFileTask = ParseFileAsync(largeFile);

    await Task.WhenAll(smallFileTask , largeFileTask );

    var smallFileResult = await smallFileTask ;
    var largeFileResult = await largeFileTask ;

    stopwatch.Stop();
    Console.WriteLine($"Small file results: {smallFileResult}");
    Console.WriteLine($"Large file results: {largeFileResult}");
}

async Task<string> ParseFileAsync(string filePath)
{
    return await File.ReadAllTextAsync(filePath);
}
```

## Debugging Asynchronous Code in C#

Debugging asynchronous code can be challenging. The different threads and processes add extra layers of complexity compared to stepping through more traditional and synchronous code. Some common issues with asynchronous programming include **deadlocks** and **race conditions**.

**Deadlocks** occur when tasks block the main thread of a program. This is particularly relevant in UI applications; imagine if every time an asynchronous action happened in a web application the page would freeze, that would be pretty annoying.

A common cause of deadlocks is `.Result` . This blocks the current thread while waiting for the task to complete. In the below example, `await CallDatabaseAsync()`  will yield a result back to the thread where it was called, while `.Result` blocks the thread from continuing. In UI applications, this would cause a deadlock.

```csharp
public string GetData()
{
		var result = CallDatabaseAsync().Result; // This will block the main thread
		return result;
}

public async Task<string> CallDatabaseAsync()
{
		await Task.Delay(1000); // Simulate getting some data from a database
		return "Some data.";
}
```

This could be rewritten to be [Async All the Way](https://learn.microsoft.com/en-us/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming#async-all-the-way). It is generally a good idea to keep asynchronous called by and calling other asynchronous code, and to ***never mix synchronous and asynchronous code.***

```csharp
public async Task<string> GetDataAsync()
{
		var result = await CallDatabaseAsync(); // This no longer deadlocks
		return result;
}

public async Task<string> CallDatabaseAsync()
{
		await Task.Delay(1000); // Simulate getting some data from a database
		return "Some data.";
}
```

**Race conditions** are when two or more asynchronous processes are trying to access the same data at once, and the desired outcome of the program requires this to happen in a certain order.

For example, if you need to create resources in two different systems, System A and System B, but the resource in System B relies on a value you only get returned after creating a resource in System A, you’ll need to yield the response of System A first. Below, this is demonstrated by being able to use the response from `CreateResourceInSystemAAsync` after it has been returned, and continue on to `CreateResourceInSystemBAsynch` in a synchronous manner.

```csharp
public void CreateResources(Resource resourceA, Resource resourceB)
{
		var resultA = await CreateInSystemAAsync(resourceA);
		await CreateInSystemBAsync(resourceB, resultA.Id);
}
```

## Wrap-Up

In this post, we went over the basics of asynchronous programming in C#. We covered when and why to use asynchronous programming, the usefulness of `async` and `await` for simplifying asynchronous code, and common challenges like deadlocks and race conditions.

Getting a good grasp on asynchronous programming is essential for modern development with C#. Experimenting with what was covered in this post and using language features like `async` and `await` in real-world projects are the best ways to get more comfortable with asynchronous programming. For further reading, refer to the links below. Thanks for reading, and happy coding!

https://learn.microsoft.com/en-us/dotnet/standard/threading/threads-and-threading

https://learn.microsoft.com/en-us/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming#async-all-the-way

https://learn.microsoft.com/en-us/dotnet/core/diagnostics/debug-deadlock?tabs=windows