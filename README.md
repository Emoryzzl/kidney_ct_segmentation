# Kidney CT Segmentation

Docker-based segmentation of kidneys, renal tumors, renal arteries, and renal veins from CT images, with GPU-accelerated inference and case-level distribution across multiple GPUs or machines. No Conda environment is required.

The project name is independent of the training framework. The model settings and commands below describe release **1.0.0**; future releases may use different architectures or inference options.

- **Docker Hub:** [zlzbme/kidney_ct_segmentation](https://hub.docker.com/r/zlzbme/kidney_ct_segmentation)
- **GitHub:** [Emoryzzl/kidney_ct_segmentation](https://github.com/Emoryzzl/kidney_ct_segmentation)
- **Published tags:** `1.0.0`, `latest`

All commands below use **Bash / WSL**. Examples use the versioned tag `1.0.0`.

## Model and segmentation labels

| Setting | Release 1.0.0 |
| --- | --- |
| Dataset | Dataset501_KiPA22 |
| Configuration | nnUNetTrainer / nnUNetPlans / 3d_fullres |
| Checkpoints | checkpoint_final.pth from folds 0, 1, 2, 3, and 4 |
| Inference | Five-fold ensemble, sliding-window step size 0.5, Gaussian weighting, mirroring |
| Input | Single-channel CT in compressed NIfTI format (.nii.gz) |
| Output | One multi-label segmentation (.nii.gz) per case |

| Label | Structure |
| --- | --- |
| 0 | Background |
| 1 | Renal vein |
| 2 | Kidney |
| 3 | Renal artery |
| 4 | Kidney tumor |

Place input files directly in the input directory. Both `case001.nii.gz` and `case001_0000.nii.gz` are supported, but do not include both names for the same case. Only channel suffix `_0000` is accepted. Subdirectories are not searched recursively. DICOM and uncompressed `.nii` files are not accepted directly.

Output filenames omit the input channel suffix: `case001_0000.nii.gz` produces `case001.nii.gz`.

## Requirements

- Linux, or Windows with WSL2 and the Docker Desktop Linux engine.
- A working Docker installation accessible from your terminal.
- For GPU inference: an NVIDIA GPU, a driver compatible with CUDA 12.6, and Docker GPU support. Native Linux hosts require NVIDIA Container Toolkit configuration.
- The image includes Python 3.11, PyTorch 2.7.1, inference dependencies, and the five model checkpoints.
- Memory and GPU memory requirements depend on the CT volume. The `--shm-size=8g` option sets shared memory capacity; it is not a statement of total RAM or GPU memory requirements.

Check the host environment:

```bash
docker version
nvidia-smi
```

## Pull the image

The image is public and can be pulled without requesting access. If anonymous pull limits are encountered, sign in to your own account with `docker login`.

```bash
export IMAGE="zlzbme/kidney_ct_segmentation:1.0.0"
docker pull "$IMAGE"
```

You do not need to clone this repository or download checkpoints separately to run the published image.

Verify GPU access inside the container:

```bash
docker run --rm --gpus all --entrypoint python "$IMAGE" -c \
  "import torch; print('PyTorch:', torch.__version__); print('CUDA:', torch.version.cuda); print('GPU available:', torch.cuda.is_available()); print(torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'NONE')"
```

Confirm that `GPU available: True` is printed before running GPU inference.

## Quick start

### 1. Set the input and output directories

Replace these paths with existing absolute paths on your host:

```bash
export INPUT="/absolute/path/to/ct"
export OUTPUT="/absolute/path/to/predictions"
mkdir -p "$OUTPUT"
find "$INPUT" -maxdepth 1 -type f -name '*.nii.gz' | sort
```

### 2. Validate the file list

```bash
docker run --rm \
  -v "$INPUT:/input:ro" \
  -v "$OUTPUT:/output" \
  "$IMAGE" --dry-run
```

This checks model file presence and input naming, and prints the selected cases and labels. It does not load checkpoint contents, validate NIfTI image contents, or test GPU execution.

### 3. Run five-fold inference

```bash
docker run --rm --gpus '"device=0"' --shm-size=8g \
  -v "$INPUT:/input:ro" \
  -v "$OUTPUT:/output" \
  "$IMAGE" --workers 1
```

The input directory is mounted read-only. Predictions are written to the host output directory.

### 4. Inspect the results

```bash
ls -lh "$OUTPUT"/*.nii.gz
```

Each file contains the segmentation labels listed above. Overlay a prediction on its original CT in a NIfTI viewer such as 3D Slicer. Model metadata and inference arguments are also written as JSON files.

## WSL example: three test cases

This example uses the local test directory shown below. Keep only the three intended CT files in its `ct` subdirectory: the container processes every matching file in that directory.

```bash
export IMAGE="zlzbme/kidney_ct_segmentation:1.0.0"
export INPUT="/mnt/d/Prostate_MRI_projects/Volupace/nnUNet/Datasets/KiPA22/test_docker/ct"
export OUTPUT="/mnt/d/Prostate_MRI_projects/Volupace/nnUNet/Datasets/KiPA22/test_docker/predictions"

mkdir -p "$OUTPUT"
find "$INPUT" -maxdepth 1 -type f -name '*.nii.gz' | sort

docker run --rm \
  -v "$INPUT:/input:ro" -v "$OUTPUT:/output" \
  "$IMAGE" --dry-run

docker run --rm --gpus '"device=0"' --shm-size=8g \
  -v "$INPUT:/input:ro" -v "$OUTPUT:/output" \
  "$IMAGE" --workers 1

ls -lh "$OUTPUT"/*.nii.gz
```

The corresponding Windows output directory is:

```text
D:\Prostate_MRI_projects\Volupace\nnUNet\Datasets\KiPA22\test_docker\predictions
```

## Inference options

```bash
docker run --rm zlzbme/kidney_ct_segmentation:1.0.0 --help
```

| Option | Default | Description |
| --- | --- | --- |
| `--input` | `/input` | Input directory inside the container |
| `--output` | `/output` | Output directory inside the container |
| `--model-dir` | `/app/models` | Model directory |
| `--folds` | `0 1 2 3 4` | Folds used for inference; changing these changes the ensemble |
| `--device` | `cuda` | `cuda` or `cpu` |
| `--workers` | `3` | Number of preprocessing workers and segmentation export workers, respectively |
| `--num-parts` | `1` | Total number of case partitions |
| `--part-id` | `0` | Zero-based partition index |
| `--continue-prediction` | Disabled | Skip existing predictions |
| `--save-probabilities` | Disabled | Save additional probability data; requires more disk space |
| `--dry-run` | Disabled | Check files and print the selected case list |

### Resume an interrupted run

Keep the image, inputs, output path, and partition settings unchanged:

```bash
docker run --rm --gpus '"device=0"' --shm-size=8g \
  -v "$INPUT:/input:ro" -v "$OUTPUT:/output" \
  "$IMAGE" --workers 1 --continue-prediction
```

Use a new output directory when changing the model or input images to avoid reusing stale results.

### CPU inference

CPU inference is supported but is slower. Omit `--gpus` and select the CPU explicitly:

```bash
docker run --rm --shm-size=8g \
  -v "$INPUT:/input:ro" -v "$OUTPUT:/output" \
  "$IMAGE" --device cpu --workers 1
```

## Multiple GPUs or machines

Each container processes a different subset of cases, and each assigned case still uses the complete five-fold ensemble. This is case-level partitioning, not synchronized inference for a single case across machines. Nodes are launched manually; no scheduler is included.

All containers must use:

- The same image and input case list.
- The same `--num-parts` value.
- A unique `--part-id` from `0` to `num-parts - 1`.

Case IDs are sorted and partitioned by index. Do not modify the input list during a distributed run.

For two GPUs, run the following commands in separate terminals. Set `IMAGE`, `INPUT`, and `OUTPUT` in **each terminal** first.

```bash
# GPU 0, partition 0
docker run --rm --gpus '"device=0"' --shm-size=8g \
  -v "$INPUT:/input:ro" -v "$OUTPUT:/output" \
  "$IMAGE" --num-parts 2 --part-id 0 --workers 1
```

```bash
# GPU 1, partition 1
docker run --rm --gpus '"device=1"' --shm-size=8g \
  -v "$INPUT:/input:ro" -v "$OUTPUT:/output" \
  "$IMAGE" --num-parts 2 --part-id 1 --workers 1
```

For multiple machines, mount identical input copies or shared storage on each host. Single-GPU hosts can all use `device=0`, while retaining distinct partition IDs.

Outputs are stored under `part_000/`, `part_001/`, and so on, preventing inference metadata from being overwritten by other partitions. Collect the segmentation NIfTI files from these directories when combining results; retain each partition's JSON files for traceability.

## Building from source

The following layout describes the local build context. A documentation-only checkout must be supplemented with the implementation files and model artifacts before building.

```text
.
├── Dockerfile
├── constraints.txt
├── predict.py
├── source/                   # Inference source and upstream license
├── checkpoint_manifest.json  # Checkpoint SHA256 hashes
└── models/                   # Model artifacts excluded from Git
    ├── dataset.json
    ├── plans.json
    ├── fold_0/checkpoint_final.pth
    ├── fold_1/checkpoint_final.pth
    ├── fold_2/checkpoint_final.pth
    ├── fold_3/checkpoint_final.pth
    └── fold_4/checkpoint_final.pth
```

Model checkpoints, test CT images, and exported Docker archives are excluded from Git. The published Docker image already includes the model files needed for inference.

From a complete build context:

```bash
docker build --progress=plain -t kidney_ct_segmentation:latest .
```

The Dockerfile installs dependencies using Python and pip without Conda. PyTorch and torchvision are pinned; other dependencies follow the source project's constraints. Inspect the installed dependency versions with:

```bash
docker run --rm --entrypoint cat \
  kidney_ct_segmentation:latest /app/installed-requirements.txt
```

## Publishing an image

The following commands require push access to `zlzbme/kidney_ct_segmentation`. Other publishers should use their own namespace.

```bash
export NAMESPACE="zlzbme"
export VERSION="1.0.0"

docker login

docker tag kidney_ct_segmentation:latest \
  "$NAMESPACE/kidney_ct_segmentation:$VERSION"
docker tag kidney_ct_segmentation:latest \
  "$NAMESPACE/kidney_ct_segmentation:latest"

docker push "$NAMESPACE/kidney_ct_segmentation:$VERSION"
docker push "$NAMESPACE/kidney_ct_segmentation:latest"
```

Use a new version tag for future releases. For identical deployments across nodes, retrieve the pulled image's digest:

```bash
docker image inspect zlzbme/kidney_ct_segmentation:1.0.0 \
  --format '{{index .RepoDigests 0}}'
```

Set `IMAGE` to the complete returned `zlzbme/kidney_ct_segmentation@sha256:...` address on every node.

## Offline export and import

Export the published image after pulling it:

```bash
docker save -o kidney_ct_segmentation.tar \
  zlzbme/kidney_ct_segmentation:1.0.0
```

Transfer the archive to the target host, then import it:

```bash
docker load -i kidney_ct_segmentation.tar
export IMAGE="zlzbme/kidney_ct_segmentation:1.0.0"
```

Run inference using the same commands shown above. Loading the image does not configure host GPU support.

## Troubleshooting

| Problem | Suggested checks |
| --- | --- |
| Cannot connect to the Docker daemon | Start Docker Desktop, enable integration for your WSL distribution, and run docker version |
| Pull access denied or image not found | Check the namespace, tag, and network connection; sign in if pull limits are reached |
| CUDA unavailable | Check host nvidia-smi, Docker GPU configuration, and the --gpus option |
| GPU out of memory | Stop competing GPU processes or use CPU inference |
| Preprocessing worker exits or insufficient RAM | Use --workers 1 and inspect host RAM, container limits, and shared memory |
| No .nii.gz inputs found | Check the host mount path and place files directly in the input directory |
| Duplicate case ID | Keep only one supported filename per case |
| Cannot write predictions | Check write permissions on the host output directory |

## Implementation and attribution

Release 1.0.0 uses [nnU-Net](https://github.com/MIC-DKFZ/nnUNet). The upstream source license is retained in `source/LICENSE` in the build context. Model weight usage and redistribution terms must be considered separately from the source code license.
