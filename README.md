# ldc-iphone-dev
An LDC (LLVM-base D Compiler) development sandbox for iPhone iOS.

This [repo](https://github.com/smolt/ldc-iphone-dev) glues together various pieces needed to build an LDC cross compiler targeting iPhoneOS.  It also includes a few samples to show how to get started.  The compiler and libraries are in good enough shape to pass the druntime/phobos unittests with a few minor test failures (see [Unittest Status](#unittest-status) below).  This means someone could, if so inclined, build their D library and use it in an iOS App.  In theory.

Versions derived from: LDC 0.15.1 (DMD v2.066.1) and LLVM 3.5.1.

There is still stuff to [work on](#what-is-missing), but overall the core D language is ready to try on iOS.

## License 
Please read the [APPLE_LICENSE](https://github.com/smolt/iphoneos-apple-support/blob/master/APPLE_LICENSE) in directory iphoneos-apple-support before using.  This subdirectory has some modified source code derived from http://www.opensource.apple.com that makes TLS work on iOS.  As I understand it, if you publish an app or source that uses that code, you need to follow the provisions of the license.

LLVM also has its [LICENSE.TXT](https://github.com/smolt/llvm/blob/ios/LICENSE.TXT) and LDC its combined [LICENSE](https://github.com/smolt/ldc/blob/ios/LICENSE).

## Prerequisites
You will need an OS X host and Xcode.  I am currently on Yosemite 10.10.3 and Xcode 6.3.2.

The prerequisite packages are pretty much the same as listed for [building LDC](http://wiki.dlang.org/Building_LDC_from_source) with my comments in parentheses:

- git
- a C++ toolchain (use Xcode since iPhoneSDK is needed anyway to cross compile C in druntime/phobos and to run on an iOS device)
- CMake 2.8+ (I am using cmake-3.1.0-Darwin64.dmg from http://www.cmake.org/download/.  You will need to install command line tools by running CMake app and using install command line menu choice)
- libconfig++ and its header files (I built from source downloaded from http://www.hyperrealm.com/libconfig/)
- libcurl (needed by std.net.curl but I have not built for iOS yet. On TODO list)

LLVM is included as a submodule since it has been modified to support TLS on iOS.  No other LLVM will work.

To really have fun, you will need an iOS device.  There is also news
that with Xcode 7 you can download to your own devices without
enrolling in the Developer Program.

## Build
Download with git and build:

```
$ git clone --recursive https://github.com/smolt/ldc-iphone-dev.git
$ cd ldc-iphone-dev
$ tools/build-all
```

and grab a cup of coffee.  It will build LLMV, LDC, druntime, phobos,
and iphone-apple-support.  LLVM takes the longest by far, but probably
only needs to be built once.  The shell script `build-all` may eventually do some nice checking, but for now mostly just calls `make -j n` where `n` is all your cores but one.

If updating an existing clone and want a clean rebuild, do something like:

```
$ git submodule update --init --recursive
$ tools/build-all -f
```

You can quickly try the resulting compiler to generate armv7 code by typing:

```
$ tools/iphoneos-ldc2 -arch armv7 -c hello.d
```

This only gives you a .o file but using Xcode and a provisioning profile you could link, codesign, bundle, and run it on an iOS device.  A sample Xcode [project](#sample-hellod-project) does just that if you have a provisioning profile.
Without a provisioning profile, you can still try out D with the iOS Simulator.

To generate code for the iOS Simulator, just change the `-arch`
option:

```
$ tools/iphoneos-ldc2 -arch i386 -c hello.d
```

Note that if you don't specify `-arch`, the default target is now the i386
iOS Simulator (used to be armv7).

At this point, you have an LDC toolchain and universal druntime/phobos libs
in `build/ldc` for 32-bit armv7, armv7s (iPhone 3gs to iPhone 5c),
and i386 iOS Simulator.  The compiler can also target armv6 target
(original iPhone and other older devices), but it is left out of the
universal libs to speed build time.  Missing is arm64 for iPhone 6 and iPhone 5s, although
these 64-bit devices can run 32-bit instructions too.  For more
information
[see Device Compatibility](https://developer.apple.com/library/ios/documentation/DeviceInformation/Reference/iOSDeviceCompatibility/DeviceCompatibilityMatrix/DeviceCompatibilityMatrix.html).

My only hardware is armv7 so that is all I can really test for the moment.

`tools/iphoneos-ldc2` is nothing more than a script that redirects to what
is built in `build/ldc/bin/iphoneos-ldc2`.  The binaries have a
`iphoneos-` prefix to remind you that the defaut target is iPhoneOS.

## Sample helloD Project
[helloD](https://github.com/smolt/ldc-iphone-dev/tree/master/helloD) is a barebones Xcode project with four simple targets.  It only uses the console to demonstrate LDC compiled code running on an iOS device.

- hello_nolibs - the simplest barebones D without reliance on any D libs or libiphoneossup (not needed if TLS not used).
- helloD_druntime - a demo of various things in druntime including threads, TLS vars, and garbage collection.
- helloD - just hello using phobos std.stdio.writeln.
- objc_helloD - demo of how an Objective-C (or C or C++) main can use D.

## Unittests
An Xcode project called [unittester](https://github.com/smolt/ldc-iphone-dev/tree/master/unittester) is included that has targets for running the druntime and phobos unittests.  Two are D only with output to the console (nothing to see on the iOS device besides a black screen).  The other two are simple scrolling text apps that show the D unittest output as it runs.  These apps manage the UI with Objective-C and run the D unittests in another thread.

You can build and run the console druntime/phobos unittests from the shell.  Here I am running on my iPad mini (cortex-a9):

```
$ make -j 3 unittest
$ xcodebuild -project unittester/unittester.xcodeproj -destination "platform=iOS,name=Dan's iPad" test -scheme debug

=== BUILD TARGET unittester-debug OF PROJECT unittester WITH CONFIGURATION Debug ===
...
Testing 1 core.atomic: OK (took 0ms)
Testing 2 core.bitop: OK (took 0ms)
Testing 3 core.checkedint: OK (took 0ms)
Testing 4 core.demangle: OK (took 17ms)
...
Testing 113 std.zlib: OK (took 96ms)
Passed 112 of 113 (3 have tailored tests), 60 other modules did not have tests
Restoring FPU mode

$ xcodebuild -project unittester/unittester.xcodeproj -destination "platform=iOS,name=Dan's iPad" test -scheme release

=== BUILD TARGET unittester-release Tests OF PROJECT unittester WITH CONFIGURATION Debug ===
...
Testing 1 core.atomic: OK (took 0ms)
Testing 2 core.bitop: OK (took 0ms)
Testing 3 core.checkedint: OK (took 0ms)
Testing 4 core.demangle: OK (took 10ms)
...
Testing 113 std.zlib: OK (took 92ms)
Passed 112 of 113 (3 have tailored tests), 60 other modules did not have tests
Restoring FPU mode
```

Note: that iOS by default runs with the ARM FPU "Default NaN" and "Flush to Zero" modes enabled.  In order to pass many of the math unittests, these modes are disabled first.  This is something to consider if you are doing some fancy math and expect full subnormal and NaN behavior.

### Unittest Status
Most druntime and phobos unittests pass with the exceptions being math
related.

- std.internal.math.gammafunction - needs update for 64-bit reals
- std.math - floating point off-by-one LSB error in a few cases (I think this is a don't care)

All the failures are marked in the druntime and phobos source with
versions that begin with "WIP" to workaround the failure so rest of
test can run.  Grep for "WIP" to see all the details.

## What is Missing
Or what is left to do.

In work now:
- Get arm64 target working
- Xcode/D integration (could use someone who loves working with Xcode plugins)
- Update to future release LDC 0.16.0 (DMD FE 2.067)

Back burner
- Add libcurl to enable std.net.curl
- Fix extern(C) ABI for small struct return values
- Objective-C interop - work in progress under [DIP 43](http://wiki.dlang.org/DIP43)
- APIs for iPhone SDK - [DStep](https://github.com/jacob-carlborg/dstep) helps here
- A D-based iOS App submitted to Apple App Store
- A D-based iOS App accepted by the Apple App Store!
- Hack lldb to understand D symbols and types so debugging is better
- Figure out what else is left to do
