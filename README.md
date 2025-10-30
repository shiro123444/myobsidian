# myobsidian

个人 Obsidian 笔记同步仓库 / Personal Obsidian Notes Sync Repository

## 简介 / Introduction

这是一个用于通过 Git 同步 Obsidian 笔记的仓库。使用 Git 进行版本控制可以实现跨设备同步，并保留完整的修改历史。

This repository is for syncing Obsidian notes via Git. Using Git for version control enables cross-device synchronization while maintaining a complete history of changes.

## 使用方法 / Usage

### 初次设置 / Initial Setup

1. 克隆此仓库到本地：
   ```bash
   git clone https://github.com/shiro123444/myobsidian.git
   ```

2. 在 Obsidian 中打开此文件夹作为 vault（文件 -> 打开文件夹作为仓库）

3. 开始编写笔记！

### 同步笔记 / Syncing Notes

#### 推送更改到远程仓库 / Push Changes to Remote

```bash
git add .
git commit -m "更新笔记"
git push
```

#### 从远程仓库拉取更新 / Pull Updates from Remote

```bash
git pull
```

### 建议的工作流程 / Recommended Workflow

1. 开始工作前先拉取最新更改：`git pull`
2. 编写和修改笔记
3. 定期提交更改：`git add . && git commit -m "描述更改"`
4. 推送到远程：`git push`

## 注意事项 / Notes

- `.gitignore` 文件已配置为排除 Obsidian 的工作区和缓存文件
- 建议使用有意义的提交信息，方便追踪笔记变化
- 在多设备间同步时，注意先拉取再推送，避免冲突

## 许可证 / License

详见 LICENSE 文件。
