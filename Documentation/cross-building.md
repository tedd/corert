Cross Compilation for ARM on Linux
==================================

Through cross compilation, on Linux it is possible to build CoreRT for arm or arm64.

Requirements
------------

You need a Debian based host and the following packages needs to be installed:

    $ sudo apt-get install qemu qemu-user-static binfmt-support debootstrap

In addition, to cross compile CoreCLR the binutils for the target are required. So for arm you need:

    $ sudo apt-get install binutils-arm-linux-gnueabihf

and conversely for arm64:

    $ sudo apt-get install binutils-aarch64-linux-gnu


Generating the rootfs
---------------------
The `cross\build-rootfs.sh` script can be used to download the files needed for cross compilation. It will generate an Ubuntu 14.04 rootfs as this is what CoreRT targets.

    Usage: build-rootfs.sh [BuildArch]
    BuildArch can be: arm, arm64

The `build-rootfs.sh` script must be run as root as it has to make some symlinks to the system, it will by default generate the rootfs in `cross\rootfs\<BuildArch>` however this can be changed by setting the `ROOTFS_DIR` environment variable.

For example, to generate an arm rootfs:

    $ sudo ./cross/build-rootfs.sh arm

and if you wanted to generate the rootfs elsewhere:

    $ sudo ROOTFS_DIR=~/coreclr-cross/arm ./build-rootfs.sh arm

Cross compiling CoreCLR
-----------------------
Once the rootfs has been generated, it will be possible to cross compile CoreRT. If `ROOTFS_DIR` was set when generating the rootfs, then it must also be set when running `build.sh`.

So, without `ROOTFS_DIR`:

    $ ./build.sh arm debug verbose clean cross

And with:

    $ ROOTFS_DIR=~/coreclr-cross/arm ./build.sh arm debug verbose clean cross

As usual the resulting binaries will be found in `bin/Product/BuildOS.BuildArch.BuildType/`

Using CoreRT for cross compiling under arm on x86 host
-----------------------
It is possible to use CoreRT for compiling under arm/armel on x86 host (or on x64 machine using roots).

1. Build CoreCLR for x86 (`checked` version)
```
sudo ./cross/build-rootfs.sh x86 xenial
./build.sh clang3.9 x86 checked verbose cross skiptests
```

2. Build CoreFX (`Debug` version)
3. Build CoreRT for armel, x64, x86
```
sudo ./cross/build-rootfs.sh armel tizen
sudo ./cross/build-rootfs.sh x86 xenial
./build.sh clang3.9 armel debug verbose cross
./build.sh debug verbose skiptests
./build.sh clang3.9 x86 debug verbose cross skiptests
```

4. Copy necessary binaries to working directory (in x86 rootfs)
```
cp ${CORECLR}/bin/Product/Linux.x86.Checked ${WORKING_DIR}
cp ${CORERT}/bin/Linux.x86.Debug/tools/ilc.dll ${WORKING_DIR}
cp ${CORERT}/bin/Linux.x86.Debug/tools/ILCompiler.* ${WORKING_DIR}
cp ${CORERT}/bin/Linux.x86.Debug/tools/System.CommandLine.dll ${WORKING_DIR}
cp ${CORERT}/bin/Linux.x86.Debug/tools/Microsoft.DiaSymReader.dll ${WORKING_DIR}
cp ${CORERT}/bin/Linux.x86.Debug/tools/jitinterface.so ${WORKING_DIR}
cp -r ${CORERT}/bin/Linux.x86.Debug/framework ${WORKING_DIR}

# Copy CoreRT sdk binaries from target (armel) output folder
cp -r ${CORERT}/bin/Linux.armel.Debug/sdk ${WORKING_DIR}
```

5. Rename RyuJIT compiler library
```
# Use cross-compiler library as default for ILC
cp ${WORKING_DIR}/libarmelnonjit.so ${WORKING_DIR}/libclrjitilc.so

# ... or ARM version instead if it's needed
# cp ${WORKING_DIR}/libprotojit.so ${WORKING_DIR}/libclrjitilc.so
```

6. Build libobjwriter. You have to compile it on x86 chroot. Before compiling put coreclr/bin/Product/Linux.x86.Debug/ to some folder on x86 chroot as well. Versions which to used are mentioned on GitHub:
https://github.com/dotnet/corert/issues/3776#issuecomment-337682166

   And apply patch:
https://gist.github.com/alpencolt/ec75fcc05d8c4ffbf143a052f7c115a8
```
mkdir build
cd build
cmake ../ -DWITH_CORECLR=../../coreclr/bin/Product/Linux.x86.Debug/ -DLLVM_TARGET_ARCH="ARM;X86" -DLLVM_TARGETS_TO_BUILD="ARM;X86" -DLLVM_DEFAULT_TARGET_TRIPLE=thumbv7-linux-gnueabi -DCMAKE_BUILD_TYPE=Release -DLLVM_BUILD_LLVM_DYLIB=1 -DLLVM_LINK_LLVM_DYLIB=1 -DLLVM_OPTIMIZED_TABLEGEN=1 -DHAVE_POSIX_SPAWN=0 -DLLVM_ENABLE_PIC=1 -DLLVM_BUILD_TESTS=0 -DLLVM_ENABLE_DOXYGEN=0 -DLLVM_INCLUDE_DOCS=0 -DLLVM_INCLUDE_TESTS=0 -DLLVM_BINUTILS_INCDIR=/usr/include
make -j8 objwriter
```

7. And to execute use:
```
./corerun ilc.dll --verbose @Hello.ilc.rsp

# For linking
clang-3.9 -target arm-linux-gnueabi --sysroot=corert/cross/rootfs/armel -Bcorert/cross/rootfs/armel/usr/lib/gcc/armv7l-tizen-linux-gnueabi/6.2.1 -Lcorert/cross/rootfs/armel/usr/lib/gcc/armv7l-tizen-linux-gnueabi/6.2.1 Hello.o -o Hello corert/bin/Linux.armel.Debug/sdk/libbootstrapper.a corert/bin/Linux.armel.Debug/sdk/libRuntime.a corert/bin/Linux.armel.Debug/sdk/libSystem.Private.CoreLib.Native.a corert/bin/Linux.armel.Debug/framework/System.Native.a corert/bin/Linux.armel.Debug/framework/libSystem.Globalization.Native.a -g -Wl,-rpath,'$ORIGIN' -pthread -lstdc++ -ldl -lm -luuid -lrt -fPIC
```
