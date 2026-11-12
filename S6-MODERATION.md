# S6 内容审核（补丁 + 管理面板）使用说明

本仓库 = Steve3184/new-api 的镜像 + 两样东西：
1. `patches/s6-moderation-for-steve3184.patch` —— 内容审核补丁（抽检 + 对话标记 + 拦截 + 预检）
2. `moderation-console/` —— 独立管理面板（单二进制小服务，不侵入 new-api）

CI（`.github/workflows/verify-moderation-patch.yml`）每天自动拉 Steve3184 上游最新代码，
试打补丁并编译 + 跑单测：绿 = 补丁仍然可用；红 = 上游改动导致补丁失效，需要重新对位。

---

## 一、更新你部署的 new-api 并带上审核功能

```bash
# 1. 拿代码（本仓库主分支 = 上游干净代码 + patches/ + 面板源码）
git clone https://github.com/freedom-winds/new-api.git
cd new-api

# 2. 打审核补丁
git apply patches/s6-moderation-for-steve3184.patch

# 3. 构建并重启（Dockerfile 无需修改；前端照常构建）
docker build -t new-api:s6-mod .
#   然后按你自己的方式重启容器（docker run / docker compose up -d）
#   首次启动自动建三张表：moderation_settings / moderation_marks / moderation_hits
```

**以后更新**：拉上游最新代码 → 重新 `git apply` 补丁 → 重新构建 → 重启。
CI 每天替你验证补丁对上游是否仍可打，红了再动手。

## 二、开启审核（必须先配检测通道）

往 new-api 的数据库执行（表启动时已自动建好）：

```sql
INSERT INTO moderation_settings(id, config, updated_at) VALUES (1, '{
  "enabled": true,
  "mode": "observe",
  "sample_rate_bp": 500,
  "deny": ["nsfw","cyberattack","reverse_web","reverse_commercial"],
  "min_confidence": 0.8,
  "min_hash_len": 24,
  "check_timeout_sec": 60,
  "preflight_timeout_sec": 30,
  "max_check_chars": 6000,
  "tail_grace_sec": 3,
  "endpoint": {
    "base_url": "https://你的检测通道地址/v1",
    "api_key": "你自己的key",
    "model": "glm-5.3-flash"
  },
  "templates": {
    "block": "【S6AI提示】我们的系统抽样检测到当前对话存在【{category}】问题，已截停当前对话。若有误报请记录并联系站点管理员，并开新对话重试。",
    "marked": "【S6AI提示】当前对话已被标记存在【{category}】问题，请勿再次请求。若有误报请记录并联系站点管理员，并开新对话重试。"
  },
  "groups": {}
}', extract(epoch from now())::bigint) ON CONFLICT (id) DO UPDATE SET config = EXCLUDED.config;
```

（上面是 PostgreSQL 写法；MySQL 把最后的时间戳换成 `UNIX_TIMESTAMP()`，SQLite 用 `strftime('%s','now')`。）

要点：
- **检测通道填你自己的 GLM 渠道**（任何 OpenAI 兼容地址 + key + 模型），不要用别人的。
- 网关每 30 秒轮询一次配置，改完自动生效，不用重启。
- 模式：`observe` 只记录不拦截（先跑一两周看误报率）→ `enforce` 拦截。每个分组可在 `groups` 里单独覆盖（模式/抽检比例/违规类别/预检）。
- **预检（preflight）**：分组级开关，默认关。开启后被抽检到的请求先扣住不发模型，等检测结果：无违规才放行、违规直接截停（零费用）；仅 enforce 模式生效，检测超时自动放行。

## 三、部署管理面板（可选，但强烈建议）

面板已独立成单独仓库（含一键部署脚本）：

- 仓库：https://github.com/freedom-winds/newapi-antiAnti
- 一键部署：`git clone https://github.com/freedom-winds/newapi-antiAnti.git && cd newapi-antiAnti && sudo bash deploy.sh`
  （问域名 → 问数据库 DSN → 端口 28906 被占自动顺延 → 自动定位 nginx server 块注入 location，
  注入前备份、nginx -t 失败自动还原；未检测到 nginx 自动取消）
- 预编译二进制：https://github.com/freedom-winds/newapi-antiAnti/releases

面板是独立服务，不碰 new-api 代码，直连 new-api 的 PostgreSQL 读写三张审核表。
数据库不是 PG 的站点：先用 SQL 手工管理 `moderation_settings`，或提 issue 催加驱动。

## 四、行为速览

- 每请求固定概率抽检，检测异步进行（glm-5.3-flash 带思考约 8~15 秒），不影响正常转发。
- 检测失败/超时 = 放行（fail-open），只记一条 error。
- 对话标识用 AI 回复哈希：被标记的对话继续追问会被直接拒答；**重新开新对话不受影响**（哪怕首句一字不差）。
- 一期不审：Responses API / embedding / 图像 / 音频；非流式请求只做"标记、下次拦"。
