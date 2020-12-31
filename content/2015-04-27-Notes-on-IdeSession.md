+++
title = "Notes on IdeSession"
date = 2015-04-27

[taxonomies]
tags = ["Haskell", "GHC"]
categories = ["Posts"]
+++

These are literally notes I wrote down while reading on ide-backend and I chose to turn into a blog post.

<!-- more -->

#### What is IdeSession?
This a module from the package [ide-backend] from [FP Complete] that provides an interface to ide-backend.
As far as I can tell it is the sole interface to ide-backend.  
I choose think of it as the sole API to ide-backend.

#### What is ide-backend?
There's a blog post from [FP Complete] explaining what ide-backend is: [ide-backend blog post]  

I'll summarize in my own words.  
FP Complete have been creating an IDE for haskell and in this IDE they have code that they use for communication between the IDE and GHC (the most widely used haskell compiler).  
Duncan Coutts, Edsko de Vries, and Mikolaj Konarski implemented a library that would act as a wrapper around the GHC API from this this code.  
It's this library (ide-backend) that is being used by people in the haskell community as general a wrapper around the GHC API.

Copied and pasted from the [ide-backend blog post]. The functions of ide-backend are:

* Compiling code
* Get compile error messages
* **Submit** updated code for recompilation
* Extract type information
* Find usage locations for identifiers *- works for both local and top level identifiers*
* Run generated bytecode
* Produce optimized executables

## IdeSession

You may want to read [official IdeSession documentation].

IdeSession is centered around:

* A single threaded IDE session.
* Operations for updating the session (changes in files, data, compiler parameters etc.)
* Running querries given the current state of the session.

> Note that everything going on here is taking place in a single threaded environment.

#### Interaction with the compiler
This interface is rather sequential; in part because we are dealing with files and data which are mutable.  
The general pattern of interation with the compiler is as follows:

1. Update phase (update source files, data et cetera).
2. Compile phase
3. Query phase (query the compiler on matters regarding the code).
4. Run phase

**Update phase**: We don't directly mutate the files since we don't want to end up in a situation where ide-backend has a different state of files and data while our client has a different state of the files and data. However, we describle the changes we want to make to the files and let ide-backend effect them. That is, give ide-backend, via IdeSession, the new state of the files.  
**Compile phase**: We apply the relevant updates and invoke the compiler. It incrementally compiles some modules. This may take a while therefore we want progress information.  
**Query phase**: After compilation we collect info related to the compilation: source errors, list of successfully loaded modules et cetera.  
**Run phase**: Regardless of compilation results; we may want to run code from a certain module, interact with the code, interrupt its execution.

In haskell we follow types so naturally there are types associated with each of these phases.

1. IdeSession: *Query phase* - This is the default mode (we start here because at the start the files are in some state).
2. IdeSessionUpdate: *Update phase* - Accumulate updates.
3. Progress: *Compile phase* - Progress info.
4. RunActions: *Run phase* - For handles on the running code, through which one can interact with the code.

## Additional notes.
**Managing and mutating files in the source directory.**

> Trust the session. Trust IdeSession.

In this environment we should coordinate updating and changing source files through IdeSession.  
Ide session manages files in the source directory. This is important because we don't want the client and ide-backend have different versions of the files.  
All file changes and file reading must be done via the session (sequenced relative to other session state changes).  
The session will manage the files carefully including the case of exceptions and things going awry.
The caller needn't duplicate file state.


The caller should be able to:

* Put files into the session
* Apply updates to files via the session
* Extract files at any time before the session is closed.

**Morally pure querries.**

Purity:

* The property of a function to always gives the same output given the same input.
* The property of a function not to have side effects.

In this case we want to regard the compiler as a pure function disregarding the side effects part of purity because we have a lot of IO going on here.  
It should always be the case that we can throw away all the compilation results and recover them just from the file state and user parameters.  
*In case of warnings:* Traditionally compilers show warnings for the modules they compile skipping warning for modules they didn't have to recompile. This however doesn't match the pure function principle of same results for the same parameters. So IdeSession provides purity in cases such as these.

> So we try to maintain the compiler as:  
 compiler (modules, args, env) -> (object code, compiler warnings, errors....)


**Persistent and transitory state.**

The persistent state regards:

* The source files
* Data files
* User supplied arguments for compilation.

Internally there is a lot of cached and transitory state. In memory or on disk; none of these persist in the case of a fatal error; for example, they are wiped before shutdown and only the source and data files persit in case of a power failure.

It should be possible to drop all transitory state and recover (somewhat) as long as the original session value is available. The [`restartSession`] function serves this purpose.

[ide-backend]: http://hackage.haskell.org/package/ide-backend-0.9.0.7
[ide-backend blog post]: https://www.fpcomplete.com/blog/2015/03/announce-ide-backend
[FP complete]: https://www.fpcomplete.com/business/about/about-us/
[official IdeSession documentation]: http://hackage.haskell.org/package/ide-backend-0.9.0.7/docs/IdeSession.html
[`restartSession`]: http://hackage.haskell.org/package/ide-backend-0.9.0.7/docs/IdeSession.html#v:restartSession