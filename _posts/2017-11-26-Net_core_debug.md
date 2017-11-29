---
layout: post
title: Debugging .NET Core CLR afterwords 
tags: .NET Core
---

Some time ago i decided to discover internal details of .NET Core CLR and today i'd like to share with you the most surprised things that i found during this simple investigation.

## Start to debug .NET Core CLR source codes is pretty easy

Microsoft took care of developers who are interesting in development of their products. Repository documentation is not only very detailed and well-designed but also keeping up to date and contains lots of how-to information. 

Depending on the developer's OS it contains a particular build instructions and requirements ([this one](https://github.com/dotnet/coreclr/blob/master/Documentation/building/windows-instructions.md) for Windows for example). As you could notice there are IDE and necessary component requirements, secondary packages and environment setup which should be performed for a successfully building. 

The build process itself is pretty straightforward and contains the only one step: execute `build.cmd` with an optional arguments. Detailed [debugging instructions](https://github.com/dotnet/coreclr/blob/master/Documentation/building/debugging-instructions.md) on a different OS could also be found at a separate page.

In my case the build process took about 40 minutes (including tests) and didn't contain any issues.

## Implicit relationship between C++ and C# in memory laout

Once i've found it i was really surprised but some time later it became really obvious. For a better explanation i'd like to show you a code:

![csharp_cpp_memory_relationship](/images/post/csharp_cpp_memory_relationship.png)

Have you also already realized it? The left part of the code (C#) has exactly the same memory layout as the right one (C++). And if we think for a while actually there is nothing to wonder. Futhermore we gain some great opportunities which i'd like to discover shortly (we need to go deeper).

## Two-way interraction

I won't go into low-level details how call actually works, but instead I'd like to provide a small part of a diagram which confirm the current thesis:

![csharp_cpp_two_way](/images/post/csharp_cpp_twoway.png)

The left part contains C++ code which is used for application domain setup. During this process CoreRun initialize an instance of `AppDomain`'s type and make a `setupDomain.Call` which pass execution control to C#'s AppDomain implementation. When the C# part of work will be completed execution control passes once again to C++ with the `nCreateContext()` method call. 

As I've already mentioned before the root cause of this opportunity is the same memory layout (probably, it's better to say 'exactly the same mapping between type design and memory structure regardless programming language'). In reality in both cases we allocate the same size of memory and every property or method has the same offset. Why shouldn't it be possible to make such kind of calls? It's quite real and obvious.

## C# access modifiers does not affect availability from outside

Once again an attentive reader was able to figure out the confirmation of this thesis with the previous image. Haven't noticed yet? Ok, i'm going to clarify a little bit:

```csharp
private static object PrepareDataForSetup(
    String friendlyName,
    AppDomainSetup setup,                                                
    string[] propertyNames,                                                
    string[] propertyValues) 
{
    //..
}
```

The signature of this method contains access modifier `private` but it's not a barrier to make a call from C++. And once again it becomes quite obvious if we start to think from memory point of view and pointers management (actually it's also even available to call `private` method from a C# via reflection but this is a complete different story).

## Contribute to Microsoft repository easier than you think

The last but not least thing I'd like to discover today is: contribution to microsoft repository is pretty easy, straightforward and really exciting. If you have any proposal, issue or question feel free to create a new issue with a an explanation in details and be confident- a competent feedback will be posted very soon. 

In my case I found that an implementation of a several methods was missed and @jkotas suggested me to be a responsible for this approvement. Honestly it's a pretty simple improvement and my contribution wasn't too important, but i'm not able to tell you in words how exciting it was.

![coreclr_oss](/images/post/coreclr_oss.png)

Finally I'd like to recommend an awesome video- [Adam Sitnik — My awesome journey with Open Source](https://www.youtube.com/watch?v=2HSPKyAyuik) for everyone who haven't seen it before. In this video Adam shares his experience and give a lot of usefull advices and recomendations for people who have been planing to start contribute but still do not know how to start.

Thanks everyone for reading and feel free to comment.