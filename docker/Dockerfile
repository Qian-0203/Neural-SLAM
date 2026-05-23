FROM nvcr.io/nvidia/cuda:10.0-cudnn7-devel-ubuntu18.04

ARG DEBIAN_FRONTEND=noninteractive
ARG HABITAT_SIM_COMMIT=9575dcd45fe6f55d2a44043833af08972a7895a9
ARG HABITAT_API_REPO=https://github.com/devendrachaplot/habitat-api.git
ARG HABITAT_API_COMMIT=8ac1b16aa0554b10748a925f8a937954c77c1563
ARG CONDA_DIR=/opt/conda

ENV LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    PATH=${CONDA_DIR}/bin:${PATH} \
    PYTHONUNBUFFERED=1 \
    GLOG_minloglevel=2 \
    MAGNUM_LOG=quiet \
    HABITAT_SIM_LOG=quiet \
    PIP_NO_CACHE_DIR=1

SHELL ["/bin/bash", "-lc"]

RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        cmake \
        curl \
        git \
        libegl1-mesa-dev \
        libgl1-mesa-dev \
        libglvnd-dev \
        libsm6 \
        libxext6 \
        libxrender1 \
        libjpeg-dev \
        libpng-dev \
        ninja-build \
        patchelf \
        pkg-config \
        unzip \
        wget \
        xz-utils \
    && rm -rf /var/lib/apt/lists/*

RUN curl -fsSL https://repo.anaconda.com/miniconda/Miniconda3-4.7.12.1-Linux-x86_64.sh -o /tmp/miniconda.sh \
    && bash /tmp/miniconda.sh -b -p "${CONDA_DIR}" \
    && rm /tmp/miniconda.sh \
    && conda config --set always_yes yes --set changeps1 no \
    && conda install python=3.6 pip \
    && conda install -c pytorch pytorch==1.2.0 torchvision==0.4.0 cudatoolkit=10.0 \
    && conda clean -afy

COPY docker/constraints.txt /tmp/neural_slam_constraints.txt
ENV PIP_CONSTRAINT=/tmp/neural_slam_constraints.txt

RUN python -m pip install --upgrade "pip==20.2.4" \
    && python -m pip install cython==0.29.14 numpy==1.17.5

WORKDIR /opt
RUN git clone --recursive https://github.com/facebookresearch/habitat-sim.git \
    && cd habitat-sim \
    && git checkout "${HABITAT_SIM_COMMIT}" \
    && git submodule update --init --recursive \
    && python -m pip install -r requirements.txt \
    && python setup.py install --headless

WORKDIR /workspace/Neural-SLAM
COPY . /workspace/Neural-SLAM

RUN rm -rf env/habitat/habitat_api \
    && git clone "${HABITAT_API_REPO}" env/habitat/habitat_api \
    && git -C env/habitat/habitat_api checkout "${HABITAT_API_COMMIT}" \
    && python -m pip install -e env/habitat/habitat_api

RUN python -m pip install -r requirements.txt \
    && python -m pip install "numpy-quaternion==2019.12.11.22.25.52"

ENV PYTHONPATH=/workspace/Neural-SLAM:/workspace/Neural-SLAM/env/habitat/habitat_api:${PYTHONPATH}

RUN mkdir -p /usr/share/glvnd/egl_vendor.d \
    && printf '%s\n' \
        '{' \
        '    "file_format_version": "1.0.0",' \
        '    "ICD": {' \
        '        "library_path": "libEGL_nvidia.so.0"' \
        '    }' \
        '}' \
        > /usr/share/glvnd/egl_vendor.d/10_nvidia.json

CMD ["/bin/bash"]
