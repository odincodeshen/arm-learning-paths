---
title: Build Isaac Sim 6.0 from source
weight: 3

### FIXED, DO NOT MODIFY
layout: learningpathall
---

Isaac Sim doesn't ship pre-built binaries for `aarch64` yet. On DGX Spark you compile it from source using the `develop` branch of the [Isaac Sim GitHub repository](https://github.com/isaac-sim/IsaacSim), which contains the 6.0 Early Developer Release.

## Install build dependencies

Isaac Sim requires GCC 11 and Git LFS:

```bash
sudo apt update && sudo apt install -y gcc-11 g++-11 git-lfs
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 200
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-11 200
```

Confirm GCC 11 is active:

```bash
gcc --version
```

## Clone and build

Clone the `develop` branch:

```bash
cd ~
git clone --depth=1 --branch develop --recursive https://github.com/isaac-sim/IsaacSim
cd IsaacSim
git lfs install
git lfs pull
```

Run the build. It will ask you to accept the license, type `yes`:

```bash
./build.sh
```

This takes around 12 minutes on the Spark (longer on first build as it downloads dependencies like PyTorch and CUDA libraries). When it finishes you will see:

```output
BUILD (RELEASE) SUCCEEDED (Took 707.50 seconds)
```

## Set up your environment

```bash
echo "export ISAACSIM_PATH=\"${HOME}/IsaacSim/_build/linux-aarch64/release\"" >> ~/.bashrc
echo "export ISAACSIM_PYTHON_EXE=\"\${ISAACSIM_PATH}/python.sh\"" >> ~/.bashrc
echo 'export LD_PRELOAD="/lib/aarch64-linux-gnu/libgomp.so.1"' >> ~/.bashrc
source ~/.bashrc
```

The `LD_PRELOAD` for `libgomp` is a required workaround on DGX Spark. Without it, Isaac Sim crashes on launch.

## Test the build

```bash
${ISAACSIM_PATH}/isaac-sim.sh
```

The first launch takes a while as shaders get compiled and cached. Once the viewport appears and you can orbit/pan the scene, the build is good. Close Isaac Sim for now.
