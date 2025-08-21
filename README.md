# Cesium for Unity(v1.15.3) – Developer Setup (Ubuntu 22.04, Unity 2022.3)

This guide documents how to clone and configure **Cesium for Unity(v1.15.3)** so that it builds correctly on **Ubuntu 22.04** with Unity **2022.3 LTS**. Some extra adjustments are needed for **Reinterop** and the CMake build files in order to compile the editor and runtime native libraries on Linux.

---

## System Environment

- **OS:** Ubuntu 22.04.5 LTS  
  - Architecture: 64-bit  
  - Windowing System: X11  
  - GNOME Version: 42.9
- **Unity Editor:** 2022.3.55f1
- **Unity Hub:** 3.11.0
- **.NET SDKs Installed:** .NET 6.0 / .NET 8.0 (with the `dotnet` CLI)

---

## Original Documentation

The official starting point is the Cesium for Unity [developer-setup.md](https://github.com/CesiumGS/cesium-unity/blob/main/Documentation~/developer-setup.md), but we’ll override some steps for Linux compatibility and Reinterop adjustments.

---

## 1. Clone the Repository

We typically place this in a `Packages/` folder if you’re integrating into a Unity project.  
For example:

```bash
cd /path/to/YourUnityProject/Packages
git clone --recurse-submodules -b v1.15.3 git@github.com:CesiumGS/cesium-unity.git com.cesium.unity
```

This clones the **cesium-unity** repo into a folder named `com.cesium.unity`, including its submodules.

---

## 2. Adjust `Reinterop.csproj` in `Reinterop~`

By default, `Reinterop.csproj` might be targeting something newer than `netstandard2.0` or have `LangVersion=Preview`, which Unity’s Roslyn won’t accept on Linux. Replace it.

```bash
cd com.cesium.unity
gedit Reinterop~/Reinterop.csproj
```

**`Replace by copy paste`:**

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <LangVersion>latest</LangVersion>
    <IsRoslynComponent>true</IsRoslynComponent>
    <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
    <CompilerGeneratedFilesOutputPath>Generated</CompilerGeneratedFilesOutputPath>
  </PropertyGroup>

  <ItemGroup>
    <Compile Remove="RoslynIncrementalGenerator.cs" />
  </ItemGroup>
  
  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="3.8.0" PrivateAssets="all" />
  </ItemGroup>

  <!-- This ensures the library will be packaged as a source generator when we use `dotnet pack` -->
  <ItemGroup>
    <None Include="$(OutputPath)\$(AssemblyName).dll" Pack="true" PackagePath="analyzers/dotnet/cs" Visible="false" />
  </ItemGroup>

</Project>
```

---

## 3. Change needed for allowing ceisum to be built for runtime unity build

open com.cesium.unity/Runtime/CesiumRuntime.asmdef

and add "LinuxStandalone64" in includePlatforms
like 

```json
  {
      "name": "CesiumRuntime",
      "rootNamespace": "",
      "references": [
          "Unity.InputSystem",
          "Unity.Mathematics",
          "Unity.Splines"
      ],
      "includePlatforms": [
          "Android",
          "Editor",
          "iOS",
          "macOSStandalone",
          "WSA",
          "WindowsStandalone64",
          "LinuxStandalone64"
      ],
      "excludePlatforms": [],
      "allowUnsafeCode": true,
      "overrideReferences": false,
      "precompiledReferences": [],
      "autoReferenced": true,
      "defineConstraints": [],
      "versionDefines": [
          {
              "name": "com.unity.splines",
              "expression": "1.0.0",
              "define": "SUPPORTS_SPLINES"
          }
      ],
      "noEngineReferences": false
  }
```


---

## 4. Publish Reinterop

From **inside** `com.cesium.unity`, do:

```bash
# Build/publish Reinterop to ensure we have the Reinterop.dll we want.
pwd # you should be in com.cesium.unity folder
dotnet publish Reinterop~ -o .
git restore Reinterop.dll.meta
```

This generates a `Reinterop.dll` in the `com.cesium.unity/` folder (copied from `Reinterop~/bin/Release/netstandard2.0/Reinterop.dll`).

---

## 5. Open the project in unity editor to compile and create DotNet folder.

**Go to Unity Hub and open the project!**.
- Unity might show **DllNotFoundException** for the native plugin until you build it, which is expected.

After it finishes compiling you can close it. Now back to the console, you should be able to see some DotNet folders generated if you enter in the console:

```bash
pwd # you should be in com.cesium.unity folder
find . -iname "*dotnet*"
```

you should get something like:
```
./native~/Runtime/generated-Editor/src/DotNet
./native~/Runtime/generated-Editor/include/DotNet
./native~/Editor/generated-Editor/src/DotNet
./native~/Editor/generated-Editor/include/DotNet
```

If you do see DotNet folders, you can move on to the next step, to build the cpp side of the cesium package.

---

## 6. fix Cesium3DTilesSelection/src/TilesetJsonLoader.cpp

https://github.com/CesiumGS/cesium-unity/issues/513#issuecomment-2655169841

```bash
pwd # you should be in com.cesium.unity folder
gedit native~/extern/cesium-native/Cesium3DTilesSelection/src/TilesetJsonLoader.cpp
```
**`Copy paste the following file's content`:**

[`TilesetJsonLoader.cpp`](./TilesetJsonLoader.cpp)

---

## 7. create custom x64-linux-unity.cmake
https://github.com/CesiumGS/cesium-unity/issues/513#issuecomment-2655169841

```bash
pwd # you should be in com.cesium.unity folder
gedit native~/vcpkg/triplets/x64-linux-unity.cmake
```

**`Copy paste`:**

```cmake
include("${CMAKE_CURRENT_LIST_DIR}/shared/common.cmake")
# Custom triplet for x64-linux with Unity-specific settings.
set(VCPKG_TARGET_ARCHITECTURE x64)
set(VCPKG_CRT_LINKAGE static)
set(VCPKG_LIBRARY_LINKAGE static)
set(VCPKG_CMAKE_SYSTEM_NAME Linux)
set(VCPKG_LIBRARY_PREFIX "")
set(VCPKG_CMAKE_CONFIGURE_OPTIONS "-DCMAKE_POLICY_VERSION_MINIMUM=3.5")
```

---

## 8. Build using x64-linux-unity.cmake you created

1. **Build the Native Editor Library**  
    - Close Unity.
    - Remove `-Werror` in native~/extern/cesium-native/cmake/marcros/configure_cesium_library.cmake and native~/extern/cesium-native/cmake/compiler.cmake
    - In a terminal:

    ```bash
    pwd # you should be in com.cesium.unity/ folder
    cd native~
    cmake -B build-Standalone -S . -DCMAKE_BUILD_TYPE=RelWithDebInfo -DVCPKG_TRIPLET=x64-linux-unity -DVCPKG_OVERLAY_TRIPLETS=$(pwd)/vcpkg/triplets -DEDITOR=OFF
    cmake --build build-Standalone --target install --parallel $(nproc)

    ```

    - This produces `CesiumForUnityNative-Editor.so` (on Linux) and installs it to `../Editor` by default.

2. **Restart Unity**  
    - After installing the Editor library, reopen Unity.  
    - Unity should now load `CesiumForUnityNative-Editor.so` without throwing `DllNotFoundException`.

---

## 9. Done!

With these changes:

- **Reinterop** is recognized as a netstandard2.0 analyzer.  
- The Editor code is compiled separately from runtime code to avoid partial-definition conflicts.  
- Unity Editor can load `CesiumForUnityNative-Editor.so`.  
- The runtime plugin is built only if needed, using `-DEDITOR=OFF`.

Now you can open scenes from Cesium for Unity, test your Ion integration, and avoid all the “No such file or directory: DotNet/…” or `DllNotFoundException` issues.
