# ldc-iphone-dev
An LDC (LLVM-base D Compiler) development sandbox for targetting
iOS and soon its siblings tvOS, watchOS.

This [repo](https://github.com/smolt/ldc-iphone-dev) glues together
various pieces needed to build an LDC cross compiler targeting Apple's
iOS for iPhone, iPad, and iPod Touch.  It also includes a few samples
to show how to get started.  The compiler and libraries are in good
enough shape to pass the druntime/phobos unittests (see
[Unittest Status](#unittest-status) below).  So yes, you can build
your D library and use it in an iOS App.

Versions derived from: LDC 0.17.0 alpha (DMD v2.068.2) and LLVM 3.6.2.

There is still stuff to [work on](#what-is-missing), but overall the core D language is ready for iOS.

## License 
Please read the [APPLE_LICENSE](https://github.com/smolt/iphoneos-apple-support/blob/master/APPLE_LICENSE) in directory iphoneos-apple-support before using.  This subdirectory has some modified source code derived from http://www.opensource.apple.com that makes TLS work on iOS.  As I understand it, if you publish an app or source that uses that code, you need to follow the provisions of the license.

LLVM also has its [LICENSE.TXT](https://github.com/smolt/llvm/blob/ios/LICENSE.TXT) and LDC its combined [LICENSE](https://github.com/smolt/ldc/blob/ios/LICENSE).

## Prerequisites

You will need an OS X host and Xcode. I am currently using Yosemite
10.10.5 and test against Xcode 6.4 and 7.1. Both seem to work fine,
but Xcode 7.1 sometimes outputs warnings about missing debug symbols
whereas Xcode 6.4 is quiet. You also need to set Enable Bitcode to NO
in Xcode 7 for now.

The prerequisite packages are pretty much the same as listed for [building LDC](http://wiki.dlang.org/Building_LDC_from_source) with my comments in parentheses:

- git
- a C++ toolchain (use Xcode since iPhoneSDK is needed anyway to cross compile C in druntime/phobos and to run on an iOS device)
- CMake 2.8+ (I am using cmake-3.1.0-Darwin64.dmg from http://www.cmake.org/download/.  You will need to install command line tools by running CMake app and using install command line menu choice)
- libconfig++ and its header files (I built from source downloaded from http://www.hyperrealm.com/libconfig/)
- libcurl (it will be automatically downloaded during build-all and placed in the
  extras directory.  Comes from http://seiryu.home.comcast.net/~seiryu/libcurl-ios.html
  : *Hmmm, this download is no longer available as of 11/3/2015, will look for alternative*)

LLVM is included as a submodule since it has been modified to support TLS on iOS.  No other LLVM will work.

To really have fun, you will need an iOS device.  Beginning with Xcode
7 you can download to your own devices without enrolling in the
Developer Program.

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

This only gives you a .o file but using Xcode and a provisioning
profile you could link, code sign, bundle, and run it on an iOS
device.  A sample Xcode [project](#sample-hellod-project) does just
that if you have a provisioning profile.  Without a provisioning
profile, you can still try out D with the iOS Simulator.

To generate code for the iOS Simulator, just change the `-arch`
option:

```
$ tools/iphoneos-ldc2 -arch i386 -c hello.d
```

Note that if you don't specify `-arch`, the default target is now the i386
iOS Simulator (used to be armv7).

At this point, you have an LDC toolchain and universal druntime/phobos libs
in `build/ldc` for armv7, armv7s, arm64 (iPhone 3gs to iPhone 6),
and i386/x86_64 iOS Simulator.  The compiler can also target armv6 target
(original iPhone and other older devices), but it is left out of the
universal libs to speed build time.  For more
information
[see Device Compatibility](https://developer.apple.com/library/ios/documentation/DeviceInformation/Reference/iOSDeviceCompatibility/DeviceCompatibilityMatrix/DeviceCompatibilityMatrix.html).

Testing to date is on armv7 and arm64, and iOS 6, 7, and 8.

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
=-=-= Testing druntime/phobos debug build =-=-=
Setting dir to TMPDIR so unittests can write files if needed.
cd /private/var/mobile/Applications/F3F25E2E-0C8C-4B6F-A976-54D747D5D4BE/tmp
FPU Flush to Zero and Default NaN disabled for tests
Testing 1 core.atomic: OK (took 0ms)
Testing 2 core.bitop: OK (took 124ms)
Testing 3 core.checkedint: OK (took 0ms)
Testing 4 core.demangle: OK (took 49ms)
...
Testing 136 std.zlib: OK (took 248ms)
Testing 137 rt.lifetime: OK (took 111ms)
Passed 136 of 137 (2 have tailored tests), 182 other modules did not have tests
Restoring FPU mode
```

Note: that iOS by default runs with the ARM FPU "Default NaN" (arm,arm64) and "Flush to Zero" (arm only) modes enabled.  In order to pass many of the math unittests, these modes are disabled first.  This is something to consider if you are doing some fancy math and expect full subnormal and NaN behavior.

### Unittest Status
Everything seems to be working!  A couple comparisons in
std.internal.math.gammafunction are off by a ulp, but no big deal.

## What is Missing
- Support tvOS and watchOS.  Working on this thanks to LLVM 3.8, hopefully done
  before new years!
- Xcode/D integration.  I think this is the next most import thing.
- Objective-C interop - work in progress under [DIP 43](http://wiki.dlang.org/DIP43)
- APIs for iPhone SDK - [DStep](https://github.com/jacob-carlborg/dstep) helps here
- A D-based iOS App submitted to Apple App Store
- A D-based iOS App accepted by the Apple App Store!
- Hack lldb to understand D symbols and types so debugging is better
- Figure out what else is left to do
