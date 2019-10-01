# Getting started with NVDLA

NVDLA enables accelerating neural network inference job which is achieved in two steps
1. Optimize trained neural network for DLA hardware and convert the graph to DLA HW instructions. This converted graph is saved to a flatbuffer file called as loadable. This is achieved using NVDLA compiler and performed offline on host system.
2. Run inference job on DLA using loadable from step 1. This is achieved using NVDLA runtime and performed on target system.

* [NVDLA Compiler](#nvdla-compiler)
    * [Help](#nvdla-compiler-help)
    * [Example compiling ResNet-50 for nv_small](#nvdla-compiler-example)
    * [Output](#output)
    * [Modifying NVDLA Compiler](#nvdla-compiler-update)
* [NVDLA Runtime](#nvdla-runtime)
    * [Help](#nvdla-runtime-help)
    * [Example running ResNet-50 on nv_small](#nvdla-runtime-example)
    * [Modifying NVDLA Runtime](#nvdla-runtime-update)
* [NVDLA Platforms](#nvdla-platforms)
* [System requirements for Virtual Platform](#system-requirements)
* [Buildroot](#buildroot)
* [FireSim](#firesim)

## NVDLA Compiler

NVDLA compiler is used to optimize neural network for DLA HW architecture and create list of HW instructions to run inference on DLA.  NVDLA compiler can be built from [source code](https://github.com/nvdla/sw/tree/master/umd/core/src/compiler) or directly use [pre-compiled binary](https://github.com/nvdla/sw/tree/master/prebuilt/x86-ubuntu)

<a name="nvdla-compiler-help"></a>
### Help

    Usage: ./nvdla_compiler [options] --prototxt <prototxt_file> --caffemodel <caffemodel_file>
    where options include:
    -h                                              print this help message
    -o <outputpath>                                 outputs wisdom files in 'outputpath' directory
    --profile <basic|default|performance|fast-math> computation profile (default: fast-math)
    --cprecision <fp16|int8>                        compute precision (default: fp16)
    --configtarget <nv_full|nv_large|nv_small>      target platform (default: nv_full)
    --calibtable <int8 calibration table>           calibration table for INT8 networks (default: 0.00787)
    --quantizationMode <per-kernel|per-filter>      quantization mode for INT8 (default: per-kernel)
    --batch                                         batch size (default: 1)
    --informat <ncxhwx|nchw|nhwc>                   input data format (default: nhwc)

<a name="nvdla-compiler-example"></a>
### Example compiling ResNet-50 for nv_small

    ./nvdla_compiler --prototxt ResNet-50-deploy.prototxt --caffemodel ResNet-50-model.caffemodel -o . --profile fast-math --cprecision int8 --configtarget nv_small --calibtable resnet50.json --quantizationMode per-filter --batch 1 --informat nhwc

<a name="nvdla-compiler-output"></a>
### Output

Once the compilation is successful, it will generate <profile-name>.nvdla file in output directort specified using -o argument. For example, in above case it will generate fast-math.nvdla in curren directory.

<a name="nvdla-compiler-update"></a>
### Modifying NVDLA Compiler

NVDLA Compiler can be updated using [source code](https://github.com/nvdla/sw/tree/master/umd/core/src/compiler) and rebuild as below

    export TOP={sw-repo-root}/umd
    make compiler
    
    Note :
    In some cases if compiler build fails because of linking error with protobuf library then rebuild protobuf library as below
    ./configure --enable-shared
    make
    make check
    sudo make install

## NVDLA Runtime

NVDLA compiler is used to run inference on DLA platform using loadable generated from NVDLA compiler. NVDLA runtime can be built from [source code](https://github.com/nvdla/sw/tree/master/umd/core/src/runtime) or directly use pre-compiled binary [arm64](https://github.com/nvdla/sw/tree/master/prebuilt/arm64-linux) or [risc-v](https://github.com/nvdla/sw/tree/master/prebuilt/riscv-linux)

<a name="nvdla-runtime-help"></a>
### Help

    Usage: ./nvdla_runtime [-options] --loadable <loadable_file>
    where options include:
    -h                    print this help message
    -s                    launch test in server mode
    --image <file>        input jpg/pgm file
    --normalize <value>   normalize value for input image
    --mean <value>        comma separated mean value for input image
    --rawdump             dump raw dimg data

<a name="nvdla-runtime-example"></a>
### Example running ResNet-50 on nv_small

    ./nvdla_runtime --loadable fast-math.nvdla --image 0000.jpg --rawdump

<a name="nvdla-runtime-update"></a>
### Modifying NVDLA Runtime

NVDLA Runtime can be updated using [source code](https://github.com/nvdla/sw/tree/master/umd/core/src/runtime) and rebuild as below

    export TOP={sw-repo-root}/umd
    make TOOLCHAIN_PREFIX=<path_to_toolchanin> runtime

For example:

    ARM64
    export TOP={sw-repo-root}/umd
    make TOOLCHAIN_PREFIX={buildroot-root}/output/host/bin/aarch64-linux-gnu- runtime
    
    RISC-V
    export TOP={sw-repo-root}/umd
    make TOOLCHAIN_PREFIX={firesim-nvdla-repo}/riscv-tools-install/bin/riscv64-unknown-linux-gnu- runtime

Note:

ARM64 build is dependent on [buildroot](#buildroot) installation.

RISC-V build is dependent on [RISC-V tools](#firesim) installation

## NVDLA Kernel Driver

### ARM64

NVDLA Kernel Driver for ARM64 virtual platform is loaded as a kernel module. It's source code is at https://github.com/nvdla/sw/tree/master/kmd and pre-built binary is at https://github.com/nvdla/sw/blob/master/prebuilt/arm64-linux/

Register mappings for nv_small/nv_large and nv_full configurations are different and hence pre-built includes two binaries:
* opendla_1.ko : for nv_full
* opendla_2.ko : for nv_large and nv_small

#### Updating driver

Note:
```
define DLA_2_CONFIG if you want to build driver for nv_small or nv_large configuration otherwise keep it undefined
```

```
make KDIR={buildroot-root}/output/build/linux-4.13.3 ARCH=arm64 CROSS_COMPILE={buildroot-root}/output/host/bin/aarch64-linux-gnu-
```

Refer to [buildroot](#buildroot) for Linux kernel and toolchain

### RISC-V

Currently only FireSim is available as an RISC-V platform. NVDLA Kernel Driver is integrated as part of Linux kernel and present at https://github.com/nvdla/riscv-linux/tree/firesim-nvdla/drivers/nvdla riscv-linux repo is present as sub-module in https://github.com/nvdla/firesim-nvdla and not required to clone separately. It will get cloned and built as part of [FireSim](#firesim) setup.

#### Updating driver

If you want to update NVDLA kernel driver then update code at {firesim-nvdla-repo-root}/sw/firesim-software/risc-linux/drivers/nvdla and run below commands to build and install driver from {firesim-nvdla-repo-root}/sw/firesim-software/

```
./marshal -v build workloads/nvdla.json
./marshal install workloads/nvdla.json
```

## NVDLA Platforms

Below platforms are available for NVDLA development and verification
1. [Virtual Platform](http://nvdla.org/vp.html)
2. [Virtual Platform on AWS FPGA](http://nvdla.org/vp_fpga.html)
3. [FireSim](https://github.com/nvdla/firesim-nvdla)

* [Virtual Platform](#virtual-platform)
    * [Using pre-built virtual simulator in docker](#prebuilt-docker)
    * [Build virtual simulator in docker](#build-vp)
        * [nv_full](#nv_full)
        * [nv_large](#nv_large)
        * [nv_small](#nv_small)
* [Virtual Platform on AWS FPGA](#virtual-platform-on-aws-fpga)
* [FireSim](#firesim)

### Virtual Platform

There are two options to use virtual platform
1. Pre-built virtual simulator in docker
2. Build and install virtual simulator

<a name="prebuilt-docker"></a>
#### 1. Using pre-built virtual simulator in docker

This is easy step to start getting introduced to NVDLA. [Docker container](https://hub.docker.com/r/nvdla/vp) includes pre-built CMOD and software for *nv_full* configuration along with it all the system requirements to build simulator and virtual platform for difference NVDLA configuration.

Steps to use it

##### Install docker
https://docs.docker.com/engine/installation

##### Start the container
docker pull nvdla/vp # pull the docker image docker run -it -v /home:/home nvdla/vp # create and start the container

##### Start NVDLA virtual simulator
Inside the container:
cd /usr/local/nvdla
aarch64_toplevel -c aarch64_nvdla.lua # start the virtual simulator # Login the kernel with account 'root' and password 'nvdla'

##### Install NVDLA demo kernel driver
After login the kernel: mount -t 9p -o trans=virtio r /mnt 
cd /mnt
insmod drm.ko # install drm driver
insmod opendla_1.ko # install nvdla driver

After this follow guideline for NVDLA [compiler](#nvdla-compiler) and [runtime](#nvdla-runtime) to run inference on NVDLA

##### Exit NVDLA virtual simulator
ctrl+a x

<a name="build-vp"></a>
#### 2. Build virtual simulator in docker

Docker container has pre-installed all system requirements to build virtual simulator. If not using docker container then refer to [installing system requirements](installing-system-requirements).

#### nv_full

```
git clone https://github.com/nvdla/hw.git 
cd hw
git checkout origin/nvdla1
```

```
make
```

Options to select for nv_full configuration
```
Enter project names      (Press ENTER to use: nv_full):nv_full
Enter c pre-processor path (Press ENTER to use: /home/utils/gcc-4.9.3/bin/cpp):/usr/bin/cpp
Enter g++ path           (Press ENTER to use: /home/utils/gcc-4.9.3/bin/g++):/usr/bin/g++
Enter perl path          (Press ENTER to use: /home/utils/perl-5.8.8/bin/perl):/usr/bin/perl
Enter java path          (Press ENTER to use: /home/utils/java/jdk1.8.0_131/bin/java):/usr/bin/java
Enter systemc path       (Press ENTER to use: /usr/local/systemc-2.3.0/):
OPTIONAL: Enter verilator path (Press ENTER to use: verilator):
OPTIONAL: Enter clang path     (Press ENTER to use: clang):
```

```
 tools/bin/tmake -build cmod_top
 ```

Download and Build VP

```
git clone https://github.com/nvdla/vp.git
cd vp
git submodule update --init --recursive
```

```
cmake -DCMAKE_INSTALL_PREFIX=[install dir] -DSYSTEMC_PREFIX=[systemc prefix] -DNVDLA_HW_PREFIX=[nvdla_hw prefix] -DNVDLA_HW_PROJECT=[nvdla_hw project name]

For example:
cmake -DCMAKE_INSTALL_PREFIX=build -DSYSTEMC_PREFIX=/usr/local/systemc-2.3.0/ -DNVDLA_HW_PREFIX=/odla/vpr/nv_full -DNVDLA_HW_PROJECT=nv_full
```
```
make
make install
```

#### nv_large

```
git clone https://github.com/nvdla/hw.git 
cd hw
git checkout origin/master
```

```
make
```

Options to select for nv_large configuration
```
Enter project names      (Press ENTER if use: nv_small nv_small_256 nv_small_256_full nv_medium_512 nv_medium_1024_full nv_large):nv_large
Using designware or not [1 for use/0 for not use] (Press ENTER if use: 1):
Enter design ware path (Press ENTER if use: /home/tools/synopsys/syn_2011.09/dw/sim_ver):
Enter c pre-processor path (Press ENTER if use: /home/utils/gcc-4.8.2/bin/cpp):/usr/bin/cpp
Enter gcc path             (Press ENTER if use: /home/utils/gcc-4.8.2/bin/gcc):/usr/bin/gcc
Enter g++ path             (Press ENTER if use: /home/utils/gcc-4.8.2/bin/g++):/usr/bin/g++
Enter perl path            (Press ENTER if use: /home/utils/perl-5.10/5.10.0-threads-64/bin/perl):/usr/bin/perl
Enter java path            (Press ENTER if use: /home/utils/java/jdk1.8.0_131/bin/java):/usr/bin/java
Enter systemc path         (Press ENTER if use: /home/ip/shared/inf/SystemC/1.0/20151112/systemc-2.3.0/GCC472_64_DBG):/usr/local/systemc-2.3.0
Enter python path          (Press ENTER if use: /home/tools/continuum/Anaconda3-5.0.1/bin/python):/usr/bin/python
Enter vcs_home path        (Press ENTER if use: /home/tools/vcs/mx-2016.06-SP2-4):
Enter novas_home path      (Press ENTER if use: /home/tools/debussy/verdi3_2016.06-SP2-9):
Enter verdi_home path      (Press ENTER if use: /home/tools/debussy/verdi3_2016.06-SP2-9):
OPTIONAL: Enter verilator path (Press ENTER to use: verilator):
OPTIONAL: Enter clang path     (Press ENTER to use: /home/utils/llvm-4.0.1/bin/clang):
```

```
 tools/bin/tmake -build cmod_top
```

Download and Build VP

```
git clone https://github.com/nvdla/vp.git
cd vp
git submodule update --init --recursive
```

```
cmake -DCMAKE_INSTALL_PREFIX=[install dir] -DSYSTEMC_PREFIX=[systemc prefix] -DNVDLA_HW_PREFIX=[nvdla_hw prefix] -DNVDLA_HW_PROJECT=[nvdla_hw project name]

For example:
cmake -DCMAKE_INSTALL_PREFIX=build -DSYSTEMC_PREFIX=/usr/local/systemc-2.3.0/ -DNVDLA_HW_PREFIX=/odla/vpr/nv_large -DNVDLA_HW_PROJECT=nv_large
```
```
make
make install
```

##### nv_small

```
git clone https://github.com/nvdla/hw.git 
cd hw
git checkout origin/master
```

```
make
```

Options to select for nv_small configuration
```
Enter project names      (Press ENTER if use: nv_small nv_small_256 nv_small_256_full nv_medium_512 nv_medium_1024_full nv_large):nv_small
Using designware or not [1 for use/0 for not use] (Press ENTER if use: 1):
Enter design ware path (Press ENTER if use: /home/tools/synopsys/syn_2011.09/dw/sim_ver):
Enter c pre-processor path (Press ENTER if use: /home/utils/gcc-4.8.2/bin/cpp):/usr/bin/cpp
Enter gcc path             (Press ENTER if use: /home/utils/gcc-4.8.2/bin/gcc):/usr/bin/gcc
Enter g++ path             (Press ENTER if use: /home/utils/gcc-4.8.2/bin/g++):/usr/bin/g++
Enter perl path            (Press ENTER if use: /home/utils/perl-5.10/5.10.0-threads-64/bin/perl):/usr/bin/perl
Enter java path            (Press ENTER if use: /home/utils/java/jdk1.8.0_131/bin/java):/usr/bin/java
Enter systemc path         (Press ENTER if use: /home/ip/shared/inf/SystemC/1.0/20151112/systemc-2.3.0/GCC472_64_DBG):/usr/local/systemc-2.3.0
Enter python path          (Press ENTER if use: /home/tools/continuum/Anaconda3-5.0.1/bin/python):/usr/bin/python
Enter vcs_home path        (Press ENTER if use: /home/tools/vcs/mx-2016.06-SP2-4):
Enter novas_home path      (Press ENTER if use: /home/tools/debussy/verdi3_2016.06-SP2-9):
Enter verdi_home path      (Press ENTER if use: /home/tools/debussy/verdi3_2016.06-SP2-9):
OPTIONAL: Enter verilator path (Press ENTER to use: verilator):
OPTIONAL: Enter clang path     (Press ENTER to use: /home/utils/llvm-4.0.1/bin/clang):
```

```
 tools/bin/tmake -build cmod_top
 ```

Download and Build VP

```
git clone https://github.com/nvdla/vp.git
cd vp
git submodule update --init --recursive
```

```
cmake -DCMAKE_INSTALL_PREFIX=[install dir] -DSYSTEMC_PREFIX=[systemc prefix] -DNVDLA_HW_PREFIX=[nvdla_hw prefix] -DNVDLA_HW_PROJECT=[nvdla_hw project name]

For example:
cmake -DCMAKE_INSTALL_PREFIX=build -DSYSTEMC_PREFIX=/usr/local/systemc-2.3.0/ -DNVDLA_HW_PREFIX=/odla/vpr/nv_small -DNVDLA_HW_PROJECT=nv_small
```

```
make
make install
```

<a name="system-requirements"></a>
## System requirements for Virtual Platform

### Install tools and libraries

```
sudo apt-get update
sudo apt-get install g++ cmake libboost-dev python-dev libglib2.0-dev libpixman-1-dev liblua5.2-dev swig libcap-dev libattr1-dev default-jdk

Steps required if using Ubuntu higher than 14.04

sudo apt-get install python-software-properties
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt-get update
sudo apt-get install gcc-4.8
sudo apt-get install g++-4.8
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8 50
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.8 50
```

### Install SystemC 2.3.0 (Note: SystemC 2.3.1/2.3.2 not supported)

```
wget -O systemc-2.3.0a.tar.gz http://www.accellera.org/images/downloads/standards/systemc/systemc-2.3.0a.tar.gz 
tar -xzvf systemc-2.3.0a.tar.gz
cd systemc-2.3.0a
sudo mkdir -p /usr/local/systemc-2.3.0/
mkdir objdir
cd objdir
../configure --prefix=/usr/local/systemc-2.3.0
make
sudo make install
```

### Install Perl package

```
wget -O YAML-1.24.tar.gz http://search.cpan.org/CPAN/authors/id/T/TI/TINITA/YAML-1.24.tar.gz 
tar -xzvf YAML-1.24.tar.gz 
cd YAML-1.24
perl Makefile.PL
make
sudo make install
wget -O IO-Tee-0.65.tar.gz http://search.cpan.org/CPAN/authors/id/N/NE/NEILB/IO-Tee-0.65.tar.gz 
tar -xzvf IO-Tee-0.65.tar.gz
cd IO-Tee-0.65
perl Makefile.PL
make
sudo make install
cpan -i Capture::Tiny [Note: Fix nvdla.org for it]
cpan -i XML::Simple [Note: Fix nvdla.org for it]
```

## Buildroot

```
git clone https://github.com/nvdla/buildroot
```

```
make qemu_aarch64_virt_defconfig
make menuconfig
   * Target Options -> Target Architecture -> AArch64 (little endian)
	* Target Options -> Target Architecture Variant -> cortex-A57
	* Toolchain -> Custom kernel headers series -> 4.13.x
	* Toolchain -> Toolchain type -> External toolchain
	* Toolchain -> Toolchain -> Linaro AArch64 2017.08
	* Toolchain -> Toolchain origin -> Toolchain to be downloaded and installed
	* Toolchain -> Copy gdb server to the Target
	* Kernel -> () Kernel version -> 4.13.3
	* Kernel -> Kernel configuration -> Use the architecture default configuration
	* System configuration -> Enable root login with password -> Y
	* System configuration -> Root password -> nvdla
	* Target Packages -> Show packages that are also provided by busybox -> Y
	* Target Packages -> Networking applications -> openssh -> Y
	* Target Packages -> Debugging, profiling and benchmark -> gdb -> Y
   * Target Packages -> Debugging, profiling and benchmark -> full debugger -> Y
make -j4
```

### Files to use from build

```
{buildroot-root}/output/images/Image
{buildroot-root}/output/images/rootfs.ext4
{buildroot-root}/output/build/linux-4.13.3/drivers/gpu/drm/drm.ko
```

### Toolchain

Toolchain is downloaded at below location which can be used to build NVDLA kernel driver and NVDLA runtime for ARM64

```
{buildroot-root}/output/host/bin/aarch64-linux-gnu-
```

### Linux kernel 4.13.3

Linux kernel 4.13.3 is downloaded at below location which can be used to build NVDLA kernel driver for ARM64

```
{buildroot-root}/output/build/linux-4.13.3
```

## FireSim


## Run Test Application

This section explains how to run test application on different platforms and dependencies for it. First dependency to run test application is loadable generated from NVDLA compiler. Refer to [NVDLA Compiler](#nvdla-compiler) for more details to generate loadable for a network.

ResNet-50 model from https://github.com/KaimingHe/deep-residual-networks is verified on this platform for all configurations (nv_full/nv_large/nv_small) and it can be used to start with.

<a name="test-app-on-vp"></a>
### Virtual platform on docker container

This section explains how to run test application on docker container which has pre-built binaries for nv_full configuration.

```
docker pull nvdla/vp
docker run -it -v /home:/home nvdla/vp
cd /usr/local/nvdla
export SC_SIGNAL_WRITE_CHECK=DISABLE
aarch64_toplevel -c aarch64_nvdla.lua
mount -t 9p -o trans=virtio r /mnt
cd /mnt
insmod drm.ko
insmod opendla_1.ko
```

Expected output after installing NVDLA driver
```
[  310.625140] opendla: loading out-of-tree module taints kernel.
[  310.629362] 0 . 12 . 5
[  310.629567] reset engine done
[  310.633122] [drm] Initialized nvdla 0.0.0 20171017 for 10200000.nvdla on minor 0
```

Run test application
```
./nvdla_runtime --loadable fast-math.nvdla --image 0000.jpg
```
```
fast-math.nvdla : loadable generated from NVDLA compiler
0000.jpg : 224x224 image for ResNet-50 model
```

