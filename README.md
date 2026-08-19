# dsh-plugin-openviking

把 [OpenViking](https://github.com/volcengine/OpenViking) 作为记忆层接入
[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)。

**不需要写代码。** OpenViking 的 MCP 端点 + DSH 自带的 `dsh-mcp-client` 桥即可。

本文所有配置和输出都经过实测（2026-08-19，OpenViking 1.29.0 / dsh-mcp-client）。

---

## 一、起 OpenViking

```bash
pip install openviking          # 注意：装完约 1.1 GB
```

密钥建议放 `.env`（已在 .gitignore 中）：

```bash
cp .env.example .env    # 填入你的 key
source .env
```

### ⚠️ 配置文件的四个坑

`ov.conf` 的格式 README 里没写清楚，这四条是我逐个试出来的：

1. **它是 JSON，不是 TOML**（名字有误导）
2. **没有 `llm` 字段，大模型配在 `vlm` 下面**（写错时它会提示 `did you mean 'vlm'?`）
3. **是 `api_base`，不是 `base_url`**
4. **embedding 要再嵌一层 `dense`**

可用配置（以阿里云百炼为例，任何 OpenAI 兼容端点同理）：

```json
{
  "vlm": {
    "provider": "openai",
    "backend": "openai",
    "api_base": "https://dashscope.aliyuncs.com/compatible-mode/v1",
    "api_key": "sk-...",
    "model": "qwen-plus"
  },
  "embedding": {
    "dense": {
      "provider": "openai",
      "backend": "openai",
      "api_base": "https://dashscope.aliyuncs.com/compatible-mode/v1",
      "api_key": "sk-...",
      "model": "qwen3.7-text-embedding",
      "dimension": 1024,
      "input": "text"
    }
  },
  "auto_generate_l0": true,
  "auto_generate_l1": true
}
```

`auto_generate_l0/l1` 是三层加载的开关，务必开着，省 token 全靠它。

**embedding 是必需的。** 没有 embedding 模型，`find` / `search` 用不了，只剩 `grep` / `glob` / `list`。
挑 provider 前先确认对方有 `text-embedding` 类模型（很多聚合站只有对话模型）。

```bash
openviking-server --config ov.conf --port 18790
```

看到这行就是好了：

```
StreamableHTTP session manager started
```

## 二、接进 DSH

把 [`cordis.yml`](cordis.yml) 那段并进你的 DSH 配置即可。核心就三行：

```yaml
- id: openviking-memory
  name: '@deepseek-ai/dsh-mcp-client'
  config:
    serverName: viking
    transport: streamable-http
    url: http://127.0.0.1:18790/mcp
```

DSH 的 mcp-client 支持 HMR，**改配置不用重启进程**。

## 三、模型能用到的 15 个工具

实测 `tools/list` 返回 15 个，在 DSH 里显示为 `mcp__viking__*`：

| 工具 | 作用 |
|---|---|
| `find` | 快速语义检索 |
| `search` | 深度检索，可带会话上下文和意图 |
| `read` | 读 viking:// 文件全文（L2） |
| `list` / `tree` | 列目录 / 看目录树 |
| `grep` / `glob` | 正则 / 通配检索（不需要 embedding） |
| `remember` | 把对话存进长期记忆 |
| `write` / `edit` | 写入 / 精确替换 |
| `add_resource` | 加资源（异步处理） |
| `forget` | 永久删除，不可逆 |
| `list_watches` / `cancel_watch` | 自动刷新订阅 |
| `health` | 健康检查 |

## 四、实测：自动记忆确实成立

**之前判断「自动沉淀要写代码」是错的。** `remember` 收的是消息数组，
不是一段文本，所以它本来就是设计来吃会话轨迹的。

```json
{"name":"remember","arguments":{"messages":[
  {"role":"user","content":"我要把 Word 转 markdown 喂给 agent，anydoc 和 pandoc 选哪个"},
  {"role":"assistant","content":"实测 30 份 docx：anydoc 残留 HTML 1887 个，pandoc 16837 个…"}
]}}
```

返回 `Stored 2 message(s) and committed for memory extraction.`，约 30 秒后异步抽取完成。

然后**换一个字面上毫不重合的说法**去检索：

```
find("文档转换工具怎么选")

- [memory 66%] viking://user/default/memories/preferences/user/文档转换工具偏好.md
    在将 Word 文档转为 Markdown 供 agent 使用时，偏好使用 anydoc 而非 pandoc，
    因其生成的 Markdown 更干净、表格保留为原生 Markdown 表格格式。
```

注意它做了抽象：输入是「1887 vs 16837」这种一次性数字，
**存下来的是可复用的偏好**，文件名和分类都是它自己起的。

DSH 侧只要在会话边界调一次 `remember`，或者在系统提示词里要求模型自己调，
记忆就自动成型。**不需要碰生命周期钩子。**

## 五、要注意的

**它会推断并写入你的身份。** 上面那一次调用，除了偏好文件，
它还顺手改了 `memories/identity.md`，写进了「Vibe: 务实、有判断力、倾向实证结论」
这类判断。这是设计如此，但跑真实工作前你该知道。

**靠模型自觉调 `remember` 不如钩子可靠。** MCP 协议不覆盖会话生命周期，
所以这条路上「什么时候存」是提示词决定的，不是约束。要确定性就得写 DSH 插件。

**AGPL-3.0。** OpenViking 主项目是 AGPL，`ov_cli` 和 examples 是 Apache-2.0。
当网络服务对外提供且改过代码，需要开源。商用前让法务看一眼。

**装完 1.1 GB。** 一个「上下文数据库」的客户端，体量要有心理准备。

## TODO

- [ ] 用 DSH 插件生命周期在会话边界自动调 `remember`，去掉对提示词的依赖
- [ ] `viking://` URI 在 DSH 侧的路径补全
- [ ] 多 workspace / project 隔离映射到 DSH workspace

## 许可

本配置 MIT。OpenViking 主项目 AGPL-3.0。
