# 本地小模型：装了又删的经历（llama.cpp + Qwen2.5-3B）


想在本地跑个小模型离线用。折腾了一圈 `llama.cpp` + `Qwen2.5-3B`，最后结论是**删掉**——但过程中关于「国内怎么把模型下下来」的经验值得留档。

## 1. 方案

- 推理：`llama.cpp`（b10298），放 `C:\Programs\llama.cpp`
- 模型：`Qwen2.5-3B` Q4_K_M（约 2GB）
- 接入：起 `llama-server:8080`，在 opencode 里加一个 `llama.cpp` provider

## 2. 实测

生成速度约 **29–30 t/s**，能跑通。但模型太小，实际效果不理想，最终**全部删除**（模型、llama.cpp、provider 配置一并移除）。

## 3. 留档：国内下载模型源实测

| 源 | 状态 |
|---|---|
| **ModelScope**（modelscope.cn） | ✅ 可达 ~27MB/s，有官方 Qwen2.5 GGUF |
| Ollama registry | ⚠️ manifest 可达，但 blob GET 被拦 |
| HuggingFace | ❌ 全拦（连 README 都 401） |
| GitHub releases | ❌ `/releases/download` 全拦 |
| ghfast.top | ⚠️ <1GB 可用；大文件 302；**分块(range)稳** |

可行的下载路径：

- **GGUF 模型**：ModelScope 直连
  ```
  https://modelscope.cn/models/{org}/{repo}/resolve/master/{file}
  ```
- **GitHub 小文件**（<1GB）：`https://ghfast.top/<github-url>`
- **大文件**：用 curl **分块**下载（ghfast 的 range 请求返回 206，稳定）
- 查 ModelScope 文件列表：`https://modelscope.cn/api/v1/models/{org}/{repo}/repo/files`（JSON，可解析）

## 4. 小结

| 项 | 结论 |
|---|---|
| 能跑吗 | 能，3B Q4 约 30 t/s |
| 好用吗 | 太小，效果一般 |
| 最终 | 删除，保留下载源经验 |

**一句话**：本地小模型「能跑」和「好用」是两回事。3B 级别更适合当玩具验证链路；真要干活，要么上更大的模型（吃显存），要么用云端 API。但这次摸清的 ModelScope / 分块下载路径，以后下模型/大文件能直接复用。

