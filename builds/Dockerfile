ARG VARIANT

# Base image: Ubuntu 20.04 for older glibc compatibility
ARG BASE_IMAGE="hexpm/elixir:1.15.8-erlang-24.3.4.17-ubuntu-focal-20240427"

# Pre-stages for base image variants

FROM ${BASE_IMAGE} AS base-cpu

FROM ${BASE_IMAGE} AS base-cuda

# Now we use hermetic CUDA, so we no longer need to install it. We
# leave this commented out for now, for reference.

# ARG CUDA_VERSION
# ARG CUDNN_VERSION

# ARG DEBIAN_FRONTEND=noninteractive

# RUN distro="ubuntu$(. /etc/lsb-release; echo "$DISTRIB_RELEASE" | tr -d '.')" && \
#   # Official Docker images use the sbsa packages when targetting arm64.
#   # See https://gitlab.com/nvidia/container-images/cuda/-/blob/85f465ea3343a2d7f7753a0a838701999ed58a01/dist/12.5.1/ubuntu2204/base/Dockerfile#L12
#   arch="$(if [ "$(uname -m)" = "aarch64" ]; then echo "sbsa"; else echo "x86_64"; fi)" && \
#   apt-get update && apt-get install -y ca-certificates wget && \
#   wget -qO /tmp/cuda-keyring.deb https://developer.download.nvidia.com/compute/cuda/repos/$distro/$arch/cuda-keyring_1.1-1_all.deb && \
#   dpkg -i /tmp/cuda-keyring.deb && apt-get update && \
#   apt-get install -y git cuda-toolkit-${CUDA_VERSION} libcudnn9-cuda-12=${CUDNN_VERSION}-1 libcudnn9-dev-cuda-12=${CUDNN_VERSION}-1 && \
#   apt-get clean -y && rm -rf /var/lib/apt/lists/*

FROM ${BASE_IMAGE} AS base-rocm

ARG ROCM_VERSION

ARG DEBIAN_FRONTEND=noninteractive

# Use Clang for ROCm toolchain instead of GCC
ENV TF_ROCM_CLANG="1"

# Install ROCm from official AMD repository
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates curl gnupg && \
  distro="$(. /etc/lsb-release && echo "$DISTRIB_CODENAME")" && \
  curl -sL https://repo.radeon.com/rocm/rocm.gpg.key | apt-key add - && \
  echo "deb [arch=amd64] https://repo.radeon.com/rocm/apt/${ROCM_VERSION}/ $distro main" | tee /etc/apt/sources.list.d/rocm.list && \
  printf 'Package: *\nPin: release o=repo.radeon.com\nPin-Priority: 600\n' | tee /etc/apt/preferences.d/rocm-pin-600 && \
  apt-get update && \
  apt-get install -y rocm-dev rocm-libs && \
  apt-get clean -y && rm -rf /var/lib/apt/lists/*

ENV ROCM_PATH "/opt/rocm"

# TheRock nightly builds for newer GPU support (e.g., gfx1151 Strix Halo)
# Downloads pre-built tarballs from TheRock S3 bucket (same method as amd-strix-halo-toolboxes)
# Use with: --build-arg VARIANT=rocm-therock --build-arg THEROCK_TARGET=gfx1151
FROM ${BASE_IMAGE} AS base-rocm-therock

ARG THEROCK_TARGET=gfx1151
ARG THEROCK_ROCM_MAJOR=7

ARG DEBIAN_FRONTEND=noninteractive

# Use Clang for ROCm toolchain instead of GCC
ENV TF_ROCM_CLANG="1"

# Install dependencies for downloading and extracting TheRock
RUN apt-get update && apt-get install -y --no-install-recommends \
  ca-certificates curl aria2 python3 python3-pip && \
  ln -sf /usr/bin/python3 /usr/bin/python && \
  apt-get clean -y && rm -rf /var/lib/apt/lists/*

# Download and install ROCm from TheRock nightly tarballs
# See https://github.com/kyuz0/amd-strix-halo-toolboxes and https://github.com/ROCm/TheRock
WORKDIR /tmp
RUN bash -c 'set -euo pipefail; \
  BASE="https://therock-nightly-tarball.s3.amazonaws.com"; \
  PREFIX="therock-dist-linux-${THEROCK_TARGET}-${THEROCK_ROCM_MAJOR}"; \
  KEY="$(curl -s "${BASE}?list-type=2&prefix=${PREFIX}" \
    | tr "<" "\n" \
    | grep -o "therock-dist-linux-${THEROCK_TARGET}-${THEROCK_ROCM_MAJOR}\..*\.tar\.gz" \
    | sort -V | tail -n1)"; \
  echo "Downloading TheRock tarball: ${KEY}"; \
  aria2c -x 16 -s 16 -j 16 --file-allocation=none "${BASE}/${KEY}" -o therock.tar.gz && \
  mkdir -p /opt/rocm && \
  tar xzf therock.tar.gz -C /opt/rocm --strip-components=1 && \
  rm -f therock.tar.gz && \
  # Remove any circular symlinks that cause issues with bazel globbing
  find /opt/rocm -maxdepth 10 -type l -exec sh -c "readlink -f {} >/dev/null 2>&1 || rm -f {}" \; 2>/dev/null || true && \
  # Also remove self-referential symlinks like llvm -> . or rocm_dist -> .
  find /opt/rocm -type l -lname "." -delete 2>/dev/null || true && \
  find /opt/rocm -type l -lname ".." -delete 2>/dev/null || true && \
  echo "/opt/rocm/lib" > /etc/ld.so.conf.d/rocm.conf && \
  echo "/opt/rocm/lib64" >> /etc/ld.so.conf.d/rocm.conf && \
  echo "/opt/rocm/lib/llvm/lib" >> /etc/ld.so.conf.d/rocm.conf && \
  ldconfig && \
  echo "=== ROCm structure ===" && ls -la /opt/rocm/ && \
  echo "=== Checking for broken symlinks ===" && find /opt/rocm -xtype l 2>/dev/null | head -10 || echo "No broken symlinks"'

WORKDIR /

# TheRock paths compatible with XLA ROCm configuration
# Note: TheRock has lib/llvm/bin not llvm/bin
ENV ROCM_PATH="/opt/rocm"
ENV HIP_PLATFORM="amd"
ENV HIP_PATH="/opt/rocm"
ENV HIP_CLANG_PATH="/opt/rocm/lib/llvm/bin"
ENV HIP_INCLUDE_PATH="/opt/rocm/include"
ENV HIP_LIB_PATH="/opt/rocm/lib"
ENV HIP_DEVICE_LIB_PATH="/opt/rocm/lib/llvm/amdgcn/bitcode"
ENV PATH="/opt/rocm/bin:/opt/rocm/lib/llvm/bin:${PATH}"
ENV LD_LIBRARY_PATH="/opt/rocm/lib:/opt/rocm/lib64:/opt/rocm/lib/llvm/lib:${LD_LIBRARY_PATH}"
ENV LIBRARY_PATH="/opt/rocm/lib:/opt/rocm/lib64"
ENV CPATH="/opt/rocm/include"
ENV PKG_CONFIG_PATH="/opt/rocm/lib/pkgconfig"

# Create llvm -> lib/llvm symlink for XLA rocm_configure.bzl compatibility
# XLA hardcodes llvm/bin/clang but TheRock uses lib/llvm/bin/clang
RUN ln -sf lib/llvm /opt/rocm/llvm

FROM base-${VARIANT}

# Set the missing UTF-8 locale, otherwise Elixir warns
ENV LC_ALL=C.UTF-8

# Make sure installing packages (like tzdata) doesn't prompt for configuration
ARG DEBIAN_FRONTEND=noninteractive

# We need to install "add-apt-repository" first
RUN apt-get update && \
  # Install basic system dependencies
  apt-get update && apt-get install -y ca-certificates curl git unzip wget xxd make build-essential && \
  # Install Clang
  clang_version="18" && \
  apt-get install -y wget gnupg software-properties-common lsb-release && \
  wget -qO- https://apt.llvm.org/llvm.sh | bash -s -- $clang_version && \
  update-alternatives --install /usr/bin/clang clang /usr/bin/clang-$clang_version 100 && \
  update-alternatives --install /usr/bin/clang++ clang++ /usr/bin/clang++-$clang_version 100 && \
  apt-get clean -y && rm -rf /var/lib/apt/lists/*

# Install Bazel using Bazelisk (works for both amd and arm)
RUN wget -O bazel "https://github.com/bazelbuild/bazelisk/releases/download/v1.26.0/bazelisk-linux-$(dpkg --print-architecture)" && \
  chmod +x bazel && \
  mv bazel /usr/local/bin/bazel

ENV USE_BAZEL_VERSION=7.7.0

# Install Python and the necessary global dependencies
RUN apt-get update && apt-get install -y python3 python3-pip && \
  ln -sf /usr/bin/python3 /usr/bin/python && \
  python -m pip install --upgrade pip numpy && \
  apt-get clean -y && rm -rf /var/lib/apt/lists/*

# Setup project files

WORKDIR /xla

ARG XLA_TARGET

ENV XLA_TARGET=${XLA_TARGET}
ENV XLA_CACHE_DIR=/cache
ENV XLA_BUILD=true

COPY mix.exs mix.lock ./
RUN mix deps.get

COPY lib lib
COPY README.md Makefile ./
COPY extension extension

CMD [ "mix", "compile" ]
