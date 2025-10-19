---
layout: post
title: Mockingbird
subtitle: A Win32 API code generator
gh-repo: zaddach/mockingbird
gh-badge: [star, fork, follow]
tags: [win32]
comments: true
mathjax: true
author: Jonas Zaddach
---

It's been a while that I've used the Rust [windows](https://crates.io/crates/windows) crate and wished that I could generate code for the Win32 API in a similar way. Well, it turns out I can! The `windows` crate actually takes the metadata used to generate APIs from the [Win32 metadata](https://github.com/microsoft/win32metadata) project. The project provides the tooling to generate API metadata from header files. Metadata is stored in winmd format, which is a special form of a Concurrent Language Runtime (CLR) binary that contains only type information.

Microsoft has already done the laborous part of the work to index all Win32 API header files, and they are
publishing the resulting winmd file as a [nuget package](https://www.nuget.org/packages/Microsoft.Windows.SDK.Win32Metadata).

This is where `mockingbird` enters the picture. It downloads the nuget package, extracts the winmd file,  and uses the [windows-metadata](https://crates.io/crates/windows-metadata) crate to access the contained type information. You can configure the APIs you're interested in in a YAML configuration file, and provide [tera](https://keats.github.io/tera/docs/) templates to be filled with the type data.

Mind that this is a proof of concept, there's currently no error handling in the application (it is littered with `.unwrap()`).

## Example: A Win32 API mock

### The configuration file

My driving motivation for this project was to have a layer on top of the Win32 API that allows for easy mocking. Usually I ended up writing API mocks again and again for each project, but that is error-prone and cumbersome. So let's try to generate a mock from the metadata!

First we need the YAML configuration:
```
# Example configuration for GDI functions
api:
  "Windows.Win32.Foundation":
    - CloseHandle
  "Windows.Win32.Storage.FileSystem":
    - CreateFileW

type_aliases:
  "FILE_CREATION_DISPOSITION": "uint32_t"
  "FILE_FLAGS_AND_ATTRIBUTES": "uint32_t"
  "FILE_SHARE_MODE": "uint32_t"
  "WIN32_ERROR": "uint32_t"

templates:
  - "templates/IWin32Api.hpp"
  - "templates/Win32Api.hpp"
  - "templates/MockWin32Api.hpp"
  - "templates/MockWin32Api.cpp"
  - "templates/Win32Api.cpp"

includes:
  "../common":
    - "macros.hpp"
```

We are just mocking two functions here, `CloseHandle` and `CreateFileW`, which are listed in the `api` section of the configuration. If you want to look up the function's namespace, you can either use [ILSpy](https://github.com/icsharpcode/ILSpy) to inspect the winmd file, or you can use the [API documentation](https://microsoft.github.io/windows-docs-rs/) of the windows crate to search for it.

Because the windows metadata APIs often provide more specific types for function arguments that represent either flags or enumerations, but are just plain data types such as `DWORD` in the C headers, we need to alias those more specific types to their plain data type. That is done in the `type_aliases` configuration section, which just provides the mappings. In the generated C++ code, this will expand to `using alias = plain_data_type` statements.

Next, we need templates to be filled with the metadata. Those are specified in the `templates` section of the configuration file.

And finally, templates can include other templates. I'm using this mechanism in this project to provide a template macro library for expanding C++ types and function argument lists. Include files are specified in the `includes` section.

### A template

Here's one of the template files:
```
{% raw %}{% import "macros.hpp" as macros  %}
// This file is generated. Do not edit.
#pragma once

#include <Windows.h>
#include <cstdint>
#include <cstddef>

{% for alias, type in type_aliases %}
using {{ alias }} = {{ type }};
{% endfor %}

class IWin32Api {
public:
    {% for function in functions %}
    virtual {{ macros::cxx_type(type = function.return_type) }} {{ function.name -}}({{- macros::cxx_arguments_with_names(args = function.params) -}}) = 0;
    {% endfor %}
};{% endraw %}
```

Most of the code should be pretty self-explanatory, especially if you have worked with Django templates before (tera is heavily inspired by the Django templating language). You can see that there are two iterations, one for the type aliases loaded straight from the configuration, and a second one for the API functions. The API function one uses macros. For example, the `macros::cxx_type` is imported from the file `macros.hpp` in the first line of the file.

### The template macro library
Here's an excerpt of the `macro.hpp` file:
```
{% raw %}{%- macro cxx_type(type) -%}
%- if type.type == TYPES.Void -%}
void
{%- elif type.type == TYPES.Bool -%}
bool
{%- elif type.type == TYPES.Char -%}
char
...
{%- elif type.type == TYPES.Name -%}
{{ type.name }}
{%- elif type.type == TYPES.PtrMut -%}
{{ self::cxx_type(type = type.inner) }}*
...
{%- else -%}
{{ throw(message = "Unsupported type") }}
{%- endif -%}
{%- endmacro -%}{% endraw %}
...
```

You can see that the tera "scripting" language is pretty powerful and even allows recursive macro invocations. We need recursion here to unpack nested types such as pointers and references. I can completely convert the type metadata into C++ types without writing a line of Rust code.

### Generating the files
Run
```
cargo run -- ./examples/mocking/mocking.yaml
```
to generate the code from the templates. You'll find the files `IWin32Api.hpp`, `Win32Api.hpp`, `Win32Api.cpp`, `MockWin32Api.hpp`, `MockWin32Api.cpp` in your current directory.

Let's check the `IWin32Api.hpp` output:
```c++
// This file is generated. Do not edit.
#pragma once

#include <Windows.h>
#include <cstdint>
#include <cstddef>


using FILE_CREATION_DISPOSITION = uint32_t;

using FILE_FLAGS_AND_ATTRIBUTES = uint32_t;

using FILE_SHARE_MODE = uint32_t;

using WIN32_ERROR = uint32_t;


class IWin32Api {
public:

    virtual HANDLE CreateFileW(const PWSTR lpFileName, uint32_t dwDesiredAccess, FILE_SHARE_MODE dwShareMode, SECURITY_ATTRIBUTES* lpSecurityAttributes, FILE_CREATION_DISPOSITION dwCreationDisposition, FILE_FLAGS_AND_ATTRIBUTES dwFlagsAndAttributes, HANDLE hTemplateFile) = 0;

    virtual BOOL CloseHandle(HANDLE hObject) = 0;

};
```

Looks good! The type aliases as well as the function declarations have been generated.

Now let's do a final check to see our C++ code is compiling:
```ps1
cl.exe -std:c++20 -EHsc -I. -c -o Win32Api.cpp.obj Win32Api.cpp
# I'm getting the gtest include files from the conan package
$gtest_path = conan cache path gtest/1.17.0:3d1ae27944dad02a17ad08beed39ed6081da8ac6
cl.exe -std:c++20 -EHsc -I. "-I${gtest_path}/include" -c -o MockWin32Api.cpp.obj MockWin32Api.cpp
```

The sources are compiling.

## Conclusion
It took me some tinkering to understand the `windows-metadata` crate, but once you know how to extract types it is fairly straightforward. I was pleasantly surprised how easy the `tera` crate was to use, and how powerful the macro system is.

You can find the code for this project [here](https://github.com/zaddach/mockingbird).
