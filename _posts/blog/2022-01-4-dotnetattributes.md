---
title: Add Custom C# Attributes to any API Endpoint
layout: post
author: Kyle Cooper
category: blog 
excerpt_separator: 
---

A method for attaching a C# custom attribute to a .NET 5.0+ API endpoint method using middleware.
<!--more-->

## What is the Use Case? 

I came across a need for checking a 'requirement' inside both the HttpContext 
and headers based on what endpoint the user was communicating with. I did not want to write this inside the 
method body and return some kind of failure in there, I also wanted a way to quickly 
and easily mark methods for checking this 'requirement'. 

This operation will allow a requirement to be 'checked' and handled before even touching the 
method. This way you can throw an Exception and return an error or other status from middleware.

An example of why you may want this is because you need to check 
that a request requires a 'header' from the client. Maybe this is true for all 
your endpoints, but maybe not. If not you can apply an attribute to the controller. 
Then have middleware check the controller has the applied attribute. If it does 
take an action, if not continue on with the request. 

**The endpoint method might look like this**
```C#
[CustomAttributeToBeCheckedInMiddleware] // Our custom attribute
[HttpPost("PostExampleModel")]
public async Task<> MySetterMethod(DataModelExample data)
{
    Response<DataModelExample> result = null;
    /* 
    do something cool and populate the result 
    details 
    */
    if (result.Succeed == true)
    {
        return Success(result);
    }
    return Failure(result);
}
```

### Warnings
This method may not be best practice, so use at your own risk. Due to this being in middleware, the
functionality will be called every time a client sends a request, so you may want 
to keep performance considerations in mind. This makes **no** guarantees of security in your application.

# Prerequisites 
These methods will work with API controllers and C# .NET APIs.

- Understand how [middleware](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-6.0) works in C#
- Understand [C# attributes](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/attributes/) and how to create custom attributes

## Create an Attribute
Create a C# attribute, below is an example of a basic attribute, you can also do much more with them! Check 
out the Microsoft docs for details.

Here is an example of a C# attribute:
```C#
using System;

[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, AllowMultiple = false, Inherited = true)]
public class CustomAttributeToBeCheckedInMiddleware : Attribute
{
    public CustomAttributeToBeCheckedInMiddleware() 
    {
    }
}
```

## Setup the Middleware
Make sure you know how middleware works before continuing as not everything is explained here.

Crete a piece of middleware like in the example below.

1. Get the endpoint that the API wants to communicate with, if nulll 
you can throw an exception. This, usually, means that you added middleware into the middleware 
stack incorrectly.
2. See if the attribute is attached.

```C#
using System; 
using System.Threading.Tasks;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Http.Features;

public class CustomAttributeToBeCheckedInMiddleware
{
    private readonly RequestDelegate _Next;

    public CustomAttributeToBeCheckedInMiddleware(RequestDelegate next)
    {
        _Next = next; 
    }

    public Task Invoke(HttpContext context) 
    {
        // 1. Check we know what endpoint we want
        var endpoint = context.Features.Get<IEndpointFeature>()?.Endpoint;
        if (endpoint == null)
        {
            throw new Exception("Could not determine endpoint");
        }
        // 2. Check to see if the endpoint has the attribute added
        CustomAttributeToBeCheckedInMiddleware attr = endpoint?.Metadata.GetMetadata<CustomAttributeToBeCheckedInMiddleware>();
        if (attr != null)
        {
            bool isSomethingSuccesfull = false;
            // This has the attribute added, do something here!

            // Make sure you invoke the next piece of middleware if the conditions are right
            isSomethingSuccesfull = true;
            if (isSomethingSuccesfull == true) 
            {
                await _Next.Invoke(context); 
            }
        }
        else 
        {
            await _Next.Invoke(context); 
        }
    }
}

```

### Place the New Middleware Component in the Configure method

Place the implementation right before ```app.UseEnpoints(...)``` and after ```app.UseRouting(...)```. 
This is important because at this point the Middleware 
is aware of the endpoints method body and what attributes are applied to it.

A vague example in Startup.cs, Configure(...) method body.
```C#
app.UseEndpoints(...);
/*
.
.
*/
app.UseMiddleware<CustomAttributeToBeCheckedInMiddleware>();
/*
.
.
*/
app.UseRotuing(...);
```
