# onert-micro

`onert-micro` - Ultra Light AI Runtime for inference/training of Neural Networks.

## ✨ Benefits

- No need network connectivity (on-device)
- Local computations
- Help to preserve user privacy
- Lower latency
- Increasing reliability
- Increasing smart device functionality including **MCU**

## ⚡Key Features

**Supported OS:**

- TizenRT
- Mbed OS
- Ubuntu

**Supported Architectures:**

- X86-64
- ARM Cortex-R (ARMv7-R)
- ARM Cortex-M (ARMv7E-M,ARMv8-M)
- ARM64

**On-device training for MCU**

**Quantized kernels support**

**CMSIS-NN support**


**onert-micro contains cmake infrastructure to build:**
- `onert_micro_interpreter` - Main interpreter library for on-device inference
- `onert_micro_training_interpreter` - Interpreter library with on-device training feature
- `onert_micro_eval_driver` - Evaluation driver tool for running inference
- `onert_micro_training_eval_driver` - Training evaluation driver
- `train_config_tool` - Training configuration tool
- previuos version based on luci-interpreter(`./luci-interpreter`)

## 🛠️ How to build stand alone library

Stand-alone library is simply built by `onert_micro_arm` target.
Result library will be placed in  `./build/standalone_arm/onert-micro/src/libonert_micro_interpreter.a`.

### Prerequisites

- arm-none-eabi-gcc and arm-none-eabi-g++ compilers

To install needed arm compilers on ubuntu:
```
$ sudo apt-get install gcc-arm-none-eabi
```

**cmake build**

``` bash
sudo apt-get install cmake gcc g++
```
``` bash
$ sudo apt-get install \
build-essential \
cmake \
git \
libboost-all-dev \
libgflags-dev \
libgoogle-glog-dev \
libatlas-base-dev \
libhdf5-dev \
libprotobuf-dev \
protobuf-compiler \
wget \
zip \
unzip \
python3 \
python3-pip \
python3-venv \
python3-dev \
hdf5-tools \
curl
```

``` bash
$ cd <path to onert-micro>
$ mkdir build
# cd build
$ cmake ..
$ make -j$(nproc) onert_micro_arm
```

## 🚀 How to use

### Convert tflite model to circle model

For preparation of `circle` file you need to use [ONE Toolchain](https://github.com/Samsung/ONE)
Everything you need for ONE project: see [how-to-build-compiler.md](https://github.com/Samsung/ONE/blob/master/docs/howto/how-to-build-compiler.md)
To inference with tflite model, you need to convert it to circle model format(../res/CircleSchema/0.8/circle_schema.fbs).
Please refer to `tflite2circle` tool ([ONE/compiler/tflite2circle](https://github.com/Samsung/ONE/tree/master/compiler/tflite2circle)) for this purpose.

### Convert to c array model

Many MCU platforms are lack of file system support. The proper way to provide a model to onert-micro is to convert it into c array so that it can be compiled into MCU binary.

``` bash
xxi -i model.circle > model.h
```

Then, model.h looks like this:

``` cpp
unsigned char model_circle[] = {
  0x22, 0x01, 0x00, 0x00, 0xf0, 0x00, 0x0e, 0x00,
  // .....
};
unsigned int model_circle_len = 1004;
```

### API

Working example of API usage can be found by the link: [Example](eval-driver/Driver.cpp)

Once you have c array model, you are ready to use onert-micro.

To run a model with onert-micro, follow the instruction:

1. Include onert-micro header

``` cpp
  #include "OMInterpreter.h"
```

2. Create interpreter instance

onert-micro interpreter expects model as c array as mentioned in [Previous Section](#convert-to-c-array-model).

``` cpp
  #include "model.h"

  luci_interpreter::Interpreter interpreter(model_circle, true);
  // Create interpreter.
  onert_micro::OMInterpreter interpreter;
  onert_micro::OMConfig config;
  interpreter.importModel(model_circle.data(), config);

```

3. Feed input data

To feed input data into interpreter, we need to do two steps: 1) allocate input tensors and 2) copy input into input tensors.

``` cpp
    interpreter.reset();
    interpreter.allocateInputs();

    for (int32_t i = 0; i < num_inputs; i++)
    {
      auto input_data = reinterpret_cast<char *>(interpreter.getInputDataAt(i));
      size_t input_size = interpreter.getInputSizeAt(i);
      readDataFromFile(input_prefix + std::to_string(i), input_data, input_size);
    }
```

4. Do inference

``` cpp
    // Do inference.
    interpreter.run(config);
```

5. Get output data

``` cpp
    auto data = interpreter.getOutputDataAt(i);
```


### Reduce Binary Size

onert-micro provides compile flags to generate reduced-size binary.

- `DIS_QUANT` : Flag for Disabling Quantized Type Operation
- `DIS_FLOAT` : Flag for Disabling Float Operation
- `DIS_DYN_SHAPES` : Flag for Disabling Dynamic Shape Support

Also, you can build onert-micro library only with kernels in target models.
For this, please remove all the kernels from [KernelsToBuild.lst](./onert-micro/include/pal/mcu/KernelsToBuild.lst) except kernels in your target model.

