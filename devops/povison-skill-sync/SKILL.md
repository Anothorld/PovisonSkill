---
name: povison-skill-sync
version: 1.0.0
description: 从 PovisonSkill Git 仓库同步技能到 Hermes，支持上传（commit+push）和下载（pull+link）。跨机器技能管理一站式工具。
triggers:
  - 同步技能
  - 上传技能
  - 下载技能
  - skill sync
  - 技能同步
  - povison skill
---

# PovisonSkill Sync

从 PovisonSkill Git 仓库同步技能到本地 Hermes，支持双向操作。

## 配置

在 Hermes profile `.env` 或 config 中设置（仅首次需要）：

```
POVISON_SKILL_REPO=/Users/arnold/agent_prj/PovisonSkill
```

若未设置，默认路径为 `~/agent_prj/PovisonSkill`。

获取当前 repo 路径：
```python
import os
repo = os.environ.get('POVISON_SKILL_REPO', os.path.expanduser('~/agent_prj/PovisonSkill'))
# named profile 下 ~ 会重定向，用绝对路径更安全
profile_home = os.environ.get('HERMES_HOME', os.path.expanduser('~/.hermes'))
```

## 操作一：下载同步（pull + link）

从远程拉取最新技能，并自动软链接到 Hermes skills 目录。

### 步骤

1. `git pull` 拉取最新
2. 扫描 repo 中所有含 `SKILL.md` 的目录
3. 对比 Hermes skills 目录，找出新增/未链接的技能
4. 自动创建软链接

### 代码模板

```python
import subprocess, os, glob

# 配置
repo = os.environ.get('POVISON_SKILL_REPO', '/Users/arnold/agent_prj/PovisonSkill')
profile_home = '/Users/arnold/.hermes/profiles/kol-orchestrator'  # 或 ~/.hermes（默认profile）
hermes_skills = os.path.join(profile_home, 'skills')

# 1. Pull
r = subprocess.run(['git', 'pull'], cwd=repo, capture_output=True, text=True, timeout=30)
print(f"Pull: {r.stdout.strip()}")
if r.returncode != 0:
    print(f"Error: {r.stderr.strip()}")
    # 可能需要先 commit 本地变更
    raise Exception("git pull failed")

# 2. 扫描 repo 中的技能
repo_skills = []
for sk in glob.glob(os.path.join(repo, '*', '*', 'SKILL.md')):
    # 路径格式: repo/category/skill-name/SKILL.md
    rel = os.path.relpath(sk, repo)           # category/skill-name/SKILL.md
    category_skill = os.path.dirname(rel)      # category/skill-name
    repo_skills.append(category_skill)

print(f"Repo skills: {len(repo_skills)}")
for s in repo_skills:
    print(f"  {s}")

# 3. 检查并创建软链接
linked = 0
skipped = 0
for skill_rel in repo_skills:
    repo_path = os.path.join(repo, skill_rel)
    hermes_path = os.path.join(hermes_skills, skill_rel)

    if os.path.islink(hermes_path):
        # 已是软链接，检查是否指向正确
        target = os.readlink(hermes_path)
        if os.path.samefile(hermes_path, repo_path):
            skipped += 1
            continue
        else:
            print(f"⚠️ {skill_rel} 链接指向不同位置，需手动处理")
            continue
    elif os.path.exists(hermes_path):
        # 存在实体目录，需先备份再链接
        backup = hermes_path + '.bak'
        os.rename(hermes_path, backup)
        print(f"📦 备份旧目录: {skill_rel} -> {os.path.basename(backup)}")
        os.symlink(repo_path, hermes_path)
        linked += 1
        continue

    # 不存在，创建软链接
    os.makedirs(os.path.dirname(hermes_path), exist_ok=True)
    os.symlink(repo_path, hermes_path)
    linked += 1
    print(f"✅ 链接: {skill_rel}")

print(f"\n结果: 新链接 {linked}, 已存在 {skipped}")
```

## 操作二：上传同步（commit + push）

将新技能添加到 repo，提交并推送到远程。

### 步骤

1. 确认技能目录含有效 `SKILL.md`
2. 复制或移动技能到 repo 对应分类
3. 更新 Hermes skills 为软链接（如果原来是实体目录）
4. 更新 README.md 技能列表
5. `git add + commit + push`

### 代码模板

```python
import subprocess, os, re

repo = os.environ.get('POVISON_SKILL_REPO', '/Users/arnold/agent_prj/PovisonSkill')
profile_home = '/Users/arnold/.hermes/profiles/kol-orchestrator'
hermes_skills = os.path.join(profile_home, 'skills')

def upload_skill(skill_name, category='productivity', source_path=None):
    """
    上传技能到 PovisonSkill repo。

    Args:
        skill_name: 技能名称（目录名）
        category: 分类（productivity/social-media/kol/devops/data-science/creative）
        source_path: 源路径。若为 None，从 hermes_skills 中查找。
    """
    # 确定源路径
    if source_path is None:
        # 在 hermes skills 中查找
        for root, dirs, files in os.walk(hermes_skills):
            if 'SKILL.md' in files and os.path.basename(root) == skill_name:
                source_path = root
                break

    if source_path is None or not os.path.exists(os.path.join(source_path, 'SKILL.md')):
        raise FileNotFoundError(f"技能 {skill_name} 未找到或缺少 SKILL.md")

    # 目标路径
    dest_path = os.path.join(repo, category, skill_name)

    # 如果已是软链接指向 repo，无需复制
    if os.path.islink(source_path) and os.path.samefile(source_path, dest_path):
        print(f"✅ {skill_name} 已在 repo 中（软链接）")
    else:
        # 复制到 repo
        if os.path.exists(dest_path):
            subprocess.run(['rm', '-rf', dest_path], capture_output=True)
        subprocess.run(['cp', '-r', source_path, dest_path], capture_output=True)
        print(f"📦 复制: {source_path} -> {dest_path}")

        # 替换 hermes 目录为软链接
        if os.path.isdir(source_path) and not os.path.islink(source_path):
            import shutil
            shutil.rmtree(source_path)
            os.symlink(dest_path, source_path)
            print(f"🔗 替换为软链接: {source_path} -> {dest_path}")

    # 读取 SKILL.md 获取描述
    skill_desc = skill_name
    with open(os.path.join(dest_path, 'SKILL.md')) as f:
        m = re.search(r'description:\s*(.+)', f.read())
        if m:
            skill_desc = m.group(1).strip()

    # 更新 README.md
    _update_readme(repo, skill_name, category, skill_desc)

    # Git commit & push
    subprocess.run(['git', 'add', '-A'], cwd=repo, capture_output=True, timeout=10)
    r = subprocess.run(['git', 'commit', '-m', f'feat: add {skill_name} skill'],
        cwd=repo, capture_output=True, text=True, timeout=10)

    if r.returncode == 0:
        print(f"✅ Committed: feat: add {skill_name} skill")
    else:
        print(f"ℹ️ No changes to commit")

    r = subprocess.run(['git', 'push'], cwd=repo, capture_output=True, text=True, timeout=30)
    if r.returncode == 0:
        print(f"✅ Pushed to remote")
    else:
        print(f"❌ Push failed: {r.stderr.strip()[:200]}")

    return dest_path


def _update_readme(repo, skill_name, category, description):
    """更新 README.md 中的技能列表表格"""
    readme_path = os.path.join(repo, 'README.md')

    with open(readme_path) as f:
        content = f.read()

    # 检查是否已存在
    if skill_name in content:
        return

    new_row = f"| {skill_name} | {category} | {description} |"

    # 找到表格末尾追加
    lines = content.split('\n')
    table_end = -1
    for i, line in enumerate(lines):
        if line.startswith('|') and i > table_end:
            table_end = i

    if table_end > 0:
        lines.insert(table_end + 1, new_row)
        with open(readme_path, 'w') as f:
            f.write('\n'.join(lines))
        print(f"📝 Updated README.md")
```

### 使用示例

```python
# 上传现有技能到 repo
upload_skill('kol-outreach-orchestrator-flow', category='kol')

# 从自定义路径上传
upload_skill('my-custom-skill', category='productivity',
             source_path='/path/to/my-custom-skill')
```

## 操作三：一键全量同步

先 pull 再检查差异，确保本地与远程完全一致。

```python
import subprocess, os, glob

repo = os.environ.get('POVISON_SKILL_REPO', '/Users/arnold/agent_prj/PovisonSkill')
profile_home = '/Users/arnold/.hermes/profiles/kol-orchestrator'
hermes_skills = os.path.join(profile_home, 'skills')

# 1. Stash 本地变更 + Pull
subprocess.run(['git', 'stash'], cwd=repo, capture_output=True, timeout=10)
subprocess.run(['git', 'pull', '--rebase'], cwd=repo, capture_output=True, text=True, timeout=30)
subprocess.run(['git', 'stash', 'pop'], cwd=repo, capture_output=True, timeout=10)

# 2. 扫描 repo 技能
repo_skills = []
for sk in glob.glob(os.path.join(repo, '*', '*', 'SKILL.md')):
    rel = os.path.relpath(sk, repo)
    repo_skills.append(os.path.dirname(rel))

# 3. 扫描 hermes 技能（仅非软链接的 = 本地独有的）
local_only = []
for sk in glob.glob(os.path.join(hermes_skills, '*', '*', 'SKILL.md')):
    full_path = os.path.dirname(sk)
    if not os.path.islink(full_path):
        rel = os.path.relpath(sk, hermes_skills)
        local_only.append(os.path.dirname(rel))

print(f"Repo 技能: {len(repo_skills)}")
print(f"本地独有（未上传）: {len(local_only)}")
for s in local_only:
    print(f"  📤 待上传: {s}")

# 4. 链接 repo 技能（见「下载同步」代码）
# ... 复用上面的链接逻辑

# 5. 如需上传本地独有技能
# for skill_rel in local_only:
#     category = skill_rel.split('/')[0]
#     upload_skill(os.path.basename(skill_rel), category=category)
```

## 注意事项

1. **软链接优先** — repo 中的技能在 Hermes 中必须是软链接，这样修改任一侧都自动同步
2. **.gitignore 排除 .env** — 绝不将凭据提交到 repo
3. **named profile 路径** — `~` 在 named profile 下会重定向，务必用绝对路径
4. **pull 冲突** — 如果本地和远程都修改了同一 SKILL.md，需要手动解决冲突
5. **SKILL.md 必须存在** — 没有该文件的目录不会被识别为技能

## 快速参考

| 命令 | 操作 |
|------|------|
| 下载同步 | `git pull` + 扫描 + 软链接 |
| 上传技能 | 复制到 repo + `git add/commit/push` |
| 全量同步 | pull → 链接 → 上传本地独有 |
