# Getting started with NVDLA

NVDLA enables accelerating neural network inference job which is achieved in two steps
1. Optimize trained neural network for DLA hardware and convert the graph to DLA HW instructions. This converted graph is saved to a flatbuffer file called as loadable. This is achieved using NVDLA compiler and performed offline on host system.
2. Run inference job on DLA using loadable from step 1. This is achieved using NVDLA runtime and performed on target system.

## NVDLA Compiler

NVDLA compiler is used to optimize neural network for DLA HW architecture and create list of HW instructions to run inference on DLA.  NVDLA compiler can be built from [source code](https://github.com/nvdla/sw/tree/master/umd/core/src/compiler) or directly use [pre-compiled binary](https://github.com/nvdla/sw/tree/master/prebuilt/x86-ubuntu)

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

### Example compiling ResNet-50 for nv_small

    ./nvdla_compiler --prototxt ResNet-50-deploy.prototxt --caffemodel ResNet-50-model.caffemodel -o . --profile fast-math --cprecision int8 --configtarget nv_small --calibtable resnet50.json --quantizationMode per-filter --batch 1 --informat nhwc

### Output

Once the compilation is successful, it will generate <profile-name>.nvdla file in output directort specified using -o argument. For example, in above case it will generate fast-math.nvdla in curren directory.

### Modifying NVDLA Compiler

NVDLA Compiler can be updated using [source code](https://github.com/nvdla/sw/tree/master/umd/core/src/compiler) and rebuild as below

    export TOP={sw-repo-root}/umd
    make compiler

## NVDLA Runtime

NVDLA compiler is used to run inference on DLA platform using loadable generated from NVDLA compiler. NVDLA runtime can be built from [source code](https://github.com/nvdla/sw/tree/master/umd/core/src/runtime) or directly use pre-compiled binary [arm64](https://github.com/nvdla/sw/tree/master/prebuilt/arm64-linux) or [risc-v](https://github.com/nvdla/sw/tree/master/prebuilt/riscv-linux)

### Help

    Usage: ./nvdla_runtime [-options] --loadable <loadable_file>
    where options include:
    -h                    print this help message
    -s                    launch test in server mode
    --image <file>        input jpg/pgm file
    --normalize <value>   normalize value for input image
    --mean <value>        comma separated mean value for input image
    --rawdump             dump raw dimg data

### Example running ResNet-50 on nv_small

    ./nvdla_runtime --loadable fast-math.nvdla --image 0000.jpg --rawdump

### Modifying NVDLA Compiler

NVDLA Runtime can be updated using [source code](https://github.com/nvdla/sw/tree/master/umd/core/src/runtime) and rebuild as below

    export TOP={sw-repo-root}/umd
    make TOOLCHAIN_PREFIX=<path_to_toolchanin> runtime

#### For example

    ARM64
    export TOP={sw-repo-root}/umd
    make TOOLCHAIN_PREFIX={buildroot-root}/output/host/bin/aarch64-linux-gnu- runtime
    
    RISC-V
    export TOP={sw-repo-root}/umd
    make TOOLCHAIN_PREFIX={firesim-nvdla-repo}/riscv-tools-install/bin/riscv64-unknown-linux-gnu- runtime

See for buildroot instructions and for riscv-tools installation instructions


## NVDLA Platforms

Below platforms are available for NVDLA development and verification
1. [Virtual Platform](http://nvdla.org/vp.html)
2. [Virtual Platform on AWS FPGA](http://nvdla.org/vp_fpga.html)
3. [FireSim](https://github.com/nvdla/firesim-nvdla)

### Virtual Platform

There are two options to use virtual platform
1. Pre-built virtual simulator in docker
2. Build and install virtual simulator

#### Using pre-built virtual simulator in docker

This is easy step to start getting introduced to NVDLA. [Docker container](https://hub.docker.com/r/nvdla/vp) includes pre-built CMOD and software for nv_full configuration along with it all the system requirements to build simulator and virtual platform for difference NVDLA configuration.

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

After this follow guideline for NVDLA [compiler](#nvdla-ompiler) and [runtime](#nvdla-runtime) to run inference on NVDLA

##### Exit NVDLA virtual simulator
ctrl+a x
