# Getting started with NVDLA

NVDLA enables accelerating neural network inference job which is achieved in two steps
1. Optimize trained neural network for DLA hardware and convert the graph to DLA HW instructions. This converted graph is saved to a flatbuffer file called as loadable. This is achieved using NVDLA compiler and performed offline on host system.
2. Run inference job on DLA using loadable from step 1. This is achieved using NVDLA runtime and performed on target system.

## NVDLA Compiler

NVDLA compiler is used to optimize neural network for DLA HW architecture and create list of HW instructions to run inference on DLA.  NVDLA compiler can be built from [source code](https://github.com/nvdla/sw/tree/master/umd/core/src/compiler) or directly use [pre-compiled binary](https://github.com/nvdla/sw/tree/master/prebuilt/x86-ubuntu)

### Compiling network using NVDLA Compiler

#### Help

<p align="center">
<img src="https://github.com/prasshantg/personal/blob/master/compiler_log.png">
</p>

#### Example

<p align="center">
<img src="https://github.com/prasshantg/personal/blob/master/compiler_use.png">
</p>

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
