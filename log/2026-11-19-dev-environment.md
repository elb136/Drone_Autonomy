# 2026-11-19: Dev environment, part 1

**Phase:** 0, Dev environment · **Time:** one evening

## Done
- Ubuntu 22.04 VM on arm64 with the C++ toolchain: gcc 11, clang/clangd 14, CMake 3.22 + Ninja, gdb, ccache, bear
- Git identity and SSH authentication to GitHub
- VS Code with clangd for code navigation and the C/C++ extension for debugging; CMake set to export `compile_commands.json`
- End-to-end check on a CMake hello-world: build, breakpoint and variable inspection in gdb, clangd hover

## What broke, and the fix
- **SSH `Permission denied (publickey)`.** The key was saved under a custom filename outside `~/.ssh/`, so SSH never offered it. Moved it to `~/.ssh/id_ed25519` and set permissions to 700/600.
- **VS Code snap not available on arm64.** Installed from Microsoft's apt repository instead, which ships arm64 builds.

## Learned
- How a C++ build is structured: compiler (one file → object file), linker (object files → program), build system (what to rebuild, with which flags), and why ccache makes rebuilds fast.
- Why clangd needs `compile_commands.json`: it must see the same include paths and defines as the compiler, or it can't resolve anything.
- clangd and Microsoft's IntelliSense conflict. Use one for code understanding (clangd), the other only for debugging.

## Risk noted
Gazebo needs 3D acceleration that a QEMU VM may not provide. To be decided before Phase 3.

## Next
ArduPilot clone → SITL build → `compile_commands.json` for the ArduPilot tree → first SITL flight → SITL under gdb. ([guide 01](../guides/01-dev-environment.md))
