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

## System binaries

There are certain packages for which we really want to make use of the system binaries already installed and configured on _Sherlock_. 
The most obvious examples are `MPI.jl`, for multi-processing using MPI, and `CUDA.jl` for running on cuda GPUs.
Both packages do this via `LocalPreferences.jl` and have a convenient interface.

### MPI.jl

- Step 1: load a Sherlock mpi module. 
    - At the time of writing I recommend `module load openmpi/5.0.5` which is the latest and has cuda support.
- Step 2: add the `MPIPreferences.jl` package to our project.
    - In the project directory run Julia 
        ```bash
        $ julia --project
        ```
    - If this is a clean project with no `Project.toml` file you will need to run 
        ```bash
        julia --project="."
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