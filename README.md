# LLMLite - Azure OpenAI 代理服务

基于 [LiteLLM](https://github.com/BerriAI/litellm) 的 Docker 部署方案，支持 Azure OpenAI GPT-5.2 Codex 模型。

## 功能特性

- ✅ 支持 Azure OpenAI GPT-5.2 Codex 模型
- ✅ 个人 API Key 管理
- ✅ 用户用量追踪和限流
- ✅ Docker 一键部署
- ✅ 自动故障转移
- ✅ 成本追踪

## 快速开始

### 1. 配置环境变量

```bash
# 复制环境变量模板
cp .env.example .env

# 编辑配置文件，填入你的 Azure OpenAI 凭据
vim .env
```

**必须配置的变量：**

| 变量 | 说明 |
|------|------|
| `LITELLM_MASTER_KEY` | 管理员主密钥，用于创建用户和 API Keys |
| `AZURE_API_BASE` | Azure OpenAI 端点，格式: `https://<resource-name>.openai.azure.com/` |
| `AZURE_API_KEY` | Azure OpenAI API 密钥 |

### 2. 启动服务

```bash
# 启动所有服务
docker compose up -d

# 查看日志
docker compose logs -f litellm
```

### 3. 验证服务

```bash
# 健康检查
curl http://localhost:4000/health

# 查看可用模型
curl http://localhost:4000/v1/models \
  -H "Authorization: Bearer sk-your-master-key"
```

## API Key 管理

### 创建用户

```bash
curl -X POST http://localhost:4000/user/new \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user-001",
    "user_email": "user@example.com",
    "max_budget": 50,
    "budget_duration": "30d"
  }'
```

### 为用户创建 API Key

```bash
curl -X POST http://localhost:4000/key/generate \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user-001",
    "key_alias": "user-001-key",
    "duration": "30d",
    "max_budget": 20,
    "models": ["gpt-5.2-codex"],
    "metadata": {
      "team": "engineering"
    }
  }'
```

### 查看所有 API Keys

```bash
curl http://localhost:4000/key/list \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

### 删除 API Key

```bash
curl -X POST http://localhost:4000/key/delete \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "keys": ["sk-key-to-delete"]
  }'
```

## 使用 API

### OpenAI 兼容接口

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-user-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.2-codex",
    "messages": [
      {"role": "system", "content": "You are a helpful coding assistant."},
      {"role": "user", "content": "Write a Python function to calculate fibonacci numbers"}
    ],
    "temperature": 0.7
  }'
```

### 流式响应

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-user-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.2-codex",
    "messages": [{"role": "user", "content": "Hello"}],
    "stream": true
  }'
```

### Python 客户端示例

```python
from openai import OpenAI

client = OpenAI(
    api_key="sk-your-api-key",
    base_url="http://localhost:4000/v1"
)

response = client.chat.completions.create(
    model="gpt-5.2-codex",
    messages=[
        {"role": "user", "content": "Write a Python hello world"}
    ]
)

print(response.choices[0].message.content)
```

## 用量查询

### 查看用户用量

```bash
curl "http://localhost:4000/user/info?user_id=user-001" \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

### 查看 Key 用量

```bash
curl "http://localhost:4000/key/info?key=sk-user-key" \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

## 管理界面

LiteLLM 提供了内置的管理 UI：

```
http://localhost:4000/ui
```

使用 `LITELLM_MASTER_KEY` 登录。

## 常用命令

```bash
# 启动服务
docker compose up -d

# 停止服务
docker compose down

# 查看日志
docker compose logs -f

# 重启服务
docker compose restart litellm

# 查看服务状态
docker compose ps

# 清理数据 (谨慎使用)
docker compose down -v
```

## 文件结构

```
LLMLite/
├── docker-compose.yml    # Docker Compose 配置
├── config.yaml           # LiteLLM 代理配置
├── .env.example          # 环境变量模板
├── .env                  # 环境变量 (不要提交到Git)
└── README.md             # 本文档
```

## 故障排除

### 服务无法启动

```bash
# 检查日志
docker compose logs litellm

# 检查配置文件语法
docker run --rm -v $(pwd)/config.yaml:/app/config.yaml \
  ghcr.io/berriai/litellm:main-latest \
  --config=/app/config.yaml --validate
```

### 数据库连接失败

```bash
# 检查 PostgreSQL 状态
docker compose exec postgres pg_isready

# 重建数据库
docker compose down -v
docker compose up -d
```

### Azure OpenAI 连接失败

1. 确认 `AZURE_API_BASE` 格式正确 (以 `/` 结尾)
2. 确认 `AZURE_API_KEY` 正确
3. 确认模型部署名称与 `config.yaml` 中配置一致

## 安全建议

1. **更换默认密钥**: 生产环境务必更换 `LITELLM_MASTER_KEY`
2. **限制网络访问**: 使用防火墙限制 4000 端口访问
3. **启用 HTTPS**: 生产环境使用反向代理添加 TLS
4. **定期轮换密钥**: 定期更新 API Keys

## 参考链接

- [LiteLLM 文档](https://docs.litellm.ai/)
- [Azure OpenAI 文档](https://learn.microsoft.com/azure/ai-services/openai/)
