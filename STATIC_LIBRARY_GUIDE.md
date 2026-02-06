# GameNetworkingSockets Static Library Guide

## Table of Contents
1. [Overview](#overview)
2. [Building as a Static Library](#building-as-a-static-library)
3. [Using the Static Library in Your Project](#using-the-static-library-in-your-project)
4. [Platform-Specific Instructions](#platform-specific-instructions)
5. [Linking Requirements](#linking-requirements)
6. [CMake Integration](#cmake-integration)
7. [Manual Integration](#manual-integration)
8. [Troubleshooting](#troubleshooting)

## Overview

GameNetworkingSockets can be built as either a static library (`.a` on Linux/Mac, `.lib` on Windows) or a shared/dynamic library (`.so`, `.dylib`, `.dll`). This guide focuses on building and using the **static library** version, which:

- **Simplifies deployment**: No need to distribute separate DLL/SO files
- **Reduces conflicts**: No version conflicts with other software
- **Enables optimization**: Linker can optimize across library boundaries
- **Improves portability**: Single executable with all dependencies

### Prerequisites

Before building, ensure you have:

- **CMake 3.10 or later**
- **C++11 compatible compiler** (GCC 7.3+, Clang 3.3+, MSVC 2017+)
- **Crypto library**: OpenSSL 1.1.1+, libsodium, or BCrypt (Windows)
- **Google protobuf** 2.6.1+
- **Build tool**: Ninja (recommended), GNU Make, or Visual Studio

## Building as a Static Library

### Quick Build (Linux/Mac)

```bash
# Clone the repository
git clone https://github.com/ValveSoftware/GameNetworkingSockets.git
cd GameNetworkingSockets

# Install dependencies (Ubuntu/Debian)
sudo apt install libssl-dev libprotobuf-dev protobuf-compiler

# Create build directory
mkdir build
cd build

# Configure with CMake (static library is ON by default)
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_STATIC_LIB=ON \
  -DBUILD_SHARED_LIB=OFF \
  ..

# Build
ninja

# The static library will be in:
# build/bin/libGameNetworkingSockets.a
```

### Quick Build (Windows with Visual Studio)

```powershell
# Clone the repository
git clone https://github.com/ValveSoftware/GameNetworkingSockets.git
cd GameNetworkingSockets

# Bootstrap vcpkg for dependencies (recommended)
git clone https://github.com/microsoft/vcpkg
.\vcpkg\bootstrap-vcpkg.bat

# Install dependencies
.\vcpkg\vcpkg install --triplet=x64-windows

# Configure with CMake
# Note: vcpkg toolchain is automatically detected if in recommended location
cmake -S . -B build -G "Visual Studio 17 2022" -A x64 `
  -DCMAKE_BUILD_TYPE=Release `
  -DBUILD_STATIC_LIB=ON `
  -DBUILD_SHARED_LIB=OFF

# Build
cmake --build build --config Release

# The static library will be in:
# build\bin\Release\GameNetworkingSockets.lib
```

### Build Options

Control what gets built with CMake options:

```bash
cmake .. \
  -DBUILD_STATIC_LIB=ON \          # Build static library (default: ON)
  -DBUILD_SHARED_LIB=OFF \         # Don't build shared library (default: ON)
  -DBUILD_EXAMPLES=OFF \           # Don't build examples (default: OFF)
  -DBUILD_TESTS=OFF \              # Don't build tests (default: OFF)
  -DBUILD_TOOLS=OFF \              # Don't build tools (default: OFF)
  -DUSE_CRYPTO=OpenSSL \           # Crypto backend: OpenSSL, libsodium, BCrypt
  -DUSE_STEAMWEBRTC=OFF \          # Disable P2P/ICE support (default: OFF)
  -DCMAKE_INSTALL_PREFIX=/usr/local  # Install location
```

### Installing the Library

After building, install to system directories:

```bash
# Install (may require sudo on Linux/Mac)
sudo cmake --install build

# This installs:
# - Library: /usr/local/lib/libGameNetworkingSockets.a
# - Headers: /usr/local/include/steam/
# - CMake config: /usr/local/lib/cmake/GameNetworkingSockets/
```

Or install to a custom directory:

```bash
cmake --install build --prefix /path/to/install/dir
```

## Using the Static Library in Your Project

### Option 1: CMake (Recommended)

If you use CMake for your project, integration is straightforward:

#### If Library is Installed System-Wide

```cmake
cmake_minimum_required(VERSION 3.10)
project(MyGame)

# Find the installed package
find_package(GameNetworkingSockets REQUIRED)

# Add your executable
add_executable(mygame
    src/main.cpp
    src/network.cpp
)

# Link against GameNetworkingSockets
target_link_libraries(mygame PRIVATE GameNetworkingSockets::GameNetworkingSockets)

# Note: Dependencies (OpenSSL, protobuf) are automatically linked
```

#### If Library is in Custom Location

```cmake
cmake_minimum_required(VERSION 3.10)
project(MyGame)

# Tell CMake where to find the library
set(GameNetworkingSockets_DIR "/path/to/install/lib/cmake/GameNetworkingSockets")

# Or use CMAKE_PREFIX_PATH
# list(APPEND CMAKE_PREFIX_PATH "/path/to/install")

find_package(GameNetworkingSockets REQUIRED)

add_executable(mygame src/main.cpp)
target_link_libraries(mygame PRIVATE GameNetworkingSockets::GameNetworkingSockets)
```

#### Building Your Project

```bash
mkdir mybuild
cd mybuild
cmake ..
make
```

### Option 2: Manual Linking (Non-CMake)

If you don't use CMake, you need to manually specify include paths and libraries:

#### Include Paths

Add to your compiler include search path:
```
/usr/local/include
/usr/local/include/steam
```

#### Library Paths

Add to your linker library search path:
```
/usr/local/lib
```

#### Libraries to Link

Link against these libraries (order matters on some platforms):

**Linux:**
```
-lGameNetworkingSockets -lssl -lcrypto -lprotobuf -lpthread
```

**macOS:**
```
-lGameNetworkingSockets -lssl -lcrypto -lprotobuf -lc++
```

**Windows (MSVC):**
```
GameNetworkingSockets.lib ws2_32.lib crypt32.lib
libssl.lib libcrypto.lib libprotobuf.lib
```

#### Example Makefile (Linux)

```makefile
CXX = g++
CXXFLAGS = -std=c++11 -I/usr/local/include
LDFLAGS = -L/usr/local/lib
LIBS = -lGameNetworkingSockets -lssl -lcrypto -lprotobuf -lpthread

SOURCES = main.cpp network.cpp
OBJECTS = $(SOURCES:.cpp=.o)
TARGET = mygame

$(TARGET): $(OBJECTS)
	$(CXX) $(LDFLAGS) -o $@ $^ $(LIBS)

%.o: %.cpp
	$(CXX) $(CXXFLAGS) -c $< -o $@

clean:
	rm -f $(OBJECTS) $(TARGET)
```

#### Example Compilation Command (Linux)

```bash
g++ -std=c++11 \
    -I/usr/local/include \
    -L/usr/local/lib \
    main.cpp network.cpp \
    -lGameNetworkingSockets \
    -lssl -lcrypto \
    -lprotobuf \
    -lpthread \
    -o mygame
```

## Platform-Specific Instructions

### Linux (Ubuntu/Debian)

#### Install Dependencies

```bash
sudo apt update
sudo apt install \
    build-essential \
    cmake \
    ninja-build \
    libssl-dev \
    libprotobuf-dev \
    protobuf-compiler
```

#### Build Static Library

```bash
git clone https://github.com/ValveSoftware/GameNetworkingSockets.git
cd GameNetworkingSockets

mkdir build && cd build
cmake -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_STATIC_LIB=ON \
    -DBUILD_SHARED_LIB=OFF \
    ..
ninja

# Install
sudo ninja install
```

#### Use in Your Project

```bash
# Compile your game
g++ -std=c++11 \
    -I/usr/local/include \
    main.cpp \
    -L/usr/local/lib \
    -lGameNetworkingSockets \
    -lssl -lcrypto -lprotobuf -lpthread \
    -o mygame
```

### macOS

#### Install Dependencies via Homebrew

```bash
brew install cmake ninja openssl protobuf

# OpenSSL may need explicit path
export PKG_CONFIG_PATH="/usr/local/opt/openssl@1.1/lib/pkgconfig:$PKG_CONFIG_PATH"
```

#### Build Static Library

```bash
git clone https://github.com/ValveSoftware/GameNetworkingSockets.git
cd GameNetworkingSockets

mkdir build && cd build
cmake -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_STATIC_LIB=ON \
    -DBUILD_SHARED_LIB=OFF \
    ..
ninja

# Install
sudo ninja install
```

#### Use in Your Project

```bash
clang++ -std=c++11 \
    -I/usr/local/include \
    main.cpp \
    -L/usr/local/lib \
    -lGameNetworkingSockets \
    -lssl -lcrypto -lprotobuf \
    -o mygame
```

### Windows (Visual Studio)

#### Install Dependencies via vcpkg

```powershell
# Clone GameNetworkingSockets
git clone https://github.com/ValveSoftware/GameNetworkingSockets.git
cd GameNetworkingSockets

# Bootstrap vcpkg (as subdirectory - recommended)
git clone https://github.com/microsoft/vcpkg
.\vcpkg\bootstrap-vcpkg.bat

# Install dependencies for x64
.\vcpkg\vcpkg install --triplet=x64-windows
```

#### Build Static Library

```powershell
# Configure (vcpkg toolchain auto-detected)
cmake -S . -B build -G "Visual Studio 17 2022" -A x64 `
    -DCMAKE_BUILD_TYPE=Release `
    -DBUILD_STATIC_LIB=ON `
    -DBUILD_SHARED_LIB=OFF

# Build
cmake --build build --config Release

# Install to custom location
cmake --install build --prefix C:\GameNetworkingSockets
```

#### Use in Visual Studio Project

1. **Open Project Properties**
2. **Configuration: All Configurations**
3. **Platform: x64**

4. **Add Include Directories**:
   - Configuration Properties → C/C++ → General → Additional Include Directories
   - Add: `C:\GameNetworkingSockets\include`

5. **Add Library Directory**:
   - Configuration Properties → Linker → General → Additional Library Directories
   - Add: `C:\GameNetworkingSockets\lib`

6. **Add Library Dependencies**:
   - Configuration Properties → Linker → Input → Additional Dependencies
   - Add:
     ```
     GameNetworkingSockets.lib
     libssl.lib
     libcrypto.lib
     libprotobuf.lib
     ws2_32.lib
     crypt32.lib
     ```

7. **Runtime Library** (important for static linking):
   - Configuration Properties → C/C++ → Code Generation → Runtime Library
   - Set to: `/MT` (Multi-threaded) for Release or `/MTd` (Multi-threaded Debug) for Debug
   - Must match how GameNetworkingSockets was built

#### Use with MSBuild (Command Line)

```powershell
# Compile
cl /EHsc /MT /std:c++11 ^
    /I"C:\GameNetworkingSockets\include" ^
    main.cpp ^
    /link ^
    /LIBPATH:"C:\GameNetworkingSockets\lib" ^
    GameNetworkingSockets.lib ^
    libssl.lib libcrypto.lib libprotobuf.lib ^
    ws2_32.lib crypt32.lib
```

### Windows (MinGW/MSYS2)

#### Install Dependencies

```bash
# Open MSYS2 MinGW 64-bit terminal
pacman -S \
    mingw-w64-x86_64-gcc \
    mingw-w64-x86_64-cmake \
    mingw-w64-x86_64-ninja \
    mingw-w64-x86_64-openssl \
    mingw-w64-x86_64-protobuf
```

#### Build Static Library

```bash
git clone https://github.com/ValveSoftware/GameNetworkingSockets.git
cd GameNetworkingSockets

mkdir build && cd build
cmake -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_STATIC_LIB=ON \
    -DBUILD_SHARED_LIB=OFF \
    ..
ninja

# Install
cmake --install . --prefix /mingw64
```

#### Use in Your Project

```bash
g++ -std=c++11 \
    main.cpp \
    -lGameNetworkingSockets \
    -lssl -lcrypto -lprotobuf \
    -lws2_32 -lcrypt32 \
    -o mygame.exe
```

## Linking Requirements

### Required System Libraries

Different platforms require different system libraries:

**Linux:**
- `pthread` - POSIX threads
- `m` - Math library (usually linked automatically)
- `dl` - Dynamic loading (for OpenSSL)

**macOS:**
- System frameworks are usually linked automatically
- May need `-framework CoreFoundation -framework Security`

**Windows:**
- `ws2_32.lib` - Winsock 2
- `crypt32.lib` - Windows cryptography API
- `iphlpapi.lib` - IP helper API (sometimes needed)

### Crypto Library Dependencies

Depending on which crypto backend you built with:

**OpenSSL:**
- `libssl` or `ssl`
- `libcrypto` or `crypto`

**libsodium:**
- `libsodium` or `sodium`

**BCrypt (Windows only):**
- `bcrypt.lib`

### Protobuf

- `libprotobuf` or `protobuf`

### WebRTC (if built with P2P support)

If you built with `-DUSE_STEAMWEBRTC=ON`:
- Additional WebRTC libraries will be needed
- These are typically bundled with the main library as static libs

## CMake Integration

### Option 1: Install and Use find_package

After building and installing GameNetworkingSockets:

```cmake
cmake_minimum_required(VERSION 3.10)
project(MyGame)

# Find the installed library
find_package(GameNetworkingSockets REQUIRED)

add_executable(mygame src/main.cpp)

# This automatically handles all dependencies
target_link_libraries(mygame PRIVATE GameNetworkingSockets::GameNetworkingSockets)
```

### Option 2: Add as Subdirectory

Include GameNetworkingSockets directly in your project:

```cmake
cmake_minimum_required(VERSION 3.10)
project(MyGame)

# Add GameNetworkingSockets as subdirectory
add_subdirectory(external/GameNetworkingSockets)

add_executable(mygame src/main.cpp)

# Link against the static library
target_link_libraries(mygame PRIVATE GameNetworkingSockets::static)
```

Project structure:
```
MyGame/
├── CMakeLists.txt
├── src/
│   └── main.cpp
└── external/
    └── GameNetworkingSockets/  (git submodule or copy)
```

### Option 3: FetchContent (CMake 3.11+)

Download and build automatically:

```cmake
cmake_minimum_required(VERSION 3.11)
project(MyGame)

include(FetchContent)

FetchContent_Declare(
    GameNetworkingSockets
    GIT_REPOSITORY https://github.com/ValveSoftware/GameNetworkingSockets.git
    GIT_TAG        master  # Or specific version tag
)

# Configure before fetching
set(BUILD_STATIC_LIB ON CACHE BOOL "")
set(BUILD_SHARED_LIB OFF CACHE BOOL "")
set(BUILD_EXAMPLES OFF CACHE BOOL "")
set(BUILD_TESTS OFF CACHE BOOL "")

FetchContent_MakeAvailable(GameNetworkingSockets)

add_executable(mygame src/main.cpp)
target_link_libraries(mygame PRIVATE GameNetworkingSockets::static)
```

### Advanced CMake: Handling Dependencies

If you want more control over dependencies:

```cmake
# Find dependencies yourself
find_package(OpenSSL REQUIRED)
find_package(Protobuf REQUIRED)

# Link GameNetworkingSockets and its dependencies
target_link_libraries(mygame PRIVATE
    GameNetworkingSockets::static
    OpenSSL::SSL
    OpenSSL::Crypto
    protobuf::libprotobuf
)

# Platform-specific libraries
if(WIN32)
    target_link_libraries(mygame PRIVATE ws2_32 crypt32)
elseif(UNIX AND NOT APPLE)
    target_link_libraries(mygame PRIVATE pthread dl)
endif()
```

## Manual Integration

### Step-by-Step Manual Integration

If not using CMake, follow these steps:

#### 1. Build the Static Library

Follow platform-specific build instructions above to create:
- Linux: `libGameNetworkingSockets.a`
- macOS: `libGameNetworkingSockets.a`
- Windows: `GameNetworkingSockets.lib`

#### 2. Copy Files to Your Project

```
YourProject/
├── lib/
│   ├── libGameNetworkingSockets.a  (or .lib on Windows)
│   ├── libssl.a                    (crypto dependency)
│   ├── libcrypto.a
│   └── libprotobuf.a
└── include/
    └── steam/
        ├── isteamnetworkingsockets.h
        ├── isteamnetworkingmessages.h
        ├── isteamnetworkingutils.h
        ├── steamnetworkingtypes.h
        └── ...
```

#### 3. Update Build System

**Makefile example:**

```makefile
INCLUDES = -Iinclude
LIBDIRS = -Llib
LIBS = -lGameNetworkingSockets -lssl -lcrypto -lprotobuf -lpthread

mygame: main.o network.o
	$(CXX) $(LIBDIRS) -o $@ $^ $(LIBS)

%.o: %.cpp
	$(CXX) $(INCLUDES) -c $< -o $@
```

**Visual Studio Project Properties (.vcxproj):**

Can be edited manually or through the IDE:

```xml
<ItemDefinitionGroup>
  <ClCompile>
    <AdditionalIncludeDirectories>$(ProjectDir)include;%(AdditionalIncludeDirectories)</AdditionalIncludeDirectories>
  </ClCompile>
  <Link>
    <AdditionalLibraryDirectories>$(ProjectDir)lib;%(AdditionalLibraryDirectories)</AdditionalLibraryDirectories>
    <AdditionalDependencies>GameNetworkingSockets.lib;libssl.lib;libcrypto.lib;libprotobuf.lib;ws2_32.lib;crypt32.lib;%(AdditionalDependencies)</AdditionalDependencies>
  </Link>
</ItemDefinitionGroup>
```

#### 4. Verify Linking

Compile a minimal test program:

```cpp
#include <steam/steamnetworkingsockets.h>
#include <stdio.h>

int main() {
    SteamDatagramErrMsg errMsg;
    if (GameNetworkingSockets_Init(nullptr, errMsg)) {
        printf("GameNetworkingSockets initialized successfully!\n");
        GameNetworkingSockets_Kill();
        return 0;
    }
    printf("Failed to initialize: %s\n", errMsg);
    return 1;
}
```

If it compiles and runs, integration is successful!

## Troubleshooting

### Build Errors

#### "Cannot find -lssl" or "Cannot find openssl/ssl.h"

**Solution**: Install OpenSSL development files
```bash
# Ubuntu/Debian
sudo apt install libssl-dev

# macOS
brew install openssl
export PKG_CONFIG_PATH="/usr/local/opt/openssl@1.1/lib/pkgconfig"

# Windows
# Use vcpkg: .\vcpkg\vcpkg install openssl:x64-windows
```

#### "Cannot find -lprotobuf" or protobuf headers

**Solution**: Install protobuf
```bash
# Ubuntu/Debian
sudo apt install libprotobuf-dev protobuf-compiler

# macOS
brew install protobuf

# Windows
# Use vcpkg: .\vcpkg\vcpkg install protobuf:x64-windows
```

#### CMake Error: "Could not find WebRTC"

**Solution**: Either disable P2P or initialize submodules
```bash
# Disable P2P (easier)
cmake -DUSE_STEAMWEBRTC=OFF ..

# Or enable WebRTC submodule
git submodule update --init --recursive
cmake -DUSE_STEAMWEBRTC=ON ..
```

### Linking Errors

#### Undefined reference to pthread_create

**Solution**: Add pthread library
```bash
g++ ... -lGameNetworkingSockets -lpthread
```

#### Unresolved external symbol (Windows)

**Solution**: Check runtime library settings
- Ensure `/MT` (static runtime) or `/MD` (dynamic runtime) matches between your app and GameNetworkingSockets
- Rebuild GameNetworkingSockets with matching setting if needed:
  ```
  cmake -DMSVC_CRT_STATIC=ON ..  # For /MT
  ```

#### Multiple definition errors

**Solution**: This happens when linking both static and shared versions
- Ensure you're only linking one version
- Check that `BUILD_SHARED_LIB=OFF` when building

#### "Undefined reference to SSL_*" or "Unresolved external symbol SSL_*"

**Solution**: Link OpenSSL libraries
```bash
# Linux/Mac
-lssl -lcrypto

# Windows
libssl.lib libcrypto.lib
```

### Runtime Errors

#### "error while loading shared libraries: libssl.so.1.1"

**Solution**: Static library still depends on shared crypto libraries
- Install OpenSSL runtime: `sudo apt install libssl1.1`
- Or build with statically linked OpenSSL (complex, see vcpkg static triplets)

#### Crash on GameNetworkingSockets_Init()

**Possible causes**:
1. **Missing crypto library**: Ensure OpenSSL/libsodium is installed
2. **ABI mismatch**: Rebuild library with same compiler/settings as your app
3. **Memory corruption**: Check that you're not mixing debug/release builds

### Platform-Specific Issues

#### Linux: "version GLIBCXX_3.4.X not found"

**Solution**: Compiler mismatch - rebuild with same GCC version as your project

#### macOS: "dyld: Library not loaded: libssl.1.1.dylib"

**Solution**: Add OpenSSL to library path
```bash
export DYLD_LIBRARY_PATH="/usr/local/opt/openssl@1.1/lib:$DYLD_LIBRARY_PATH"
```

#### Windows: "MSVCP140.dll not found"

**Solution**: Install Visual C++ Redistributable or use static runtime (`/MT`)

## Best Practices

### 1. Version Control Integration

Track the specific version you're using:

```bash
# As git submodule
git submodule add https://github.com/ValveSoftware/GameNetworkingSockets.git external/GNS
git submodule update --init

# Or record the commit hash
echo "Using GameNetworkingSockets commit: abc1234" > DEPENDENCIES.txt
```

### 2. Reproducible Builds

Lock dependency versions:

```cmake
# In CMakeLists.txt
FetchContent_Declare(
    GameNetworkingSockets
    GIT_REPOSITORY https://github.com/ValveSoftware/GameNetworkingSockets.git
    GIT_TAG        v1.4.1  # Specific version, not 'master'
)
```

### 3. Cross-Platform Builds

Use CMake for easier cross-platform support:

```cmake
# Detect platform and link appropriate libraries
if(WIN32)
    target_link_libraries(mygame PRIVATE ws2_32 crypt32)
elseif(APPLE)
    target_link_libraries(mygame PRIVATE "-framework CoreFoundation")
else()
    target_link_libraries(mygame PRIVATE pthread dl)
endif()
```

### 4. Separate Debug and Release Builds

```bash
# Debug build
cmake -DCMAKE_BUILD_TYPE=Debug -B build-debug
cmake --build build-debug

# Release build
cmake -DCMAKE_BUILD_TYPE=Release -B build-release
cmake --build build-release
```

Keep separate library builds or use CMake's multi-config generators.

### 5. Minimal Dependencies

If you don't need certain features, disable them:

```bash
cmake \
    -DUSE_STEAMWEBRTC=OFF \  # Disable P2P (removes WebRTC dependency)
    -DBUILD_EXAMPLES=OFF \   # Don't build examples
    -DBUILD_TESTS=OFF \      # Don't build tests
    ..
```

## Additional Resources

- **Building Guide**: [BUILDING.md](BUILDING.md) - Complete build instructions
- **Integration Guide**: [INTEGRATION_GUIDE.md](INTEGRATION_GUIDE.md) - How to use the library in your game
- **Implementation Guide**: [IMPLEMENTATION_GUIDE.md](IMPLEMENTATION_GUIDE.md) - How the library works internally
- **Examples**: [examples/](examples/) - Working code samples
- **API Headers**: [include/steam/](include/steam/) - Full API documentation in headers

## Getting Help

If you encounter issues:

1. Check this troubleshooting section
2. Review the build logs for specific error messages
3. Search existing GitHub issues
4. Create a new issue with:
   - Platform and compiler version
   - CMake command used
   - Full error output
   - Steps to reproduce

Remember: Static library linking requires all dependencies to be available at link time. When in doubt, ensure all required libraries are installed and paths are correctly specified.
