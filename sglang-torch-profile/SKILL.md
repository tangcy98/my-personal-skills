---
name: sglang-torch-profile
description: 启动 sglang 服务并通过 torch profiler 采集 prefill 和 decode 阶段的 trace 数据，下载到本地协助分析性能瓶颈。用户提到"sglang-torch-profile"、"sglang profile"、"torch profile"、"trace分析"时使用。
argument-hint: "<MODEL_NAME_OR_SGLANG_ARGS> [AIPERF_PARAMS] [ENV_VARS]"
---

# sglang-torch-profile：sglang torch profiler 采集

## 目的
启动 sglang 推理服务（开启 torch profiler），在稳态下分别采集 prefill 和 decode 阶段的 trace 数据，下载到本地供工程师分析性能瓶颈。

## 输入参数

用户可能提供以下三类输入，从 `$ARGUMENTS` 和上下文中解析：

### 1. sglang 启动参数

用户可能提供完整的 sglang 启动命令（如 `python -m sglang.launch_server --port 9999 ...`），直接使用即可。

如果用户**没有提供完整的 sglang 启动命令**，使用以下默认配置（用户至少需要提供模型名，否则必须确认）：

```bash
python3 -m sglang.launch_server \
  --port 9999 \
  --enable-metrics \
  --model-path /workspace/models/<MODEL_NAME> \
  --tp-size 4 \
  --served-model-name <MODEL_NAME> \
  --trust-remote-code \
  --disable-radix-cache \
  --chunked-prefill-size 12288 \
  --quantization fp8 \
  --kv-cache-dtype fp8_e4m3 \
  --speculative-algo NEXTN \
  --speculative-num-steps 3 \
  --speculative-eagle-topk 1 \
  --speculative-num-draft-tokens 4
```

如果用户**既没有提供完整命令也没有提供模型名**，必须向用户确认模型名。

### 2. aiperf 压测参数（用于制造稳态负载）

| 参数 | 默认值 | 说明 |
|------|--------|------|
| ISL (--synthetic-input-tokens-mean) | `10000` | 输入序列长度 |
| OSL (--output-tokens-mean) | `726` | 输出序列长度（**prefill 阶段会被强制设为 1**） |
| REQUEST_COUNT (--request-count) | `1024` | 请求总数（需要足够多以维持稳态） |
| QPS (--request-rate) | `1.0` | 请求速率 |
| CONCURRENCY (--concurrency) | 不指定（使用 aiperf 默认） | 并发数，仅在用户明确指定时添加 |
| WARMUP_COUNT (--warmup-request-count) | `5` | warmup 请求数 |
| PORT | 从 sglang 启动参数中提取 | 服务端口，与 sglang 端口一致 |

### 3. 环境变量（可选）

用户可能提供额外环境变量，会与以下**必需的默认环境变量**合并（用户指定的优先）：
```bash
NCCL_MIN_NCHANNELS=32 NCCL_ENABLE_QUANTIZATION=1 NCCL_SHM_DISABLE=1 NCCL_P2P_DIRECT_DISABLE=1 NCCL_IB_DISABLE=1 SGLANG_CONV1D_ADAPTIVE_TRITON=1
```

**注意**：启动 sglang 时必须额外加上 `SGLANG_TORCH_PROFILER_DIR=/workspace/profile_log`，这是开启 torch profiler 的关键环境变量。

## 操作步骤

### 1. 参数解析
从 `$ARGUMENTS` 和对话上下文中解析所有参数。如果无法确定模型名，**必须向用户确认**。

### 2. 检查 aiperf 是否安装
```bash
which aiperf
```
如果未安装，自动安装：
```bash
pip install aiperf -i http://pypi.devops.xiaohongshu.com/simple/ --trusted-host pypi.devops.xiaohongshu.com
```

### 3. 停止已有 sglang 进程
```bash
PID=$(pgrep -f "sglang.launch_server" | head -1); if [ -n "$PID" ]; then kill $PID; sleep 2; kill -9 $PID 2>/dev/null; fi
```

### 4. 清理旧 profile 数据
```bash
rm -rf /workspace/profile_log/*
```

### 5. 启动 sglang（开启 torch profiler）
使用 `run_in_background` 在后台启动 sglang 服务，**timeout 设为 3000000ms（3000秒）**。

环境变量中**必须包含** `SGLANG_TORCH_PROFILER_DIR=/workspace/profile_log`：
```bash
SGLANG_TORCH_PROFILER_DIR=/workspace/profile_log <其他ENV_VARS> python3 -m sglang.launch_server <ARGS> 2>&1 | tee /tmp/sglang_profile.log
```

### 6. 等待端口就绪
轮询端口是否可用（最多等待 300 秒）：
```bash
PORT=<从sglang参数中提取的端口>
for i in $(seq 1 300); do
  if curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${PORT}/health 2>/dev/null | grep -q '200'; then
    echo "sglang 服务就绪"; break
  fi
  if [ "$i" -eq 300 ]; then
    echo "错误：sglang 启动超时（300秒）"
    echo "最后 50 行日志："
    tail -50 /tmp/sglang_profile.log
    exit 1
  fi
  sleep 1
done
```
**timeout 设为 3000000ms（3000秒）**。

### 7. Profile Prefill 阶段

#### 7a. 后台启动 aiperf（OSL=1）
在后台启动 aiperf，**OSL 强制设为 1**（使请求主要在 prefill 阶段）：
```bash
aiperf profile \
  --model <MODEL_PATH> \
  --endpoint-type chat \
  --endpoint /v1/chat/completions \
  --streaming \
  --url http://127.0.0.1:<PORT> \
  --request-rate-mode constant \
  --request-count <REQUEST_COUNT> \
  --synthetic-input-tokens-mean <ISL> \
  --output-tokens-mean 1 \
  --extra-inputs ignore_eos:true \
  --extra-inputs "{\"nvext\":{\"ignore_eos\":true}}" \
  --request-rate <QPS> \
  --warmup-request-count <WARMUP_COUNT> &
```
使用 `run_in_background` 或 `nohup ... &` 启动，**timeout 设为 3000000ms**。

#### 7b. 等待进入稳态
等待 aiperf warmup 完成并进入稳态（等待约 30-60 秒，或观察 aiperf 日志出现 "profiling started" 字样）：
```bash
sleep 30
```

#### 7c. 触发 profiler 采集
发送 HTTP 请求触发 torch profiler（**num_steps=3**）：
```bash
curl -X POST -H 'Content-Type: application/json' \
  "http://127.0.0.1:<PORT>/start_profile" \
  -d '{"num_steps":3,"merge_profiles":true,"output_dir":"/workspace/profile_log"}'
```

#### 7d. 等待 trace 文件生成完成
轮询 `/workspace/profile_log` 目录，等待 `merged*.trace.json.gz` 文件出现，并确认文件写入完成（文件大小不为 0 且连续 30 秒不变）：
```bash
# 等待文件出现（最多 300 秒）
for i in $(seq 1 300); do
  TRACE_FILE=$(ls /workspace/profile_log/merged*.trace.json.gz 2>/dev/null | head -1)
  if [ -n "$TRACE_FILE" ]; then
    echo "找到 trace 文件: $TRACE_FILE"; break
  fi
  sleep 1
done

# 等待文件写入完成（文件大小不为 0 且连续 30 秒不变）
LAST_SIZE=-1
STABLE_COUNT=0
while [ "$STABLE_COUNT" -lt 30 ]; do
  CURRENT_SIZE=$(stat -c%s "$TRACE_FILE" 2>/dev/null || echo 0)
  if [ "$CURRENT_SIZE" -gt 0 ] && [ "$CURRENT_SIZE" -eq "$LAST_SIZE" ]; then
    STABLE_COUNT=$((STABLE_COUNT + 1))
  else
    STABLE_COUNT=0
  fi
  LAST_SIZE=$CURRENT_SIZE
  sleep 1
done
echo "trace 文件写入完成: $TRACE_FILE (大小: ${CURRENT_SIZE} bytes)"
```

#### 7e. 重命名 trace 文件
将文件重命名为包含 `prefill` 和时间戳的格式：
```bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
NEW_NAME="/workspace/profile_log/<MODEL_NAME>_prefill_${TIMESTAMP}.trace.json.gz"
mv "$TRACE_FILE" "$NEW_NAME"
```

#### 7f. 停止 aiperf
```bash
PID=$(pgrep -f "aiperf" | head -1); if [ -n "$PID" ]; then kill $PID; sleep 2; kill -9 $PID 2>/dev/null; fi
```

#### 7g. 下载 prefill trace 到本地
将文件从远程 pod 下载到本地：
```bash
kubectl cp <namespace>/<pod>:<remote_path> <local_path>
```
本地保存路径：`/data/shiyuan/workspace/bench_results/`

### 8. Profile Decode 阶段

#### 8a. 后台启动 aiperf（使用正常 OSL）
与 prefill 不同，这里使用**正常的 OSL**（默认 726 或用户指定值）：
```bash
aiperf profile \
  --model <MODEL_PATH> \
  --endpoint-type chat \
  --endpoint /v1/chat/completions \
  --streaming \
  --url http://127.0.0.1:<PORT> \
  --request-rate-mode constant \
  --request-count <REQUEST_COUNT> \
  --synthetic-input-tokens-mean <ISL> \
  --output-tokens-mean <OSL> \
  --extra-inputs ignore_eos:true \
  --extra-inputs "{\"nvext\":{\"ignore_eos\":true}}" \
  --request-rate <QPS> \
  --warmup-request-count <WARMUP_COUNT> &
```

#### 8b. 等待进入稳态
```bash
sleep 30
```

#### 8c. 触发 profiler 采集（num_steps=10）
```bash
curl -X POST -H 'Content-Type: application/json' \
  "http://127.0.0.1:<PORT>/start_profile" \
  -d '{"num_steps":10,"merge_profiles":true,"output_dir":"/workspace/profile_log"}'
```

#### 8d. 等待 trace 文件生成完成
同 7d 的逻辑，轮询等待新的 `merged*.trace.json.gz` 文件出现，文件大小不为 0 且连续 30 秒不变即视为完成。

#### 8e. 重命名 trace 文件
```bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
NEW_NAME="/workspace/profile_log/<MODEL_NAME>_decode_${TIMESTAMP}.trace.json.gz"
mv "$TRACE_FILE" "$NEW_NAME"
```

#### 8f. 停止 aiperf
```bash
PID=$(pgrep -f "aiperf" | head -1); if [ -n "$PID" ]; then kill $PID; sleep 2; kill -9 $PID 2>/dev/null; fi
```

#### 8g. 下载 decode trace 到本地
同 7g，下载到 `/data/shiyuan/workspace/bench_results/`

### 9. 停止 sglang
Profile 完成后，**自动停止 sglang 服务**：
```bash
pkill -f "sglang.launch_server"
```

### 10. 输出结果
告知用户本地文件路径，格式如：
```
Prefill trace: /data/shiyuan/workspace/bench_results/<MODEL_NAME>_prefill_<timestamp>.trace.json.gz
Decode trace:  /data/shiyuan/workspace/bench_results/<MODEL_NAME>_decode_<timestamp>.trace.json.gz
```

## 注意事项
- 所有长耗时命令都需设置 **timeout 3000000ms（3000秒）**。
- `SGLANG_TORCH_PROFILER_DIR=/workspace/profile_log` 是**必需**的环境变量，缺少它 profiler 不会工作。
- Prefill profile 时 OSL **必须设为 1**，decode profile 时使用正常 OSL。
- Prefill profile 的 `num_steps=3`，decode profile 的 `num_steps=10`。
- 判断 trace 文件写入完成的标准：文件大小不为 0 且连续 30 秒不变。
- trace 文件生成完成后，应立即停止对应的 aiperf 进程，不需要等 aiperf 自然结束。
- 如果在远程 pod 上执行，下载文件时使用 `kubectl cp`；如果是本地执行则直接 mv/cp。
- 如果 aiperf 未安装，自动安装后再继续。
- 最终告知用户本地文件路径。
