---
sidebar_position: 2
---

# Inference Optimization pipeline

KubeFlow was the platform of choice for making a reproducible and scalable machine learning inference pipeline for the `g4fastsim` inference project. CERN IT Department has built [ml.cern.ch](https://ml.cern.ch) which is KubeFlow service accessible to all CERN members and was used for building the pipeline.

`g4fastsim` is broken into 2 parts - `Training` and `Inference`. Inference Optimization Pipeline is aimed at reducing memory footprint of the ML model by performing various types of hardware-specific quantizations and graph optimizations.

The Inference pipeline is broken down into 5 main components:
* MacroHandler
* InputChecker
* Inference
* Benchmark
* Optimization

Below image represents an overview of the approach taken to build inference optimization kubeflow pipeline:

![](/img/ML_Deployment/Kubeflow_Inference_Optimization_Pipeline/InfOptim-Workflow.jpg)

## KubeFlow Pipeline

The pipeline can be found on [ml.cern.ch](https://ml.cern.ch) under the name of `Geant4-Model-Optimization-Pipeline`. 
![](/img/ML_Deployment/Kubeflow_Inference_Optimization_Pipeline/Complete-pipeline.svg)

-   `Model Loader` - Model Loader component that acts as a central reppository for non-optimized model. The model gets downloaded and stored here.
-   `MacroHandlers` - Macro Handler components which output a macro file that gets passed to the respective `Par04`s.
-   `Inference` - `Par04`s which use Full Simulation and `ONNXRuntime` CPU and CUDA Execution providers for inference.
-   `Benchmark` - Benchmark component that takes in 3 .root files as input and generates comparison plots through them.
-   `Optimization` - Optimization component that takes in non-optimized model and optimizes it.

> :green_book: A memory arena is a large, contiguous piece of memory that you allocate once and then use to manage memory manually by handing out parts of that memory. To understand `arena` in relation to memory, check out this [stack overflow post](https://stackoverflow.com/questions/12825148/what-is-the-meaning-of-the-term-arena-in-relation-to-memory)

### Parameters
* `fullSimDetectionConstructionMacUrl` - Url of FullSim .mac file for detector construction
* `fastSimDetectionConstructionMacUrl` - Url of FastSim .mac file for detector construction
* `fullSimInferenceJsonlUrl` - Url of FullSim Jsonl file which generates macro file (.mac) for full sim inference
* `fastSimInferenceJsonlUrl` - Url of FastSim Jsonl file which generates macro file (.mac) for fastsim inference
* `particleEnergy` - Energy of the particle
* `particleAngle` - Angle of the particle.
* `setSizeLatentVector` - Dimension of the latent vector (encoded vector in a Variational Autoencoder model)
* `setSizeConditionVector` - Size of the condition vector (energy, angle and geometry)
* `setModelPathName` - Path of the `ONNX` model to be used for Fast Sim
* `setProfileFlag` - Whether to perform profiling during inference using `ONNXRuntime`.
* `setOptimizationFlag` - Whether to perform graph optimization on the model when using `ONNXRuntime`.
* `setDnnlFlag` - Whether to use `oneDNN` Execution Provider as compute backend when using `ONNXRuntime`.
* `setOpenVINOFlag` - Whether to use `OpenVINO` Execution Provider as compute backend when using `ONNXRuntime`.
* `setCudaFlag` - Whether to use `CUDA` Execution Provider as compute backend when using `ONNXRuntime`.
* `setTensorrtFlag` - Whether to use `TensorRT` Execution Provider as compute backend when using `ONNXRuntime`.
* `setInferenceLibrary` - Whether to use `ONNXRuntime` or `LWTNN` for inference in FastSim
* `setFileName` - Name of the the output `.root` file.
* `beamOn` - Number of events to run
* `yLogScale` - Whether to set a log scale on the y-axis of full and fast simulation comparison plots.
* `saveFig` - Whether to save plots in `Benchmark` component
* `eosDirectory` - Path of the EOS directory where output plots will be saved.

#### oneDNN
* `setDnnlEnableCpuMemArena` - Whether to enable memory arena when using `oneDNN` Execution Provider.

#### CUDA
* `setCudaDeviceId` - ID of the device on which to run `CUDA` Execution Provider.
* `setCudaGpuMemLimit` - GPU Memory Limit when using `CUDA` Execution Provider.
* `setCudaArenaExtendedStrategy` - Strategy to use for extending memory arena. For more details, go [here](https://onnxruntime.ai/docs/execution-providers/CUDA-ExecutionProvider.html#arena_extend_strategy)
* `setCudaCudnnConvAlgoSearch` - Type of search done for finding which `cuDNN` convolution algorithm to use. For more details, go [here](https://onnxruntime.ai/docs/execution-providers/CUDA-ExecutionProvider.html#cudnn_conv_algo_search)
* `setCudaDoCopyInDefaultStream` - Whether to perform data copying operation from host to device and viceversa in default stream or separate streams. For more details, check [here](https://onnxruntime.ai/docs/execution-providers/CUDA-ExecutionProvider.html#do_copy_in_default_stream)
* `setCudaCudnnConvUseMaxWorkspace` - Amount of memory to use when querying `cuDNN` to find the most optimal convolution algorithm. Lower value will result in sub-optimal querying and higher value will lead to higher peak memory usage. For more details, chech [here](https://onnxruntime.ai/docs/execution-providers/CUDA-ExecutionProvider.html#cudnn_conv_use_max_workspace).

#### TensorRT
* `setTrtDeviceId` - ID of device on which to run `TensorRT` Execution Provider
* `setTrtMaxWorksapceSize` - Maximum memory usage of TensorRT Engine.
* `setTrtMaxPartitionIterations` - Maximum no.of iterations allowed while partitioning the model for TensorRT
* `setTrtMinSubgraphSize` - Minimum node size in a subgraph after partitioning.
* `setTrtFp16Enable` - Whether to enable FP16 computation in TensorRT
* `setTrtInt8Enable` - Whether to enable INT8 computation in TensorRT
* `setTrtInt8UseNativeCalibrationTable` - Select what calibration table is used for non-QDQ (Quantize- DeQuantize) models in INT8 mode. If 1, native TensorRT generated calibration table is used; if 0, ONNXRUNTIME tool generated calibration table is used.
* `setTrtEngineCacheEnable` - Enable TensorRT engine caching. The purpose of using engine caching is to save engine build time in the case that TensorRT may take long time to optimize and build engine.
> :warning: Each engine is created for specific settings such as model path/name, precision (FP32/FP16/INT8 etc), workspace, profiles etc, and specific GPUs and it’s not portable, so it’s essential to make sure those settings are not changing, otherwise the engine needs to be rebuilt and cached again.
> 
> :warning: Please clean up any old engine and profile cache files (.engine and .profile) if any of the following changes:
> * Model changes (if there are any changes to the model topology, opset version, operators etc.)
> * ORT version changes (i.e. moving from ORT version 1.8 to 1.9)
> * TensorRT version changes (i.e. moving from TensorRT 7.0 to 8.0)
> * Hardware changes. (Engine and profile files are not portable and optimized for specific Nvidia hardware)
* `setTrtEngineCachePath` -  Specify path for TensorRT engine and profile files if `setTrtEngineCacheEnable` is True, or path for INT8 calibration table file if `setTrtInt8Enable` is True.
* `setTrtDumpSubgraphs` - Dumps the subgraphs that are transformed into TRT engines in onnx format to the filesystem. This can help debugging subgraphs.

#### OpenVINO
* `setOVDeviceType` - Type of device to run `OpenVINO` Execution Provider on
* `setOVDeviceId` - ID of the device to run `OpenVINO` Execution Provider on
* `setOVEnableVpuFastCompile` -  During initialization of the VPU device with compiled model, Fast-compile may be optionally enabled to speeds up the model’s compilation to VPU device specific format.
* `setOVNumOfThreads` - No.of threads to use while performing computation using `OpenVINO` Execution Provider
* `setOVUseCompiledNetwork` - It can be used to directly import pre-compiled blobs if exists or dump a pre-compiled blob at the executable path. 
* `setOVEnableOpenCLThrottling` - This option enables OpenCL queue throttling for GPU devices (reduces CPU utilization when using GPU).

#### Optimization
* `enable_cpu_optimization` - Whether to enable CPU Optimization
* `cpu_optimization_json_url` - URL of JSON containing optimization configs to be used for CPU optimization. 
* `cpu_optim_model_eos_save_path` - Path in EOS where to save the CPU optimized model.
* `enable_cuda_optimization` - Whether to enable CUDA Optimization
* `cuda_optimization_json_url` - URL of JSON containing optimization configs to be used for CUDA optimization.
* `cuda_optim_model_eos_save_path` - Path in EOS where to save the CUDA optimized model.
* `enable_onednn_optimization` - Whether to enable oneDNN Optimization
* `onednn_optimization_json_url` - URL of JSON containing optimization configs to be used for oneDNN optimization.
* `onednn_optim_model_eos_save_path` - Path in EOS where to save the oneDNN optimized model.
* `enable_tensorrt_optimization` - Whether to enable TensorRT Optimization
* `tensorrt_optimization_json_url` - URL of JSON containing optimization configs to be used for TensorRT optimization.
* `tensorrt_optim_model_eos_save_path` - Path in EOS where to save the TensorRT optimized model.
## MacroHandler

`Par04` takes in input a macro file. This macro file contains a list of commands in the order they are to be run. Normally, if we want to change a macro file we will have to change the `.mac` file, upload it to gitlab, EOS or someplace from where it will be downloaded and pass the URL in KubeFlow Run UI Dashboard. This will become a tedious process.

`MacroHandler` component is implemented to solve this issue by generating a `.mac` file based on the inputs from the KubeFlow Run UI. 

The input to `MacroHandler` component is a `.jsonl` file containing `jsonlines`, where each `jsonline` contain 2 keys - `command` and `value`. An example of `.jsonl` macro file can be seen [here](https://gitlab.cern.ch/prmehta/geant4_par04/-/blob/master/pipelines/model-optimization/macros/examplePar04_onnx.jsonl).

`.jsonl` file is converted to `.mac` file after value modifications and passed to the respective `Par04` components. 
> **Eg:** As per the pipeline image above, the output of `FullSimMacroHandler` component will be passed to `FullSim` and output of `OnnxFastSimNoEPMacroHandler` component will be passed to `OnnxFastSimNoEP`.

> ***This component is specifically designed for KubeFlow usage***

### Macro file

This macro file contains Geant4 commands as well as custom commands for manipulating the simulation process.
An example of the macro file can be seen below. It uses `CUDA` backend for performing inference in `ONNXRuntime`.
```bash
#  examplePar04_onnx.mac
#
# Detector Construction 
/Par04/detector/setDetectorInnerRadius 80 cm
/Par04/detector/setDetectorLength 2 m
/Par04/detector/setNbOfLayers 90
/Par04/detector/setAbsorber 0 G4_W 1.4 mm true
/Par04/detector/setAbsorber 1 G4_Si 0.3 mm true
## 2.325 mm of tungsten =~ 0.25 * 9.327 mm = 0.25 * R_Moliere
/Par04/mesh/setSizeOfRhoCells 2.325 mm
## 2 * 1.4 mm of tungsten =~ 0.65 X_0
/Par04/mesh/setSizeOfZCells 3.4 mm
/Par04/mesh/setNbOfRhoCells 18
/Par04/mesh/setNbOfPhiCells 50
/Par04/mesh/setNbOfZCells 45

# Initialize
/run/numberOfThreads 1
/run/initialize

/gun/energy 10 GeV
/gun/position 0 0 0
/gun/direction 0 1 0

# Inference Setup
## dimension of the latent vector (encoded vector in a Variational Autoencoder model)
/Par04/inference/setSizeLatentVector 10
## size of the condition vector (energy, angle and geometry)  
/Par04/inference/setSizeConditionVector 4
## path to the model which is set to download by cmake
/Par04/inference/setModelPathName ./MLModels/Generator.onnx
/Par04/inference/setProfileFlag 1
/Par04/inference/setOptimizationFlag 0
/Par04/inference/setCudaFlag 1

# cuda options
/Par04/inference/cuda/setDeviceId 0
/Par04/inference/cuda/setGpuMemLimit 2147483648
/Par04/inference/cuda/setArenaExtendedStrategy kSameAsRequested
/Par04/inference/cuda/setCudnnConvAlgoSearch DEFAULT
/Par04/inference/cuda/setDoCopyInDefaultStream 1
/Par04/inference/cuda/setCudnnConvUseMaxWorkspace 1

/Par04/inference/setInferenceLibrary ONNX

## set mesh size for inference == mesh size of a full sim that
## was used for training; it coincides with readout mesh size
/Par04/inference/setSizeOfRhoCells 2.325 mm
/Par04/inference/setSizeOfZCells 3.4 mm
/Par04/inference/setNbOfRhoCells 18
/Par04/inference/setNbOfPhiCells 50
/Par04/inference/setNbOfZCells 45

# Fast Simulation
/analysis/setFileName 10GeV_100events_fastsim_onnx.root
## dynamically set readout mesh from particle direction
## needs to be the first fast sim model!
/param/ActivateModel defineMesh
## ML fast sim, configured with the inference setup /Par04/inference
/param/ActivateModel inferenceModel
/run/beamOn 100
```

There are additional commands which can be used to configure the settings of each execution provider. A detailed example of macro file for `ONNXRuntime` can be viewed [here](https://gitlab.cern.ch/prmehta/geant4_par04/-/blob/master/examplePar04_onnx.mac).

## InputChecker

KubeFlow passes all the parameters as string from its run UI. In order to debug type issues a small input checker component is build which will output the value of every KubeFlow UI parameter and its value type.
This an independent component of the pipeline.

> ***This component is specifically designed for KubeFlow usage***

## Inference

`Par04` is a C++ application which takes in as input an ONNX model, performs inference using it and outputs a `.root` file containing the simulated physics quantities which can be used for downstream analysis of particle showers. 

`Par04` currently supports 3 libraries - `ONNXRuntime`, `LWTNN` and `PyTorch`. The inference optimization pipeline currently being built is geared towards using `ONNXRuntime`. The execution providers that will be investigated for optimizing the ONNX model are - 
* ***Nvidia TensorRT*** - GPU
* ***Nvidia CUDA*** - GPU
* ***Intel oneDNN*** - CPU
* ***Intel OpenVINO*** - CPU
> *Support for **Intel OpenVINO** is currently on halt due to compatibility issues.*

### Output .root file

The output of `Par04` is a `.root` file which can be opened using `TSystem` (C++) and `uproot` (python). The `.root` file contains the following structure:

```text
output_file.root
├── energyDeposited;1
├── energyParticle;1
├── energyRatio;1
├── events;1
│   ├── CPUResMem
│   ├── CPUVirMem
│   ├── EnergyCell
│   ├── EnergyMC
│   ├── GPUMem
│   ├── phiCell
│   ├── rhoCell
│   ├── SimTime
│   └── zCell
├── hitType:1
├── longFirstMoment;1
├── longProfile;1
├── longSecondMoment;1
├── time;1
├── transFirstMoment;1
├── transProfile;1
└── transSecondMoment;1
```

More details on these quantities are explained in this [link](https://geant4-userdoc.web.cern.ch/UsersGuides/ForApplicationDeveloper/html/Examples/extended/Par04.html#output-data).

`CPUResMem`, `CPUVirMem`, `GPUMem` are added into the updated version of the `Par04` application:
`CPUResMem` - Resident Memory used by the inference of the `Par04` application
`CPUVirMem` - Virtual Memory used by the inference of the `Par04` application
`GPUMem` - GPU Memory used by the inference of the `Par04` application
More details on Resident and Virtual Memory can be found [here](https://stackoverflow.com/questions/7880784/what-is-rss-and-vsz-in-linux-memory-management)

### Dependencies

`Par04` has a number of optional dependencies which can be added to customise it for different host hardware and usecase. 

> Running FastSim in `Par04` will require `ONNXRuntime`.

#### Mandatory
- `Geant4` - A platform for simulation of the passage of particles through matter.
---
#### Optional
- `ONNXRuntime` - Machine learning platform developed by microsoft which is acting as the backend for using different hardware specific execution providers.

- `CUDA` - Parallel computing interface for Nvidia GPUs developed by Nvidia.

- `TensorRT` - Machine learning framework developed by Nvidia for optimizing ML model deployment on Nvidia GPUs. It is built on top of CUDA.

- `oneDNN` - Machine learning library which optimizes model deployment on Intel hardware. It is part of oneAPI - a cross platform performance library for deep learning applications.

- `OpenVINO` - Machine Learning toolkit facilitating optimization of deep learning models from a framework and deployment on Intel Hardware.

- `ROOT` - An object oriented library developed by CERN for for particle physics data analysis.
  - In `Par04`, `ROOT` is used to get host memory usage.

- `Valgrind` - Valgrind is a applicationming tool for memory debugging, memory leak detection, and profiling.
  - In `Par04`, `Valgrind` is used for function call profiling of inference block.

### Run Inference

To run `Par04` simulation application, follow the below given steps:

1. Clone the git repo
```bash
git clone --recursive https://gitlab.cern.ch/prmehta/geant4_par04.git
```
2. Build the application
```bash
cmake -DCMAKE_BUILD_TYPE="Debug" -DCMAKE_CXX_FLAGS_DEBUG="-ggdb3" <source folder>
```

If you want build `Par04` using dependencies, then use the `DCMAKE_PREFIX_PATH` flag.
```bash
cmake -DCMAKE_BUILD_TYPE="Debug" \
    -DCMAKE_PREFIX_PATH="/opt/valgrind/install;/opt/root/src/root;/opt/geant4/install;/opt/onnxruntime/install;/opt/onnxruntime/data;/usr/local/cuda;" \
    -DCMAKE_CXX_FLAGS_DEBUG="-ggdb3" \
    <source folder>
```

3. To build and install the application, go into the build directory and run
```bash
make -j`nproc`
make install
```

4. Go to the install directory and run:
```bash
./examplePar04 -g True -k -kd <detector construction macro file> -ki <inference macro file> -o <output filename> -do <output directory> -fk -f <custom fastsim inference model path> -s <optimized fastsim model save path> -p <profiling json save path>
```

> :warning: Make sure to add path to dependencies in `PATH` and `LD_LIBRARY_PATH` respectively

Checkout [dependencies section](#dependencies) for more info on customising `Par04` for your purpose.

#### Flags
- `-g` - Whether to enable GPU memory usage collection or not.
- `-k` - Whether it is kubeflow deployment or not. Adding `-k` will indicate `True`.

Below given commands will only be used when `-k` is set as the below given flags are specific to KubeFlow to make the inference component KubeFlow compatible.
- `kd` - Path to detector construction macro.
- `ki` - Path to inference construction macro.
- `-o` - Name of the output file. Make sure it ends with `.root`.
- `-do` - Path of the directory where output file will be saved.
- `-fk` - Running fastsim or not. Setting this will mean `true`.

Below given commands will only be used when `-k` and `-fk` is set as the below given flags are make kubeflow deployment of fastsim inference component compatible with Kubeflow pipeline.
- `-f` - Path to `.onnx` model to use for inference.
- `-s` - Path where optimized model created by `ONNXRuntime` session will be stored once model specified in `-f` is loaded into the session.
- `-p` - Path where profiling json will be set. It will be used when profiling flag is set to true.

> :warning: They are added to make the `Par04` compatible with KubeFlow's data passing mechanism. In order to make KubeFlow component's reusable, KubeFlow recommends allowing the KubeFlow sdk to generate paths at compile time.
> 
> The paths generated can or cannot be present in the docker container, so, we need to handle both the cases and when path is not present, then our code should auto generate the directories as the per the path. In order to understand KubeFlow's data passing mechanism, please refer to [this guide](https://www.kubeflow.org/docs/components/pipelines/sdk-v2/v2-component-io/#review-and-update-inputsoutputs-placeholders-if-applicable).
> 
> If you are using `Par04` as a standalone application, using macro file is recommended.

## Benchmark

The benchmark component is built for comparing the output of different `Par04` runs. Currently, it supports `2` output files and the plan to support 3 files is in roadmap. The current benchmark component is tailored a lot towards using in a pipeline rather than a standalone application. 

> :warning: If you are running `benchmark.py` as a standalone script, make sure the output `.root` filenames of Full Sim and Fast Sim(s) is same and both are in different folders.
> ```
> .
> ├── FastSim
> │   └── output.root
> └── FullSim
>     └── output.root
> ```

The output of the benchmark component is a set of plots which will help with comparison between Full Sim and Fast Sim(s) as well as a `.json` file which will contain Jensen Shannon Divergence values for different physics quantity histograms.

> For more details on Jensen Shannon Divergence, refer to this [link](https://docs.scipy.org/doc/scipy/reference/generated/scipy.spatial.distance.jensenshannon.html)

`Benchmark` is a python component and requires the following dependencies:
-   `scipy`
-   `uproot`
-   `numpy`
-   `matplotlib`
-   `json`

### Run Benchmark

1.  To run `Benchmark`, clone the `Par04` repository
```bash
git clone --recursive https://gitlab.cern.ch/prmehta/geant4_par04.git
```
2.  Go into the `Benchmark` component directory
```bash
cd <source directory>/pipelines/model-optimization/benchmark
```
3.  Run `benchmark.py`
```bash
python3 benchmark.py \
    --input_path_A <Path of first input directory> \
    --input_path_B <Path of second input directory> \
    --rootFilename <Filename of the output root file> \
    --truthE <Energy of the particles> \
    --truthAngle <Angle of the particles> \
    --yLogScale <Whether to set a log scale on the y-axis of full and fast simulation comparison plots.> \
    --saveFig <Whether to save the plots> \
    --eosDirectory <EOS directory path where you want to save the files> \
    --eosSaveFolderPath <EOS folder inside the EOS directory where you want to save the output> \
    --saveRootDir <Directory path inside docker container where you want the data to be saved for further passing>
    --bo <Whether to add 3rd plot>
    --ipo <Path of third input directory>
    --ep "Cpu, Optimized Cpu" or "Cpu, Cuda" etc.
```

> :warning: If you are running `Benchmark` as a kubeflow component, use `runBenchmark.sh`. It performs Kerberos authentication for EOS access and then runs `benchmark.py`. Everytime a Kubeflow component is created it needs to authenticate itself for EOS access, it is not feasible to do it manually and `runBenchmark.sh` takes care of that.
> 
> Before running a pipeline which requires EOS access, make sure your `krb-secret` is up-to-date and also run `kinit CERN_ID`. If you want to renew your `krb-secret` perform the steps mentioned [here](https://gitlab.cern.ch/ai-ml/examples/-/tree/master/pipelines/argo-workflows/access_eos).

An example plot is given below:
![](/img/ML_Deployment/Kubeflow_Inference_Optimization_Pipeline/transProfile_1_E_10_GeV_A_90.png)

## Optimizations

Optimization contains 4 blocks - CPU, CUDA, oneDNN and TensorRT. oneDNN and TensorRT are in development. All optimization blocks follow a common structure:

![](/img/ML_Deployment/Kubeflow_Inference_Optimization_Pipeline/optim-block.png)

ONNXRuntime has 2 types of optimization available: `static` and `dynamic`. In `static`, the model is optimized using a representative dataset, saved and then used for inference. In `dynamic`, the model is optimized when it is loaded in ONNXRuntime and uses statistics of the input data.

`dynamic` optimization thus has an extra overhead as compared to `static` optimization but, can have less accuracy drop.

Currently, for CPU and CUDA optimization, `static` optimization is used. For other execution providers like oneDNN and TensorRT, either dynamic or mixed will be implemented due to the restriction placed by ONNXRuntime's design. For more info, refer this this issue: https://github.com/microsoft/onnxruntime/issues/12804

Optimization component performs optimization as a 2 step process which can be mixed:
1. Graph Optimization
2. Quantization

> :warning: When performing quantization, make sure the .onnx model has been converted from it native framework using the latest `opset`. Currently, the minimum opset requirement for quantization is 10. But, it is always better to use latest version as it supports more layers.  

### Graph Optimization

ONNXRuntime has 4 modes of Graph Optimization - 
1. `DISABLE`
2. `BASIC`
3. `EXTENDED`
4. `ALL`

All the modes are supported in this optimization component. For more details on this mode, refer to ONNXRuntime Graph Optimization doc:
https://onnxruntime.ai/docs/performance/graph-optimizations.html

In brief, `basic` adds hardware agnostic optimizations, `extended` applies graph optimizations only to subgraphs placed on CPU, CUDA (NVIDIA GPU) and ROCm (AMD GPU) and `all` applied basic + extended + layout (hardware dependent) optimizations.  

### Quantization

ONNXRuntime has a very rich `INT8` quantization API. Our experiments showed considerable reduction in CPU and GPU memory usage for CPU and CUDA Execution Providers respectively with no to very little accuracy drop. Quantization is very dependent on Calirabtion data / representative data, its quality and quantity both. 

![](/img/ML_Deployment/Kubeflow_Inference_Optimization_Pipeline/longProfile_1_E_10_GeV_A_90.png)
![](/img/ML_Deployment/Kubeflow_Inference_Optimization_Pipeline/CPUResMem_E_10_GeV_A_90.png)

### Config JSON

```json
{
    "strides_count": 30,  
    "latent_vector_dim": 10,
    "min_angle": 50,                  
    "max_angle": 90,
    "min_energy": 1,                          
    "max_energy": 1024,
    "stride": 5,   
    "input_name": null,  
    "optimize_model": true,                
    "graph_optim_lvl": "all",
    "only_graph_optim": false,                    
    "quant_format": "QDQ",
    "op_types_to_quantize": [],
    "per_channel": false,  
    "reduce_range": false,           
    "activation_type": "int8",                
    "weight_type": "int8",
    "nodes_to_quantize": [],
    "nodes_to_exclude": [],  
    "use_external_data_format": false,             
    "calibrate_method": "MinMax", 
    "extra_options": {
        "extra.Sigmoid.nnapi": false,
        "ActivationSymmetric": false,
        "WeightSymmetric": true,
        "EnableSubgraph": false,
        "ForceQuantizeNoInputCheck": true,
        "MatMulConstBOnly": false,
        "AddQDQPairToWeight": false,
        "OpTypesToExcludeOutputQuantizatioin": [],
        "DedicatedQDQPair": false,
        "QDQOpTypePerChannelSupportToAxis": {},
        "CalibTensorRangeSymmetric": false,
        "CalibMovingAverage": false,
        "CalibMovingAverageConstant": 0.01
    },            
    "batch_size": 10,
    "execution_providers": ["CPUExecutionProvider"], 
    "for_tensorrt": false                           
}
```

#### Description
The description for all the keys are JSON given below:
##### Calibration Data
- `strides_count` - Number of strides/loops to use for generating the calibration data. Useful when handling OOM issue.
- `latent_vector_dim` - Dimension of latent vectors to be generated for calibration data.
- `min_angle` - Minimum angle for which to generate calibration data.
- `max_angle` - Maximum angle for which to generate calibration data.
- `min_energy` - Minimum particle energy for which to generate calibration data.
- `max_energy` - Maximum particle energy for which to generate calibration data.
- `stride` - Number of events to generate for each pair of [condE, condA, condGeo].
- `batch_size` - Batch size to use when performing inference for getting calibration data output. Calibrator uses the output to generate statistics which get used downstream in quantization to ensure minimal accuracy drop possible.
- `calibrate_method` - Calibration method / Calibrator to use. Currently, supported [`MinMax`, `Entropy`, `Percentile`]. Preferred, `MinMax` as it has less memory requirement and performs very well.
- `execution_providers` - Execution providers to use 
  
##### Graph Optimization
- `graph_optim_lvl` - Which graph optimization lvl to use. Supported [`disable`, `basic`, `extended`, `all`].
- `only_graph_optim` - Whether to only perform graph optimization and skip quantization.
- `optimize_model` - Whether to perform graph optimization before quantization. Should be set to `True` even if `only_graph_optim` is set to `True`.
  
##### Quantization

For detailed read on ONNXRuntime Quantization, refer https://onnxruntime.ai/docs/performance/quantization.html

- `quant_format` - Type of quantization format to use. Supported [`QDQ`, `QOperator`]. Perferred, `QOQ` as it gives good performance to accuracy tradeoff. ONNXRuntime suggests to use `QDQ` on x86 CPU and `QOperator` for arm64 CPU.
- `op_types_to_quantize` - List of operations to quantize. Only the operations listed here will be quantized in the model.
- `per_channel` - Whether to perform `per_channel` quantization or not.
- `reduce_range` - Whether to perform `7bit` quantization or not.
- `activation_type` - Which `INT8` quantization to perform on activations. Supported [`int8`, `uint8`]. Peferred, `int8` as it is more versatile and works on a wide array of models.
- `weight_type` - Which `INT8` quantization to perform on model weights. Supported [`int8`, `uint8`]. Peferred, `int8` as it is more versatile and works on a wide array of models.
- `nodes_to_quantize` - List of nodes to quantize. Specify exact node names of the .onnx model. Only the nodes present in the list will be quantized. If empty, all nodes will be quanized.
- `nodes_to_exclude` - List of nodes to exclude. Specify exact node names of the .onnx model. Only the nodes present in the list will be exclued. If empty, no nodes will be excluded.
- `use_external_data_format` -  Saving models > 2GB creates problems in ONNXRuntime. It is preferred to set this option to `True` if dealing with models > 2GB.
- `extra_options` - Refer [onnxruntime/quantization/quantize.py](https://github.com/microsoft/onnxruntime/blob/main/onnxruntime/python/tools/quantization/quantize.py#L113)
- `for_tensorrt` - If this optimization is for tensorrt or not. If set to `True`, only the calibration table will be created. TensorRT performs it own graph optimization and quantization. Hence, TensorRT optimization can only be performed dynamically in ONNXRuntime. The path of calibration table generated can be given as input to ONNXRuntime Session, TensorRT will use this Calibration cache to optimize the model at runtime.

> :warning: Optimization Config JSON can be changed as per the need but currently, these are the supported params. Any additional params added to JSON will be ignored.


## Complete Pipeline Image

![](/img/ML_Deployment/Kubeflow_Inference_Optimization_Pipeline/current_gsoc_pipeline.png)