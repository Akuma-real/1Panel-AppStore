# 1Panel-AppStore 使用说明

## 简介
1Panel-AppStore 是一个本人个人维护的应用商店项目，感谢 okxlin 提供的部署脚本及相关应用。该项目旨在解决官方应用商店无法满足用户需求的问题。

## 安装第三方商店的步骤

### 设置计划任务
1. **打开并登录1Panel面板**。
2. **点击计划任务**，选择新增任务。

![计划任务](https://cdn.sa.net/2024/05/19/Z9mJLzqjN6vMubI.png)
3. **填写任务执行周期和脚本内容**。脚本内容如下所示。

### 完整脚本代码
```bash
#!/bin/bash

# 1panel本地app的目录（如果不是默认安装，需修改该目录）
app_local_dir="/opt/1panel/resource/apps/local"

# AppStore的git仓库地址（必选）
git_repo_url="https://github.com/Akuma-real/1Panel-AppStore"

# 访问git仓库的access token，访问私有仓库时用，优先级高于账密（可选）
git_access_token=""

# 访问git仓库的用户名，访问私有仓库时用（可选）
git_username=""
# 访问git仓库的密码，访问私有仓库时用（可选）
git_password=""

# 指定克隆的分支（可选）
git_branch=""
# 指定克隆的深度（可选）
git_depth=1

# 拉取远程仓库前是否清空本地app目录（可选）
clean_local_app=true
# 拉取远程仓库前是否清空远程app缓存（可选）
clean_remote_app_cache=true

# 设置克隆或拉取远程仓库时使用的代理（可选）
proxy_url=""

# 将远程app store工程克隆到本地的工作目录
work_dir="/opt/1panel/appstore"

set -e

mkdir -p "$work_dir/logs"
log_file="$work_dir/logs/local_appstore_sync_helper_$(date +"%Y-%m-%d").log"

log() {
  local message="$1"
  mkdir -p "$(dirname "$log_file")"
  echo -e "[$(date +"%Y-%m-%d %H:%M:%S")] $message" | tee -a "$log_file"
}

url_encode() {
  local string="$1"
  local length="${#string}"
  local url_encoded_string=""
  for ((i = 0; i < length; i++)); do
    local c=${string:i:1}
    case "$c" in
      [a-zA-Z0-9.~_-]) url_encoded_string+=$c ;;
      *) url_encoded_string+=$(printf '%%%02X' "'$c") ;;
    esac
  done
  echo "$url_encoded_string"
}

replace_protocol() {
  local url="$1"
  local replacement="$2"
  if [[ -z $replacement ]]; then
    echo "${url/http:\/\//}" | sed "s/https:\/\///"
  else
    echo "${url/http:\/\//$replacement}" | sed "s/https:\/\//$replacement/"
  fi
}

clone_git_repo() {
  local url="$1"
  local username="$2"
  local password="$3"
  local access_token="$4"
  local branch="$5"
  local depth="$6"

  branch=${branch:+--branch $branch}
  depth=${depth:+--depth $depth}

  log "branch: $branch, depth: $depth"

  if [[ -n $access_token ]]; then
    log "use access_token to clone"
    local fix_url=$(replace_protocol "$url")
    git clone "https://oauth2:$access_token@$fix_url" $branch $depth
  elif [[ -n $username && -n $password ]]; then
    local encoded_username=$(url_encode "$username")
    local encoded_password=$(url_encode "$password")
    local fix_url=$(replace_protocol "$url")
    log "use username and password to clone"
    git clone "https://$encoded_username:$encoded_password@$fix_url" $branch $depth
  else
    log "use default clone"
    git clone "$url" $branch $depth
  fi
}

proxy_off() {
  unset http_proxy https_proxy ALL_PROXY no_proxy
  log "Proxy Off"
}

proxy_on() {
  local proxy_url="${1:-http://127.0.0.1:7890}"
  if [[ $proxy_url =~ :// ]]; then
    export http_proxy=$proxy_url https_proxy=$proxy_url ALL_PROXY=$proxy_url
    export no_proxy="localhost,127.0.0.1,localaddress,.localdomain.com"
    log "Proxy On $proxy_url"
  else
    log "Incorrect proxy_url, use default proxy_url"
  fi
}

script_info() {
  echo ""
  log "##################################################################"
  log "#    Name: local appstore sync helper for 1Panel                 #"
  log "# Version: v1.0.0                                                #"
  log "#  Author: xxxily                                                #"
  log "#  Github: https://github.com/xxxily/local-appstore-for-1Panel   #"
  log "##################################################################"
  echo ""
}

main() {
  script_info

  if [ ! -d "$app_local_dir" ]; then
    log "未检测到1panel的app目录，请检查1panel是否安装正确，或修改脚本中的app_local_dir变量"
    exit 1
  fi

  if [[ "$git_repo_url" != *.git ]]; then
    git_repo_url="${git_repo_url}.git"
  fi

  if [[ $git_repo_url =~ .*\/(.*)\/(.*)\.git ]]; then
    repo_username=${BASH_REMATCH[1]}
    repo_projectname=${BASH_REMATCH[2]}
  else
    log "无法提取用户名和项目名，请检查git_repo_url变量提供的地址是否正确"
    exit 1
  fi

  mkdir -p "$work_dir/temp"
  local repo_user_dir="$work_dir/temp/$repo_username"
  local repo_dir="$repo_user_dir/$repo_projectname"

  if [ "$clean_remote_app_cache" = true ] && [ -d "$repo_dir" ]; then
    rm -rf "$repo_dir"
    log "已清空远程app的缓存数据"
  fi

  if [ -n "$proxy_url" ]; then
    proxy_on "$proxy_url"
  fi

  log "准备获取远程仓库最新代码：$git_repo_url"
  if [ -d "$repo_dir" ]; then
    log "执行git pull操作"
    cd "$repo_dir"
    git pull --force 2>>"$log_file"
  else
    log "执行git clone操作"
    mkdir -p "$repo_user_dir"
    cd "$repo_user_dir"
    clone_git_repo "$git_repo_url" "$git_username" "$git_password" "$git_access_token" "$git_branch" "$git_depth" 2>>"$log_file"
  fi

  log "远程仓库最新代码获取完成"

  if [ ! -d "$repo_dir/apps" ]; then
    log "未检测到apps目录，请检查远程仓库是否正确"
    exit 1
  fi

  if [ "$clean_local_app" = true ]; then
    rm -rf "$app_local_dir"/*
    log "已清空本地原有的app"
  fi

  cd "$repo_dir"
  cp -rf apps/* "$app_local_dir"

  pwd
  ls -lah
  du -sh

  if [ "$clean_remote_app_cache" = true ]; then
    rm -rf "$repo_dir"
  fi

  if [ -n "$proxy_url" ]; then
    proxy_off
  fi

  log "1panel本地app同步成功，enjoy it!"
}

main "$@"
```
如果1Panel不是默认目录安装，需要修改以下部分的代码

![代码](https://cdn.sa.net/2024/05/19/M4ta52TKVxoXOvS.png)
### 使用方法
1. **点击应用商店**。
2. **点击本地**。
3. **点击更新应用列表**。

![使用方法](https://cdn.sa.net/2024/05/19/iO2grzBh8bf5CVl.png)
