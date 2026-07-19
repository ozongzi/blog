---
title: 把 Claude Code 当成一个 LLM API 用
date: 2026-07-19 10:00:00
tags:
  - Rust
  - LLM
  - agentix
categories:
  - 技术
description: agentix 的 claude_code 策略：桩工具、MITM 回放代理，以及一场和 prompt cache 的拉锯战。
---

agentix 是一个 Rust 库，作用是把各家 LLM 的 API 抹平成同一套接口：给它消息和工具定义，它吐回一条 `LlmEvent` 流。Anthropic、OpenAI、Gemini、DeepSeek、Kimi 这些都是标准做法，读 API 文档，写 HTTP 客户端，完事。

麻烦的是 Claude Code。Claude 订阅里的额度只能通过 `claude` 这个 CLI 用。我想让它也变成 agentix 里的一个普通 provider——和其他家用同一个接口，同一套工具分发逻辑。

问题在于 `claude -p` 不是 API，是一个 agent。你给它一个任务，它自己规划、自己调工具、自己循环，最后给你结果。而我要的是最底层的东西：发一轮消息，拿回模型的 `tool_use` 块，就此打住，工具由我在外面执行。

这篇记录一下我是怎么把一个 agent 变成一个单轮 LLM 的。整个过程分两步。

## MCP

`claude` 支持 MCP 服务器，模型能看到 MCP 工具的 schema 并发出调用。这就是入口。

agentix 在进程内起一个 MCP 服务器，把调用方传进来的工具定义原样注册上去，但每个工具都是桩：`call()` 被调到时立刻返回一个空结果，什么都不做。然后 spawn `claude -p --input-format stream-json --output-format stream-json`，把它指向这个回环地址上的 MCP 服务器，最后一条用户消息从 stdin 喂进去。

模型看到工具 schema，正常输出 `tool_use`。我们在 stdout 的 stream-json 里解析到第一条 assistant 消息，把里面的工具调用和 usage 统计发出去，然后直接杀掉子进程。工具真正的执行、结果回填、下一轮请求，全是调用方的事。

到这里,已经能用了。工具名上的 `mcp__agentix__` 前缀剥掉,工具调用 ID 重映射成 `toolu_` 格式,细节琐碎但直接。

然后 prompt cache 出了问题。

单轮没问题,多轮工具循环把问题暴露出来了。

一个正常的 agent 循环是:模型说要调工具 → 我执行 → 把结果连同历史发回去 → 模型继续。历史怎么发给 `claude`？走 `--resume` 加 stdin。我一开始就是这么做的，然后发现 Anthropic 的 prompt cache 命中率低到不正常，几乎只有 system prompt 那一段命中。

抓包看了才明白。CLI 拿到 resume 的历史后会重塑它：往里面注入一句 "Continue from where you left off."，而且每轮注入的位置都不一样，落在历史中间的不同消息上。prompt cache 的机制是前缀匹配，中间任何一个字节变了，后面全部失效。等于每一轮工具循环都在重新付整个上下文的钱。

交互式的 Claude Code 没有这个问题，因为它是一个连续 session，历史只增不改，缓存前缀天然稳定。所以修法不是去对抗 CLI 的重塑逻辑，而是让它以为自己就在一个连续 session 里。

## 回放

思路是：resume 只 resume 到最后一条"落定"的用户消息为止，当前这一轮工具循环不通过历史喂给它，而是让 CLI 现场把这段循环重新经历一遍。它自己发模型请求、自己调工具、自己走到循环末尾，会话形状和交互式完全一致，缓存自然就对了。

但重新经历一遍不能重新付一遍钱，也不能不确定。所以这些"已知步骤"全部用录好的数据应答：

- CLI 的模型请求，由一个进程内的 MITM 代理接住，用之前录下的 assistant 消息伪造成 SSE 回给它；
- 模型（按剧本）发出的工具调用，由桩 MCP 服务器用录好的工具结果应答。

代理的接法是 `HTTPS_PROXY` 指向本地，再用 `NODE_EXTRA_CA_CERTS` 让 CLI 信任我们现场铸造的 CA 证书（TLS 拆包的活交给了 hudsucker 这个库）。CLI 以为自己在跟 `api.anthropic.com` 说话，OAuth 凭证也照常带上——因为录好的步骤耗尽之后，下一个请求代理会原样放行给真正的 Anthropic，这才是本轮唯一一次真实生成。再之后的请求，代理回一个空的 `end_turn`，CLI 的循环安静地停下来，不再多花一分钱。

## 结果

现在 claude_code 在 agentix 里就是一个普通 provider。缓存命中恢复到和交互式 Claude Code 相同的水平,因为发到 Anthropic 的请求形状本来就和交互式一模一样。

代码在 [agentix/src/raw/claude_code](https://github.com/ozongzi/agentix/tree/main/agentix/src/raw/claude_code)，`mod.rs`、`proxy.rs`、`replay.rs`、`session.rs` 四个文件。
