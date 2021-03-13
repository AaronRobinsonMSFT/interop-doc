# .NET Interop Document

### Contents

- [Introduction](#intro): Introduction on how interoperability works in .NET.
    - [Terminology](#terms)
    - [Transitioning between Managed and Native](#transition)

- [IL Stubs and Reverse IL Stubs](#ilstubs): How stubs are generated and runtime optimizations.
<!-- 
- [COM and `IUnknown`](#comiunknown) - COM interoperability in .NET.
    - [WinRT](#winrt)
- [C++/CLI](#cppcli)
-->
- [Third party tools](#thirdpartytools)

## Introduction <a name="intro"></a>

The .NET interop system is an intrinsic part of the .NET platform and its impact can be observed down to the binary level &ndash; defined in [ECMA-335][spec_ecma335]. The representation of C language function exports with the common [x86 calling conventions][wiki_x86callconv] at the time of writing are literally able to be encoded at the lowest readable level. The lowest level in this case is termed "Intermediate Language" or "IL". The point of interop in .NET is to express in IL how software _not_ defined in IL and _not_ the .NET platform itself should work with logic and data owned and managed by a .NET runtime implementation.

An example is warranted. Imagine wanting to call the following C function from a .NET language (for example, C#).

```C
// Return the sum of the supplied array of integers.
int32_t sum_ints(int32_t length, int32_t* integers)
{
    int32_t acc = 0;
    for (int32_t i = 0; i < length; ++i)
        acc += integers[i];
    return acc;
}
```

In order to call this function it must first be declared in a manner that permits being called in C#. The .NET runtime will also need to know what library the function is defined in. The arguments to this function must also be considered. Since the first is a primitive type in both C and .NET (that is, a 4 byte integer) how it is passed to this function is relatively straight forward. The second argument is more complicated since it is an array of `int32_t`s. The .NET runtime must convert a C# `int[]` to a C `int32_t*` in order to pass the argument. This is the nature of .NET interoperability &ndash; working with a system that is not .NET.

### Terminology <a name="terms"></a>

Understanding the .NET interop domain is difficult without defining common jargon. Below is a list of terms used throughout this document. Terms are **bolded** in definitions when defined in this section.

[**Blittable**][overview_blittable] - Data that has the same representation within and outside the .NET platform. Blittable data doesn't need to be **Marshalled**.

**IL Stub** - A function generated by the .NET runtime at run time to facilitate a **managed** to **native** transition.

**Managed** - Code running under the control of a virtual machine where memory is managed automatically by a Garbage Collector (for example, .NET runtime). When referring to memory or data its lifetime is managed automatically.

**Marshal** - The process of converting data when transitioning between **managed** and **native** code.

**Native** - Code running where memory is managed manually by the developer (for example, C or C++). When referring to memory or data its lifetime is managed manually.

[**Pinning**][overview_pinning] - Mechanism used to indicate to the Garbage Collector not to move the **managed** data.

**Reverse IL Stub** - A function generated by the .NET runtime at run time to facilitate a **native**  to **managed** transition.

### Transitioning between Managed and Native <a name="transition"></a>

Declaring the previous C function to be callable in C# is accomplished via [`DllImportAttribute`][api_dllimport] &ndash; also called a [P/Invoke][overview_pinvoke]. Assuming the C function is contained within a native binary named `NativeLib.dll`, the following C# would declare the native function. Other mechanisms for defining callable native functions in IL are discussed in subsequent sections.

```csharp
[DllImport("NativeLib.dll")]
static extern int sum_ints(int length, int[] integers);
```

This P/Invoke declaration enables the C function to be called in C# but this call is not as simple as it may appear. Given this P/Invoke declaration an implicit function will also need to be constructed. In this case that is not technically true due to a JIT optimization, but the concept is the point here &ndash; see caveat of one-dimensional arrays of blittable primitive types [here][overview_blittable].

The implicit function is what is called an IL Stub. This IL Stub is a stream of IL instructions computed at run time by the interop system to marshal the function arguments from managed to native. There are a number of considerations that must be accounted for by the IL Stub which are discussed in another section. For this example the purpose of the IL Stub is to ensure the two arguments are safely passed to the native function and the native function's return value is returned to the C# caller. An example of an IL Stub is below, it has been annotated and slightly modified from what would be generated by the .NET runtime.

```
int32 ILStub_sum_ints(int32,int32[])
.locals (int64,native int,object pinned)
          ldarg.0
          conv.i8
          stloc.0       // Store input length as 64-bit number.
          ldc.i4.0
          conv.i
          stloc.1       // Initialize pointer local to null.
          ldarg.1       // Load and check if array is null.
          brfalse     NO_ARRAY
          ldarg.1
          stloc.2       // Pin the array so GC won't move it.
          ldloc.2
          conv.i
          ldc.i4.s    0x10
          add
          stloc.1       // Store pointer to managed array data.
NO_ARRAY :ldloc.0       // Load input length.
          ldloc.1       // Load pointer to managed array data.
          call        native int GetStubContext()
          ldc.i4.s    0x20
          add
          ldind.i
          ldind.i       // Load pointer to native function.
          calli       unmanaged stdcall int32(int64,native int)
          ret           // Return value on stack.
```

The precise details of the IL Stub aren't necessary to understand but the general flow is described to help illustrate its purpose. With the above IL Stub the a C# application can now call the `sum_ints()` function written in C.

```csharp
int[] arr = new[] { 1, 2, 3 };
int len = arr.Length;

int sum = sum_ints(len, arr);

// During the call to the P/Invoke the call stack
// will be the following:
//  - ILStub_sum_ints(len, arr) // C#
//      Locals:
//          int64  l0  = len
//          int*   l1  = pointer into arr's data
//          object l2  = pinned arr
//      - sum_ints(l0, l1)      // C
```

During a transitions from native to managed a Reverse IL Stub is generated. This stub performs similar duties to the one defined above but in the opposite direction &ndash; marshals data from native to managed.

This represents a conceptual introduction into how the .NET runtime handles interop. Additional details and specifics are discussed in other sections.

## IL Stubs and Reverse IL Stubs <a name="ilstubs"></a>

<!--
## COM and `IUnknown` <a name="comiunknown"></a>

### WinRT <a name="winrt"></a>

## C++/CLI <a name="cppcli"></a>
-->

## Third party tools <a name="thirdpartytools"></a>

Multiple tools exist for building interop solutions in .NET interop. Below is a list of tools that may be useful for .NET applications with interop requirements.

[SharpGenTools](https://github.com/SharpGenTools/SharpGenTools) - Compile time generation of interop marshalling code for P/Invokes and the `IUnknown` ABI.

[DNNE][repo_dnne] - Export C functions from a managed assembly.

## Additional Resources

[Investigating GC managed memory](https://github.com/Maoni0/mem-doc).


<!-- Reusable links -->

[api_dllimport]:https://docs.microsoft.com/dotnet/api/system.runtime.interopservices.dllimportattribute
[api_iunknown]:https://docs.microsoft.com/windows/win32/api/unknwn/nn-unknwn-iunknown

[overview_blittable]:https://docs.microsoft.com/dotnet/framework/interop/blittable-and-non-blittable-types
[overview_com]:https://docs.microsoft.com/windows/win32/com/the-component-object-model
[overview_pinning]:https://docs.microsoft.com/dotnet/framework/interop/copying-and-pinning
[overview_pinvoke]:https://docs.microsoft.com/dotnet/standard/native-interop/pinvoke

[spec_ecma335]:https://www.ecma-international.org/publications-and-standards/standards/ecma-335/

[repo_cswinrt]:https://github.com/microsoft/CsWinRT
[repo_dnne]:https://github.com/AaronRobinsonMSFT/DNNE

[wiki_x86callconv]:https://en.wikipedia.org/wiki/X86_calling_conventions