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
- On _Sherlock_ add the following line to your `.bashrc` script. 
    - `export JULIA_DEPOT_PATH=$GROUP_HOME/<USER_DIRECTORY>/.julia`

### `JULIAUP_DEPOT_PATH`
- Similar to `JULIA_DEPOT_PATH` but where Juliaup stores Julia versions and configuration files.
- Also defaults to `~/.julia`.
- Like `JULIA_DEPOT_PATH` this needs setting on _Sherlock_ via the `.bashrc` script.
    - `export JULIAUP_DEPOT_PATH=$GROUP_HOME/<USER_DIRECTORY>/.julia`




