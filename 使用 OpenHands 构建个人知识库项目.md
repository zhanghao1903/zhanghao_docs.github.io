# 使用 OpenHands 构建个人知识库项目

## 背景

现在 **vibe coding** 很火，网上说得神乎其技，但我始终没有亲自感受过。

前一阵子，我主要在阅读 OpenHands 的文档，对它的主要使用场景有了一些了解。同时也在关注 AI 相关技术栈，主要是因为感觉未来岗位可能会越来越少，希望增强自己在 AI 方向的能力。

在看了几篇关于“向量数据库 + 知识库”的文章之后，我产生了一个想法：

> 要不自己动手搭建一个知识库项目？

---

## 为什么要做这个项目？

我思考了一下，这个项目能带来的价值：

1. **动手远比看文章更有效**  
   看再多文章，不如自己做一遍。只有真正实践，才能理解问题、限制以及技术边界。

2. **弥补 AI 技术栈的断层**  
   我明显感觉自己和当前 AI 技术有些脱节，通过项目实践，比单纯看文档更清晰、更高效。

3. **体验 vibe coding + 学习 OpenHands**  
   这个项目正好可以结合 OpenHands 来做，一举两得。

4. **提升简历竞争力**  
   有完整项目经验，比空谈技术更有说服力。

> 总体来说，这个项目是非常值得投入时间去做的。

于是，我开始动手了。

---

## OpenHands 部署选择

OpenHands 提供三种部署方式：

- Cloud
- CLI
- 本地 GUI

我选择的是：**本地 GUI**

---

## 硬件环境选择

我手头有三台设备：

- Mac mini M2（8G + 256G）
- Windows mini 主机（AMD 7940HS，32G + 1T）
- MacBook Pro M2（16G + 512G）

在 GitHub 上看了两周文档后，我得出一个结论：

> 当前很多 AI 项目对 Windows 的支持并不好，也不够及时

因此：

- ❌ 放弃 Windows
- ✅ 选择 MacBook Pro（内存更大，更适合本地 GUI）

---

## 本地环境搭建

OpenHands 对 macOS 的支持还是不错的，整体安装体验较顺畅。

### 1. 安装 uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 2. 安装 OpenHands

```bash
uv tool install openhands --python 3.12
```

---

## 遇到的问题

### 问题现象

刚开始使用时，遇到一个比较诡异的问题：

> 会话一直报错：`OpenAI API 403 Unauthorized`

但问题是：

- 我用的是 OpenHands Cloud
- 账户里已经充值了 10 美元

---

## 配置排查

### 查看全局配置

```bash
cat ~/.openhands/agent_settings.json
```

### 查找会话配置

```bash
grep -RniE '"model"|"llm"|"provider"|"base_url"|"api_key"' ~/.openhands/conversations
```

---

## 关键结论

- OpenHands 的 LLM 配置是 **会话级别**
- 会话启动后无法修改
- 修改配置必须新建会话

---

## 总结

这次实践让我收获很大：

- 对 OpenHands 架构有更深理解
- 理解了 LLM 调用机制
- 完整跑通一个 AI 项目流程

---

## 后续计划

- 支持 URL 知识源
- 使用Claude design完善UI界面
- 云端部署
- 做文档站 + 演示视频
