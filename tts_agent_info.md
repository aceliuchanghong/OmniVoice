# OmniVoice 本地 TTS 调用手册（供 Agent 使用）

## 固定环境信息

- 项目目录：`C:\Users\lawrence\PycharmProjects\OmniVoice`
- Python：`C:\Users\lawrence\PycharmProjects\OmniVoice\.venv\Scripts\python.exe`
- 本地模型：`C:\Users\lawrence\PycharmProjects\Models\OmniVoice`
- 输出格式：单声道 WAV，采样率 `24000 Hz`
- 当前机器已检测到 CUDA GPU；依赖已经安装，无需再次安装或下载模型。

执行命令前先进入项目目录：

```powershell
Set-Location C:\Users\lawrence\PycharmProjects\OmniVoice
```

## 最简单的调用：CLI

自动选择音色并生成中文语音：

```powershell
.\.venv\Scripts\omnivoice-infer.exe `
  --model "C:\Users\lawrence\PycharmProjects\Models\OmniVoice" `
  --device cuda:0 `
  --language zh `
  --text "你好，这是 OmniVoice 本地语音合成测试。" `
  --output "output.wav"
```

生成完成后，文件位于当前目录的 `output.wav`。

### 指定声音特征

不需要参考音频；用 `--instruct` 描述音色。声音设计模式主要针对中文和英文训练：

```powershell
.\.venv\Scripts\omnivoice-infer.exe `
  --model "C:\Users\lawrence\PycharmProjects\Models\OmniVoice" `
  --device cuda:0 `
  --language zh `
  --text "欢迎使用本地文本转语音服务。" `
  --instruct "female, young adult, low pitch" `
  --output "designed_voice.wav"
```

### 克隆参考音色

参考音频建议为清晰、单人、无背景音乐的 `3～10` 秒音频。`--ref_text` 必须准确对应参考音频内容：

```powershell
.\.venv\Scripts\omnivoice-infer.exe `
  --model "C:\Users\lawrence\PycharmProjects\Models\OmniVoice" `
  --device cuda:0 `
  --language zh `
  --text "这是用参考音色合成的新句子。" `
  --ref_audio "C:\path\to\reference.wav" `
  --ref_text "参考音频中实际说出的文字。" `
  --output "cloned_voice.wav"
```

离线调用时不要省略 `ref_text`。省略后程序会尝试使用 Whisper 自动转写，可能需要额外下载 ASR 模型。

## Python API（适合在程序中复用模型）

一次加载模型，多次调用 `generate()`，不要为每一句话重复加载权重：

```python
import soundfile as sf
import torch
from omnivoice import OmniVoice

MODEL_PATH = r"C:\Users\lawrence\PycharmProjects\Models\OmniVoice"

model = OmniVoice.from_pretrained(
    MODEL_PATH,
    device_map="cuda:0",
    dtype=torch.float16,
)

# 自动音色；指定 language 通常效果更好。
audios = model.generate(
    text="你好，这是通过 Python API 生成的语音。",
    language="zh",
    num_step=32,
    speed=1.0,
)

# generate() 返回 list[np.ndarray]；单条输入取 audios[0]。
sf.write("python_output.wav", audios[0], model.sampling_rate)
```

三种模式只需改变 `generate()` 参数：

```python
# 1. 自动音色
audio = model.generate(text="测试文本", language="zh")

# 2. 声音设计
audio = model.generate(
    text="测试文本",
    language="zh",
    instruct="female, low pitch",
)

# 3. 音色克隆
audio = model.generate(
    text="测试文本",
    language="zh",
    ref_audio=r"C:\path\to\reference.wav",
    ref_text="参考音频的准确文字稿。",
)
```

### 复用克隆音色

同一参考音色会反复使用时，先编码并保存 prompt，可避免每次重新处理参考音频：

```python
from omnivoice import VoiceClonePrompt

# 首次创建
prompt = model.create_voice_clone_prompt(
    ref_audio=r"C:\path\to\reference.wav",
    ref_text="参考音频的准确文字稿。",
)
prompt.save("my_voice.pt")

# 后续进程中加载（model 仍需正常加载）
prompt = VoiceClonePrompt.load("my_voice.pt")
audio = model.generate(
    text="使用已保存音色生成的新文本。",
    language="zh",
    voice_clone_prompt=prompt,
)
sf.write("reused_voice.wav", audio[0], model.sampling_rate)
```

## 常用参数

| 参数 | 默认值 | 作用 |
|---|---:|---|
| `language` / `--language` | `None` | 语言名称或代码，如 `zh`、`en`；建议明确传入 |
| `num_step` / `--num_step` | `32` | 扩散步数；`32` 质量优先，`16` 更快 |
| `speed` / `--speed` | `1.0` | 大于 `1` 更快，小于 `1` 更慢 |
| `duration` / `--duration` | 自动估计 | 强制输出时长（秒）；设置后优先于 `speed` |
| `guidance_scale` / `--guidance_scale` | `2.0` | 引导强度；一般保留默认值 |

长文本会自动分段生成再拼接。默认参数通常足够，不确定时只设置 `language`、`text` 和所需的音色参数。

## Agent 调用约定

1. 始终使用上述本地模型绝对路径，不要写 Hugging Face 仓库名，以免触发联网下载。
2. 优先使用项目现有 `.venv`，不要重新安装依赖。
3. 批量生成时用 Python API 加载一次模型并循环调用，避免反复加载约 2.45 GB 的权重。
4. 克隆模式必须同时提供 `ref_audio` 和准确的 `ref_text`，以保持纯本地执行。
5. `generate()` 返回数组列表；单句保存 `audios[0]`，采样率使用 `model.sampling_rate`（当前为 24000）。
6. 中文或英文数字若读音不理想，先在文本中把数字改写成文字；不要假定文本正规化扩展一定可用。
7. 只在用户明确要求固定时长时传 `duration`；否则让模型自动估计，通常更自然。

## 快速排错

- 显存不足：先关闭占用 GPU 的进程，并改用 `num_step=16`；不要并行加载多个模型实例。
- 克隆结果不像参考音色：换用清晰的 3～10 秒单人录音，并检查 `ref_text` 是否逐字匹配。
- 跨语言克隆有口音：尽量让参考音频与目标文本使用相同语言。
- 命令找不到：使用本文给出的 `.\.venv\Scripts\omnivoice-infer.exe`，不要依赖全局 PATH。
- 查看全部 CLI 参数：`.\.venv\Scripts\omnivoice-infer.exe --help`。

主要依据：仓库根目录 `README.md`、`docs/generation-parameters.md` 与 `omnivoice/cli/infer.py`。
