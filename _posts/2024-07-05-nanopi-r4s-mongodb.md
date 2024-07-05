---
layout: post
title: "NanoPi R4S: Compiling MongoDB and Installing Unifi for Debian LXC Container"
date: 2024-07-05 10:31:27 +00:00
---

To recap,
my goal is to run OpenWRT on a NanoPi R4S,
and also run Unifi Network Application inside an LXC container on the same hardware.
I specifically do not want to run it dockerized,
so that means I have to bring all my dependencies.
Let's start with MongoDB.

## Choosing MongoDB Version

MongoDB does provide arm64 packages for some distributions,
but 4.4 is already end of life,
and stating from 5.0,
["requires the ARMv8.2-A or later microarchitecture."](https://www.mongodb.com/docs/manual/administration/production-notes/#arm64)
The SoC inside the NanoPi R4S, the RK3399, has Cortex A72 / A53 cores implementing ARMv8-A.
So the official binaries will not work here.

There are people who provide MongoDB binaries built for ARMv8,
such as [this](https://github.com/Inqnuam/MongoDB-ARMv8),
but I want to build it from source myself.

Unifi Network Application 8.1 finally [added support for MongoDB "up to MongoDB 7.0"](https://community.ui.com/releases/UniFi-Network-Application-8-1-127/571d2218-216c-4769-a292-796cff379561),
so let's use 7.0.

## Compiling

Read the [documentation on building **from the versioned branch**,](https://github.com/mongodb/mongo/blob/v7.0/docs/building.md)
otherwise you'll get really confused about things mentioned that do not exist.

Don't even think about compiling natively on device.
Use a beefy desktop or server for the job.

I'm running Debian Bookworm inside the container,
so my build environment is also Debian Bookworm.
It's also `amd64` instead of `aarch64`,
so I need to install cross compilers:

```shell
$ sudo apt install crossbuild-essential-arm64
```

Clone the source repo and checkout a tagged release.
At the time of writing, the latest 7.0 release tag is `r7.0.12`.
Follow the docs carefully to set up the build environment.

The dev packages of runtime dependencies need to be installed in the [host (Debian nomenclature)](https://wiki.debian.org/CrossCompiling) architecture:

```shell
$ sudo dpkg --add-architecture arm64
$ sudo apt update
$ sudo apt install libssl-dev:arm64 libcurl4-openssl-dev:arm64
```

When it finally comes to invoke the build command, note:

- We only need the server, so use target `install-mongod`.
- Specify the correct cross compilers.
- Specify appropriate `march` and `mtune`.
We can discover the correct `march` by [testing for it on the host](https://stackoverflow.com/questions/5470257/how-to-see-which-flags-march-native-will-activate).
It's OK to leave `mtune` as `generic`,
but I decided to pick the [value corresponding to the actual core configuration according to the docs.](https://gcc.gnu.org/onlinedocs/gcc/AArch64-Options.html#index-mtune)
- We are using C++ compilers newer than the supported version (bookworm comes with GCC 12),
so do what the doc says.
- Once we try to run the command,
it will tell us the default linker won't work and to use `--linker=gold` instead
- The final binary will be huge (4+ GB!) due to embedded debug info.
We can strip it manually,
but having the build process strip it for us is nice.
Oh, and we must specify a cross `objcopy` as well.

Taking all the above into consideration, the final magic incantation is:

```shell
$ python buildscripts/scons.py install-mongod CC=aarch64-linux-gnu-gcc CXX=aarch64-linux-gnu-g++ OBJCOPY=aarch64-linux-gnu-objcopy CCFLAGS="-march=armv8-a+crypto+crc -mtune=cortex-a72.cortex-a53" --linker=gold --disable-warnings-as-errors --separate-debug
```

It took a few hours on 12 cores, using more than 16GB memory.
Again, don't even think about building this natively on device.

At the end, the `mongod` binary can be found in `build/install/bin/`.

## Satisfying dependencies

Now that `mongod` is installed somewhere in the LXC container,
we can try [installing `unifi`.](https://help.ui.com/hc/en-us/articles/220066768-Updating-and-Installing-Self-Hosted-UniFi-Network-Servers-Linux)

Alas, the installation will fail because the `unifi` package depends on mongodb packages.
Since we installed the binary manually, there is no package.

The proper solution here is, of course,
package the binary into a proper package and installing it.
But that's too hard.
Instead,
[we can create a fake package just to satisfy the dependency.](https://wiki.debian.org/Packaging/HackingDependencies)

Just use an `.equiv` file like this:

```
Package: mongodb-server
Version: 1:7.0.12-1
Maintainer: Your Name <your.email@example.com>
Architecture: all
Description: dummy mongodb-server package
  A dummy package to satisfy dependencies.
```

and follow the instructions above.

Apart from MongoDB, `unifi` pulls in its other dependencies like Java, so that's nice.

## Installing `mongod` so that Unifi Network Application can use it

There is no need to start a separate `mongod` service,
because `unifi` starts one for itself.
Presumably it will try to find `mongod` on `$PATH`.
If you did not install `mongod` somewhere in `$PATH`,
just make a symlink in `/usr/lib/unifi/bin`.
