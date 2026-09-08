# Kidney CT Segmentation

用于 CT 肾脏、肾肿瘤、肾动脉和肾静脉自动分割的 Docker 推理项目，支持 GPU 加速及多机病例分片，无需 Conda。项目名称不绑定训练框架；以下参数与模型配置适用于当前 1.0.0 版本。

## 模型和输入输出

| 项目 | 设置 |
| --- | --- |
| 数据集 | Dataset501_KiPA22 |
| 配置 | nnUNetTrainer / nnUNetPlans / 3d_fullres |
| 权重 | fold 0、1、2、3、4 的 checkpoint_final.pth |
| 推理 | 五折集成，滑窗步长 0.5，高斯加权，镜像增强 |
| 输入 | 单通道 CT，`.nii.gz` |
| 输出 | 每病例一个多标签 `.nii.gz` 分割文件 |

| 标签值 | 结构 |
| --- | --- |
| 0 | 背景 |
| 1 | 肾静脉 |
| 2 | 肾脏 |
| 3 | 肾动脉 |
| 4 | 肾肿瘤 |

输入文件直接放在输入目录下，支持 `case001.nii.gz` 或 `case001_0000.nii.gz`。同一病例不能同时存在这两种命名；不接受其他通道编号。入口不递归搜索子目录，也不直接接受 DICOM 或未压缩 `.nii` 文件。输出病例名会去掉输入的 `_0000` 后缀。

## 环境要求

- Linux，或 Windows WSL2 + Docker Desktop Linux engine；WSL 中需能执行 `docker version`。
- GPU 推理需要 NVIDIA GPU、兼容镜像 CUDA 12.6 的驱动，以及 Docker GPU 支持。Linux 主机需配置 NVIDIA Container Toolkit。
- 镜像包含 Python 3.11、PyTorch 2.7.1 和五折权重。
- 显存、内存需求取决于 CT 大小；示例设置 8 GB 共享内存，不代表总内存或显存要求。

以下命令均在 Bash / WSL 中运行。镜像地址：[zlzbme/kidney_ct_segmentation](https://hub.docker.com/r/zlzbme/kidney_ct_segmentation)，已发布标签 `1.0.0` 和 `latest`。示例固定使用 `1.0.0`；未来版本的框架和参数可能变化，请查看对应版本说明。

## 获取公开镜像

镜像公开，可直接拉取。无需申请仓库权限；如遇匿名拉取限额，可运行 `docker login` 登录自己的账号。

```bash
export IMAGE="zlzbme/kidney_ct_segmentation:1.0.0"
docker pull "$IMAGE"
```

镜像内包含推理依赖和模型权重，无需克隆源码或单独下载 checkpoint。

检查 GPU：

```bash
docker run --rm --gpus all --entrypoint python "$IMAGE" -c "import torch; print('PyTorch:', torch.__version__); print('CUDA:', torch.version.cuda); print('GPU available:', torch.cuda.is_available()); print(torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'NONE')"
```

## 快速推理

使用实际绝对路径设置输入输出目录：

```bash
export INPUT="/absolute/path/to/ct"
export OUTPUT="/absolute/path/to/predictions"
mkdir -p "$OUTPUT"
find "$INPUT" -maxdepth 1 -type f -name '*.nii.gz' | sort
```

仅检查输入、模型文件和病例清单：

```bash
docker run --rm \
  -v "$INPUT:/input:ro" -v "$OUTPUT:/output" \
  "$IMAGE" --dry-run
```

`--dry-run` 不加载权重，也不验证 GPU 或 NIfTI 内容是否有效。

完整五折推理：

```bash
docker run --rm --gpus '"device=0"' --shm-size=8g \
  -v "$INPUT:/input:ro" -v "$OUTPUT:/output" \
  "$IMAGE" --workers 1
```

查看结果：

```bash
ls -lh "$OUTPUT"/*.nii.gz
```

可用 3D Slicer 将预测标签叠加到原始 CT 查看。推理也会写出模型配置和调用参数 JSON。

### WSL 三病例测试路径示例

```bash
export IMAGE="zlzbme/kidney_ct_segmentation:1.0.0"
export INPUT="/mnt/d/Prostate_MRI_projects/Volupace/nnUNet/Datasets/KiPA22/test_docker/ct"
export OUTPUT="/mnt/d/Prostate_MRI_projects/Volupace/nnUNet/Datasets/KiPA22/test_docker/predictions"
mkdir -p "$OUTPUT"
docker run --rm --gpus '"device=0"' --shm-size=8g \
  -v "$INPUT:/input:ro" -v "$OUTPUT:/output" \
  "$IMAGE" --workers 1
```

容器会处理该目录下所有符合命名规则的文件；只放入需要测试的三个病例。

## 常用参数

查看完整帮助：

```bash
docker run --rm zlzbme/kidney_ct_segmentation:1.0.0 --help
```

中断后继续（保持模型、输入和分片设置一致）：

```bash
docker run --rm --gpus '"device=0"' --shm-size=8g \
  -v "$INPUT:/input:ro" -v "$OUTPUT:/output" \
  "$IMAGE" --workers 1 --continue-prediction
```

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `--input` | `/input` | 容器内输入目录 |
| `--output` | `/output` | 容器内输出目录 |
| `--model-dir` | `/app/models` | 模型目录 |
| `--folds` | `0 1 2 3 4` | 参与集成的折；减少折数会改变结果 |
| `--device` | `cuda` | `cuda` 或 `cpu` |
| `--workers` | `3` | 预处理和分割导出各自的进程数 |
| `--num-parts` | `1` | 病例分片总数 |
| `--part-id` | `0` | 当前分片编号，从 0 开始 |
| `--continue-prediction` | 关闭 | 跳过已有结果；更换输入或模型后应使用新输出目录 |
| `--save-probabilities` | 关闭 | 额外保存概率数据，占用更多磁盘空间 |
| `--dry-run` | 关闭 | 检查文件及显示病例清单 |

CPU 推理示例（速度较慢）：

```bash
docker run --rm --shm-size=8g \
  -v "$INPUT:/input:ro" -v "$OUTPUT:/output" \
  "$IMAGE" --device cpu --workers 1
```

## 多 GPU / 多机分片

所有容器使用相同镜像、相同输入病例清单和相同 `--num-parts`，每个容器分配唯一 `--part-id`。按病例 ID 排序后分片，每个病例仍执行完整五折集成。不自动调度节点，不跨节点同步单病例计算。

两个终端分别运行；每个终端都需先设置 IMAGE、INPUT、OUTPUT：

```bash
# GPU 0 / 分片 0
docker run --rm --gpus '"device=0"' --shm-size=8g \
  -v "$INPUT:/input:ro" -v "$OUTPUT:/output" \
  "$IMAGE" --num-parts 2 --part-id 0 --workers 1
```

```bash
# GPU 1 / 分片 1
docker run --rm --gpus '"device=1"' --shm-size=8g \
  -v "$INPUT:/input:ro" -v "$OUTPUT:/output" \
  "$IMAGE" --num-parts 2 --part-id 1 --workers 1
```

多机运行时，每台机器分别设置 IMAGE、INPUT、OUTPUT；单 GPU 节点均可使用 device=0，但分片编号必须不同。运行期间保持输入清单不变。

结果分别保存在 `part_000/` 和 `part_001/`。汇总时收集各分片的 NIfTI 结果，保留分片 JSON 供追溯。

## 源码构建

```text
.
├── Dockerfile
├── constraints.txt
├── predict.py
├── source/                   # nnunetv2 源码和原项目许可证
├── checkpoint_manifest.json  # 权重 SHA256
└── models/                   # 不提交 Git；构建前准备模型文件
    ├── dataset.json
    ├── plans.json
    ├── fold_0/checkpoint_final.pth
    ├── fold_1/checkpoint_final.pth
    ├── fold_2/checkpoint_final.pth
    ├── fold_3/checkpoint_final.pth
    └── fold_4/checkpoint_final.pth
```

GitHub 源码仓库不包含训练权重、测试 CT 或镜像 tar。克隆源码后必须另外准备上述模型文件才能构建；用户可直接拉取完整公开镜像进行推理。

```bash
docker build --progress=plain -t kidney_ct_segmentation:latest .
```

构建使用 pip 安装依赖，无需 Conda。PyTorch 和 torchvision 固定版本，其他依赖遵循源项目约束。查看已构建镜像的依赖版本：

```bash
docker run --rm --entrypoint cat kidney_ct_segmentation:latest /app/installed-requirements.txt
```

## 镜像发布与离线分发

维护者向公开仓库发布镜像（需要该仓库的 push 权限）：

```bash
export NAMESPACE="zlzbme"
docker login
docker tag kidney_ct_segmentation:latest "$NAMESPACE/kidney_ct_segmentation:1.0.0"
docker tag kidney_ct_segmentation:latest "$NAMESPACE/kidney_ct_segmentation:latest"
docker push "$NAMESPACE/kidney_ct_segmentation:1.0.0"
docker push "$NAMESPACE/kidney_ct_segmentation:latest"
```

保留版本标签；后续发布使用新的版本号。跨节点需要严格一致时，可按发布镜像的 digest 拉取并运行。

查询已拉取镜像的完整 digest：

```bash
docker image inspect zlzbme/kidney_ct_segmentation:1.0.0 \
  --format '{{index .RepoDigests 0}}'
```

将输出的完整 `zlzbme/kidney_ct_segmentation@sha256:...` 地址设为 IMAGE，再在各节点使用相同地址运行。

离线导出 / 导入：

```bash
docker save -o kidney_ct_segmentation.tar kidney_ct_segmentation:latest
docker load -i kidney_ct_segmentation.tar
```

## 常见问题

| 问题 | 检查方式 |
| --- | --- |
| 无法连接 Docker daemon | 启动 Docker Desktop，启用对应 WSL 发行版集成，检查 docker version |
| pull access denied | 检查镜像命名空间、版本标签和网络；如遇拉取限额则 docker login |
| CUDA unavailable | 检查宿主机 nvidia-smi、Docker GPU 配置及 --gpus 参数 |
| 显存不足 | 停止同 GPU 上其他任务；必要时使用 CPU 推理 |
| 预处理进程退出或内存不足 | 使用 --workers 1 并检查主机内存、容器限制及共享内存 |
| No .nii.gz inputs | 检查挂载源路径；文件应直接位于输入目录 |
| duplicate case ID | 同病例仅保留一种命名格式 |
| 输出权限错误 | 确认宿主机输出目录允许容器写入 |

## 来源

基于 [nnU-Net](https://github.com/MIC-DKFZ/nnUNet)。源代码保留上游许可证 `source/LICENSE`；模型权重的使用和再分发授权需单独确认，不能由源码许可证推定。
