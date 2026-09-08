# Kidney CT Segmentation

Automated segmentation of kidneys, renal tumors, renal arteries, and renal veins from CT images. The Docker image includes the trained model and all inference dependencies, with GPU acceleration and no local Python or Conda setup required.

**Docker image:** [zlzbme/kidney_ct_segmentation](https://hub.docker.com/r/zlzbme/kidney_ct_segmentation)

## Requirements

- Linux or Windows with WSL2 and Docker Desktop.
- An NVIDIA GPU with Docker GPU support for accelerated inference.
- Single-channel CT images in compressed NIfTI format (`.nii.gz`).

Place all CT files directly in one input folder. Supported filenames are `case001.nii.gz` or `case001_0000.nii.gz`; include only one file per case. DICOM input is not supported directly.

All commands below run in **Bash / WSL**.

## Run segmentation

### 1. Download the image

```bash
export IMAGE="zlzbme/kidney_ct_segmentation:1.0.0"
docker pull "$IMAGE"
```

### 2. Set your input and output folders

Replace the paths below with your own absolute paths. In WSL, a Windows path such as `D:\data\ct` becomes `/mnt/d/data/ct`.

```bash
export INPUT="/mnt/d/data/ct"
export OUTPUT="/mnt/d/data/predictions"
mkdir -p "$OUTPUT"
```

### 3. Run inference

```bash
docker run --rm --gpus '"device=0"' --shm-size=8g \
  -v "$INPUT:/input:ro" \
  -v "$OUTPUT:/output" \
  "$IMAGE" --workers 1
```

This processes all `.nii.gz` files in the input folder using the five-fold model ensemble. Predictions are saved to your output folder. Input images are mounted read-only.

## Output

Each case produces one segmentation file, such as `case001.nii.gz`, with the following labels:

| Value | Structure |
| --- | --- |
| 0 | Background |
| 1 | Renal vein |
| 2 | Kidney |
| 3 | Renal artery |
| 4 | Kidney tumor |

Open the original CT and its segmentation in a viewer such as **3D Slicer** to inspect the overlay. Additional JSON files record the model configuration and inference settings.

## Useful options

Append these options to the inference command as needed:

| Option | Purpose |
| --- | --- |
| `--dry-run` | Check input filenames and display the case list without running inference |
| `--continue-prediction` | Skip existing predictions when resuming the same run |

For CPU inference, remove `--gpus '"device=0"'` and add `--device cpu`. CPU processing is slower.

If GPU inference fails, check that `nvidia-smi` works on the host and that Docker has GPU access. To view all available options:

```bash
docker run --rm "$IMAGE" --help
```
