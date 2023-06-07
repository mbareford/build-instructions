Instructions for building Horovod for the Cirrus GPU nodes
==========================================================

These instructions show how to build a Python virtual environment (venv) that provides Horovod 0.26.1, a distributed deep learning training framework,
one that encompasses three ML libraries, [TensorFlow](https://www.tensorflow.org/) and [PyTorch](https://pytorch.org/).
The instructions will attempt to install the latest versions of those libraries; as of Apr 2023, these are TensorFlow 2.11.0 and PyTorch 1.13.1.

The Horovod environment is intended to run on the Cirrus GPU nodes (Cascade Lake, NVIDIA Tesla V100-SXM2-16GB).

This venv is an extension of the Miniconda3 (Python 3.10.8) environment provided by the `python/3.10.8-gpu` module.
MPI comms is handled by the [Horovod](https://horovod.readthedocs.io/en/stable/index.html) 0.26.1 package (built with NCCL 2.11.4).
Horovod is required for running TensorFlow/PyTorch over multiple GPUs distributed across multiple compute nodes.


Setup initial environment
-------------------------

```bash
PRFX=/path/to/work  # e.g., PRFX=/mnt/lustre/indy2lfs/sw
cd ${PRFX}

HOROVOD_LABEL=horovod
HOROVOD_VERSION=0.26.1
HOROVOD_ROOT=${PRFX}/${HOROVOD_LABEL}

module load python/3.10.8-gpu

PYTHON_VER=`echo ${MINICONDA3_PYTHON_VERSION} | cut -d'.' -f1-2`
PYTHON_DIR=${PRFX}/${HOROVOD_LABEL}/${HOROVOD_VERSION}-gpu/python
PYTHON_BIN=${PYTHON_DIR}/${MINICONDA3_PYTHON_VERSION}/bin


mkdir -p ${PYTHON_BIN}

export PIP_CACHE_DIR=${PYTHON_DIR}/.cache/pip

export PYTHONUSERBASE=${PYTHON_DIR}/${MINICONDA3_PYTHON_VERSION}
export PATH=${PYTHONUSERBASE}/bin:${PATH}
export PYTHONPATH=${PYTHONUSERBASE}/lib/python${PYTHON_VER}/site-packages:${PYTHONPATH}
```

Remember to change the setting for `PRFX` to a path appropriate for your Cirrus project.


Install the machine learning packages
-------------------------------------

```bash
pip install --user pyspark
pip install --user scikit-learn
pip install --user scikit-image

pip install --user tensorflow
pip install --user tensorflow-gpu
pip install --user tensorboard_plugin_profile

pip install --user torch
pip install --user torchvision
pip install --user pytorch-lightning
pip install --user pytorch-lightning-bolts
pip install --user pytorch-lightning-bolts["extra"]
pip install --user lightning-flash
pip install --user 'lightning-flash[all]'

pip install --user mxnet
pip install --user mxnet-cu112

pip install --user fastai
pip install --user opencv-python
```


Install Horovod linking with the Nvidia Collective Communications Library (NCCL)
--------------------------------------------------------------------------------

Please note, in preparation for the Horovod install, you must check that `libcuda.so.1`
exists as soft link to `libcuda.so` in `${NVHPC_ROOT}/cuda/lib64/stubs`.

```bash
module load cmake

export LD_LIBRARY_PATH=${NVHPC_ROOT}/cuda/lib64/stubs:${LD_LIBRARY_PATH}

CC=mpicc CXX=mpicxx FC=mpifort HOROVOD_CUDA_HOME=${NVHPC_ROOT}/cuda/11.6 HOROVOD_NCCL_HOME=${NVHPC_ROOT}/comm_libs/nccl HOROVOD_GPU=CUDA HOROVOD_BUILD_CUDA_CC_LIST=70 HOROVOD_CPU_OPERATIONS=MPI HOROVOD_GPU_OPERATIONS=NCCL HOROVOD_WITH_MPI=1 HOROVOD_WITH_TENSORFLOW=1 HOROVOD_WITH_PYTORCH=1 HOROVOD_WITH_MXNET=0 CUDA_PATH=${NVHPC_ROOT}/cuda/11.6 pip install --user --no-cache-dir horovod[tensorflow,pytorch]==${HOROVOD_VERSION}
```

Now run `horovodrun --check-build` to confirm that [Horovod](https://horovod.readthedocs.io/en/stable/index.html) has been installed
correctly. That command should return something like the following output

```
Horovod v0.26.1:

Available Frameworks:
    [X] TensorFlow
    [X] PyTorch
    [ ] MXNet

Available Controllers:
    [X] MPI
    [X] Gloo

Available Tensor Operations:
    [X] NCCL
    [ ] DDL
    [ ] CCL
    [X] MPI
    [X] Gloo 
```