# Getting started with NVDLA

NVDLA enables accelerating neural network inference job which is achieved in two steps
1. Optimize trained neural network for DLA hardware and convert the graph to DLA HW instructions. This converted graph is saved to a flatbuffer file called as loadable. This is achieved using NVDLA compiler and performed offline on host system.
2. Run inference job on DLA using loadable from step 1. This is achieved using NVDLA runtime and performed on target system.

## NVDLA Compiler

NVDLA compiler is used to optimize neural network for DLA HW architecture and create list of HW instructions to run inference on DLA.  NVDLA compiler can be built from [source code](https://github.com/nvdla/sw/tree/master/umd/core/src/compiler) or directly use [pre-compiled binary](https://github.com/nvdla/sw/tree/master/prebuilt/x86-ubuntu)

### Compiling network using NVDLA Compiler

#### Help

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

#### Example compiling ResNet-50 for nv_small

    ./nvdla_compiler --prototxt ResNet-50-deploy.prototxt --caffemodel ResNet-50-model.caffemodel -o . --profile fast-math --cprecision int8 --configtarget nv_small --calibtable resnet50.json --quantizationMode per-filter --batch 1 --informat nhwc

#### Output

Once the compilation is successful, it will generate <profile-name>.nvdla file in output directort specified using -o argument. For example, in above case it will generate fast-math.nvdla in curren directory.

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

# Low Precision inference using NVDLA

NVDLA supports inference using int8 precision. Since low precision is not used for training, it requires to quantize trained models from higher precision such as fp32 to int8 for inference.

## Calibrator

This caffe based calibrator tool is used to calculate int8 scales for low precision inference using DLA. Scales are calculated for int8 symmetric quantization in which 0 is mapped to 0 in target range and min = -max.

## Tuning

Calibrator output can be tuned using combination of below 2 parameters

    bins : number of bins decide how is the values from activation tensors are distributed in bins, if numbers of bins is very small then values might causing dense bins while more number of bins causing sparse bins. Hence it is important to select right number of bins
    threshold : this is percentile of data to consider for calculating scales, it ignores all bins above this percentile threshold

## Usage

    ./caffe test -model <deploy_prototxt> -weights <caffe_model> -iterations <n> -calib 1 -histdir <path_to_histogram_dir> -bins <n> -threshold <0.n>

-calib : enable or disable calibration in test phase, default is disabled

-bins : number of bins used for histogram in calibration, default is 2048

-threshold : threshold for selecting dynamic range in calibration, default is 1

-histdir : path to histogram directory to store histogram and calibration output

-iterations : number of iterations to run, default is 50

## Dependencies

    apt-get install libprotobuf-dev libleveldb-dev libsnappy-dev libopencv-dev libhdf5-serial-dev protobuf-compiler
    apt-get install --no-install-recommends libboost-all-dev
    apt-get install libgflags-dev libgoogle-glog-dev liblmdb-dev
    apt-get install libopenblas-dev

## Output

Calibrator dumps histogram data for each layer which can be used for analysis and int8.json file containing scales for each layer which is input to compiler.

int8.json sample
{
  "data" : {
  "scale": 0.868586,
  "min": 0,
  "max": 110.31,
  "offset": 0
  }

}

## Example

    ./caffe test -model <path>/ResNet-50-deploy.prototxt -iterations 3 -weights <path>/ResNet-50-model.caffemodel -calib 1 -histdir <path>/histogram/ -bins 1024 -threshold 0.9
