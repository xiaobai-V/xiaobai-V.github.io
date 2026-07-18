# 在 WSL2 中配置 Claude Code 接入智谱 GLM API 教程

## 前置条件

| 依赖 | 最低版本 | 验证命令 |
|---|---|---|
| WSL2 (Ubuntu 22.04+) | - | `uname -r` 输出含 `microsoft-standard-WSL2` |
| Node.js | v18+ | `node --version` |
| npm | v9+ | `npm --version` |
| 智谱开放平台账号 | - | 已获取 API Key |

## 第一步：安装 Node.js（如未安装）

推荐使用 NodeSource 安装 Node.js 22 LTS：

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
```

验证：

```bash
node --version   # v22.x.x
npm --version    # 10.x.x
```

## 第二步：安装 Claude Code

通过 npm 全局安装：

```bash
sudo npm install -g @anthropic-ai/claude-code
```

验证：

```bash
claude --version
```

## 第三步：获取智谱 API Key

1. 访问 [智谱开放平台](https://open.bigmodel.cn/)
2. 注册并登录
3. 进入 **API Keys** 管理页面
4. 创建新的 API Key 并复制保存

## 第四步：配置 Claude Code 接入 GLM API

编辑 Claude Code 的全局配置文件：

```bash
mkdir -p ~/.claude
nano ~/.claude/settings.json
```

写入以下内容（将 `你的API Key` 替换为实际的智谱 API Key）：

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "你的API Key",
    "ANTHROPIC_BASE_URL": "https://open.bigmodel.cn/api/anthropic",
    "API_TIMEOUT_MS": "3000000",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": 1,
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "GLM-4.5-air",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "GLM-5.1",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "GLM-5.1"
  }
}
```

### 配置项说明

| 环境变量 | 值 | 作用 |
|---|---|---|
| `ANTHROPIC_BASE_URL` | `https://open.bigmodel.cn/api/anthropic` | 将 API 请求重定向到智谱的 Anthropic 兼容端点 |
| `ANTHROPIC_AUTH_TOKEN` | 你的智谱 API Key | 使用智谱平台的认证密钥 |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | `GLM-5.1` | Sonnet 模型映射为 GLM-5.1（主模型） |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | `GLM-5.1` | Opus 模型映射为 GLM-5.1（主模型） |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | `GLM-4.5-air` | Haiku 模型映射为 GLM-4.5-air（轻量模型） |
| `API_TIMEOUT_MS` | `3000000` | API 超时设为 50 分钟（防止长任务中断） |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | `1` | 禁用遥测等非必要网络请求 |

## 第五步：启动 Claude Code

```bash
cd 你的项目目录
claude
```

首次启动会提示登录——由于使用的是自定义 API 端点，认证已通过 `ANTHROPIC_AUTH_TOKEN` 完成，直接跳过登录流程即可开始使用。

## 第六步（可选）：配置网络代理

如果需要通过代理访问 API，可在 `~/.bashrc` 中添加快捷命令：

```bash
# 添加代理别名（端口号改为你自己的代理端口）
alias proxy='export all_proxy=http://$(cat /etc/resolv.conf | grep nameserver | awk "{print \$2}"):7897'
alias unproxy='unset all_proxy'
```

使用方式：

```bash
proxy    # 开启代理
unproxy  # 关闭代理
```

> 提示：`7897` 是 Clash Verge 的默认端口，请根据你实际使用的代理软件修改。

## 验证配置是否生效

启动 `claude` 后，输入简单问题测试：

```
> 你好，请告诉我你是什么模型
```

如果正常回复，说明配置成功。

## 工作原理

Claude Code 原生使用 Anthropic API 协议。智谱开放平台提供了 **Anthropic 兼容的 API 端点**（`/api/anthropic`），协议格式与 Anthropic 官方一致。通过修改 `ANTHROPIC_BASE_URL`，将请求重定向到智谱平台，再用 `ANTHROPIC_DEFAULT_*_MODEL` 环境变量将模型名称映射为 GLM 系列，即可在不修改 Claude Code 源码的情况下使用 GLM 模型。

```
Claude Code  →  ANTHROPIC_BASE_URL 重定向  →  智谱 Anthropic 兼容端点  →  GLM 模型
```

## 常见问题

### Q: 启动后报认证错误

确认 `ANTHROPIC_AUTH_TOKEN` 填写的是完整的智谱 API Key（通常以数字开头，长度约 32 位）。检查 `settings.json` 的 JSON 格式是否正确（无多余逗号、引号闭合）。

### Q: 请求超时

增大 `API_TIMEOUT_MS` 的值，或在代理环境下确认代理连通性：

```bash
curl -x http://代理地址:端口 https://open.bigmodel.cn/api/anthropic
```

### Q: 模型名称错误

智谱支持的模型名称可能随时间更新，查看[智谱官方文档](https://open.bigmodel.cn/dev/api/normal-model/glm-5)确认当前可用的模型 ID。

### Q: 工具调用（Tool Use）不正常

智谱的 Anthropic 兼容端点支持 Tool Use，但兼容性可能与 Anthropic 原生略有差异。如遇到工具调用异常，可尝试更换映射的 GLM 模型版本。
