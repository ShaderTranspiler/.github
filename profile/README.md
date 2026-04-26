# ShaderTranspiler

A performance-focused bridge between high-level CPU-side user code and native GPU execution.

The ShaderTranspiler organization hosts the STC (Shader Transpiler Core) ecosystem — a suite of tools designed to compile Julia-based shader definitions into optimized, production-ready GLSL. By leveraging a high-performance C++ engine and a seamless Julia interface, we enable developers to write expressive, type-safe shaders without leaving the Julia environment.

## Quick Note on the Current State

The project currently only supports Julia to GLSL transpilation, as this was the initial motivation behind the project, and is currently the only real application of it.
The transpiler is currently being developed as part of my BSc thesis (for which this was the intended endgoal), and thus this will remain so until the deadline at least.

The transpiler is still heavily under development, and one of the first goals to achieve after an initial release will be providing support to other high-level shader languages as backends.

The structure of the core library (stc) is already much more general than the above limitation would imply: GLSL code generation is completely decoupled from the frontend Julia parsing, the two stages separated by a general Shader AST (SIR).
The SIR itself relies partially on GLSL-specific semantics, but it will be generalized later on, when new backend targets are implemented.

The library also provides general data structures and algorithms for high-performance ASTs, type systems, interning capabilities, visitors and much more. This shared foundation both allows modular development without having to reimplement everything from scratch per language, but also serves as an optimization possibility for general transpilation speed.

Overall, while the current capabilities of the transpiler are limited to Julia -> GLSL transpilation, it was designed with modularity and extensibility in mind from the start.

# Motivation

When designing high-level computer graphics libraries, scenarios come up sometimes where high-level user code needs to be executed a large number of times with different inputs.
One such case is the tessellation of parametric geometry: a general geometry visualization libary (e.g. [Juliagebra](https://github.com/Csabix/Juliagebra)) probably wants to let end users define their own parametric function.
However, this can quickly become a bottleneck in a real time rendering pipeline, if the geometry needs to be tessellated frequently.

One solution to the problem would be to rely on general GPU computing focused libraries, but that quickly introduces two problems. The end user is now responsible for "thinking in terms of the GPU", which may include having to use custom types, or provide extra annotation for variables.
It also might be the case that the language doesn't provide libraries for interacting with higher level GPU frameworks. Julia, for example, does not have a stable OpenGL package, it only provides wrappers around general-purpose frameworks like CUDA or OpenCL.
For applications that use OpenGL, while it is possible to interop between these two, it can quickly lead to introducing achitecture fragmentation into the codebase.

ShaderTranspiler aims to provide a slightly different approach: source-to-source transpilation that bridges the gap between high-level mathematical code and lower level GPU logic.
Instead of requiring the user to manage low-level compute kernels, it transforms a subset of standard Julia functions directly into GLSL code.
This approach ensures that geometric modeling remains both accessible to non-developers, and efficient enough for real-time visualization.

Naturally, this also means that the possibilities are more limited than what is allowed by general purpose GPU computing libraries.
For use cases where there isn't a clear separation between some "blackbox" end-user code, and the rest of the graphics pipeline, there are already plenty of options out there which provide more flexibility and maturity.

# The Ecosystem

## Transpiler

The architecture is built around a core library (stc), ensuring that the heavy lifting of transpilation is handled by a robust, cross-platform engine written in C++20.

- [core](https://github.com/ShaderTranspiler/core): The heart of the project: a 10K+ SLOC C++ library (stc) that handles frontend AST parsing, semantic analysis, and target code generation. It is designed and developed with speed and modularity in mind. It contains both general transpilation code, as well language-specific passes.

- [stc_jll.jl](https://github.com/ShaderTranspiler/stc_jll.jl) The binary distribution layer for Julia usage. It's powered by [BinaryBuilder.jl](https://github.com/JuliaPackaging/BinaryBuilder.jl), and provides pre-compiled versions of stc for multiple platforms (currently Linux and Windows, with MacOS hopefully coming soon) across multiple Julia versions (currently 1.10-1.12).

- [ShaderTranspiler.jl](https://github.com/ShaderTranspiler/ShaderTranspiler.jl) The primary entry point for Julia users. A high-level Pkg wrapper that provides both memory-safe, and lower level APIs for interacting with the core library, handling Julia-based transpilations. It relies on `stc_jll.jl` to handle binary distribution.

## Tooling

- [ShaderSandbox.jl](https://github.com/ShaderTranspiler/ShaderSandbox.jl) A simple app that allows quickly and easily testing the transpiler through a [Shadertoy](https://shadertoy.com)-like sandbox environment. Though it was mainly designed for testing the transpiler, it can be used as a general sandbox environment as well for writing vertex and fragment shaders.

# Getting Started

To begin transpiling shaders in your Julia project:

```julia
using Pkg
Pkg.add("https://github.com/ShaderTranspiler/ShaderTranspiler.jl")

using ShaderTranspiler

julia_code = quote
    @gl_out     global pos::Vec3
    @gl_in      global uv::Vec2
    @gl_uniform global r::Float32
    @gl_uniform global R::Float32

    function main()
        u = uv[:x]
        v = uv[:y]

        # any expensive calculations may come here
        # (e.g. tessellation of a parametric surface at parameters u and v)

        x = r * sin(v)
        y = (R + r * cos(v)) * sin(u)
        z = (R + r * cos(v)) * cos(u)

        pos = Vec3(x, y, z)
    end
end

shader_code = transpile(julia_code)

println(shader_code)
```

Notice how the only place where GPU-specific code needs to be written is when applying type qualifiers.
This enables automatized generation of shader code that performs complex calculations from end-user code written in plain Julia, by wrapping it in a `main` function, and prepending some fixed code snippet to define the shader interface.

## Contributing

The project is currently being developed and maintained solely by [me](https://github.com/szgerii). As the base architecture is still under heavy development and faces constant restructuring, this will remain so for the foreseeable future.

The primary reason for creating this organization was to group the various repos of the project together, not fostering an open source community (though, obviously, that'd be highly unlikely to happen anyways). 
**Any major external contributions will be rejected. Small bug fixes or minor QoL improvements are welcome.**

## License

Despite not accepting any external contributions, most of the ShaderTranspiler project's source code is licensed under the MIT License, allowing free use and redistribution of the codebase. See individual repositories for specific licensing details.

