# Docker Reproduction Guide

This Docker setup targets the dependency versions used by the original Active Neural SLAM release:

- Ubuntu 18.04
- CUDA 10.0 / cuDNN 7
- Python 3.6
- PyTorch 1.2.0 / torchvision 0.4.0
- `habitat-sim` commit `9575dcd45fe6f55d2a44043833af08972a7895a9`
- `devendrachaplot/habitat-api` submodule commit `8ac1b16aa0554b10748a925f8a937954c77c1563`

## 1. Host Requirements

Install Docker with the NVIDIA Container Toolkit on a Linux host with a working
NVIDIA driver. Check the host driver before debugging the container:

```bash
nvidia-smi
```

If this reports `Failed to initialize NVML: Driver/library version mismatch`,
reboot after the driver update or reinstall matching NVIDIA driver packages
before running Docker. The container cannot start with GPU access until
`nvidia-smi` works on the host.

Check GPU container access:

```bash
docker run --rm --gpus all nvcr.io/nvidia/cuda:10.0-cudnn7-devel-ubuntu18.04 nvidia-smi
```

The old Docker Hub tag `nvidia/cuda:10.0-base-ubuntu18.04` is no longer
available in current registries; use NVIDIA NGC's `nvcr.io/nvidia/cuda` image
for this legacy CUDA 10.0 stack.

## 2. Build the Image

From the repository root:

```bash
docker compose build neural-slam
```

The build compiles `habitat-sim`, so it can take a while.

If a previous build appears stuck while pip repeatedly downloads old
`pyparsing` versions, stop it with `Ctrl+C` and rebuild. The Dockerfile pins
`pyparsing` and uses pip `20.2.4` to avoid slow dependency backtracking on this
legacy Python 3.6 environment.

## 3. Prepare Data

Download the Gibson scene dataset and Habitat PointNav Gibson dataset following the original Habitat data instructions. The final layout must be:

```text
Neural-SLAM/
  data/
    scene_datasets/
      gibson/
        Adrian.glb
        Adrian.navmesh
        ...
    datasets/
      pointnav/
        gibson/
          v1/
            train/
            val/
            ...
```

The compose file mounts local `./data` into the container, so keep the dataset in the repository root under `data/`.

Check the two common layout mistakes before running Habitat:

```bash
find data/datasets/pointnav/gibson/v1 -maxdepth 3 -type f | head
find data/scene_datasets/gibson -maxdepth 1 -name "*.glb" | head
find data/scene_datasets/gibson -maxdepth 1 -name "*.navmesh" | head
```

The PointNav episodes must be under `data/datasets/pointnav/...`, not
`data/pointnav/...`. The scene dataset must contain per-scene files such as
`Cantwell.glb` and `Cantwell.navmesh`; panorama folders or `mesh.obj` alone
are not enough for this Habitat evaluation.

## 4. Download Pretrained Models

The original CMU URLs may now return an HTML page instead of model weights.
Use the Google Drive links from the project README:

```bash
mkdir -p pretrained_models
wget --no-check-certificate 'https://drive.google.com/uc?export=download&id=1UK2hT0GWzoTaVR5lAI6i8o27tqEmYeyY' -O pretrained_models/model_best.global
wget --no-check-certificate 'https://drive.google.com/uc?export=download&id=1A1s_HNnbpvdYBUAiw2y1JmmELRLfAJb8' -O pretrained_models/model_best.local
wget --no-check-certificate 'https://drive.google.com/uc?export=download&id=1o5OG7DIUKZyvi5stozSqRpAEae1F2BmX' -O pretrained_models/model_best.slam
```

Verify the downloads before evaluation:

```bash
file pretrained_models/model_best.*
du -h pretrained_models/model_best.*
```

The files should report as `data`, not `HTML document`. Expected sizes are
roughly `8.1M`, `52M`, and `74M`.

## 5. Smoke Test

Run a single-process validation episode:

```bash
docker compose run --rm neural-slam \
  python main.py -n 1 --auto_gpu_config 0 --split val
```

For CPU-only import checks, use `--no_cuda`, but full Habitat simulation and
paper reproduction are intended for the GPU-enabled `neural-slam` service.
Do not use the `neural-slam-cpu` service for Habitat evaluation: this
Habitat-Sim build uses headless EGL and still needs the NVIDIA runtime even if
PyTorch runs on CPU.

## 6. Convert Validation Split for Evaluation

```bash
docker compose run --rm neural-slam \
  python scripts/convert_datasets.py
```

This creates `data/datasets/pointnav/gibson/v1/val_mt/`.

Successful conversion prints 14 validation scenes with 71 episodes each and
writes `data/datasets/pointnav/gibson/v1/val_mt/`.

## 7. Reproduce Pretrained Evaluation

Before the full 14-process evaluation, run a one-process sanity check. This
confirms that the dataset, scene files, and pretrained checkpoints can load.
`--pretrained_resnet 0` avoids downloading torchvision's ResNet18 weights at
startup; the SLAM checkpoint is loaded immediately afterwards.

```bash
docker compose run --rm neural-slam \
  python main.py --split val_mt --eval 1 \
  --auto_gpu_config 0 -n 1 --num_episodes 1 --num_processes_per_gpu 1 \
  --pretrained_resnet 0 \
  --load_global pretrained_models/model_best.global --train_global 0 \
  --load_local pretrained_models/model_best.local --train_local 0 \
  --load_slam pretrained_models/model_best.slam --train_slam 0
```

Recommended Gibson validation evaluation uses 14 processes and 71 episodes per process:

```bash
docker compose run --rm neural-slam \
  python main.py --split val_mt --eval 1 \
  --auto_gpu_config 0 -n 14 --num_episodes 71 --num_processes_per_gpu 7 \
  --pretrained_resnet 0 \
  --load_global pretrained_models/model_best.global --train_global 0 \
  --load_local pretrained_models/model_best.local --train_local 0 \
  --load_slam pretrained_models/model_best.slam --train_slam 0
```

For the larger evaluation map used in the project instructions:

```bash
docker compose run --rm neural-slam \
  python main.py --split val_mt --eval 1 \
  --auto_gpu_config 0 -n 14 --num_episodes 71 --num_processes_per_gpu 7 \
  --map_size_cm 4800 --global_downscaling 4 \
  --pretrained_resnet 0 \
  --load_global pretrained_models/model_best.global --train_global 0 \
  --load_local pretrained_models/model_best.local --train_local 0 \
  --load_slam pretrained_models/model_best.slam --train_slam 0
```

### Newer NVIDIA GPUs

This image intentionally pins CUDA 10.0, cuDNN 7, PyTorch 1.2.0, and
torchvision 0.4.0. These libraries can fail on modern GPUs during PyTorch
CUDA inference, for example with `CUDNN_STATUS_EXECUTION_FAILED`. Keep Habitat
on the GPU-enabled service, but add `--no_cuda` so neural network inference
runs on CPU:

```bash
docker compose run --rm neural-slam \
  python main.py --split val_mt --eval 1 --no_cuda \
  --auto_gpu_config 0 -n 14 --num_episodes 71 --num_processes_per_gpu 7 \
  --pretrained_resnet 0 \
  --load_global pretrained_models/model_best.global --train_global 0 \
  --load_local pretrained_models/model_best.local --train_local 0 \
  --load_slam pretrained_models/model_best.slam --train_slam 0
```

This is slower, but it avoids using the legacy PyTorch CUDA kernels while
still letting Habitat-Sim render through EGL. If you need fast PyTorch GPU
inference on a modern GPU, port the environment to a newer CUDA/PyTorch and
Habitat stack.

### Small Evaluation and Visualization

For a quick finished run, reduce `--num_episodes`:

```bash
docker compose run --rm neural-slam \
  python main.py --split val_mt --eval 1 --no_cuda \
  --auto_gpu_config 0 -n 14 --num_episodes 1 --num_processes_per_gpu 7 \
  --pretrained_resnet 0 \
  --load_global pretrained_models/model_best.global --train_global 0 \
  --load_local pretrained_models/model_best.local --train_local 0 \
  --load_slam pretrained_models/model_best.slam --train_slam 0
```

To dump visualization PNGs on a headless machine, add `--print_images 1`,
choose a results directory, and set an experiment name:

```bash
docker compose run --rm neural-slam \
  python main.py --split val_mt --eval 1 --no_cuda \
  --auto_gpu_config 0 -n 14 --num_episodes 1 --num_processes_per_gpu 7 \
  --print_images 1 -d results/ --exp_name vis_test \
  --pretrained_resnet 0 \
  --load_global pretrained_models/model_best.global --train_global 0 \
  --load_local pretrained_models/model_best.local --train_local 0 \
  --load_slam pretrained_models/model_best.slam --train_slam 0
```

The PNGs are written under:

```text
results/dump/vis_test/episodes/
```

Use `--vis_type 1` for predicted map and pose, or `--vis_type 2` for the
ground-truth map and pose. Avoid `-v 1` unless GUI display forwarding is set
up.

Evaluation logs and dumps are written under the mounted `tmp/`, `saved/`, or `results/` folders depending on the command arguments.

The evaluation can be quiet while Habitat initializes scenes. Check whether it
is still doing work with:

```bash
docker ps
docker stats
nvidia-smi
tail -f tmp/models/exp1/train.log
```

If a terminal was killed directly, Docker containers can remain running in the
background. Stop stale runs before starting a new evaluation:

```bash
docker ps
docker stop <container_id>
docker compose down
```

## 8. Train From Scratch

Full training uses the default command:

```bash
docker compose run --rm neural-slam python main.py
```

For fewer GPUs or a quick run, disable auto GPU configuration and choose fewer processes:

```bash
docker compose run --rm neural-slam \
  python main.py --auto_gpu_config 0 -n 4 --num_processes_per_gpu 4
```

Training from scratch is expensive and the original setup assumes the full Gibson train split.
