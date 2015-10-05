As of version 2.2.0, we've change the library structure of this library to a header-only implementation. So as of now, all the M2X library implementation will be in `M2XStreamClient/M2XStreamClient.h` file. For each platform, this header file might include one more addtional platform specific header file, the naming of this file follows `m2x-<platform>.h` convention. So for Arduino, this is `m2x-arduino.h`, for ESP8266, this is `m2x-esp8266.h`.

We will provide a little rationale on how we decide to make this change.

#### I thought using headers for declarations, and sources for implementations is good practice for C/C++ programming?

This might (or might not) be true if you have a large codebase for a sophisticated application. However, for a library, one thing we cannot ignore is build system. One application might use [Make](https://www.gnu.org/software/make/), another might use [CMake](https://cmake.org/), yet another one might use [SCons](http://www.scons.org/), [Bazel](http://bazel.io/) or even [home-grown build systems](http://nethack4.org/projects/aimake/). To make things worse, each embedded system might have its own stack: Arduino has [one](https://www.arduino.cc/en/Hacking/LibraryTutorial), mbed has [one](https://developer.mbed.org/compiler/), and Cypress also has [one](http://www.cypress.com/products/psoc-creator)!

This is slowly becoming a nightmare for us since we are planning to support all those boards with one library and one set of APIs. We have a common codebase, which is the current one. Due to the limitations of each platform, we have to create [many](https://github.com/attm2x/m2x-launchpad-energia) [many](https://github.com/attm2x/m2x-galileo) [different](https://github.com/attm2x/m2x-cypress-psoc) [repo](https://github.com/attm2x/m2x-arm-mbed) for each platform here. The underlying code only differs in very small part of the code, hence we have to maintain all of them at once.

We do have a fresh rewrite of the Arduino library going on internally, this is not ready for release yet since we are still polishing the API, so we start to wonder: is there anything we can do with the current codebase?

Of course! There's something called [Unity](http://buffered.io/posts/the-magic-of-unity-builds/) [Build](http://engineering-game-dev.com/2009/12/15/the-evils-of-unity-builds/) [exists](https://cheind.wordpress.com/2009/12/10/reducing-compilation-time-unity-builds/) in the C/C++ community. The trick here is use `#include` to combine multiple `.h` or even `.c` or `.cpp` file into a single file, so compiling the single file is enough for building the project. Since this avoids recomplication brought by including the same header multiple times, some argue that this might bringing performance boosts as well as architectural benefits.

While this is [supported](http://number-none.com/blow/john_carmack_on_inlined_code.html) by some very famous name in the C++ community, some other people have questions on this, since it ruins the possibility of parallel compiling: with one single source code, you can only compile it with one core, which might contain more disadvantages.

However, while we are not aiming for the same performance benefits, we can introduce this technique to this library for the architectural benefits! We can shrink our library to a single file so it can be trivial to include it in any build system!

#### Okay I get Unity Build, but it does not explain why you are only using header files!

Well, Unity Build normally talks about compiling applications, not libraries. With an application, we will inevitably have a source code, we cannot avoid this, so Unity Build just advocates using one source file. However, for libraries, having source file means you have to include it in the host build system, while using a header file, all we need is a `#include` line, we don't need to do anything at the build system level(well, actually we have to, but the point is: we've already shrink our build system integration to the minimum).

Besides that, there's also a practical reason at Arduino side:

Remember that in Arduino, the actual user can only write source code in a single `.ino` file? With the previous version of this library, we cannot do something like:

```
#define DEBUG
#include <M2XStreamClient.h>
```

This is because the `.cpp` file in our library cannot pick up the `DEBUG` macro. Hence to debug the library, we have to manually navigate to the M2X client library, change the header file to add the `DEBUG` macro, then re-compile everything. This is very cumbersome, and it's hard to remember to change the macro back once you've done with debugging. With a single header file, we can simply define the `DEBUG` macro in the `.ino` file, which saves a whole lot of hassles.

In fact, we take this technique one step further: we also let the user specify what board platform he / she is using now! So the user would actually use:

```
#define ARDUINO_PLATFORM
#include <M2XStreamClient.h>
```

To define that he / she is now using Arduino boards. This way we can provide supports for many different boards within one single library. The maintainence hassle can also be avoided. If someone is contributing to this library, we don't have to manually port the same changes to many different libraries.

#### Ahhh, I don't want to specify the platform to use every time I'm writing some code, can't you just auto-detect this?

Of course! We do have this in mind. In the future we will definitely add auto-detection code so if you don't specify the platform to use, we can detect them for you. However, since there're many embedded development environments, and new environments are coming out everyday, we do want to make sure we are careful with this feature, since it might be worse to detect the platform wrong than not detecting at all.

And even if we have auto-detection in place, we will also support manual overrides, so if you do provide a platform definition at the top of the `#include` line, we will use your specified one and disable auto-detection.
