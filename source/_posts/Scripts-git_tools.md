---

title: Scripts-管理多个 git 仓库
top_img: transparent
date: 2024-08-11 11:37:00
updated: 2024-09-16 22:15:39
tags:
  - git
categories: Scripts
keywords:
description: 对多个 git 仓库进行批量操作。
---

## 脚本

```bash
#!/bin/bash

# ANSI颜色码
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BLUE='\033[0;34m'
NC='\033[0m' # 恢复默认颜色

# Define special characters
OK_STATUS=$'\u2714'   # ✅
FAIL_STATUS=$'\u274C' # ❌

# 保存当前目录
current_dir=$(pwd)
command_to_execute="\$1"

# git log
function git_log {

	# 获取当前分支名称
	current_branch=$(git rev-parse --abbrev-ref HEAD)

	# 获取默认远程仓库名称
	remote=$(git for-each-ref --format='%(upstream:short)' "$(git symbolic-ref -q HEAD)")
	remote_name=${remote%%/*}

	# 检查是否存在默认远程仓库
	if [ -z "$remote" ]; then
		echo -e "${YELLOW}当前分支没有追踪远程分支。${NO_COLOR}"
	else
		# 检查本地是否有未推送的提交
		local_commits=$(git log ${remote}..HEAD --oneline)
		if [ -n "$local_commits" ]; then
			echo -e "${YELLOW}存在未同步到远程仓库的提交 (${FAIL_STATUS}):${NO_COLOR}"
			echo "$local_commits"
		else
			echo -e "${GREEN}当前分支已经全部同步到远程仓库 (${OK_STATUS} ).${NO_COLOR}"
		fi

		# 检查远程是否有未拉取的提交
		git fetch ${remote_name} >/dev/null 2>&1
		remote_commits=$(git log HEAD..${remote} --oneline)
		if [ -n "$remote_commits" ]; then
			echo -e "${YELLOW}远程仓库存在未拉取的提交 (${FAIL_STATUS}):${NO_COLOR}"
			echo "$remote_commits"
		else
			echo -e "${GREEN}远程仓库的提交已经全部拉取到本地 (${OK_STATUS} ).${NO_COLOR}"
		fi
	fi
}

# git status function
function git_status {
	git status
	echo -e ""
	git_log
	echo -e "${BLUE}==========================================================================${NO_COLOR}"
}

# 遍历当前目录下的所有子目录
for dir in */ ; do
	# 进入子目录
	cd "$dir"

	# 检查子目录中是否存在.git目录
	if [ -d ".git" ]; then
		echo -e "${BLUE}Executing '$1', Current directory: $(pwd)${NC}"
		# 执行传递进来的命令
		eval "$command_to_execute"
	else
		echo -e "${YELLOW}$dir is not a git repository${NC}"
	fi

	# 返回初始目录
	cd "$current_dir"
done

echo "Operation completed for all repositories."
```

将上面的脚本添加到 `bin` 目录下，或者增加 `PATH` 环境变量，以便任意位置都可以执行。

## 用法

以上脚本命名为 gits，用法如下：

```bash
gits "git_status" # 遍历每个子目录，如果是 git 仓库，执行 git status
                  # 并检查是否有未同步到远程仓库的提交，或者未拉取到本地仓库的提交

gits "git xxx" # 遍历每个子目录，如果是 git 仓库，执行 git xxx
               # 例如 gits "git fetch"
```
