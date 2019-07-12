---
layout: post
title: Using DXC In Practice
---

[DirectX Shader Compiler (DXC)](https://github.com/microsoft/DirectXShaderCompiler) is a necessary component to compile SM6 shader code. If you only want to use it to compile shaders, all DXR tutorials will include so but that's basically it.
In my recent project, I wanted to build my own shader pipelines which requires more different operations, where I found they are not sufficient to accomplish all the tasks. At the same time, since the compiler is still in relatively early stage, the documentation is not very detailed. So here I am :) and I hope everyone wants to play with the compiler can find what they need from this post.

## Getting Started

To use DXC in your own project, we need three things: `dxcompiler.dll` and `dxil.dll` for running and header files for compiling. To get it, you can manually build the source files from the repo or install the latest Windows Dev Kit(I had 17763). For WDK, the dlls are located at `C:\Program Files (x86)\Windows Kits\10\bin\<version#>\x64`, be aware the prefix path may vary for you. When you test your code, make sure that you set up the environment and double check they are avaible at runtime to your executable file.

![Checking dll available](/public/images/19-7-11-dxcompiler-available.png)

For header files, the WDK only has `dxcapi.h`, which is not enough for my cases. I recommend for now, get the header files from the official repo and make them all available to your project.

MS provided a [bridge dll](https://github.com/microsoft/DirectXShaderCompiler/wiki/D3DCompiler-DXC-Bridge), but only with limited features. But it's a great reference to start with. The source code is [here](https://github.com/microsoft/DirectXShaderCompiler/blob/master/tools/clang/tools/d3dcomp/d3dcomp.cpp). which is the reference of the most of code I will post here.

To start with, you need to include `dxc/include/dxc/Support/dxcapi.use.h`, where it provides a nice wrapper to load in DLL and some core functions. This should be something global and only have one.
```cpp
static dxc::DxcDllSupport support;

STATIC_BLOCK {
    support.Initialize();
}
``` 

## Compile Shader

To compile a shader from a buffer in the most simple way, you only need three steps:
1. Provide a buffer in type of `IDxcBlob` - It's also ok to derive from this interface class by yourself.
2. Create `IDxcCompiler*`, which is the actual compiler instance.
3. Call `IDxcCompile::Compile` and feed in your buffer to compile the shader.

```cpp
IDxcCompiler*        compiler;
IDxcOperationResult* operationResult;
assert_win( support.CreateInstance( CLSID_DxcCompiler, &compiler ) );

HRESULT hr = compiler->Compile(pSource, "shader_name", L"entry_point", L"ps_6_1",
                  arguments, argumentCount,
                  defines, defineCount, pInclude, // `pInclude` - IDxcIncludeHandler*
                  &operationResult) );
if(SUCCEEDED( hr )) {
    HRESULT result =  operationResult->GetResult( (IDxcBlob **)ppCode );
    // handle result
} else {
    ID3DBlob **ppErrorMsgs;
    operationResult->GetErrorBuffer( (IDxcBlobEncoding **)ppErrorMsgs );
    // print, display error message
}                 
```

The `Compile` function is pretty straightforward. You can basically maps all arguments to `D3DCompile`. After compilation, the byte code is available in the blob and is the equivalent to the ones from the `D3DCompiler`... not exact the same actually. Remember when we setting up the environment, we need two dlls, the one is `dxcompiler.dll`, which is the library for the compiler obviously. But `dxil.dll` is also a important part. If you don't have it during runtime, you will get an error like this:

> D3D12 ERROR: ID3D12Device::CreateVertexShader: Vertex Shader is corrupt or in an unrecognized format. [ STATE_CREATION ERROR #322: CREATEVERTEXSHADER_INVALIDSHADERBYTECODE]

It's because `dxcompiler` will not sign the shader code if it cannot find `dxil.dll` so that it's not usable for user's computer. [here](https://blogs.msdn.microsoft.com/marcelolr/2018/03/06/using-the-github-dxcompiler-dll/) is a post with some more explanation. But it seems that warning is not a very relable test for you to determine whether missing `dxil.dll` is the reason to the corrupt error because I did not get that when I was doing the experiement. [Graham's post](https://www.wihlidal.com/blog/pipeline/2018-09-16-dxil-signing-post-compile/) had more discussion about signing the shader. You should check that out if you are interested.

Furthermore, if you want to compile shader from a file , you will need some extra steps to construct the buffer we mentioned before:
1. Create `IDxcLibrary*` to load in files, which can also handle file encoding if you need to.
2. Use `IDxcLibrary::CreateBlobXXXX` functions to create the buffer, whose type is either `IDxcBlob` or `IDxcBlobEncoding`(derived from `IDxcBlob`)

```cpp
IDxcLibrary*        library;
IDxcBlobEncoding*   source;
assert_win( support.CreateInstance( CLSID_DxcLibrary, &library ) );
assert_win( library->CreateBlobFromFile( pFileName, nullptr, &source ) );
```

In case if you want a standard include handler, instead of passing in `D3D_COMPILE_STANDARD_FILE_INCLUDE` for compile function, you also need a `IDxcLibrary` instance to create one.

```cpp
IDxcIncludeHandler* includeHandler = nullptr;
assert_win( library->CreateIncludeHandler(&includeHandler) );
```

## Shader Reflection

Creating reflection object is very similar:

```
   IDxcContainerReflection* pReflection;
   IDxcLibrary*             pLibrary;
   UINT32                   shaderIdx;

   assert_win( support.CreateInstance( CLSID_DxcContainerReflection, &pReflection ) );
   assert_win( support.CreateInstance( CLSID_DxcLibrary, &pLibrary ) );

   assert_win( pReflection->Load( &pBlob ) );
   assert_win( pReflection->FindFirstPartKind( hlsl::DFCC_DXIL, &shaderIdx ) );
   assert_win( pReflection->GetPartReflection( shaderIdx, IID_PPV_ARGS(&pShaderReflection )) );
```

There are two things may potentially bring confusion. If you were using `dxcapi.h` from WDK only, you would not be able to compile this code. Because `hlsl::DFCC_DXIL` is defined in `dxc/DxilContainer/DxilContainer.h`. The second thing is more related to Visual Studio. VS has a built-in HLSL compiler, which can compile shader files of SM6 and can be avaiable as header files or binary files. But you will not be able to feed these binary files(even they are SM6) into `IDxcContainerReflection` and it will complain. I am not sure the reason, but I would appreciate it if someone can clarify this :).

## Create Root Signature from Shader

This was an issue I kept fighting for a while. But turned out you can just pass in the shader blob itself to `ID3D12Device::CreateRootSignature`. Because the API expects a signed blob container with a root signature part. `D3D12CreateRootSignatureDeserializer` works in the same way. Here is a related [issue](https://github.com/microsoft/DirectXShaderCompiler/issues/2315) from the repo.


