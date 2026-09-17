# MuseTalk Local 安装踩坑实录（RTX 5090 / 国内网络）

本文是 [MuseTalk Local 单机部署](./local.md) 的实战补充，整理自 Ubuntu + RTX 5090 上的真实排障过程。按官方 Local 文档走完后若仍卡住，优先对照本文。

正式步骤仍以 [local.md](./local.md) 为准；本文只写**容易踩坑的分叉点**和**已验证绕过方案**。

## 0. 先选对路线

| 场景 | 建议 | 原因 |
|------|------|------|
| 单机验证、已有 OpenTalking QuickTalk local | **MuseTalk Local** | 推理走主 `.venv`（较新 Torch），更贴近现有部署 |
| RTX 5090 / Blackwell（sm_120） | **不要先上 OmniRT MuseTalk** | OmniRT MuseTalk runtime 锁死 **PyTorch 2.0.1 + cu118**，在 5090 上 CUDA 极易挂死 |
| 要进程隔离、多模型共用 gateway | OmniRT | 见 [omnirt.md](./omnirt.md)；5090 需另解预处理/推理 CUDA 兼容问题 |

## 1. 目录不要混

三个根目录必须分开，**不要互相软链**：

```text
$DIGITAL_HUMAN_HOME/
  models/                 # 权重（OPENTALKING_MUSETALK_MODEL_ROOT）
  model-repos/MuseTalk/   # 官方源码 checkout（OPENTALKING_MUSETALK_REPO）
  runtimes/musetalk-preprocess/venv/  # 独立预处理 Python
  opentalking/            # 本仓库
```

常见错法：把 `model-repos/MuseTalk` 指到权重目录，或把权重软链进源码树。

## 2. 权重：命名、断链、国内下载

### 2.1 官方脚本命名 ≠ OpenTalking 期望名

| 官方常见名 | OpenTalking 需要 |
|------------|------------------|
| `sd-vae` | `sd-vae-ft-mse` |
| `face-parse-bisent` | `face-parse-bisenet` |

可软链，但软链目标必须是**真实目录**，不要留下断链。

### 2.2 Whisper

必须是 OpenAI whisper 的 `whisper/tiny.pt`，不要用 Hugging Face `pytorch_model.bin` 改名顶替。

### 2.3 VAE 缺 `config.json`

只下了 `diffusion_pytorch_model.*` 不够，`sd-vae-ft-mse/config.json` 必须在。推荐：

```bash
export HF_ENDPOINT="${HF_ENDPOINT:-https://hf-mirror.com}"
hf download stabilityai/sd-vae-ft-mse \
  --local-dir "$OPENTALKING_MUSETALK_MODEL_ROOT/sd-vae-ft-mse" \
  --include "config.json" "diffusion_pytorch_model.bin" "diffusion_pytorch_model.safetensors"
```

### 2.4 `gdown` / Google Drive 在国内常挂死

- 新版 `gdown` **没有** `--id`，文件 ID 作位置参数：`gdown <FILE_ID> -O out.pth`
- 连不上 Google 时不要空等：改走 HuggingFace 镜像下 `79999_iter.pth`，`resnet18-5c106cde.pth` 用 `download.pytorch.org`

## 3. 环境与准备脚本

### 3.1 主环境是 uv，不是裸 pip

`prepare_local_musetalk.sh` 会调 `python -m pip`。主 `.venv` 若没装 pip 会直接失败。先执行：

```bash
cd "$OPENTALKING_HOME"
uv sync --extra models --extra dev --python 3.11
uv pip install --python .venv/bin/python pip "setuptools<81" openmim
bash scripts/quickstart/prepare_local_musetalk.sh
```

### 3.2 推理 venv ≠ 预处理 venv

| 用途 | Python | 说明 |
|------|--------|------|
| 实时推理 | OpenTalking `.venv` | 较新 Torch，可跑 5090 |
| 官方头像预处理（mmcv / DWPose） | `runtimes/musetalk-preprocess/venv` | 常锁 cu118 + 旧 Torch |

**不要**把 `OPENTALKING_MUSETALK_PREPROCESS_PYTHON` 指到主 `.venv`（通常没有完整 `mmcv._ext`）。

### 3.3 5090 上官方预处理容易卡死

症状：`prepare_musetalk_avatar_asset.py` 打出 UNet 警告后长时间无进展，GPU 利用率 ≈ 0%，`prepared/` 不增长。

根因常见组合：

1. 预处理 venv 的 `torch 2.0.1+cu118` 与 Blackwell 不兼容  
2. 同卡还有 LiveTalking / 其它进程占十几 GB 显存  
3. WebUI 自动 prepare 与手动 prepare **双开**抢 GPU  

验证预处理环境能否真正跑 CUDA：

```bash
"$OPENTALKING_MUSETALK_PREPROCESS_PYTHON" - <<'PY'
import torch
print(torch.__version__, torch.version.cuda, torch.cuda.is_available())
x = torch.randn(1024, 1024, device="cuda")
y = x @ x
torch.cuda.synchronize()
print("CUDA_OK", float(y.mean()))
PY
```

若 `CUDA_FAIL` 或长时间无输出：不要死磕官方 mmcv 预处理。

### 3.4 推荐绕过：复用 LiveTalking `genavatar` 产物

OpenTalking MuseTalk local 可读 LiveTalking 风格的 `prepared/`（`full_imgs`、`coords.pkl`、`latents.pt`、`mask` 等）。

实操：

1. 用 LiveTalking 的 `genavatar`（依赖 face_alignment / face_recognition，**不依赖 mmcv**）生成 avatar  
2. 拷到 `examples/avatars/<id>/prepared/`  
3. 启动会话前设置 **`AUTO_PREPARE=0`**（或等价配置），禁止 OpenTalking 再跑官方 prepare（否则会覆盖/拒绝 fallback）

目录名必须等于 `manifest.id`（见下一节）。

## 4. Avatar 资产坑

### 4.1 目录名必须等于 avatar id

错误示例：

```bash
# 错：目录 my-musetalk，id 却写成 avatar-nurse
--out examples/avatars/my-musetalk --avatar-id avatar-nurse
```

结果：WebUI 有名字、预览 404。正确做法：`--out examples/avatars/<id>` 与 `--avatar-id <id>` **一致**。

### 4.2 两个脚本是前后两步，不是二选一

| 脚本 | 输入 | 输出 |
|------|------|------|
| `prepare_wav2lip_video_asset.py` | 视频 | 通用 Avatar（`frames/`、`manifest.json`） |
| `prepare_musetalk_avatar_asset.py` | **已有** Avatar 目录 | `prepared/` |

`prepare_musetalk_avatar_asset.py` **不能直接吃 mp4**。

### 4.3 改完 `.env` 必须停干净旧 API

`start_unified.sh` 若复用旧 PID，不会吃到新的 `OPENTALKING_MUSETALK_BACKEND=local`。先：

```bash
bash scripts/quickstart/stop_all.sh
# 再 start_unified.sh
```

## 5. WebRTC 黑屏 / ICE 失败

MuseTalk 接通但舞台空白，优先查 ICE，不要先怀疑模型。

- 公网 / 跨网访问常需要 **TURN**（仅 STUN 不够）  
- 在 `.env` 配好转发，重启 API 后再测会话  
- 用浏览器 `chrome://webrtc-internals` 看 candidate 是否成对连通

## 6. 装完 MuseTalk 后 QuickTalk 挂了：`onnxruntime` 无 `InferenceSession`

### 症状

```text
failed to prewarm local quicktalk: module 'onnxruntime' has no attribute 'InferenceSession'
```

诊断：

```bash
python -c "import onnxruntime as ort; print(ort.__file__, hasattr(ort,'InferenceSession'))"
# 坏包常见：__file__ is None，hasattr -> False
ls .venv/lib/python3.11/site-packages/onnxruntime
# 坏包常见：只剩 capi/ quantization/ transformers/，没有 __init__.py
```

### 修复（注意钉回 numpy / protobuf）

```bash
python -m pip uninstall -y onnxruntime
rm -rf .venv/lib/python3.11/site-packages/onnxruntime
python -m pip install --force-reinstall --no-cache-dir "onnxruntime==1.19.2"
# 上一步常会把 numpy/protobuf 拉飞，必须钉回：
python -m pip install --force-reinstall "numpy==1.26.4" "protobuf==4.25.9"
python -c "import onnxruntime as ort, numpy; print(ort.__version__, hasattr(ort,'InferenceSession'), numpy.__version__)"
```

期望类似：`1.19.2 True 1.26.4`。然后**重启** OpenTalking。

> 不要只 `pip install onnxruntime` 完事：`--force-reinstall` 可能升到 `numpy 2.x` / `protobuf 7.x`，与 `opentalking`、`mediapipe` 冲突。

## 7. GPU 绑定

多卡时：

```bash
export CUDA_VISIBLE_DEVICES=1          # 选物理 GPU1
export OPENTALKING_MUSETALK_DEVICE=cuda:0
export OPENTALKING_TORCH_DEVICE=cuda:0
```

`CUDA_VISIBLE_DEVICES` 之后，进程内看到的永远是 `cuda:0`。

预处理期间建议独占一张卡：停掉同卡 LiveTalking / ComfyUI，且只跑一个 prepare。

## 8. 建议验收清单

1. `curl http://127.0.0.1:8000/models` → `musetalk`：`backend=local, connected=true`  
2. Avatar 目录名 = `manifest.id`，且存在可用 `prepared/`（或关闭自动 prepare 的旁路资产）  
3. 短文本对话：有口型、WebRTC 有画面  
4. 切回 QuickTalk：prewarm 不再报 `InferenceSession`  
5. （可选）沉浸模式竖屏全屏：前端需 `immersiveFill` / `object-cover`；超高竖屏（如 1632×3840）用工作台「微调」校正裁切

## 9. 相关文档

- [MuseTalk Local 单机部署](./local.md)
- [MuseTalk OmniRT](./omnirt.md)
- [MuseTalk 模型概述](../../avatar_models/musetalk.md)
- [会说话头像部署索引](../talking-head/index.md)
