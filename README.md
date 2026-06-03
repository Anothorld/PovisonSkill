# PovisonSkill

Povison 自定义 Hermes Agent 技能仓库。所有自研技能统一存放于此，方便跨机器 clone 同步。

**只同步你明确指定的技能，不会触碰 Hermes 内置或第三方技能。**

---

## 快速开始（新机器从零安装）

### 1. Clone 仓库

```bash
git clone git@github.com:Anothorld/PovisonSkill.git ~/agent_prj/PovisonSkill
```

### 2. 告诉 Hermes Agent 同步技能

在 Hermes 对话中直接说：

> "同步技能" 或 "skill sync"

Agent 会自动：
- `git pull` 拉取最新
- 读取 `MANIFEST.txt` 白名单
- 将白名单中的技能软链接到 Hermes skills 目录

### 3. 完成！

技能已可用。以后新增技能只需在任意机器说"同步技能"即可。

---

## 配置

### 环境变量（可选）

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `POVISON_SKILL_REPO` | `~/agent_prj/PovisonSkill` | 仓库本地路径 |

### 飞书应用凭据

`feishu-doc-writer` 技能需要飞书凭据，在 profile 的 `.env` 中配置：

```bash
# 默认 profile
echo 'FEISHU_APP_ID=cli_xxxxx' >> ~/.hermes/.env
echo 'FEISHU_APP_SECRET=*** >> ~/.hermes/.env

# Named profile（如 kol-orchestrator）
echo 'FEISHU_APP_ID=cli_xxxxx' >> ~/.hermes/profiles/kol-orchestrator/.env
echo 'FEISHU_APP_SECRET=*** >> ~/.hermes/profiles/kol-orchestrator/.env
```

飞书应用权限（开发者后台一次配置，全平台生效）：
- `docx:document` — 创建/写入文档
- `drive:drive` — 设置文档分享权限
- `authen:v3.tenant_access_token` — 获取 tenant token

---

## 日常使用

所有操作通过 Hermes 对话完成，不需要手动敲命令。

| 你对 Agent 说 | Agent 做什么 |
|---------------|-------------|
| "同步技能" | `git pull` + 软链接 MANIFEST 中的技能 |
| "上传 xxx 技能" | 复制到 repo + 加 MANIFEST + commit + push |
| "移除 xxx 技能" | 从 MANIFEST/repo/Hermes 三处移除 |
| "查看技能清单" | 列出 MANIFEST 内容 |

### 上传技能示例

```
你: 上传 kol-outreach-orchestrator-flow 技能到 repo
Agent: 
  📦 复制: kol-outreach-orchestrator-flow -> repo/kol/...
  🔗 替换为软链接
  📝 已添加到 MANIFEST.txt
  ✅ Committed & Pushed
```

---

## MANIFEST.txt 白名单

`MANIFEST.txt` 是唯一的同步清单，**只有列在其中的技能才会被同步**。

```
# 格式：skill-name | category | 描述
feishu-doc-writer | productivity | 飞书云文档创建/追加/权限设置
povison-skill-sync | devops | 技能同步（上传/下载）
```

- 添加技能 = 对 agent 说"上传 xxx 技能"
- 移除技能 = 对 agent 说"移除 xxx 技能"
- **不要手动编辑 MANIFEST.txt**，让 agent 来管理以确保一致性

---

## 目录结构

```
PovisonSkill/
├── MANIFEST.txt                    # 同步白名单
├── README.md                       # 本文档
├── .gitignore                      # 排除 .env 等敏感文件
├── .env.example                    # 凭据模板
├── productivity/
│   └── feishu-doc-writer/
│       └── SKILL.md                # 飞书云文档技能
├── devops/
│   └── povison-skill-sync/
│       └── SKILL.md                # 技能同步技能
├── social-media/                   # 社交媒体技能
├── kol/                            # KOL 运营技能
├── data-science/                   # 数据分析技能
└── creative/                       # 创意内容技能
```

---

## 技能列表

| 技能 | 分类 | 说明 |
|------|------|------|
| feishu-doc-writer | productivity | 飞书云文档创建/追加/权限设置 |
| povison-skill-sync | devops | 技能同步（上传/下载/白名单管理） |

---

## 多机器工作流

```
机器A: "上传 xxx 技能" → git push
机器B: "同步技能"       → git pull + 软链接
```

技能在 Hermes 中以软链接形式存在，修改 repo 或 Hermes 侧的 SKILL.md 都会自动双向同步。

---

## 手动安装（不使用 Agent）

如果你不用 Hermes Agent 或想手动操作：

```bash
# 1. Clone
git clone git@github.com:Anothorld/PovisonSkill.git ~/agent_prj/PovisonSkill

# 2. 手动软链接需要的技能（以默认 profile 为例）
ln -s ~/agent_prj/PovisonSkill/productivity/feishu-doc-writer \
      ~/.hermes/skills/productivity/feishu-doc-writer

ln -s ~/agent_prj/PovisonSkill/devops/povison-skill-sync \
      ~/.hermes/skills/devops/povison-skill-sync

# Named profile 示例
# ln -s ~/agent_prj/PovisonSkill/productivity/feishu-doc-writer \
#       ~/.hermes/profiles/<profile-name>/skills/productivity/feishu-doc-writer

# 3. 配凭据（按需）
echo 'FEISHU_APP_ID=cli_xxxxx' >> ~/.hermes/.env
echo 'FEISHU_APP_SECRET=*** >> ~/.hermes/.env
```

---

## 注意事项

- **白名单模式**：只同步 MANIFEST.txt 中的技能，不会自动扫描全部
- **软链接优先**：Hermes 中的技能目录是软链接 → repo，修改任一侧自动同步
- **绝不提交凭据**：`.gitignore` 已排除 `.env`，敏感信息不会进入 git
- **Named profile 路径**：`~` 在 named profile 下会重定向到 `~/.hermes/profiles/<name>/home/`，使用绝对路径更安全
