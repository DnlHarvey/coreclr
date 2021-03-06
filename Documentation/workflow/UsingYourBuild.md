
# Using your .NET Core Runtime Build

We assume that you have successfully built CoreCLR repository and thus have files of the form
```
    bin\Product\<OS>.<arch>.<flavor>\
```
And now you wish to try it out.  We will be using Windows OS as an example and thus will use \ rather
than / for directory separators and things like Windows_NT instead of Linux but it should be
pretty obvious how to adapt these instructions for other operating systems.

To run your newly built .NET Core Runtime in addition to the application itself, you will need
a 'host' program that will load the Runtime as well as all the other .NET Core Framework
code that your application needs. The easiest way to get all this other stuff is to simply use the
standard 'dotnet' host that installs with .NET Core SDK.

The released version of 'dotnet' tool may not be compatible with the live CoreCLR repository. The following steps
assume use of a dogfood build of the .NET SDK.

## Acquire the latest nightly .NET Core SDK

- [Win 64-bit Latest](https://dotnetcli.blob.core.windows.net/dotnet/Sdk/master/dotnet-sdk-latest-win-x64.zip)
- [macOS 64-bit Latest](https://dotnetcli.blob.core.windows.net/dotnet/Sdk/master/dotnet-sdk-latest-osx-x64.tar.gz)
- [Others](https://github.com/dotnet/cli/blob/master/README.md#installers-and-binaries)

To setup the SDK download the zip and extract it somewhere and add the root folder to your [path](../building/windows-instructions.md#adding-to-the-default-path-variable)
or always fully qualify the path to dotnet in the root of this folder for all the instructions in this document.

After setting up dotnet you can verify you are using the newer version by:

`dotnet --info` -- the version should be greater than 2.1.0-*

For another small walkthrough see [Dogfooding .NET Core SDK](https://github.com/dotnet/corefx/blob/master/Documentation/project-docs/dogfooding.md).

## Create sample self-contained application

At this point, you can create a new 'Hello World' program in the standard way.

```bat
mkdir HelloWorld
cd HelloWorld
dotnet new console
```

### Change project to be self-contained

In order to update with your local changes, the application needs to be self-contained, as opposed to running on the
shared framework.  In order to do that you will need to add a `RuntimeIdentifier` to your project.

```xml
<PropertyGroup>
  ...
  <RuntimeIdentifier>win-x64</RuntimeIdentifier>
</PropertyGroup>
```
For Windows you will want `win-x64`, for macOS `osx-x64` and `linux-x64` for Linux.

You might also need to explicitly specify a `PlatformTarget`: it shouldn't be required though, unless for some reason the default `PlatformTarget` on your machine, for that directory, is not `x64`.

```xml
<PropertyGroup>
  ...
  <RuntimeIdentifier>win-x64</RuntimeIdentifier>
  <PlatformTarget>x64</PlatformTarget>
</PropertyGroup>
```

### Publish

Now is the time to publish. The publish step will trigger restore and build. You can iterate on build by calling `dotnet build` as
needed.

```bat
dotnet publish
```

**Note:** If publish fails to restore runtime packages you need to configure custom NuGet feed. To do so you have to:

1. run `dotnet new nugetconfig`
2. go to the `NuGet.Config` file and add `<add key="dotnet-core" value="https://dotnet.myget.org/F/dotnet-core/api/v3/index.json" />`

After you publish you will find you all the binaries needed to run your application under `bin\Debug\netcoreapp2.1\win-x64\publish\`.

```
.\bin\Debug\netcoreapp2.1\win-x64\publish\HelloWorld.exe
```

**But we are not done yet, you need to replace the published runtime files with the files from your local build!**

## Update CoreCLR from raw binary output

Updating CoreCLR from raw binary output is easier for quick one-off testing which is what this set of instructions
outline but for consuming in a real .NET Core application you should use the nuget package instructions below.

The 'dotnet publish' step above creates a directory that has all the files necessary to run your app
including the CoreCLR and the parts of CoreFX that were needed. You can use this fact to skip some steps if
you wish to update the DLLs. For example typically when you update CoreCLR you end up updating one of two DLLs

* coreclr.dll - Most modifications (with the exception of the JIT compiler and tools) that are C++ code update
  this DLL.
* System.Private.CoreLib.dll - If you modified C# it will end up here.

Thus after making a change and building, you can simply copy the updated binary from the `bin\Product\<OS>.<arch>.<flavor>`
directory to your publication directory (e.g. `helloWorld\bin\Debug\netcoreapp2.1\win-x64\publish`) to quickly
deploy your new bits. In a lot of cases it is easiest to just copy everything from here to your publication directory.

You can build just the .NET Library part of the build by doing (debug, for release add 'release' qualifier)
(on Linux / OSX us ./build.sh)
```bat
    .\build skiptests skipnative
```
Which builds System.Private.CoreLib.dll if you modify C# code. If you wish to only compile the coreclr.dll you can do
 ```bat
    .\build skiptests skipmscorlib
```
Note that this technique does not work on .NET Apps that have not been published (that is you have not created
a directory with all DLLs needed to run the all)  That is because the runtime is either fetched from the system-wide
location that dotnet.exe installed, OR it is fetched from the local nuget package cache (which is where your
build was put when you did a 'dotnet restore' and had a dependency on your particular runtime).    In theory you
could update these locations in place, but that is not recommended since they are shared more widely.

## (Optional) Confirm that the app used your new runtime

Congratulations, you have successfully used your newly built runtime. 

As a hint you could add some code like:
```
var coreAssemblyInfo = System.Diagnostics.FileVersionInfo.GetVersionInfo(typeof(object).Assembly.Location);
Console.WriteLine($"Hello World from Core {coreAssemblyInfo.ProductVersion}");
Console.WriteLine($"The location is {typeof(object).Assembly.Location}");
```

That should tell you the version and which user and machine build the assembly as well as the commit hash of the code
at the time of building:

```
Hello World from Core 4.6.26210.0 @BuiltBy: adsitnik-MININT-O513E3V @SrcCode: https://github.com/dotnet/coreclr/tree/3d6da797d1f7dc47d5934189787a4e8006ab3a04
The location is C:\coreclr\helloWorld\bin\Debug\netcoreapp2.1\win-x64\publish\System.Private.CoreLib.dll
```

## Using DotNetCli to run your .NET Core Application

If you don't like the idea of copying files manually you can follow [these instructions](UsingDotNetCli.md) to use dotnet cli to do this for you.
However the steps described here are the simplest and most commonly used by CoreCLR developers for ad-hoc testing.

## Using CoreRun to run your .NET Core Application

Generally using dotnet.exe tool to run your .NET Core application is the preferred mechanism to run .NET Core Apps.
However there is a simpler 'host' for .NET Core applications called 'CoreRun' that can also be used.   The value
of this host is that it is simpler (in particular it knows nothing about NuGet), but precisely because of this
it can be harder to use (since you are responsible for insuring all the dependencies you need are gather together)
See [Using CoreRun To Run .NET Core Application](UsingCoreRun.md) for more.
