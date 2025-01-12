# n2n on macOS

In order to use n2n on macOS, you first need to install support for TUN/TAP interfaces:

```bash
brew tap homebrew/cask
brew cask install tuntap
```

If you are on a modern version of macOS (i.e. Catalina), the commands above will ask you to enable the TUN/TAP kernel extension in System Preferences → Security & Privacy → General.

For more information refer to vendor documentation or the [Apple Technical Note](https://developer.apple.com/library/content/technotes/tn2459/_index.html).


# Build on Windows

## Requirements

In order to build on Windows the following tools should be installed:

- Visual Studio. For a minimal install, the command line only build tools can be
  downloaded and installed from https://visualstudio.microsoft.com/downloads/#build-tools-for-visual-studio-2017.

- CMake

- (optional) The OpenSSL library. Prebuild binaries can be downloaded from https://slproweb.com/products/Win32OpenSSL.html.
  The full version is required, i.e. not the "Light" version. The Win32 version of it is usually required for a standard build.

> NOTE: In order to skip OpenSSL compilation, edit `CMakeLists.txt` and replace **– is this still valid?**
>
>  ```plaintext
>  OPTION(N2N_OPTION_AES "USE AES" ON)
>  with
>  OPTION(N2N_OPTION_AES "USE AES" OFF)
>  ```

  NOTE: To statically link OpenSSL, add the `-DOPENSSL_USE_STATIC_LIBS=true` option to the `cmake` command below.

- If compilation throws a "config.h: No such file or directory" error, an `include/config.h` file needs to be obtained from an already configured Linux compilation and put into the `include/` directory as discussed [here](https://github.com/ntop/n2n/issues/366).

In order to run n2n, you will need the following:

- The TAP drivers should be installed into the system. They can be installed from
  http://build.openvpn.net/downloads/releases, search for "tap-windows".

- If OpenSSL has been linked dynamically, the corresponding `.dll` file should be available
  onto the target computer.
  
NOTE: Sticking to this tool chain ensures that resulting executables are able to communicate with Linux or other OS builds.
Especialy MinGW builds are reported to not be compatible to other OS builds, please see [#617](https://github.com/ntop/n2n/issues/617) and [#642](https://github.com/ntop/n2n/issues/642).

## Build (CLI)

In order to build from the command line, open a terminal window and run the following commands:

```batch
md build
cd build
cmake ..

MSBuild.exe edge.vcxproj /t:Build /p:Configuration=Release
MSBuild.exe supernode.vcxproj /t:Build /p:Configuration=Release
MSBuild.exe n2n-benchmark.vcxproj /t:Build /p:Configuration=Release
```

NOTE: If CMake has problems finding the installed OpenSSL library, try to download the official cmake and invoke it with
`C:\Program Files\CMake\bin\cmake`.

NOTE: Visual Studio might not add `MSBuild.exe`'s path to the environment variable %PATH% so you might have difficulties finding and executing it without giving the full path. Regular installations seem to have it reside at `"C:\Program Files (x86)\Microsoft Visual Studio\2019\BuildTools\MSBuild\Current\Bin\MSBuild.exe"`

The compiled `.exe` files should now be available in the `build\Release` directory.

## Run

The `edge.exe` program reads the `edge.conf` file located into the current directory if no option is provided.

Here is an example `edge.conf` file:

```plaintext
-c=mycommunity
-k=mysecretpass

# supernode IP address
-l=1.2.3.4:5678

# edge IP address
-a=192.168.100.1
```

The `supernode.exe` program reads the `supernode.conf` file located into the current directory if no option is provided.

Here is an example `supernode.conf` file:

```plaintext
-p=5678
```

See `edge.exe --help` and `supernode.exe --help` for a full list of supported options.

# General Building Options

## Compiler Optimizations

The easiest way to boosting speed is by allowing the compiler to apply optimization to the code. To let the compiler know, the configuration process can take in the optionally specified compiler flag `-O3`:

`./configure CFLAGS="-O3"`

The `tools/n2n-benchmark` tool reports speed-ups of 200% or more! There is no known risk in terms of instable code or so.

## Hardware Features

Some parts of the code can be compiled to benefit from available hardware acceleration. It needs to be decided at compile-time. So, if compiling for a specific platform with known features (maybe the local one), it should be specified to the compiler, for example through the `-march=sandybridge` (you name it) or just `-march=native` for local use.

So far, the following portions of n2n's code benefit from hardware features:

```
AES:               AES-NI
ChaCha20:          SSE2, SSSE3
SPECK:             SSE2, SSSE3, AVX2, AVX512, (NEON)
Pearson Hashing:   AES-NI
Random Numbers:    RDSEED, RDRND (not faster but more random seed)
```

The compilations flags could easily be combined:

`./configure CFLAGS="-O3 -march=native"`.

## OpenSSL Support

Some ciphers' speed can take advantage of OpenSSL support which is disabled by default as the built-in ciphers already prove reasonably fast in most cases. OpenSSL support can be configured using

`./configure --with-openssl`

which then will include OpenSSL 1.1 if found on the system. This can be combined with the hardware support and compiler optimizations such as

`./configure --with-openssl CFLAGS="-O3 -march=native"`

Please do no forget to `make clean` after (re-)configuration and before building (again) using `make`.

## ZSTD Compression Support

In addition to the built-in LZO1x for payload compression (`-z1` at the edge's commandline), n2n optionally supports [ZSTD](https://github.com/facebook/zstd). As of 2020, it is considered cutting edge and [praised](https://en.wikipedia.org/wiki/Zstandard) for reaching the currently technologically possible Pareto frontier in terms of CPU power versus compression ratio. ZSTD support can be configured using

`./configure --with-zstd`

which then will include ZSTD if found on the system. It will be available via `-z2` at the edges. Of course, it can be combined with the other features mentioned above:

`./configure --with-zstd --with-openssl CFLAGS="-O3 -march=native"`

Again, and this needs to be reiterated sufficiently often, please do no forget to `make clean` after (re-)configuration and before building (again) using `make`.

## Federation – Supernode Selection by Round Trip Time

If used with multiple supernodes, by default, an edge choses the least loaded supernode to connect to. This selection strategy is part of the [federation](Federation.md) feature and aims at a fair workload distribution among the supernodes. To serve special scenarios, an edge can be compiled to always connect to the supernode with the lowest round trip time, i.e. the "closest" with the lowest ping. However, this could result in not so fair workload distribution among supernodes. This option can be configured by defining the macro `SN_SELECTION_RTT` and affects edge's behaviour only:

`./configure CFLAGS="-DSN_SELECTION_RTT"`

which of course can be combined with the compiler optimizations mentioned above…

Note that the activation of this strategy requires a sufficiently accurate local day-of-time clock. It probably will fail on smaller systems using `uclibc` (instead of `glibc`) whose day-of-time clock is said to not provide sub-second accuracy.

## SPECK – ARM NEON Hardware Acceleration

By default, SPECK does not take advantage of ARM NEON hardware acceleration even if compiled with `-march=native`. The reason is that the NEON implementation proved to be slower than the 64-bit scalar code on Raspberry Pi 3B+, see [here](https://github.com/ntop/n2n/issues/563).

Your specific ARM mileage may vary, so it can be enabled by configuring the definition of the `SPECK_ARM_NEON` macro:

`./configure CFLAGS="-DSPECK_ARM_NEON"`

Just make sure that the correct architecture is set, too. `-march=native` usually works quite well.

## Disable Multicast Local Peer Detection

For better local peer detection, the edges try to detect local peers by sending REGISTER
packets to a certain multicast address. Also, edges listen to this address to eventually
fetch such packets.

If these packets disturb network's peace or even get forwarded by (other) edges through the
n2n network, this behavior can be disabled, just add

`-DSKIP_MULTICAST_PEERS_DISCOVERY`

to your `CFLAGS` when configuring, e.g.

`./configure --with-zstd CFLAGS="-O3 -march=native -DSKIP_MULTICAST_PEERS_DISCOVERY"`
