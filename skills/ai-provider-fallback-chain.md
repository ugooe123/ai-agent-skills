---
name: ai-provider-fallback-chain
description: AI Provider 保底链 — DeepSeek 等付费模型欠费/故障时自动切换免费本地模型，确保服务不断。含双端点接入、防抖动、健康探测。
tags: [provider, fallback, deepseek, freellmapi, reliability, cost-saving]
platforms: [hermes, claude-code, codex]
---

# AI Provider 保底链（防止付费模型欠费导致服务中断）

## 触发场景
- DeepSeek / OpenAI / Anthropic 欠费或故障，AI 服务全部停摆
- 需要"付费模型当主力 + 免费本地模型兜底"的高可用架构

## 核心架构
```
付费模型(DeepSeek) → 失败 → FreeLLMAPI 本地免费模型(80B级) → 最终 → 本地模板
```

## 关键步骤

### 1. 确认免费本地模型可用（实测）
```bash
# FreeLLMAPI 本地服务（端口3001，21个免费模型）
curl -s http://127.0.0.1:3001/v1/models | python3 -c "import sys,json; d=json.load(sys.stdin); print([m['id'] for m in d.get('data',[]) if 'free' in m.get('id','')])"
```

### 2. 非流式端点接入（统一调用点）
在付费模型失败后、最终兜底前插入免费模型调用：
```python
# 伪代码：付费失败 → FreeLLMAPI → 模板
try:
    result = call_deepseek(messages)  # 主链
except:
    try:
        result = call_freellmapi(messages)  # 保底链
    except:
        result = local_template(messages)   # 终极兜底
```

### 3. 流式端点接入（SSE）
付费模型流式失败时，在 except 分支切换 FreeLLMAPI 流式输出，保持前端不中断。

### 4. 防抖动（关键坑）
服务重启后需要2-3分钟加载，健康检查脚本必须加冷却时间：
- 上次重启 <10 分钟 → 不重复重启，只报告
- 用 `/tmp/patrol_restart_{service}` 时间戳文件做冷却标记

## 常见坑
| 坑 | 原因 | 解决 |
|:---|:---|:---|
| 反复 kill 循环 | watchdog 每3分钟检查，启动慢就 kill | 删多余 watchdog，只留 systemd Restart=always |
| 免费模型质量差 | 选了小模型 | 用 80B 级免费模型（qwen3-next-80b/nemotron-120b） |
| 上下文塞满 | 调免费模型也传大量消息 | 只传最近4-8条 |
| "未留联系方式" | 旧数据 | 代码改后新对话才生效，旧记录不回填 |

## 验证
1. `curl -s http://127.0.0.1:3001/v1/chat/completions` 免费模型返回正常
2. 模拟付费失败（改错 key）→ 系统自动切免费模型
3. 服务重启后 3 分钟内不重复重启
