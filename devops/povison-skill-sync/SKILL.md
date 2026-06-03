---
name: povison-skill-sync
version: 1.1.0
description: 从 PovisonSkill Git 仓库同步指定技能到 Hermes，支持上传和下载。只同步用户明确指定的技能，不自动扫描全部。
triggers:
  - 同步技能
  - 上传技能
  - 下载技能
  - skill sync
  - 技能同步
  - povison skill
---

# PovisonSkill Sync

从 PovisonSkill Git 仓库同步技能到本地 Hermes。**只同步用户明确指定的技能**，基于 `MANIFEST.txt` 白名单管理。

## 配置

```
POVISON_SKILL_REPO=/Users/arnold/agent_prj/PovisonSkill
```

默认路径 `~/agent_prj/PovisonSkill`。

## MANIFEST.txt 白名单

Repo 根目录下的 `MANIFEST.txt` 是唯一的同步清单。格式：

```
# 注释行
skill-name | category | 描述
```

**只有出现在 MANIFEST.txt 中的技能才会被同步。** 其他技能（Hermes 内置的、第三方安装的）不会被触碰。

### 管理白名单

```python
import os

repo = os.environ.get('POVISON_SKILL_REPO', '/Users/arnold/agent_prj/PovisonSkill')
manifest_path = os.path.join(repo, 'MANIFEST.txt')

def read_manifest():
    """读取 MANIFEST.txt，返回 [{name, category, desc}]"""
    skills = []
    if not os.path.exists(manifest_path):
        return skills
    with open(manifest_path) as f:
        for line in f:
            line = line.strip()
            if not line or line.startswith('#'):
                continue
            parts = [p.strip() for p in line.split('|')]
            if len(parts) >= 2:
                skills.append({
                    'name': parts[0],
                    'category': parts[1],
                    'desc': parts[2] if len(parts) > 2 else ''
                })
    return skills

def add_to_manifest(skill_name, category, desc=''):
    """添加技能到白名单（如已存在则更新）"""
    skills = read_manifest()
    skills = [s for s in skills if s['name'] != skill_name]
    skills.append({'name': skill_name, 'category': category, 'desc': desc})
    _write_manifest(skills)

def remove_from_manifest(skill_name):
    """从白名单移除技能"""
    skills = read_manifest()
    skills = [s for s in skills if s['name'] != skill_name]
    _write_manifest(skills)

def _write_manifest(skills):
    """写入 MANIFEST.txt"""
    lines = ['# PovisonSkill Manifest',
             '# 只有列在此文件中的技能才会被同步到 repo',
             '# 格式：skill-name | category | 描述', '']
    for s in skills:
        lines.append(f"{s['name']} | {s['category']} | {s['desc']}")
    with open(manifest_path, 'w') as f:
        f.write('\n'.join(lines) + '\n')
```

## 操作一：下载同步（pull + link）

只链接 MANIFEST.txt 中列出的技能。

```python
import subprocess, os

repo = os.environ.get('POVISON_SKILL_REPO', '/Users/arnold/agent_prj/PovisonSkill')
profile_home = '/Users/arnold/.hermes/profiles/kol-orchestrator'  # 按实际调整
hermes_skills = os.path.join(profile_home, 'skills')

# 1. Pull
r = subprocess.run(['git', 'pull'], cwd=repo, capture_output=True, text=True, timeout=30)
print(f"Pull: {r.stdout.strip()}")

# 2. 读取白名单
skills = read_manifest()
print(f"Manifest skills: {len(skills)}")

# 3. 只链接白名单中的技能
linked = 0
skipped = 0
for s in skills:
    skill_rel = f"{s['category']}/{s['name']}"
    repo_path = os.path.join(repo, skill_rel)
    hermes_path = os.path.join(hermes_skills, skill_rel)

    # 检查 repo 中是否实际存在
    if not os.path.exists(os.path.join(repo_path, 'SKILL.md')):
        print(f"⚠️ {s['name']}: 在 MANIFEST 中但 repo 缺少 SKILL.md，跳过")
        continue

    # 已是正确软链接
    if os.path.islink(hermes_path) and os.path.exists(hermes_path):
        skipped += 1
        continue

    # 存在实体目录，先备份
    if os.path.isdir(hermes_path) and not os.path.islink(hermes_path):
        import shutil
        backup = hermes_path + '.bak'
        if os.path.exists(backup):
            shutil.rmtree(backup)
        os.rename(hermes_path, backup)
        print(f"📦 备份: {s['name']} -> {os.path.basename(backup)}")

    # 创建软链接
    os.makedirs(os.path.dirname(hermes_path), exist_ok=True)
    if os.path.islink(hermes_path):
        os.remove(hermes_path)
    os.symlink(repo_path, hermes_path)
    linked += 1
    print(f"✅ 链接: {s['name']}")

print(f"\n结果: 新链接 {linked}, 已存在 {skipped}")
```

## 操作二：上传技能（指定技能名）

将指定技能添加到 repo 并推送。**必须明确指定技能名。**

```python
import subprocess, os, re, shutil

repo = os.environ.get('POVISON_SKILL_REPO', '/Users/arnold/agent_prj/PovisonSkill')
profile_home = '/Users/arnold/.hermes/profiles/kol-orchestrator'
hermes_skills = os.path.join(profile_home, 'skills')

def upload_skill(skill_name, category=None):
    """
    上传指定技能到 repo。

    Args:
        skill_name: 技能名称（必须明确指定）
        category: 分类。为 None 时自动从 hermes skills 路径推断。
    """
    # 1. 在 hermes skills 中查找源
    source_path = None
    for root, dirs, files in os.walk(hermes_skills):
        if os.path.basename(root) == skill_name and 'SKILL.md' in files:
            source_path = root
            break

    if source_path is None:
        raise FileNotFoundError(f"技能 {skill_name} 在 Hermes skills 中未找到")

    # 2. 推断 category
    if category is None:
        rel = os.path.relpath(source_path, hermes_skills)
        parts = rel.split(os.sep)
        category = parts[0] if len(parts) > 1 else 'productivity'

    # 3. 读取描述
    desc = skill_name
    with open(os.path.join(source_path, 'SKILL.md')) as f:
        m = re.search(r'description:\s*(.+)', f.read())
        if m:
            desc = m.group(1).strip()

    # 4. 复制到 repo
    dest_path = os.path.join(repo, category, skill_name)

    if os.path.islink(source_path) and os.path.samefile(source_path, dest_path):
        print(f"✅ {skill_name} 已在 repo 中（软链接）")
    else:
        if os.path.exists(dest_path):
            shutil.rmtree(dest_path)
        shutil.copytree(source_path, dest_path)
        print(f"📦 复制: {skill_name} -> {dest_path}")

        # 替换 hermes 目录为软链接
        if os.path.isdir(source_path) and not os.path.islink(source_path):
            shutil.rmtree(source_path)
            os.symlink(dest_path, source_path)
            print(f"🔗 替换为软链接")

    # 5. 添加到 MANIFEST
    add_to_manifest(skill_name, category, desc)
    print(f"📝 已添加到 MANIFEST.txt")

    # 6. Git commit & push
    subprocess.run(['git', 'add', '-A'], cwd=repo, capture_output=True, timeout=10)
    r = subprocess.run(['git', 'commit', '-m', f'feat: add {skill_name} skill'],
        cwd=repo, capture_output=True, text=True, timeout=10)
    if r.returncode == 0:
        print(f"✅ Committed")
    else:
        print(f"ℹ️ No changes to commit")

    r = subprocess.run(['git', 'push'], cwd=repo, capture_output=True, text=True, timeout=30)
    if r.returncode == 0:
        print(f"✅ Pushed to remote")
    else:
        print(f"❌ Push failed: {r.stderr.strip()[:200]}")

# 使用示例 — 必须明确指定技能名
# upload_skill('kol-outreach-orchestrator-flow', category='kol')
```

## 操作三：从白名单移除

从 MANIFEST.txt 移除技能，并删除 repo 中对应目录。Hermes 中的软链接也会被清除。

```python
def remove_skill(skill_name):
    """从 repo 和 MANIFEST 中移除技能"""
    skills = read_manifest()
    target = None
    for s in skills:
        if s['name'] == skill_name:
            target = s
            break

    if target is None:
        print(f"⚠️ {skill_name} 不在 MANIFEST 中")
        return

    # 1. 从 MANIFEST 移除
    remove_from_manifest(skill_name)

    # 2. 删除 repo 目录
    repo_dir = os.path.join(repo, target['category'], skill_name)
    if os.path.isdir(repo_dir):
        shutil.rmtree(repo_dir)
        print(f"🗑️ 删除 repo 目录: {target['category']}/{skill_name}")

    # 3. 删除 Hermes 软链接
    hermes_path = os.path.join(hermes_skills, target['category'], skill_name)
    if os.path.islink(hermes_path):
        os.remove(hermes_path)
        print(f"🗑️ 删除软链接")
    elif os.path.isdir(hermes_path):
        print(f"⚠️ {hermes_path} 是实体目录，需手动处理")

    # 4. Git commit & push
    subprocess.run(['git', 'add', '-A'], cwd=repo, capture_output=True, timeout=10)
    subprocess.run(['git', 'commit', '-m', f'remove: {skill_name}'],
        cwd=repo, capture_output=True, text=True, timeout=10)
    subprocess.run(['git', 'push'], cwd=repo, capture_output=True, text=True, timeout=30)
    print(f"✅ Removed {skill_name}")
```

## 操作四：查看当前白名单

```python
def list_manifest():
    """列出 MANIFEST 中所有技能"""
    skills = read_manifest()
    if not skills:
        print("MANIFEST 为空")
        return
    print(f"{'技能名':<35} {'分类':<18} {'描述'}")
    print('-' * 80)
    for s in skills:
        print(f"{s['name']:<35} {s['category']:<18} {s['desc']}")
```

## 交互指令参考

| 用户说 | 操作 |
|--------|------|
| "同步技能" / "下载技能" | pull + 链接 MANIFEST 中的技能 |
| "上传 xxx 技能" | 复制到 repo + 加 MANIFEST + commit + push |
| "移除 xxx 技能" | 从 MANIFEST/repo/Hermes 三处移除 |
| "查看技能清单" | 列出 MANIFEST 内容 |

## 注意事项

1. **白名单模式** — 只同步 MANIFEST.txt 中列出的技能，不自动扫描
2. **必须明确指定** — 上传/移除操作需要用户明确说技能名，agent 不主动添加
3. **软链接优先** — repo 中的技能在 Hermes 中用软链接，修改任一侧自动同步
4. **.gitignore 排除 .env** — 绝不提交凭据
5. **named profile 路径** — `~` 在 named profile 下会重定向，用绝对路径
