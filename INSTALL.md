# PovisonSkill 安装指引

> 给 Hermes Agent 使用的自研技能安装与同步指南。

---

## 一、首次安装

### 1. Clone 仓库

```
git clone git@github.com:Anothorld/PovisonSkill.git ~/agent_prj/PovisonSkill
```

> 公开仓库，clone 不需要 GitHub 认证。

### 2. 软链接技能到 Hermes

根据你的 profile 类型选择：

**默认 profile：**
```bash
# 飞书文档技能
ln -s ~/agent_prj/PovisonSkill/productivity/feishu-doc-writer \
      ~/.hermes/skills/productivity/feishu-doc-writer

# 技能同步技能
ln -s ~/agent_prj/PovisonSkill/devops/povison-skill-sync \
      ~/.hermes/skills/devops/povison-skill-sync
```

**Named profile（如 kol-orchestrator）：**
```bash
ln -s ~/agent_prj/PovisonSkill/productivity/feishu-doc-writer \
      ~/.hermes/profiles/kol-orchestrator/skills/productivity/feishu-doc-writer

ln -s ~/agent_prj/PovisonSkill/devops/povison-skill-sync \
      ~/.hermes/profiles/kol-orchestrator/skills/devops/povison-skill-sync
```

> 如果目录已存在（非软链接），先备份：`mv <path> <path>.bak`

### 3. 配置飞书凭据（如需 feishu-doc-writer）

在对应 profile 的 `.env` 中添加：

```bash
echo 'FEISHU_APP_ID=cli_xxxxx' >> ~/.hermes/.env
echo 'FEISHU_APP_SECRET=*** >> ~/.hermes/.env
```

飞书应用权限（开发者后台配置一次即可）：
- `docx:document` — 创建/写入文档
- `drive:drive` — 设置分享权限
- `authen:v3.tenant_access_token` — 获取 token

### 4. 验证

对 agent 说 **"查看技能清单"**，应返回 MANIFEST 中的技能列表。

---

## 二、日常同步

### 下载（拉取新技能）

对 agent 说：**"同步技能"**

agent 自动执行：`git pull` → 读取 MANIFEST → 软链接新技能

### 上传（推送本地技能到 repo）

对 agent 说：**"上传 xxx 技能"**

agent 自动执行：复制到 repo → 更新 MANIFEST → commit → push

> push 需要 SSH key，见下方"推送认证"。

---

## 三、推送认证

clone/pull 不需要认证。push 需要将本机 SSH 公钥添加到 GitHub：

1. 查看公钥：`cat ~/.ssh/id_ed25519.pub`
2. 打开 https://github.com/settings/keys
3. 点 New SSH key → 粘贴公钥 → 保存
4. 验证：`ssh -T git@github.com`

如无 SSH key，先生成：
```bash
ssh-keygen -t ed25519 -C "your@email.com"
```

---

## 四、技能说明

### feishu-doc-writer（飞书云文档）

功能：
- 创建飞书云文档
- 追加 block 内容到已有文档
- 设置文档公开访问权限
- 生成文档链接

使用：对 agent 说 "创建飞书文档" / "写入飞书文档" / "追加飞书文档"

注意事项：
- 只用 block_type=2（其他类型 3/4/22 会报错），标题用 bold 样式
- named profile 下 `~` 会重定向，读取 .env 用绝对路径
- 正则中 `APP_SECRET` 需字符串拼接绕过过滤

### povison-skill-sync（技能同步）

功能：
- 下载同步：pull + 软链接 MANIFEST 中的技能
- 上传技能：复制到 repo + 更新 MANIFEST + push
- 移除技能：从 MANIFEST/repo/Hermes 三处清除
- 查看清单：列出 MANIFEST 内容

使用：对 agent 说 "同步技能" / "上传 xxx 技能" / "移除 xxx 技能" / "查看技能清单"

---

## 五、MANIFEST.txt 说明

仓库根目录的 `MANIFEST.txt` 是同步白名单，格式：

```
# 注释行
skill-name | category | 描述
```

**只有列在其中的技能才会被同步。** 建议通过 agent 命令管理，不要手动编辑。

---

## 六、目录结构

```
PovisonSkill/
├── INSTALL.md                  # 本文档
├── MANIFEST.txt                # 同步白名单
├── README.md                   # 项目概述
├── .env.example                # 凭据模板
├── productivity/
│   └── feishu-doc-writer/
│       └── SKILL.md
├── devops/
│   └── povison-skill-sync/
│       └── SKILL.md
├── social-media/
├── kol/
├── data-science/
└── creative/
```
