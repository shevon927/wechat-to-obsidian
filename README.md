# wechat-to-obsidian Skill 安装和上传指南

这是一份给新手看的完整指南，记录了如何在 Codex 协助下，把一个本地 Codex skill 上传到 GitHub，并让别人可以下载安装到自己的 Codex 里。

本示例使用的信息：

- GitHub 用户名：`shevon927`
- GitHub 仓库名：`wechat-to-obsidian`
- Skill 名称：`wechat-to-obsidian`
- 仓库地址：`https://github.com/shevon927/wechat-to-obsidian`

## 这个仓库是什么

这个仓库里放的是一个 Codex skill：

```text
codex-skills-src/wechat-to-obsidian/
├── SKILL.md
└── agents/
    └── openai.yaml
```

其中：

- `SKILL.md` 是 skill 的核心说明文件。
- `agents/openai.yaml` 是 Codex 里展示名称、简介、默认提示词等信息的配置。

## 一、在 GitHub 创建仓库

打开 GitHub 新建仓库页面：

```text
https://github.com/new
```

按照截图里的信息填写：

- `Owner` 选择：`shevon927`
- `Repository name` 填：`wechat-to-obsidian`
- `Description` 可以填：`下载微信公众号文章到obsidian`
- `Choose visibility` 选择：`Public`
- `Add README` 先保持关闭
- `Add .gitignore` 选择：`No .gitignore`
- `Add license` 选择：`No license`

然后点击 `Create repository`。

说明：这里选择 `Public`，别人才能直接下载你的 skill。

## 二、在本地整理 skill 文件

Codex 已协助把原始 Markdown 文件整理成标准 skill 目录：

```text
codex-skills-src/wechat-to-obsidian/SKILL.md
codex-skills-src/wechat-to-obsidian/agents/openai.yaml
```

如果你以后要上传另一个 skill，也按这个格式放：

```text
codex-skills-src/你的skill名字/SKILL.md
codex-skills-src/你的skill名字/agents/openai.yaml
```

## 三、提交到本地 Git

进入项目目录：

```bash
cd /Users/chenxiaofeng/Documents/壁纸自动化设计
```

添加要上传的文件：

```bash
git add .gitignore codex-skills-src/wechat-to-obsidian
```

提交：

```bash
git commit -m "Add wechat-to-obsidian skill"
```

如果终端提示有其他 `Untracked files`，例如：

```text
mockup-studio/
outputs/
video-review/
```

这不是错误。意思只是这些文件夹还没有加入 Git。只要你这次不想上传它们，就不用管。

## 四、连接 GitHub 仓库

先把分支命名为 `main`：

```bash
git branch -M main
```

设置远程仓库地址：

```bash
git remote add origin https://github.com/shevon927/wechat-to-obsidian.git
```

如果出现：

```text
error: remote origin already exists.
```

说明已经有 `origin` 了，不要重复添加，改用：

```bash
git remote set-url origin https://github.com/shevon927/wechat-to-obsidian.git
```

## 五、推送到 GitHub

执行：

```bash
git push -u origin main
```

如果终端问：

```text
Username for 'https://github.com':
```

输入你的 GitHub 用户名：

```text
shevon927
```

如果终端问：

```text
Password for 'https://shevon927@github.com':
```

这里不要输入 GitHub 登录密码。GitHub 现在不支持密码推送，需要输入 GitHub Personal Access Token。

创建 token 的步骤：

1. 打开 `https://github.com/settings/tokens/new`
2. `Note` 填：`codex upload`
3. `Expiration` 选 `30 days` 或 `90 days`
4. 勾选 `repo`
5. 点击 `Generate token`
6. 复制生成出来的一长串 token
7. 回到终端，在 `Password` 那里粘贴 token，然后回车

注意：终端里粘贴 token 时通常不会显示任何字符，这是正常的。

看到下面内容就表示成功：

```text
[new branch] main -> main
branch 'main' set up to track 'origin/main'
```

## 六、别人如何安装这个 skill

别人可以在终端执行：

```bash
git clone https://github.com/shevon927/wechat-to-obsidian.git
mkdir -p ~/.codex/skills
cp -R wechat-to-obsidian/codex-skills-src/wechat-to-obsidian ~/.codex/skills/
```

然后重启 Codex。

重启后就可以这样使用：

```text
Use $wechat-to-obsidian to save this WeChat public account article to my Obsidian vault.
```

或者直接说：

```text
用 wechat-to-obsidian 帮我把这篇微信公众号文章保存到 Obsidian
```

## 七、常见问题

### 1. 为什么 GitHub 说用户名或 token 错误？

通常是因为把命令误输入到了用户名位置，或者把 GitHub 登录密码当成 password 输入了。

正确做法：

- 用户名输入：`shevon927`
- password 位置粘贴：GitHub token

### 2. `remote origin already exists` 是什么意思？

意思是远程仓库地址已经设置过了。用下面命令改地址：

```bash
git remote set-url origin https://github.com/shevon927/wechat-to-obsidian.git
```

### 3. `Untracked files` 是不是错误？

不是。它只是提醒你有些文件还没有被 Git 管理。

如果这次只上传 skill，就只需要：

```bash
git add .gitignore codex-skills-src/wechat-to-obsidian
```

其他文件不用管。

### 4. 怎么确认已经上传成功？

打开这个地址：

```text
https://github.com/shevon927/wechat-to-obsidian
```

能看到 `codex-skills-src/wechat-to-obsidian/SKILL.md`，就说明成功了。

