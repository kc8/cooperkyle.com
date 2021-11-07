---

layout: post
title: C# Async and Await
author: Kyle Cooper
category: notes
excerpt_separator: <!--more-->

---

C# Async and Await Techniques and ideas
<!--more-->

# Async and Await Specifics

## Gotchas
- async await wraps the function in a 100 byte size class 
- **Await all tasks**: Always await a Task because if an expection occurs it will be swallowed by the task and we will never know. This is because the C# behind the scenes creates a class to handle our operation and wraps the call in a try{} catch {} 
- Do not use ```.wait()```. Just better to use ```await function()```
	- This will through an aggregate exception and lock the main thread
	- ```.GetAwaiter().GetResult()``` is similar however, this will return the result and run synchronously
- If you do not need to return to the calling thread, you can use ```await myFunction.ConfigureAwait(false)``` at the end of the function which will tell the context to use any available thread. This is nice if you do not need to go back to the UI thread for anything, keeping it free. If the there is no context, then it won't matter (this is true in .NET Core)
	- Dont use this if you need to return to the calling thread
- If a function uses async await and returns an awaited call you can just return the task. This reduces context switching. Do not do this if you are not returning an awaitable, or the code paths do not always guarantee the awaitable task return type. So instead of awaiting what is returned in the example below: 
 ```c#
	 Task<ReturnType GetSomething(string nothing)
	 {
	 	return SomethingFromNothing(nothing); //NOTE this must be awaitable
	 }
 ```

-  If not all function calls return an await or awaitable, you can use a [[ValueTask]]
```c#
async ValueTask<string> SomethingFromNothing(string nothing)
{
	if (this.SomethingFromNothing > 1)
	{
		return SomethingFromNothing[0]; 
	}
	try 
	{
		return await ConvertNothingToSomething(nothing); 
	}
	catch 
	{
		return null;
	}
}
```

- Constructors and awaitables. Use the safe fire and forget method below
- Doing tasks ```WhenALl()```, use ConfigurationAwait(false); to get results
```
	var task2 = Task.Run( () => true); 
	var task2 = Task.Run( () => true); 
	
	await Task.WhhenAll(task1, task2);
	
	var result = await task1.configureAwait(false); 
```
 
## Async Void 
**Async Void is Not Good** AKA **Fire and Forget**
Why is it not good. It causes a lot of problems with threads executing in the correct order and catching exceptions. Sometimes this can also be bad as it can lead a higher probability of race conditions and such
```C#
public SomeClass()
{
	try 
	{
		Refresh(); // thread 1 comes back here and continues it execution
	}
	catch (Exception e)
	{
	}
	// If Refresh uses the list we can have race conditions
	this.ListOfSomething.Add(something)
}

async void Refresh() 
{
	// Calling thread runs the below and retrns back to the try catch above
	await DoSomething(); 
	//When this triggers, we will never be able to catch it because its happens in another thread
	throw new Exception(); 
}
```

We remove the async await from the event handler, however, this is not good because 
we fire PrepareCoffeAsync() and ignore any potential exception. We could mark the void function with async and await the prepare coffee call but this is also not good. 
```C#
public void OnPrepareButtonClick(object sender, EventArgs e)
{
    Button button = (Button)sender;
    PrepareCoffeeAsync(button); //This is not the best
}

public async Task PrepareCoffeeAsync(Button button)
{
    button.IsEnabled = false;
    activityIndicator.IsRunning = true;

    var coffeeService = new CoffeeService();
    await coffeeService.PrepareCoffeeAsync();

    activityIndicator.IsRunning = false;
    button.IsEnabled = true;
}
```
Into this: (See the Fire and Foreget methods below)
```C#
public void OnPrepareButtonClick(object sender, EventArgs e)
{
    IErrorHandler errorHandler = null; // Get an instance from somewhere
    Button button = (Button)sender;
    PrepareCoffeeAsync(button).FireAndForgetSafeAsync(errorHandler);
}
```

### Safe Fire and Forget
[Safe Fire and Forget](https://gist.github.com/kc8/bd4a4c0abcc7675970a0a2d34a3eb6c3)
[GitHub](https://github.com/brminnick/AsyncAwaitBestPractices)
```c#
static async void HandleSafeFireAndForget<TException>(Task task, bool continueOnCapturedContext, Action<TException>? onException) where TException : Exception
{
	try
	{
		await task.ConfigureAwait(continueOnCapturedContext);
	}
	catch (TException ex) when (onException != null)
	{
		onException(ex);
	}
}
```
***See [0] for Details***

**Note there is something like the above for ICommand 

## References
- [Async and Await Common Mistakes](https://www.youtube.com/watch?v=J0mcYVxJEl0)
- [Async and Await Blog Posts](https://johnthiriet.com/removing-async-void/) and a [Library](https://github.com/brminnick/AsyncAwaitBestPractices) to go with it
- [ConfigureAwait](https://devblogs.microsoft.com/dotnet/configureawait-faq/)

## Labeled References
[0] [Remove async void ](https://johnthiriet.com/removing-async-void/#removing-async-void)
