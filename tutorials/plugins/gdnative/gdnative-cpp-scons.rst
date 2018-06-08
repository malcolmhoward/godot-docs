.. _doc_gdnative_cpp_scons:

GDNative C++ Scons Overview
===========================

Introduction
------------
This tutorial builds on top of the information given in the :ref:`GDNative C++ example <doc_gdnative_cpp_example>` so we highly recommend you read that first.

The C++ compilation workflow from the basic C++ example is targeted specifically at working for that example alone. It has many assumptions and hardcoded values that prevent you from re-using the same build file for other projects.

This tutorial will instead cover recommendations for how to use scons and its various options to build GDNative C++ bindings and libraries.

By the end of this tutorial, we'll have made a fully customizable ``SConstruct`` build file that functions well for the most common use cases.

Note that in the previous tutorial, you first built a static library of the bindings by using scons within the ``godot-cpp`` directory. This tutorial will guide you through the process of building and using the *other* ``SConstruct`` file which you downloaded and used to build your dynamic library code (though ours won't be hardcoded). It will therefore assume you already have the ``godot_headers`` and ``godot-cpp`` directories located somewhere.

If you don't really care to get a breakdown of how it works and just **want the SConstruct file now, then click here**: :download:`SConstruct <files/scons-tutorial/SConstruct>`

How To Build Shared Libraries
-----------------------------

The most important part of your scons file will be the following lines.

.. code-block:: python

    library = env.SharedLibrary(target="", source=[])
    Default(library)

Here, we are declaring that scons should build a dynamic library. We tell it the full filepath that should be given to this library (the "target" filename) and the filepaths to all of the *.cpp files that should be included in it (the "source" array).

.. code-block:: python

    # example usage
    library = env.SharedLibrary(target="./libtest.dll", source=["./test.cpp", "./folder/test2.cpp"])
    Default(library)

Here, as an example, you can see that we are building "libtest.dll" at the current directory by including our two test C++ files in various directories.

Our next step is to setup an environment which will supply the context of our build operations.

.. code-block:: python

    env = Environment()

Not so fast though! When Windows compilation is done using Visual Studio, it will require environment variables that Windows has already setup to access Visual Studio's compiler ``cl``. As such, we can ensure that scons finds the appropriate compiler for all operating systems by by cloning the existing environment in the Windows case.

To do this, we'll need to import the ``os`` Python module (to get the OS's environment). We'll also need to determine when the user is compiling on Windows versus other operating systems.

.. code-block:: python
    import os

    platform = ARGUMENTS.get("platform", "windows")

    env = Environment()
    if platform == "windows":
        env = Environment(ENV = os.environ)

The ``ARGUMENTS`` global is an object that allows us to ``get`` a named parameter from the command line as well as supply a default value if the parameter is not given. In this case, we are asking the user to provide us with the ``platform`` parameter. If they do not provide one, we will assume it is "windows" since that is the most common case.

If we are compiling for Windows, then we set our ``env`` to be a new ``Environment`` in which its internal environment is a copy of the operating system's environment variables.

OS-Specific Information
-----------------------

We'll probably want to give our resulting library a unique name depending on what platform and bit version it's aimed at (32-bit vs. 64-bit). With that in mind, let's also collect the desired bits version.

.. code-block:: python

    platform = ARGUMENTS.get("platform", "windows")
    bits = ARGUMENTS.get("bits", 64)
    lib_name = ARGUMENTS.get("name", "libdefault")

    library_file = lib_name + "." + platform + "." + str(bits)

We'll also want to allow the user to decide where to output their various dynamic libraries (one for each platform/bit), so let's also collect the library path.

If we combine the library filename with the platform/bits-specific directory path, then we can determine the full appropriate filename to use for the dynamic library's output location.

.. code-block:: python

    platform = ARGUMENTS.get("platform", "windows")
    bits = ARGUMENTS.get("bits", 64)
    lib_name = ARGUMENTS.get("name", "libdefault")
    lib_path = ARGUMENTS.get("lib", "bin/")

    # Force expected formatting.
    # Ensures we view Windows-style and filepath-like 
    # directory paths as typical UNIX directory paths.
    lib_path = lib_path.split("\\").join("/")
    if lib_path[-1:] != "/":
        lib_path += "/"

    platform_dir = ""

    if platform == "osx":
        platform_dir = "osx" + str(bits) + "/"
    elif platform == "linux":
        platform_dir = "x11" + str(bits) + "/"
    elif platform == "windows":
        platform_dir = "win" + str(bits) + "/"

    library_file = lib_name + "." + platform + "." + str(bits)
    final_lib_path = lib_path + platform_dir + library_file

Each operating system will also have different flags to pass to the C++ compiler and linker we use.

All of the available options for each platform are really beyond the scope of this tutorial, but you can find more information by examining the compiler options for each individual compiler. In this example, the compiler options specified are related to ``g++`` for OSX/Linux and ``cl`` for Windows.

An overview of the compiler and linker options for ``g++`` can be found here: https://developers.redhat.com/blog/2018/03/21/compiler-and-linker-flags-gcc/

An overview of the compiler options for ``cl`` can be found here: https://msdn.microsoft.com/en-us/library/19z1t1wy.aspx

.. code-block:: python

    target = ARGUMENTS.get("target", "debug")
    platform = ARGUMENTS.get("platform", "windows")
    
    if platform == "osx":
        env.Append(CCFLAGS = ['-g','-O3', '-arch', 'x86_64'])
        env.Append(LINKFLAGS = ['-arch', 'x86_64'])
    
    elif platform == "linux":
        env.Append(CCFLAGS = ['-fPIC', '-g','-O3', '-std=c++14'])
    
    elif platform == "windows":
        if target == "debug":
            env.Append(CCFLAGS = ['-EHsc', '-D_DEBUG', '-MDd'])
        else:
            env.Append(CCFLAGS = ['-O2', '-EHsc', '-DNDEBUG', '-MD'])

Notice that we define the compilation and linking flags by assigning the command line options as an array of strings to special property names. These names (like ``CCFLAGS``) are specifically used by scons to build whatever we want from the ``Environment`` we've setup in the script.

Phew! Okay, so that's the name of our library, where the outputted libraries will end up, and the flags for each OS.

Let's put it all together now, and see where we're at!

.. code-block:: python
    #!python

    platform = ARGUMENTS.get("platform", "windows")
    bits = ARGUMENTS.get("bits", 64)
    lib_name = ARGUMENTS.get("name", "libdefault")
    lib_path = ARGUMENTS.get("lib", "bin/")

    # Force expected formatting.
    # Ensures we view Windows-style and filepath-like 
    # directory paths as typical UNIX directory paths.
    lib_path = lib_path.split("\\").join("/")
    if lib_path[-1:] != "/":
        lib_path += "/"

    platform_dir = ""

    if platform == "osx":
        env.Append(CCFLAGS = ['-g','-O3', '-arch', 'x86_64'])
        env.Append(LINKFLAGS = ['-arch', 'x86_64'])

        platform_dir = "osx" + str(bits) + "/"

    elif platform == "linux":
        env.Append(CCFLAGS = ['-fPIC', '-g','-O3', '-std=c++14'])

        platform_dir = "x11" + str(bits) + "/"

    elif platform == "windows":
        if target == "debug":
            env.Append(CCFLAGS = ['-EHsc', '-D_DEBUG', '-MDd'])
        else:
            env.Append(CCFLAGS = ['-O2', '-EHsc', '-DNDEBUG', '-MD'])

        platform_dir = "win" + str(bits) + "/"

    library_file = lib_name + "." + platform + "." + str(bits)
    final_lib_path = lib_path + platform_dir + library_file

Finding Libraries
-----------------

So, let's quickly review the things that we needed and where we are at.

1. The ``env.SharedLibrary`` method which needs...

    1. The full filename of the resulting dynamic library.

        - We are building this from the lib_path ("lib") argument and appending to it...
        - A combination of the lib_name ("name"), bits ("bits"), target ("target"), and platform ("platform").
    
    2. The full filenames of all **source** files to include in the result.

        - **We still need these!**

2. An ``Environment`` that contains the context of our build operations.

    1. The compilation and linker flags we intend to use.

        -   We are using our ``platform`` and ``target`` parameters to assign the proper ``CCFLAGS`` and ``LINKFLAGS`` environment variables.

    2. The full filenames of all **library** files to include in the result.

        - **We still need these!**

    3. The directories in which to search for all related C++ **source** and **header** files referenced by included *.cpp files.

        - **We still need these!**

Now, to add the library information to our environment information, we will use the following line:

.. code-block:: python

    env.Append(LIBS=[])

As you may have guessed, this method adds to our ``Environment`` the corresponding "full filenames" for the library files (*.lib/*.dll, *.a/*.so, *.a/*.dylib). Library files include both static and dynamic libraries, for each platform (windows/linux/mac), which is the reason why there are so many different file extensions.

The C++ bindings library you created at the start of the prevoius tutorial was a static library (.lib for Windows, .a for Linux/Mac). We will now need to include it when building our dynamic library.

The name of the file should be `godot-cpp.<platform>.<bits>`. That exact name is not strictly necessary, but that is the format of the library currently exported by the godot-cpp repository's ``SConstruct`` file, so we'll need to match its format if we intend to locate it properly. Because we know we will have to include the C++ bindings, we can create a dedicated parameter for the path to it.

.. code-block:: python

    cpp_bindings_library_path = ARGUMENTS.get("cpp_bindings_library", "godot-cpp/bin/godot-cpp")
    
    platform = ARGUMENTS.get("platform", "windows")
    bits = ARGUMENTS.get("bits", 64)

    cpp_bindings_library = cpp_bindings_library_path + "." + platform + "." + str(bits)

    env.Append(LIBS=[cpp_bindings_library])

This results in users passing the file path to their bindings library into the scons build file. There isn't much sense in creating a separate parameter for the name of the library ("godot-cpp" here), so we can just include the library name in the path.

However, we will have to grab a library that uses the same platform and bits compilation as we intend to produce in our dynamic library. As such, we will manually inject that part ourselves. After all, there's no sense including a Windows 32-bit static library when trying to build a MacOS 64-bit dynamic library, etc. To prepare for the possibility that our users will provide the full filepath though, rather than just up to the name of the library (or by mistake, supply the wrong extensions in their filepath!), we will strip away the extensions they provide and use only the actual name of the library.

.. code-block:: python
    import os

    cpp_bindings_library_path = ARGUMENTS.get("cpp_bindings_library", "godot-cpp/bin/godot-cpp")
    
    # repeatedly divide out extensions until the extension string is empty.
    ext = "z"
    while ext != "":
        # "godot-cpp.lib" would get split here into "godot-cpp" in the first value and ".lib" in the second
        cpp_bindings_library_path, ext = os.path.splitext(cpp_bindings_library_path)
    
    platform = ARGUMENTS.get("platform", "windows")
    bits = ARGUMENTS.get("bits", 64)

    # manually setup our own appropriate file extensions
    final_cpp_bindings_library_path = cpp_bindings_library_path + "." + platform + "." + str(bits)

Now, if our users were to use the parameter ``cpp_bindings_library=godot-cpp/bin/godot-cpp.64.windows``, we can properly convert it to the expected format of <name>.<platform>.<bits> instead where <name> is "godot-cpp" in this case.

That will give us the full filepath to our library. If we want to include other libraries, we would need to define another parameter. Perhaps a comma-separated list of filepaths?

.. code-block:: python

    # allows `scons other_libs="path/1/a.lib,path/2/b.lib"`
    other_libs = ARGUMENTS.get("other_libs", "")
    other_libs = other_libs.split(",")

    # merge the cpp_bindings_library path into the array of other libraries.
    libs = [cpp_bindings_library] + other_libs
    env.Append(LIBS=libs)

Finding Sources
---------------

For our source files, it would be tedious to ask users to provide explicit paths to every single file. Instead, we will write a function that can scan a directory and collect all *.cpp files that are present.

.. code-block:: python
    import os

    def add_sources(sources, directory):
        for file in os.listdir(directory):
            if file.endswith('.cpp'):
                sources.append(directory + '/' + file)
    
    sources = []
    # sample usage: add_sources(sources, "./src")
    # Now, every *.cpp file in the direct `src` folder is included (not the subfolders though).

If we combine this with our earlier method of providing a comma-separated list of directories to include in our search, we can make the scons file even more flexible.

.. code-block:: python
    import os

    sources = ARGUMENTS.get("sources", "")

    source_files = []
    for path in sources.split(","):
        if os.path.isdir(path):
            add_sources(source_files, path)

This way, we can provide a list of directories to search within for *.cpp files.

Finding Headers
---------------

Now that we have a means of collecting source files, the last step is to acquire our header files. For this, scons just needs us to supply the directories to search through. It works for both *.h and *.hpp files. These will be "the directories in which we search for all content referred to by the included *.cpp source files".

Right off the bat, we know we'll need to include the ``godot_headers`` directory as well as the generated contents of ``include`` and ``include/core`` from the ``godot-cpp`` directory. Because we do not know exactly where those directories may be located, we'll need the user to direct us to their location.

.. code-block:: python
    import os

    def add_sources(sources, directory):
        for file in os.listdir(directory):
            if file.endswith('.cpp'):
                sources.append(directory + '/' + file)

    sources = ARGUMENTS.get("sources", "")

    source_dirs = sources.split(",")

    source_files = []
    for path in source_dirs:
        if os.path.isdir(path):
            add_sources(source_files, path)
    
    # new content
    godot_headers_path = ARGUMENTS.get("headers", "godot_headers/")
    cpp_bindings_path = ARGUMENTS.get("cpp_bindings_path", "godot-cpp/")

    source_dirs.append(godot_headers_path)
    source_dirs.append(cpp_bindings_path + "include/")
    source_dirs.append(cpp_bindings_path + "include/core/")

    # At this point source_dirs has all source and header file directories we reference anywhere.
    # These directories will be passed to our environment so that scons can locate referenced files.

    env.Append(CPPPATH=source_dirs)

However, there is one piece missing here. Why did we collect the ``source_files`` array, but not ultimately do anything with it? Where does that go?

If you recall, while the environment needs the file locations of the libraries, and the directories of all of our source and header files, the actual source file locations are needed by the dynamic library itself, to know what to include in it. As such, we pass the ``source_files`` array directly to our ``SharedLibrary()`` method.

.. code-block:: python

    library = env.SharedLibrary(target=final_lib_path, source=source_files)
    Default(library)

Combining Sources, Headers, and Libraries, and Flags
----------------------------------------------------

Just to review where we are at, let's now pull everything together one more time before moving forward. We're almost at the end!

.. code-block:: python
    #!python
    import os

    # utility method definitions
    def add_sources(sources, directory):
        for file in os.listdir(directory):
            if file.endswith('.cpp'):
                sources.append(directory + '/' + file)

    # output lib preparation 
    target = ARGUMENTS.get("target", "debug")
    platform = ARGUMENTS.get("platform", "windows")
    bits = ARGUMENTS.get("bits", 64)
    lib_name = ARGUMENTS.get("name", "libdefault")

    library_file = lib_name + "." + platform + "." + str(bits)

    lib_path = ARGUMENTS.get("lib", "bin/")

    lib_path = lib_path.split("\\").join("/")
    if lib_path[-1:] != "/":
        lib_path += "/"

    # ------
    # While we've prepared our lib_path and library_file values now,
    # we won't create the final_lib_path value until after we've
    # done our OS-specific logic down below, since we need to know
    # the name of the <platform><bits> subdirectory.
    # ------

    # library dependencies
    
    cpp_bindings_library_path = ARGUMENTS.get("cpp_bindings_library", "godot-cpp/bin/godot-cpp")

    ext = "z"
    while ext != "":
        cpp_bindings_library_path, ext = os.path.splitext(cpp_bindings_library_path)

    cpp_bindings_library = cpp_bindings_library_path + "." + platform + "." + str(bits)

    other_libs = ARGUMENTS.get("other_libs", "")
    other_libs = other_libs.split(",")

    libs = [cpp_bindings_library] + other_libs

    # source dependencies

    godot_headers_path = ARGUMENTS.get("headers", "godot_headers/")
    cpp_bindings_path = ARGUMENTS.get("cpp_bindings_path", "godot-cpp/")
    sources = ARGUMENTS.get("sources", "")

    source_files = []

    source_dirs = sources.split(",")

    for path in source_dirs:
        if os.path.isdir(path):
            add_sources(source_files, path)

    source_dirs.append(godot_headers_path)
    source_dirs.append(cpp_bindings_path + "include/")
    source_dirs.append(cpp_bindings_path + "include/core/")

    # OS-specific logic for flags and the output lib directory

    platform_dir = ""

    if platform == "osx":
        env.Append(CCFLAGS = ['-g','-O3', '-arch', 'x86_64'])
        env.Append(LINKFLAGS = ['-arch', 'x86_64'])

        platform_dir = "osx"
    
    elif platform == "linux":
        env.Append(CCFLAGS = ['-fPIC', '-g','-O3', '-std=c++14'])

        platform_dir = "x11"
    
    elif platform == "windows":

        # Set exception handling model to avoid warnings caused by Windows system headers.
        env.Append(CCFLAGS=['-EHsc'])

        if target == "debug":
            env.Append(CCFLAGS = ['-D_DEBUG', '-MDd'])
        else:
            env.Append(CCFLAGS = ['-O2', '-DNDEBUG', '-MD'])

        platform_dir = "win"

    else:
        # do nothing if we don't recognize the platform
        print 'unrecognized platform provided. Please enter a valid platform.'
        return

    final_lib_path = lib_path + platform_dir + str(bits) + "/" + lib_name

    env.Append(LIBS=libs)
    env.Append(CPPPATH=source_dirs)
    
    library = env.SharedLibrary(target=final_lib_path, source=source_files)
    Default(library)

I know there's a lot to see here, and in some cases, this may be all you need. However, there are some other specific cases of usability that we need to look out for, and which could stand to improve this build file.

Special Operations - LLVM
-------------------------

A few things remain which can improve the usability of our ``SConstruct`` file.

The first is that we need to give users the option to use the alternative cross-platform compiler relying on LLVM: clang++. For this, all we need to do is supply another environment alteration.

.. code-block:: python
    #!python

    use_llvm = ARGUMENTS.get("use_llvm", "no")

    if use_llvm == "yes":
        env["CXX"] = "clang++"

To integrate this piece of code, we simply need to stick this snippet at the end of our current ``SConstruct`` file, just before we make the call to ``env.SharedLibrary()``.

Special Operations - Visual Studio Project Generation
-----------------------------------------------------

The second thing we want to address is for Windows users to be able to generate a Visual Studio project solution since many Windows devs rely on it.

Scons has another built-in method for doing this, similar to the ``SharedLibrary()`` method, called ``MSVSProject()``. It's syntax looks like this (taken from the scons documentation):

.. code-block:: python
    #!python

    barsrcs = ['bar.cpp']
    barincs = ['bar.h']
    barlocalincs = ['StdAfx.h']
    barresources = ['bar.rc','resource.h']
    barmisc = ['bar_readme.txt']

    dll = env.SharedLibrary(target = 'bar.dll', source = barsrcs)

    env.MSVSProject(
        target = 'Bar' + env['MSVSPROJECTSUFFIX'], srcs = barsrcs,
        incs = barincs,
        localincs = barlocalincs,
        resources = barresources,
        misc = barmisc,
        buildtarget = dll,
        variant = 'Release')

Now, this process can become very complicated, very quickly, so if you're interested in understanding how this is broken down, then follow along. Otherwise, you might as well skip past this section.

.. note:

    For those who are privy to Godot Engine's compilation, the ``vsproj`` and ``num_jobs`` parameters are identical to and serve the same purpose as the similarly named parameters for Godot Engine's main ``SConstruct`` file that builds the engine itself.

.. code-block:: python
    #!python

    # the name of our output library. We'll use this to define the name of our solution file
    lib_name = ARGUMENTS.get("name", "libdefault")

    # Whether or not we will even generate a Visual Studio project
    vsproj = ARGUMENTS.get("vsproj", "no")

    # The number of processes we'll use to aggregate files into our Visual Studio project.
    num_jobs = ARGUMENTS.get("num_jobs", 1)

    if vsproj == "yes":
        env.vs_incs = []
        env.vs_srcs = []

        def AddToVSProject(sources):
            for x in sources:
                if type(x) == type(""):
                    fname = env.File(x).path
                else:
                    fname = env.File(x)[0].path
                pieces = fname.split(".")
                if len(pieces) > 0:
                    basename = pieces[0]
                    basename = basename.replace('\\\\', '/')
                    if os.path.isfile(basename + ".h"):
                        env.vs_incs = env.vs_incs + [basename + ".h"]
                    elif os.path.isfile(basename + ".hpp"):
                        env.vs_incs = env.vs_incs + [basename + ".hpp"]
                    if os.path.isfile(basename + ".c"):
                        env.vs_srcs = env.vs_srcs + [basename + ".c"]
                    elif os.path.isfile(basename + ".cpp"):
                        env.vs_srcs = env.vs_srcs + [basename + ".cpp"]

        def build_commandline(commands):
            common_build_prefix = ['cmd /V /C set "plat=$(PlatformTarget)"',
                                    '(if "$(PlatformTarget)"=="x64" (set "plat=x86_amd64"))',
                                    'call "' + batch_file + '" !plat!']

            result = " ^& ".join(common_build_prefix + [commands])
            # print("Building commandline: ", result)
            return result

        def find_visual_c_batch_file(env):
            from  SCons.Tool.MSCommon.vc import get_default_version, get_host_target, find_batch_file

            version = get_default_version(env)
            (host_platform, target_platform, req_target_platform) = get_host_target(env)
            return find_batch_file(env, version, host_platform, target_platform)[0]

        env.AddToVSProject = AddToVSProject
        env.build_commandline = build_commandline

        env['CPPPATH'] = [Dir(path) for path in env['CPPPATH']]

        batch_file = find_visual_c_batch_file(env)
        if batch_file:
            env.AddToVSProject(source_files)

            # windows allows us to have spaces in paths, so we need
            # to double quote off the directory. However, the path ends
            # in a backslash, so we need to remove this, lest it escape the
            # last double quote off, confusing MSBuild
            env['MSVSBUILDCOM'] = build_commandline('scons --directory="$(ProjectDir.TrimEnd(\'\\\'))" platform=windows target=$(Configuration) -j' + str(num_jobs))
            env['MSVSREBUILDCOM'] = build_commandline('scons --directory="$(ProjectDir.TrimEnd(\'\\\'))" platform=windows target=$(Configuration) vsproj=yes -j' + str(num_jobs))
            env['MSVSCLEANCOM'] = build_commandline('scons --directory="$(ProjectDir.TrimEnd(\'\\\'))" --clean platform=windows target=$(Configuration) -j' + str(num_jobs))

            # This version information (Win32, x64, Debug, Release, Release_Debug seems to be
            # required for Visual Studio to understand that it needs to generate an NMAKE
            # project. Do not modify without knowing what you are doing.
            debug_variants = ['debug|Win32'] + ['debug|x64']
            release_variants = ['release|Win32'] + ['release|x64']
            release_debug_variants = ['release_debug|Win32'] + ['release_debug|x64']
            variants = debug_variants + release_variants + release_debug_variants

            # Sets up output executable names for each variant. The ordering of the final 'targets' array should match that of the final 'variants' array.

            target_name = 'bin\\' + lib_name + '.windows.'

            debug_targets = [target_name + 'tools.32.' + dl_suffix] + [target_name + 'tools.64.' + dl_suffix]
            release_targets = [target_name + 'opt.32.' + dl_suffix] + [target_name + 'opt.64.' + dl_suffix]
            release_debug_targets = [target_name + 'opt.tools.32.' + dl_suffix] + [target_name + 'opt.tools.64.' + dl_suffix]
            targets = debug_targets + release_targets + release_debug_targets

            msvproj = env.MSVSProject(target=['#' + lib_name + env['MSVSPROJECTSUFFIX']],
                                        incs=env.vs_incs,
                                        srcs=env.vs_srcs,
                                        runfile=targets,
                                        buildtarget=library, #recall that 'library' is the result of our 'env.SharedLibrary()' method call
                                        auto_build_solution=1,
                                        variant=variants)

        # handle cpp hint file
        if os.path.isfile(filename):
            # Don't overwrite an existing hint file since the user may have customized it.
            pass
        else:
            try:
                fd = open(filename, "w")
                fd.write("#define GDCLASS(m_class, m_inherits)\n")
            except IOError:
                print("Could not write cpp.hint file.")

Okay, this entire section looks incredibly complicated. Some parts of it are verbose, but ultimately simple. We'll break things down.

The first thing we do is grab our relevant parameters. One you'll recognize as the ``lib_name`` which we are using to decide on the name of our output dynamic library. In this case, we are doubling up the use of this name to also give a name to our generated solution file. The ``vsproj`` parameter, when equal to "yes", is just a flag to trigger all of this behavior. Finally, we get ``num_jobs``. This is a value that will enable us to rely on multithreading to build up our solution file, in case our dynamic library happens to be building from an exceptionally large set of files.

After grabbing our parameters and checking for the ``vsproj`` flag, we declare two arrays in our environment: one each for the *.cpp and *.h files we intend to include in our VS Project.

Then we declare a set of utiltiy methods. ``AddToVSProject`` just does some parsing of filepath structure to make sure that files have the right extension, before adding them to the two arrays. The ``build_commandline`` method does some contextual preparation for each build command specific to Visual Studio. Finally, we get the ``find_visual_c_batch_file`` method that locates the batch file used to initialize Visual Studio Compiler access. For more details, visit the ``find_batch_file`` source code here: https://scons.org/sphinx_wip/_modules/SCons/Tool/MSCommon/vc.html

With the declarations all out of the way, we add the methods to our ``Environment`` for later use by the ``build_commandline`` method and re-assert that all directories in our CPPPATH are indeed SCons Directory objects.

If we look for the Visual Studio batch file and find one, then it means we'll be capable of building a VS Project, so we proceed.

In the if-statement section, there are 4 things happening.

1. We first add the C++ code files we've collected to our respective header and source file directories by calling AddToVSProject.

2. We then define what command line operations to execute when users attempt to build, rebuild, or clean their VS Project. In these lines, you can clearly see us redefining these operations to use scons instead. This is also where you see the ``num_jobs`` parameter come into play, passing the "-j" + number of threads value at the end.

3. We define the set of platform and target combinations that will be available to the project as well as what resulting dynamic library they will each produce.

4. We go ahead and build the VS Project.

After having built the project, we top things off by building a simple C++ hint file that gives the compiler some extra definitions of C++ content for IntelliSense assistance.

In order to integrate this content, we place this content after our created ``library`` variable. We must also amend our OS-specific logic to include a determination of the appropriate dynamic library extension for the ``dl_suffix`` variable.

Conclusion
----------

And there you have it! That is the entire process. Below is the full ``SConstruct`` file that integrates all of the information we have covered.

Note that, because the Visual Studio Project Generation requires the header files to explicitly be supplied separately from the source files, we create analogous utilities to acquire those files as well.

This tutorial will likely evolve over time as more platforms are adapted into it, but it should at least get people started.

Here is a list of parameters and usage examples available for this build file. If you would like to learn more about developing your own ``SConstruct`` build file, I recommend you check out the official SCons User Guide here: https://scons.org/doc/production/PDF/scons-user.pdf

===================== ======== ================================= ================================= ==============================================
Name                  Required Default                           Format                            Description
--------------------- -------- --------------------------------- --------------------------------- ----------------------------------------------
platform              yes      "windows"                         "windows"|"linux"|"osx"           The targeted platform.
target                no       "debug"                           "debug"|"release"|"debug_release" The targeted release version.
bits                  no       64                                32|64                             The targeted bit version.
name                  no       "libdefault"                      any string                        The first portion of the library name.
lib                   no       "bin/"                            Directory Path                    The output directory.
headers               no       "godot_headers/"                  Directory Path                    The location of the godot_headers directory.
cpp_bindings_path     no       "godot-cpp/"                      Directory Path                    The location of the godot-cpp directory.
cpp_bindings_library  no       "godot-cpp/bin/godot-cpp"         Directory Path + bindings libname The location and name of the cpp bindings lib.
sources               no       "" ("." and "src/" auto-included) "DirPath,DirPath,..."             Additional source directories to include.
other_libs            no       ""                                "FilePath,FilePath,..."           Additional lib files to include.
vsproj                no       "no"                              "yes"|"no"                        Whether to generate a VS Project Solution.
num_jobs              no       1                                 Integer                           For 'vsproj', the number of threads to use.

==================================================================================================================================================

.. code-block:: python
    #!python
    import os

    ### utility method definitions ###

    def add_sources(sources, directory):
        for file in os.listdir(directory):
            if file.endswith('.cpp') or file.endswith('.c'):
                sources.append(directory + '/' + file)

    def add_headers(headers, directory):
        for file in os.listdir(directory):
            if file.endswith('.hpp') or file.endswith('.h'):
                headers.append(directory + '/' + file)

    ### output lib preparation ###

    target = ARGUMENTS.get("target", "debug")
    platform = ARGUMENTS.get("platform", "windows")
    bits = ARGUMENTS.get("bits", 64)
    lib_name = ARGUMENTS.get("name", "libdefault")

    library_file = lib_name + "." + platform + "." + str(bits)

    lib_path = ARGUMENTS.get("lib", "bin/")

    lib_path = lib_path.split("\\").join("/")
    if lib_path[-1:] != "/":
        lib_path += "/"

    ### library dependencies ###

    cpp_bindings_library_path = ARGUMENTS.get("cpp_bindings_library", "godot-cpp/bin/godot-cpp")

    ext = "z"
    while ext != "":
        cpp_bindings_library_path, ext = os.path.splitext(cpp_bindings_library_path)

    cpp_bindings_library = cpp_bindings_library_path + "." + platform + "." + str(bits)

    other_libs = ARGUMENTS.get("other_libs", "")
    other_libs = other_libs.split(",")

    libs = [cpp_bindings_library] + other_libs

    ### source dependencies ###

    godot_headers_path = ARGUMENTS.get("headers", "godot_headers/")
    cpp_bindings_path = ARGUMENTS.get("cpp_bindings_path", "godot-cpp/")
    sources = ARGUMENTS.get("sources", "")

    source_files = []
    header_files = []

    source_dirs = sources.split(",")

    for path in source_dirs:
        if os.path.isdir(path):
            add_sources(source_files, path)
            add_headers(header_files, path)

    source_dirs.append(godot_headers_path)
    source_dirs.append(cpp_bindings_path + "include/")
    source_dirs.append(cpp_bindings_path + "include/core/")

    ### OS-specific logic for flags and the output lib directory ###

    platform_dir = ""
    dl_suffix = ""

    if platform == "osx":
        env.Append(CCFLAGS = ['-g','-O3', '-arch', 'x86_64'])
        env.Append(LINKFLAGS = ['-arch', 'x86_64'])

        platform_dir = "osx"
        dl_suffix = "dylib"
    
    elif platform == "linux":
        env.Append(CCFLAGS = ['-fPIC', '-g','-O3', '-std=c++14'])

        platform_dir = "x11"
        dl_suffix = "so"
    
    elif platform == "windows":

        # Set exception handling model to avoid warnings caused by Windows system headers.
        env.Append(CCFLAGS=['-EHsc'])

        if target == "debug":
            env.Append(CCFLAGS = ['-D_DEBUG', '-MDd'])
        else:
            env.Append(CCFLAGS = ['-O2', '-DNDEBUG', '-MD'])

        platform_dir = "win"
        dl_suffix = "dll"

    else:
        # do nothing if we don't recognize the platform
        print 'unrecognized platform provided. Please enter a valid platform.'
        return

    final_lib_path = lib_path + platform_dir + str(bits) + "/" + lib_name

    env.Append(LIBS=libs)
    env.Append(CPPPATH=source_dirs)

    if ARGUMENTS.get("use_llvm", "no") == "yes":
        env["CXX"] = "clang++"
    
    library = env.SharedLibrary(target=final_lib_path, source=source_files)
    Default(library)

    ### VS Project Generation ###

    vsproj = ARGUMENTS.get("vsproj", "no")
    num_jobs = ARGUMENTS.get("num_jobs", 1)

    if vsproj == "yes":
        env.vs_incs = []
        env.vs_srcs = []

        def AddToVSProject(sources):
            for x in sources:
                if type(x) == type(""):
                    fname = env.File(x).path
                else:
                    fname = env.File(x)[0].path
                pieces = fname.split(".")
                if len(pieces) > 0:
                    basename = pieces[0]
                    basename = basename.replace('\\\\', '/')
                    if os.path.isfile(basename + ".h"):
                        env.vs_incs = env.vs_incs + [basename + ".h"]
                    elif os.path.isfile(basename + ".hpp"):
                        env.vs_incs = env.vs_incs + [basename + ".hpp"]
                    if os.path.isfile(basename + ".c"):
                        env.vs_srcs = env.vs_srcs + [basename + ".c"]
                    elif os.path.isfile(basename + ".cpp"):
                        env.vs_srcs = env.vs_srcs + [basename + ".cpp"]

        def build_commandline(commands):
            common_build_prefix = ['cmd /V /C set "plat=$(PlatformTarget)"',
                                    '(if "$(PlatformTarget)"=="x64" (set "plat=x86_amd64"))',
                                    'call "' + batch_file + '" !plat!']

            result = " ^& ".join(common_build_prefix + [commands])
            # print("Building commandline: ", result)
            return result

        def find_visual_c_batch_file(env):
            from  SCons.Tool.MSCommon.vc import get_default_version, get_host_target, find_batch_file

            version = get_default_version(env)
            (host_platform, target_platform, req_target_platform) = get_host_target(env)
            return find_batch_file(env, version, host_platform, target_platform)[0]

        env.AddToVSProject = AddToVSProject
        env.build_commandline = build_commandline

        env['CPPPATH'] = [Dir(path) for path in env['CPPPATH']]

        batch_file = find_visual_c_batch_file(env)
        if batch_file:
            env.AddToVSProject(source_files)
            env.AddToVSProject(header_files)

            # windows allows us to have spaces in paths, so we need
            # to double quote off the directory. However, the path ends
            # in a backslash, so we need to remove this, lest it escape the
            # last double quote off, confusing MSBuild
            env['MSVSBUILDCOM'] = build_commandline('scons --directory="$(ProjectDir.TrimEnd(\'\\\'))" platform=windows target=$(Configuration) -j' + str(num_jobs))
            env['MSVSREBUILDCOM'] = build_commandline('scons --directory="$(ProjectDir.TrimEnd(\'\\\'))" platform=windows target=$(Configuration) vsproj=yes -j' + str(num_jobs))
            env['MSVSCLEANCOM'] = build_commandline('scons --directory="$(ProjectDir.TrimEnd(\'\\\'))" --clean platform=windows target=$(Configuration) -j' + str(num_jobs))

            # This version information (Win32, x64, Debug, Release, Release_Debug seems to be
            # required for Visual Studio to understand that it needs to generate an NMAKE
            # project. Do not modify without knowing what you are doing.
            debug_variants = ['debug|Win32'] + ['debug|x64']
            release_variants = ['release|Win32'] + ['release|x64']
            release_debug_variants = ['release_debug|Win32'] + ['release_debug|x64']
            variants = debug_variants + release_variants + release_debug_variants

            # Sets up output executable names for each variant. The ordering of the final 'targets' array should match that of the final 'variants' array.

            target_name = 'bin\\' + lib_name + '.windows.'

            debug_targets = [target_name + 'tools.32.' + dl_suffix] + [target_name + 'tools.64.' + dl_suffix]
            release_targets = [target_name + 'opt.32.' + dl_suffix] + [target_name + 'opt.64.' + dl_suffix]
            release_debug_targets = [target_name + 'opt.tools.32.' + dl_suffix] + [target_name + 'opt.tools.64.' + dl_suffix]
            targets = debug_targets + release_targets + release_debug_targets

            msvproj = env.MSVSProject(target=['#' + lib_name + env['MSVSPROJECTSUFFIX']],
                                        incs=env.vs_incs,
                                        srcs=env.vs_srcs,
                                        runfile=targets,
                                        buildtarget=library, #recall that 'library' is the result of our 'env.SharedLibrary()' method call
                                        auto_build_solution=1,
                                        variant=variants)

        # handle cpp hint file
        if os.path.isfile(filename):
            # Don't overwrite an existing hint file since the user may have customized it.
            pass
        else:
            try:
                fd = open(filename, "w")
                fd.write("#define GDCLASS(m_class, m_inherits)\n")
            except IOError:
                print("Could not write cpp.hint file.")

