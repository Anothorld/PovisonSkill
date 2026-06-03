# PovisonSkill

Povison 自定义 Hermes Agent 技能仓库。所有自研技能统一存放于此，方便跨机器 clone 同步。

## 使用方法

```bash
# 1. Clone 到任意机器
git clone <repo-url> ~/agent_prj/PovisonSkill

# 2. 将技能目录软链接到 Hermes skills 目录
#    默认 profile：
ln -s ~/agent_prj/PovisonSkill/productivity/feishu-doc-writer ~/.hermes/skills/productivity/feishu-doc-writer

#    Named profile（如 kol-orchestrator）：
ln -s ~/agent_prj/PovisonSkill/productivity/feishu-doc-writer ~/.hermes/profiles/kol-orchestrator/skills/productivity/feishu-doc-writer
```

## 目录结构

```
PovisonSkill/
├── productivity/      # 生产力工具（文档、表格、邮件等）
│   └── feishu-doc-writer/
├── social-media/      # 社交媒体相关技能
├── kol/               # KOL 运营技能
├── devops/            # DevOps & 自动化
├── data-science/      # 数据分析
└── creative/          # 创意内容生成
```

## 添加新技能

1. 在对应分类目录下创建技能文件夹
2. 编写 `SKILL.md`（含 YAML frontmatter）
3. 提交并推送到远程仓库
4. 在目标机器 `git pull` 同步

## 技能列表

| 技能 | 分类 | 说明 |
|------|------|------|
| feishu-doc-writer | productivity | 飞书云文档创建/追加/权限设置 |
| povison-skill-sync | devops | PovisonSkill Git 仓库技能同步（上传/下载/全量同步） |
