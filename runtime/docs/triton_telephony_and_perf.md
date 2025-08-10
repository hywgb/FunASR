# Triton 生产部署建议：8k 电话音频与性能优化

## 目标
- 为 8k 电话语音优化声学前端频带，降低无效高频噪声影响。
- 在保证低延迟的前提下提升并发吞吐，通过动态批与实例化策略进行平衡。

## 前端配置（特征提取器）
- 在模型配置（`config.yaml` 或对应 `config.pbtxt` 参数链）中设置：
  - `frontend_conf.fs = 8000`
  - `frontend_conf.telephony_mode = true`
  - 可选：`frontend_conf.low_freq` 与 `frontend_conf.high_freq` 指定频带（默认约 50Hz–3800Hz，自动避让奈奎斯特）。
- 本仓库已在以下路径注入支持：
  - `model_repo_paraformer_large_online/feature_extractor/1/model.py`
  - `model_repo_paraformer_large_offline/feature_extractor/1/model.py`
  - `model_repo_sense_voice_small/feature_extractor/1/model.py`

## 动态批与并发
- 在线（流式）模型：
  - `preferred_batch_size: [1, 2, 4]`
  - `max_queue_delay_microseconds: 2000–8000`（视SLA与语音切片时长）
  - 多实例：`instance_group { count: N kind: KIND_GPU }`，N=GPU 数量 × 每卡实例数（一般 1–2），并与 ASR 编码器/解码器实例数对齐。
- 离线（文件转写）模型：
  - `preferred_batch_size: [4, 8, 16, 32]`
  - `max_queue_delay_microseconds: 20000–50000`
  - 结合 `batch_size_s`（动态按音频总时长聚合）提升吞吐。

## 线程与亲和性
- GPU：单实例内尽量减少 CPU 线程数，避免竞争；
- CPU：根据核心数设置 `numa` 亲和性与 `num_threads`，确保无过度上下文切换。

## 监控指标
- RT（p50/p95/p99）、吞吐（QPS/sps）、GPU 利用率、显存占用、动态批有效率（实际批大小统计）、队列等待时间、VAD 段落粒度与占比。

## 其他
- 在线流式的 `chunk_size_s` 与模型窗口/步长/右上下文对齐，避免重复计算与边界抖动；
- 使用热词/语言模型时，注意资源放置与实例隔离，以免抢占核心路径资源；
- 配置变更分灰度发布与回滚策略。