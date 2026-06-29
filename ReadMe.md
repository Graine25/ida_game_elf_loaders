# ida_game_elf_loaders
A collection of user mode ELF loaders for the following game consoles:
* ~~PS3~~
* ~~PS Vita~~
* Wii U

You should use other repositories for the non-Wii U ELF loaders, as this one is only aimed at providing an updated version of the RPX loader. 

## Installation
Copy loader plugins to IDA loaders directory.

## Building

### Dependencies
* IDA SDK
* [CMake](https://cmake.org/download/)

### Wii U loader
The maintained Wii U loader builds on 64-bit Windows, macOS, and Linux. The
bundled IDA SDK submodule contains libraries for these targets:

* Windows x86_64
* macOS arm64 and x86_64
* Linux arm64 and x86_64

Initialize the submodules before configuring:

```sh
git submodule update --init --recursive
```

Configure and build a native macOS or Linux release:

```sh
cmake -S src/wiiu -B build/wiiu -DCMAKE_BUILD_TYPE=Release
cmake --build build/wiiu
```

On macOS, select a non-native architecture explicitly when needed. Keep each
architecture in a separate build directory:

```sh
cmake -S src/wiiu -B build/wiiu-x64 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_OSX_ARCHITECTURES=x86_64
cmake --build build/wiiu-x64
```

On Windows, use a 64-bit Visual Studio generator:

```powershell
cmake -S src/wiiu -B build/wiiu -A x64
cmake --build build/wiiu --config Release
```

The resulting loader is named `wiiu.dll`, `wiiu.dylib`, or `wiiu.so`
according to the host platform. Copy it into IDA's `loaders` directory.

## Notes

The current commit was compiled and tested to work with IDA 9.3.
It also comes with improved Wii U RPX/RPL support, including compressed sections,
FILE_INFO parsing, import/export metadata, and a broader set of relocations.
