# Julia Best Practices
Best practices for Julia, particularly when working on Stanford's _Sherlock_ cluster.

## Installation

- Use **Juliaup** to install and manage Julia versions.
    - See https://github.com/julialang/juliaup for instructions to install and use `juliaup`.
    - Use `juliaup override set` to set the Julia version for a particular project directory.

## Environment variables

### `JULIA_DEPOT_PATH`
- Determines where the Julia package manager and code loading mechanisms look for things. 
- By default `~/.julia`.
- On HPC clusters like _Sherlock_ this is often an issue because the package cache will become large and hit the home directory quota.
- I recommend creating a `.julia` directory within your personal subdirectory in `$GROUP_HOME$`.
- On _Sherlock_ add the following line to your `.bashrc` script (this lives in your `$HOME` directory at `~/.bashrc`). 
    ```bash
    export JULIA_DEPOT_PATH=$GROUP_HOME/<USER_DIRECTORY>/.julia
    ```

### `JULIAUP_DEPOT_PATH`
- Similar to `JULIA_DEPOT_PATH` but where Juliaup stores Julia versions and configuration files.
- Also defaults to `~/.julia`.
- Like `JULIA_DEPOT_PATH` this needs setting on _Sherlock_ via the `.bashrc` script.
    ```bash
    export JULIAUP_DEPOT_PATH=$GROUP_HOME/<USER_DIRECTORY>/.julia
    ```

## Projects
- By default Julia will run in a base environment for that version.
    - For example, if I run 
        ```bash
        $ julia
        ```
        This open the Julia **REPL**. If I switch to **Package Mode** with `]` then I see
        ```julia
        (@v1.12) pkg> 
        ```
- We want to keep the base environments clean.
- All work should be done within a **project**.
- There are two ways to activate a new project.
    - If you are already in **Package Mode** you can use the `activate` command
        ```julia
        (@v1.12) pkg> activate foo
        ```
        ```plain
        Activating new project at `~/foo`
        ```
    - From the command line specify the new project directory explicitly.
        ```bash
        $ julia --project="foo"
        ```
- Once you have a project set up (i.e. there is a `Project.toml` file in the top-level directory of the project) you can run
    ```bash
    $ julia --project
    ```
    and Julia will find the `Project.toml` file (ascending up the directory tree if you run a subdirectory of the project).
- **Make lots of projects!**
    - The package cache in the Julia depot is your friend.
    - If you start something new, make a new project!


## System binaries

There are certain packages for which we really want to make use of the system binaries already installed and configured on _Sherlock_. 
The most obvious examples are `MPI.jl`, for multi-processing using MPI, and `CUDA.jl` for running on cuda GPUs.
Both packages do this via `LocalPreferences.jl` and have a convenient interface.

### MPI.jl

- These instructions largely follow https://juliaparallel.org/MPI.jl/stable/configuration/
- Step 1: load a Sherlock mpi module. 
    - At the time of writing I recommend `module load openmpi/5.0.5` which is the latest and has cuda support.
- Step 2: add the `MPIPreferences.jl` package to our project.
    - In the project directory run Julia 
        ```bash
        $ julia --project
        ```
    - If this is a clean project with no `Project.toml` file you will need to run 
        ```bash
        $ julia --project="."
        ```
    - Enter **Package Mode** by hitting `]`.
    - Add the `MPIPreferences` package with `add MPIPreferences`.
        ```julia
        (directory) pkg> add MPIPreferences
        ```
- Step 3: use the package to detect the system MPI
    - Leave **Package Mode** and enter the **REPL** by hitting `<backspace>`.
    - Load the package and run the function to detect the system MPI
        ```julia
        julia> using MPIPreferences

        julia> MPIPreferences.use_system_binary()
        ```
        ```plain
        ┌ Info: MPI implementation identified
        │   libmpi = "libmpi"
        │   version_string = "Open MPI v5.0.5, package: Open MPI kilian@sh02-hn01.SUNet Distribution, ident: 5.0.5, repo rev: v5.0.5, Jul 22, 2024\0"
        │   impl = "OpenMPI"
        │   version = v"5.0.5"
        └   abi = "OpenMPI"
        ┌ Info: MPIPreferences changed
        │   binary = "system"
        │   libmpi = "libmpi"
        │   abi = "OpenMPI"
        │   mpiexec = "mpiexec"
        │   preloads = Any[]
        └   preloads_env_switch = nothing
        ```
- Step 4: Exit Julia and inspect the `LocalPreferences.toml` file
    ```bash
    $ cat LocalPreferences.toml
    ```
    ```plain
    [MPIPreferences]
    __clear__ = ["preloads_env_switch"]
    _format = "1.0"
    abi = "OpenMPI"
    binary = "system"
    cclibs = []
    libmpi = "libmpi"
    mpiexec = "mpiexec"
    preloads = []
    ```
- Step 5: Set a UCX environment variable to stop UCX intercepting harmless Julia signals
    ```bash
    $ export UCX_ERROR_SIGNALS="SIGILL,SIGBUS,SIGFPE"
    ```
    - You will need to set this whenever working with Julia MPI.
- Step 6: Test everything is working by installing and initialising the MPI package.
    - In **Package Mode**
        ```julia
        (directory) pkg> add MPI
        ```
    - In **REPL mode**
        ```julia
        julia> using MPI

        julia> MPI.Init()

        julia> MPI.versioninfo()
        ```
        ```plain
        MPIPreferences:
            binary:  system
            abi:     OpenMPI
            libmpi:  libmpi
            mpiexec: mpiexec

        Package versions
            MPI.jl:             0.20.28
            MPIPreferences.jl:  0.1.12

        Library information:
            libmpi:  libmpi
            libmpi dlpath:  /share/software/user/open/openmpi/5.0.5/lib/libmpi.so
            MPI version:  3.1.0
            Library version:
            Open MPI v5.0.5, package: Open MPI kilian@sh02-hn01.SUNet Distribution, ident: 5.0.5, repo rev: v5.0.5, Jul 22, 2024
            MPI launcher: mpiexec
            MPI launcher path: /share/software/user/open/openmpi/5.0.5/bin/mpiexec
        ```
    - The version info should reflect the system MPI.

### CUDA.jl

- These instructions are based on https://cuda.juliagpu.org/stable/installation/overview/
- `CUDA.jl` is slightly different in that we don't need a special package just to write the `LocalPreferences.toml`.
- Step 1: We need the system cuda. Note that you don't need to be on a GPU node at this point.
    ```bash
    $ module load cuda/12.6.1
    ```
    Note that `cuda/12.6.1` is the module loaded by `openmpi/5.0.5`. For example,
    ```bash
    $ module purge
    $ module load openmpi/5.0.5
    $ module list
    ```
    ```plain
    Currently Loaded Modules:
        1) devel (S)   2) math (S)   3) gcc/12.4.0   4) cuda/12.6.1 (g)   5) ucx/1.17.0 (g)   6) nccl/2.23.4 (g)   7) ucc/1.3.0 (g)   8) openmpi/5.0.5 (g,m)

        Where:
            S:  Module is Sticky, requires --force to unload or purge
            g:  GPU support
            m:  MPI support
   ```
   You can use these together for distributed GPU work or CUDA by itself for a single GPU.
- Step 2: Add the `CUDA.jl` package to your project
    ```bash
    $ julia --project
    ```
    ```julia
    julia> ]

    (directory) pkg> add CUDA
    ```
- Step 3: Set the runtime version to the system cuda.
    ```julia 
    (directory) pkg> <backspace>

    julia> using CUDA

    julia> CUDA.set_runtime_version!(v"12.6.1", local_toolkit=true)
    ```
    The v-sting syntax `v"12.6.1"` creates a `VersionNumber` object. This needs to match the CUDA version you have loaded.
- Step 4: Check the `LocalPreferences.toml` and `Project.toml` files.
    ```bash
    $ cat LocalPreferences.toml
    ```
    ```plain
    [CUDA_Compiler_jll]
    local = "true"
    version = "12.6"

    [CUDA_Runtime_jll]
    local = "true"
    version = "12.6"
    ```
    You should see these two entries in addition to any other entries for other packages (e.g. MPIPreferences).
    ```bash
    $ cat Project.toml
    ```
    ```plain
    [deps]
    CUDA = "052768ef-5323-5732-b1bb-66c8b64840ba"

    [extras]
    CUDA_Compiler_jll = "d1e2174e-dfdc-576e-b43e-73b79eb1aca8"
    CUDA_Runtime_jll = "76a88914-d11a-5bdc-97e0-2f5a05c973a2"
    ```
    The key here is the last line `CUDA_Runtime_jll = "76a88914-d11a-5bdc-97e0-2f5a05c973a2"` which is a dummy package that tells Julia to use the local cuda runtime.
- Step 5: Request a GPU node and check everything is working. Here using the `dev` partition GPUs is a good option.
    ```bash
    $ sdev -g 1
    ```
    Ensure cuda is loaded and then run Julia
    ```bash
    $ module load cuda/12.6.1
    $ julia --project
    ```
    ```julia
    julia> using CUDA
    julia> CUDA.versioninfo()
    ```
    ```plain
    CUDA toolchain:
    - runtime 12.6.0, local installation
    - driver 550.163.1 for CUDA 12.4
    - compiler 12.6.68, local installation

    CUDA libraries:
    - cuBLAS: 12.6.1
    - cuSPARSE: 12.5.3
    - cuSOLVER: 11.6.4
    - cuFFT: 11.2.6
    - cuRAND: 10.3.7
    - CUPTI: 2024.3.1 (API 12.6.0)
    - NVML: 12.0.0+550.163.1

    Julia packages:
    - CUDACore: 6.4.2
    - GPUArrays: 11.5.16
    - GPUCompiler: 2.14.1
    - KernelAbstractions: 0.9.45
    - CUDA_Driver_jll: 13.3.5+0
    - CUDA_Compiler_jll: 0.6.3+2
    - CUDA_Runtime_jll: 0.25.0+3
    - CUDA_Runtime_Discovery: 2.1.1
    - NVPTX_LLVM_Backend_jll: 23.1.2+1

    Toolchain:
    - Julia: 1.12.6
    - LLVM: 18.1.7

    Preferences:
    - CUDA_Runtime_jll.version: 12.6
    - CUDA_Runtime_jll.local: true
    - CUDA_Compiler_jll.version: 12.6
    - CUDA_Compiler_jll.local: true

    1 device:
    0: NVIDIA A30 MIG 1g.6gb (sm_80, 5.798 GiB / 5.812 GiB available)
        compiles to sm_80 / PTX 8.5
    ```