# Qwen Image Studio

> 本地跑 Qwen-Image-2.1 Turbo 的一体化出图程序：文生图 + 指令式图片编辑，双击即用，不装环境、不联网、不依赖 ComfyUI。

![界面预览](docs/preview.png)

## 这是什么

一个 Windows 桌面程序（`QwenImageStudio.exe`），把三件事打包在一起：

1. **自带出图引擎** —— 内置 [stable-diffusion.cpp](https://github.com/leejet/stable-diffusion.cpp) 的 `sd-server.exe`，程序启动时自动拉起，退出时自动关闭；
2. **自带网页界面** —— 启动后自动打开浏览器，界面服务在 `127.0.0.1:7860`，所有请求经程序转发到引擎的 `127.0.0.1:8081`，不需要处理跨域；
3. **模型自备** —— 权重不随包分发，放到 `models\` 即可，程序按文件名自动识别。

## 功能

- **文生图**：提示词 → 图片，8 步出图（Turbo 蒸馏模型）
- **图片编辑**：上传一张图 + 描述要改什么，保留原图其余内容（走引擎的 `ref_images` 视觉通路，需要 mmproj 权重）
- **尺寸**：1K / 1536 / 2048 / 4K，8 种画幅比例，也可只手填宽高（256–4096，自动对齐 32）
- **参数**：采样步数、CFG、Seed
- **界面主题**：灯光（浅色）/ 夜晚（深色），记在浏览器本地
- **任务控制**：排队/生成进度、运行日志、生成中可取消
- **结果**：下载 PNG，或一键作为参考图继续编辑
- **设置**：服务地址、默认采样参数、重启引擎

## 系统要求

| 项 | 要求 |
|---|---|
| 系统 | Windows 10 / 11（x64） |
| 显卡 | NVIDIA，建议 **8 GB 显存以上**，需装好较新的 NVIDIA 驱动（CUDA 12 运行时随包内置） |
| 内存 | 建议 16 GB 以上（8 GB 显存 + `--offload-to-cpu` 会把权重放内存） |
| 磁盘 | 程序约 1.2 GB，模型另需约 12.8 GB |
| 网络 | 运行期**不需要**，仅下载模型时需要 |

实测参考（RTX 4060 Laptop 8 GB）：512×512 / 4 步约 25 秒；1024×1024 / 8 步几分钟；4K 不推荐。

## 快速开始

1. 解压压缩包到任意目录（**路径别太长、别放中文路径**）。
2. 把下面四个模型放进 `models\`：

   | 文件 | 大小 | 作用 | 来源 |
   |---|---|---|---|
   | `Qwen-Image-2.1-Turbo-AD-Q6_K.gguf` | 6.25 GB | 扩散模型 | [AtomicChat/Qwen-Image-2.1-Turbo-GGUF](https://huggingface.co/AtomicChat/Qwen-Image-2.1-Turbo-GGUF) |
   | `Qwen3-VL-8B-Instruct-UD-Q4_K_XL.gguf` | 4.80 GB | 文本编码器 | 同上仓库或 [Unsloth](https://huggingface.co/unsloth) 系列 |
   | `qwen_image_2.1_vae_bf16.safetensors` | 0.63 GB | VAE | [Comfy-Org/Qwen-Image-2.1](https://huggingface.co/Comfy-Org/Qwen-Image-2.1/tree/main/vae) |
   | `mmproj-Qwen-Image-2.1-Turbo-Uncensored-BF16.gguf` | 1.08 GB | 视觉投影（**图片编辑必需**） | [AtomicChat/Qwen-Image-2.1-Turbo-Uncensored-GGUF](https://huggingface.co/AtomicChat/Qwen-Image-2.1-Turbo-Uncensored-GGUF) |

3. 双击 `QwenImageStudio.exe`。
   黑色窗口会先加载权重（首次几分钟，之后因为系统缓存会快很多），随后自动打开 <http://127.0.0.1:7860/>。

关掉黑色窗口 = 退出程序，引擎会一并结束。

## 目录结构

```
QwenImageStudio\
├─ QwenImageStudio.exe     程序本体（引导器）
├─ _internal\              Python 运行时与 web\出图.html
├─ engine\                 sd-server.exe 及 DLL（勿删）
├─ models\                 权重放这里（自备）
├─ output\                 预留输出目录
├─ config.json             端口与引擎参数
└─ README.txt              简版说明
```

## 配置

`config.json`：

```json
{
  "port": 7860,
  "engine_port": 8081,
  "open_browser": true,
  "extra_engine_args": [
    "--steps", "8",
    "--cfg-scale", "1",
    "--sampling-method", "euler",
    "--sigmas", "qwen-image-2.1-turbo",
    "--offload-to-cpu",
    "--flash-attn", "off,diffusion=on"
  ]
}
```

- `port` 被占用时改这里；`engine_port` 一般不用动。
- 显存宽裕（12 GB+）可以把 `--offload-to-cpu` 去掉，出图明显更快。
- 改完重启程序生效。

## 接口

程序在 `127.0.0.1` 上暴露以下路由，方便脚本化调用：

```http
GET  /api/engine/status           # 引擎状态、识别到的权重、缺失项
POST /api/engine/start | /stop    # 重启引擎

POST /sdcpp/v1/img_gen            # 提交生成任务 → 202 { id }
GET  /sdcpp/v1/jobs/{id}          # queued / generating / completed
POST /sdcpp/v1/jobs/{id}/cancel   # 取消
```

`/sdcpp/*` 会原样转发给引擎。提交示例：

```json
{
  "prompt": "a red lantern on snow, cinematic",
  "negative_prompt": "",
  "width": 1024, "height": 1024,
  "batch_count": 1, "seed": -1,
  "ref_images": [],
  "output_format": "png",
  "sample_params": {
    "sample_steps": 8, "sample_method": "euler", "scheduler": "default",
    "guidance": { "txt_cfg": 1.0 }
  }
}
```

`ref_images` 传 `data:image/png;base64,...` 即为图片编辑；完成后图片在 `result.images[0].b64_json`。

## 技术栈

- 界面：单文件 HTML/CSS/JS，无框架、无 CDN
- 启动器：Python 3.11 标准库（`http.server`）＋ PyInstaller（onedir）
- 引擎：[stable-diffusion.cpp](https://github.com/leejet/stable-diffusion.cpp)（CUDA）
- 模型：Qwen-Image-2.1 Turbo（GGUF 量化）＋ Qwen3-VL 文本编码器

## 常见问题

**状态栏一直「引擎未就绪」**
`models\` 缺文件。看黑色窗口里打印的缺失项，或访问 `/api/engine/status`。

**图片编辑报错 / 不可用**
缺 `mmproj-*.gguf`。文生图不需要它，编辑必须。

**4K 卡死或报显存不足**
8 GB 显存请留在 1K–2K。4K 的潜空间是 1K 的 16 倍。

**Windows 提示「未签名的应用」**
程序没有数字签名，选择「仍要运行」。若被 **智能应用控制（Smart App Control）** 拦截，需在 Windows 安全中心关闭它（不可逆，请自行权衡）。

**端口占用**
改 `config.json` 的 `port`。

**想换模型**
换掉 `models\` 里的文件即可，识别规则：含 `mmproj` → 视觉投影；含 `Qwen3-VL` → 文本编码器；其余最大 `.gguf` → 扩散模型；`.safetensors` → VAE。

## 许可与致谢

- 启动器与界面：本仓库作者自写。
- 出图引擎：[stable-diffusion.cpp](https://github.com/leejet/stable-diffusion.cpp)（MIT），版权归其作者。
- 模型权重不在本仓库内分发，使用请遵守各自的许可与使用条款（Qwen-Image 系列见其 Hugging Face 模型页）。

## 免责声明

生成内容由本地模型产生，作者不对生成结果及使用后果负责。请遵守所在地法律法规与模型许可条款。
