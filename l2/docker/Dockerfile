#---------------
# This image is based on cuda and installs horovod
# Additionally it installs everything required to run process UC1 L2
#---------------

# TF 1.14 does not work with cuda10.1, so used cuda10.0
FROM nvidia/cuda:10.0-devel-ubuntu18.04

# TensorFlow version is tightly coupled to CUDA and cuDNN so it should be selected carefully
ENV TENSORFLOW_VERSION=1.14.0
ENV CUDNN_VERSION=7.6.0.64-1+cuda10.0
ENV NCCL_VERSION=2.4.7-1+cuda10.0
ARG HVD_VERSION=0.19.5

# Python 2.7 or 3.6 is supported by Ubuntu Bionic out of the box
ARG python=2.7
ENV PYTHON_VERSION=${python}

# Set default shell to /bin/bash
SHELL ["/bin/bash", "-cu"]

RUN apt-get update && apt-get install -y --allow-downgrades --allow-change-held-packages --no-install-recommends \
        build-essential \
        cmake \
        g++-4.8 \
        git \
        curl \
        vim \
        wget \
        ca-certificates \
        libcudnn7=${CUDNN_VERSION} \
        libnccl2=${NCCL_VERSION} \
        libnccl-dev=${NCCL_VERSION} \
        libjpeg-dev \
        libpng-dev \
        python${PYTHON_VERSION} \
        python${PYTHON_VERSION}-dev \
        librdmacm1 \
        libibverbs1 \
        ibverbs-providers

RUN if [[ "${PYTHON_VERSION}" == "3.6" ]]; then \
        apt-get install -y python${PYTHON_VERSION}-distutils; \
    fi
RUN ln -s /usr/bin/python${PYTHON_VERSION} /usr/bin/python

RUN curl -O https://bootstrap.pypa.io/get-pip.py && \
    python get-pip.py && \
    rm get-pip.py

# Install TensorFlow, Keras, PyTorch and MXNet
RUN pip install future typing
RUN pip install numpy \
        tensorflow-gpu==${TENSORFLOW_VERSION} \
        tensorboard==1.14.0  \
        keras==2.1.6 \
        h5py==2.10.0

# Install Open MPI
RUN mkdir /tmp/openmpi && \
    cd /tmp/openmpi && \
    wget https://www.open-mpi.org/software/ompi/v4.0/downloads/openmpi-4.0.0.tar.gz && \
    tar zxf openmpi-4.0.0.tar.gz && \
    cd openmpi-4.0.0 && \
    ./configure --enable-orterun-prefix-by-default && \
    make -j $(nproc) all && \
    make install && \
    ldconfig && \
    rm -rf /tmp/openmpi



# Install Horovod, temporarily using CUDA stubs
# Not specifying HOROVOD_GPU_BROADCAST=NCCL makes horovod hang when executing
# Only installing HOROVOD for Tensorflow, no PyTorch and MXnet support
RUN ldconfig /usr/local/cuda/targets/x86_64-linux/lib/stubs && \
    HOROVOD_GPU_ALLREDUCE=NCCL HOROVOD_GPU_BROADCAST=NCCL HOROVOD_WITH_TENSORFLOW=1 \
         pip install --no-cache-dir horovod==${HVD_VERSION} && \
    ldconfig

# Install OpenSSH for MPI to communicate between containers
RUN apt-get install -y --no-install-recommends openssh-client openssh-server && \
    mkdir -p /var/run/sshd

# Allow OpenSSH to talk to containers without asking for confirmation
RUN cat /etc/ssh/ssh_config | grep -v StrictHostKeyChecking > /etc/ssh/ssh_config.new && \
    echo "    StrictHostKeyChecking no" >> /etc/ssh/ssh_config.new && \
    mv /etc/ssh/ssh_config.new /etc/ssh/ssh_config

# Download examples
RUN apt-get install -y --no-install-recommends subversion && \
    svn checkout https://github.com/horovod/horovod/trunk/examples && \
    rm -rf /examples/.svn

WORKDIR "/examples"



# Env variables used during build only. Not available in running containers using this image.
# Prefixes: C=Container, H=Host. This should clarify if file/dir that is referenced in variable referes to host or container.
ARG C_BASE_CODE_DIR=/code
ARG C_UC1_L2_CODE_DIR=${C_BASE_CODE_DIR}/PROCESS_L2
ARG CODE_GITHUB_URL=https://github.com/medgift/PROCESS_L2.git
ARG H_PIP_REQUIREMENTS=./requirements.txt
ARG H_CONFIG_FILE=./config.cfg
ARG H_DATA_CONFIG_FILE=./data-config.cfg


#---------------
#EVERYTHING BELOW HERE REFERS TO PROCESS UC1 L2 integration
#---------------

# Install libs/applications used by training script
ARG DEBIAN_FRONTEND=noninteractive
RUN apt update && apt install -y \
    libcurl4-openssl-dev \
	libssl-dev \
	libsm6 \
	libxext6 \
    python-tk \
	openslide-tools

# Create code dir
RUN mkdir -p "$C_BASE_CODE_DIR"

# Override requirements.txt with adapted one from host.
COPY "$H_PIP_REQUIREMENTS" /requirements.txt

# Install python libs specified in requirements.txt
RUN pip install -r /requirements.txt --no-cache-dir

# Clone GitHub repo. 
# Make sure this step is reexecuted each time a new REDO_CLONE value is provided as build argument.
# This is useful for having the code in the container up to date
ARG CODE_GIT_BRANCH=origin/horovod-ivan
ARG REDO_CLONE=no
WORKDIR "$C_BASE_CODE_DIR"
RUN git clone "$CODE_GITHUB_URL"
WORKDIR "$C_UC1_L2_CODE_DIR"

# Checkout specific branch
RUN git checkout "$CODE_GIT_BRANCH"

# Override config files from from host to container
WORKDIR "$C_UC1_L2_CODE_DIR"
COPY "$H_CONFIG_FILE" lib/config.ini 
COPY "$H_DATA_CONFIG_FILE" data/data.cfg


# No entrypoint so client script decides what to execute
# CMD: /bin/bash is already exeuted by default.
