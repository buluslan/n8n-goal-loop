# n8n-ops.md — n8n 实例运维参考

> 用途：n8n 实例异常（数据库膨胀 / 损坏 / 迁移）时的运维参考。
> 生成 goal 时若遇实例异常，应在 pause 条件预置人工确认点（备份完成 / 变更后验证）。
> 面向 Claude，命令块中路径均为占位符。

## 占位符约定

| 占位符 | 含义 |
|--------|------|
| `${N8N_DATA_PATH}` | n8n 数据目录，挂载到容器内 `/home/node/.n8n` |
| `${PG_DATA_PATH}` | PostgreSQL 数据卷宿主路径 |
| `${DOCKER_DEPLOY_PATH}` | docker-compose 部署目录（含 `.env`） |
| `${N8N_CONTAINER}` | n8n 容器名（默认 `local-n8n`） |
| `${N8N_API_KEY}` | n8n REST API 密钥 |
| `${N8N_HOST}` | n8n 实例地址（默认 `http://localhost:5678`） |

---

## 1. SQLite 数据库维护

### 1.1 问题背景

n8n 默认用 SQLite 存执行数据。每条执行写入 `execution_data` 表（含每节点输入/输出中间数据），平均 2~6 MB/条。长期不清理导致：

- 数据库膨胀（100+ 条执行可达数百 MB）
- WAL 模式下大文件并发写入触发 `SQLITE_CORRUPT / SQLITE_IOERR / SQLITE_NOTADB`
- 容器文件系统同步延迟加剧写入冲突
- 执行调度卡死、API 写入失败

### 1.2 启用自动清理（必做）

在 `${DOCKER_DEPLOY_PATH}/.env` 中添加：

```bash
EXECUTIONS_DATA_PRUNE=true
EXECUTIONS_DATA_MAX_AGE=168   # 保留最近 168 小时（7 天）
```

清理范围（仅删以下）：
- `execution_entity` — 执行元数据（时间/状态/模式）
- `execution_data` — 全节点中间数据（占 DB 95%+ 空间）

不受影响（永不清理）：

| 数据 | 表 |
|------|------|
| 工作流定义 | `workflow_entity` |
| 工作流版本 | `workflow_history` |
| 凭据 | `credentials_entity` |
| 变量 | `variables` |
| Webhook 注册 | `webhook_entity` |
| 用户账号 | `user` |
| 实例设置 | `settings` |
| 社区节点 | `installed_nodes` / `installed_packages` |

### 1.3 手动清理（紧急膨胀）

```bash
# 1. 停止 n8n
docker stop ${N8N_CONTAINER}

# 2. 删除 30 天以上执行数据并压缩
sqlite3 "${N8N_DATA_PATH}/database.sqlite" "
PRAGMA foreign_keys = OFF;
BEGIN TRANSACTION;
DELETE FROM execution_data WHERE executionId IN (
  SELECT id FROM execution_entity WHERE startedAt < datetime('now', '-30 days')
);
DELETE FROM execution_entity WHERE startedAt < datetime('now', '-30 days');
COMMIT;
VACUUM;
PRAGMA foreign_keys = ON;
"

# 3. 重启 n8n
docker start ${N8N_CONTAINER}
```

### 1.4 验证清理结果

```bash
sqlite3 "${N8N_DATA_PATH}/database.sqlite" "SELECT COUNT(*) FROM execution_entity;"
ls -lh "${N8N_DATA_PATH}/database.sqlite"
curl -s -H "X-N8N-API-KEY: ${N8N_API_KEY}" \
  "${N8N_HOST}/api/v1/workflows" \
  | python3 -c "import json,sys; print(f'workflows: {len(json.load(sys.stdin).get(\"data\",[]))}')"
```

---

## 2. 数据库完整性监控

### 2.1 定期检查

```bash
sqlite3 "${N8N_DATA_PATH}/database.sqlite" "PRAGMA integrity_check;"   # 期望 ok
ls -lh "${N8N_DATA_PATH}/database.sqlite"
sqlite3 "${N8N_DATA_PATH}/database.sqlite" \
  "SELECT status, COUNT(*) FROM execution_entity GROUP BY status;"
```

### 2.2 异常等级表

| 现象 | 等级 | 处理方式 |
|------|------|----------|
| `integrity_check` 返回 `ok` 但 DB > 500 MB | 警告 | 手动清理 + 确认自动清理已启用 |
| 日志偶发 `SQLITE_IOERR` | 警告 | `docker restart ${N8N_CONTAINER}` + 清理 |
| 日志持续 `SQLITE_CORRUPT / SQLITE_NOTADB` | 严重 | 停止 → 备份 → 手动清理 → VACUUM → 重启 |
| 执行长期卡在 `Webhook Trigger` | 严重 | 多由 DB 写入阻塞导致，处理同上 |
| `integrity_check` 返回错误信息 | 致命 | 使用备份恢复或重建实例 |

> Webhook Trigger 卡死：不要只看触发器本身，优先检查 DB 写入是否阻塞（`execution_data` 表锁 / WAL 滞留）。

---

## 3. PostgreSQL 迁移方案

### 3.1 何时迁移

满足任一条件即应考虑迁移：
- 反复出现 `SQLITE_CORRUPT / SQLITE_NOTADB`
- 执行数据增长快、自动清理后仍频繁膨胀
- 需要并发稳定性更高的存储后端

### 3.2 docker-compose 配置

```yaml
services:
  postgres:
    image: postgres:16
    container_name: n8n-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}"
      POSTGRES_DB: n8n
    volumes:
      - ${PG_DATA_PATH}:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  n8n:
    image: n8nio/n8n:latest
    container_name: local-n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    env_file:
      - .env
    environment:
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_PORT: 5432
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: n8n
      DB_POSTGRESDB_PASSWORD: "${POSTGRES_PASSWORD}"
    volumes:
      - ${N8N_DATA_PATH}:/home/node/.n8n
    depends_on:
      - postgres
```

`.env` 中添加：

```bash
POSTGRES_PASSWORD=YOUR_STRONG_PASSWORD
```

### 3.3 迁移步骤

```bash
# Step 1: 备份
cp "${N8N_DATA_PATH}/database.sqlite" \
   "${N8N_DATA_PATH}/database.sqlite.bak.$(date +%Y%m%d)"
curl -s -H "X-N8N-API-KEY: ${N8N_API_KEY}" \
  "${N8N_HOST}/api/v1/workflows" > workflows_backup.json

# Step 2: 启动新栈（n8n 首次连 PG 会自动建表并迁移工作流/凭据）
cd "${DOCKER_DEPLOY_PATH}"
docker compose down
docker compose up -d

# Step 3: 观察日志，等待 "n8n ready"
docker logs -f ${N8N_CONTAINER}
```

> 注意：n8n 自动迁移的是工作流和凭据；历史执行记录不保证完整迁移，迁移前务必备份。

### 3.4 验证

```bash
curl -s -H "X-N8N-API-KEY: ${N8N_API_KEY}" "${N8N_HOST}/api/v1/workflows" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(f'workflows: {len(d.get(\"data\",d))}')"
curl -sS -X POST "${N8N_HOST}/webhook/YOUR_WEBHOOK_PATH" \
  -H 'Content-Type: application/json' -d '{}'
```

### 3.5 回滚

```bash
cd "${DOCKER_DEPLOY_PATH}"
docker compose down
# 恢复旧 docker-compose.yml（移除 postgres 服务与 DB_* 环境变量）
# 恢复 SQLite 备份
cp "${N8N_DATA_PATH}/database.sqlite.bak.YYYYMMDD" \
   "${N8N_DATA_PATH}/database.sqlite"
docker compose up -d
```

---

## 4. 运维检查清单

### 变更前

- [ ] 备份 `${N8N_DATA_PATH}/database.sqlite`
- [ ] 记录当前工作流数量与活跃状态
- [ ] 确认 `EXECUTIONS_DATA_PRUNE=true` 已启用

### 变更后

- [ ] `PRAGMA integrity_check` 返回 `ok`
- [ ] 工作流数量与变更前一致
- [ ] 凭据数量与变更前一致
- [ ] 活跃工作流状态与变更前一致
- [ ] Webhook 路径可正常触发
- [ ] Docker 日志无 `SQLITE_CORRUPT / SQLITE_IOERR`
