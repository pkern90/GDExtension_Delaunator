#!/usr/bin/env python
import os
import sys

# default values, adapt them to your setup
default_library_name = "libgddelaunator"
default_target_path = "demo/addons/delaunator_gdextension/bin/"

# Local dependency paths, adapt them to your setup
cpp_bindings_path = "godot-cpp/"
godot_headers_path = cpp_bindings_path + "godot-headers/"
cpp_library = "libgodot-cpp"

# Try to detect the host platform automatically.
# This is used if no `platform` argument is passed
if sys.platform.startswith("linux"):
    host_platform = "linux"
elif sys.platform.startswith("freebsd"):
    host_platform = "freebsd"
elif sys.platform == "darwin":
    host_platform = "osx"
elif sys.platform == "win32" or sys.platform == "msys":
    host_platform = "windows"
else:
    raise ValueError("Could not detect platform automatically, please specify with platform=<platform>")

env = Environment(ENV=os.environ)

opts = Variables([], ARGUMENTS)

# Define our options
opts.Add(EnumVariable("target", "Compilation target", "template_debug", allowed_values=("template_debug", "template_release"), ignorecase=2))
opts.Add(
    EnumVariable(
        "platform",
        "Target platform",
        host_platform,
        # We'll need to support these in due times
        # allowed_values=("linux", "freebsd", "osx", "windows", "android", "ios", "javascript"),
        allowed_values=("linux", "windows", "osx"),
        ignorecase=2,
    )
)
opts.Add(EnumVariable("bits", "Target platform bits", "64", ("32", "64")))
opts.Add(BoolVariable("use_llvm", "Use the LLVM / Clang compiler", "no"))
opts.Add(EnumVariable("macos_arch", "Target macOS architecture", "universal", ["universal", "x86_64", "arm64"]))
opts.Add(PathVariable("target_path", "The path where the lib is installed.", default_target_path, PathVariable.PathAccept))
opts.Add(PathVariable("target_name", "The library name.", default_library_name, PathVariable.PathAccept))

# only support 64 at this time..
bits = 64

# Updates the environment with the option variables.
opts.Update(env)
# Generates help for the -h scons option.
Help(opts.GenerateHelpText(env))

# This makes sure to keep the session environment variables on Windows.
# This way, you can run SCons in a Visual Studio 2017 prompt and it will find
# all the required tools
if host_platform == "windows" and env["platform"] != "android":
    if env["bits"] == "64":
        env = Environment(TARGET_ARCH="amd64")
    elif env["bits"] == "32":
        env = Environment(TARGET_ARCH="x86")

    opts.Update(env)

# Require C++17
if host_platform == "windows" and env["platform"] == "windows" and not env["use_mingw"]:
    # MSVC
    env.Append(CXXFLAGS=["/std:c++17"])
else:
    env.Append(CXXFLAGS=["-std=c++17"])
    

# Process some arguments
if env["use_llvm"]:
    env["CC"] = "clang"
    env["CXX"] = "clang++"

if env["platform"] == "":
    print("No valid target platform selected.")
    quit()

# For the reference:
# - CCFLAGS are compilation flags shared between C and C++
# - CFLAGS are for C-specific compilation flags
# - CXXFLAGS are for C++-specific compilation flags
# - CPPFLAGS are for pre-processor flags
# - CPPDEFINES are for pre-processor defines
# - LINKFLAGS are for linking flags

if env["target"] == "template_debug":
    env.Append(CPPDEFINES=["DEBUG_ENABLED", "DEBUG_METHODS_ENABLED"])

# Check our platform specifics
if env["platform"] == "osx":
    env["target_path"] += "osx/"
    cpp_library += ".osx"

    if env["bits"] == "32":
        raise ValueError("Only 64-bit builds are supported for the macOS target.")

    if env["macos_arch"] == "universal":
        env.Append(LINKFLAGS=["-arch", "x86_64", "-arch", "arm64"])
        env.Append(CCFLAGS=["-arch", "x86_64", "-arch", "arm64"])
    else:
        env.Append(LINKFLAGS=["-arch", env["macos_arch"]])
        env.Append(CCFLAGS=["-arch", env["macos_arch"]])

    env.Append(CXXFLAGS=["-std=c++17"])
    if env["target"] == "template_debug":
        env.Append(CCFLAGS=["-g", "-O2"])
    else:
        env.Append(CCFLAGS=["-g", "-O3"])

    arch_suffix = env["macos_arch"]

elif env["platform"] in ("x11", "linux"):
    cpp_library += ".linux"
    env.Append(CCFLAGS=["-fPIC"])
    if env["target"] == "template_debug":
        env.Append(CCFLAGS=["-g3", "-Og"])
    else:
        env.Append(CCFLAGS=["-g", "-O3"])

    arch_suffix = str(bits)
elif env["platform"] == "windows":
    cpp_library += ".windows"

    if host_platform == "windows" and not env["use_mingw"]:
        # MSVC
        # This makes sure to keep the session environment variables on windows,
        # that way you can run scons in a vs 2017 prompt and it will find all the required tools
        env.Append(ENV=os.environ)

        env.Append(CPPDEFINES=["WIN32", "_WIN32", "_WINDOWS", "_CRT_SECURE_NO_WARNINGS"])
        env.Append(CCFLAGS=["-W3", "-GR"])
        env.Append(CXXFLAGS=["-std:c++17"])
        if env["target"] == "template_debug":
            env.Append(CPPDEFINES=["_DEBUG"])
            env.Append(CCFLAGS=["-EHsc", "-MDd", "-ZI", "-FS"])
            env.Append(LINKFLAGS=["-DEBUG"])
        else:
            env.Append(CPPDEFINES=["NDEBUG"])
            env.Append(CCFLAGS=["-O2", "-EHsc", "-MD"])

        if not(env["use_llvm"]):
            env.Append(CPPDEFINES=["TYPED_METHOD_BIND"])
	    
    elif host_platform == "linux" or host_platform == "freebsd" or host_platform == "osx":
        # Cross-compilation using MinGW
        if env["bits"] == "64":
            env["CXX"] = "x86_64-w64-mingw32-g++"
            env["AR"] = "x86_64-w64-mingw32-ar"
            env["RANLIB"] = "x86_64-w64-mingw32-ranlib"
            env["LINK"] = "x86_64-w64-mingw32-g++"
        elif env["bits"] == "32":
            env["CXX"] = "i686-w64-mingw32-g++"
            env["AR"] = "i686-w64-mingw32-ar"
            env["RANLIB"] = "i686-w64-mingw32-ranlib"
            env["LINK"] = "i686-w64-mingw32-g++"
        # DLLs seem to be given a .so file extension even though they are DLLs unless
        # you specify the extension
        env["SHLIBSUFFIX"] = ".dll"


    elif host_platform == "windows" and env["use_mingw"]:
        # Don't Clone the environment. Because otherwise, SCons will pick up msvc stuff.
        env = Environment(ENV=os.environ, tools=["mingw"])
        opts.Update(env)

        # Still need to use C++17.
        env.Append(CCFLAGS=["-std=c++17"])
        # Don't want lib prefixes
        env["IMPLIBPREFIX"] = ""
        env["SHLIBPREFIX"] = ""

        # Long line hack. Use custom spawn, quick AR append (to avoid files with the same names to override each other).
        env["SPAWN"] = mySpawn
        env.Replace(ARFLAGS=["q"])

    # Native or cross-compilation using MinGW
    if host_platform == "linux" or host_platform == "freebsd" or host_platform == "osx" or env["use_mingw"]:
        # These options are for a release build even using target=debug
        env.Append(CCFLAGS=["-O3", "-Wwrite-strings"])
        env.Append(
            LINKFLAGS=[
                "--static",
                "-Wl,--no-undefined",
                "-static-libgcc",
                "-static-libstdc++",
            ]
        )

    arch_suffix = str(bits)

# suffix our godot-cpp library
cpp_library += "." + env["target"] + "." + arch_suffix

# make sure our binding library is properly includes
env.Append(CPPPATH=[".", godot_headers_path, cpp_bindings_path + "include/", cpp_bindings_path + "gen/include/"])
env.Append(LIBPATH=[cpp_bindings_path + "bin/"])
env.Append(LIBS=[cpp_library])

# tweak this if you want to use different folders, or more folders, to store your source code in.
env.Append(CPPPATH=["src/"])
sources = Glob("src/*.cpp")

target_name = "{}.{}.{}.{}".format(env["target_name"], env["platform"], env["target"], arch_suffix)
print(target_name)
library = env.SharedLibrary(target=env["target_path"] + target_name, source=sources)

Default(library)
