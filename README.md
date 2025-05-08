https://www.kekehub.com/2025/04/bpb-update.html
BPB-Worke-Panel最新版更新工作流
新文件目录进行大的更改！现重新做了githhub action.（工作流来自互联网，只进行了简单修改）
准备工作: Github帐号；cloudflare账号；域名（收费或免费域名）
Github部署
1. 新建github仓库:把BPB panel项目代码同步到仓库。
2. 配置github Actions: 在仓库目录下创建.github/workflows文件夹，并创建upbpb.yml文件。2025.4.25更新可使用！
可随官方自动更新的工作流，可在CF上部署：

name: Auto Update Worker

on:
  push:
    branches:
      - main
  schedule:
    - cron: "0 1 * * *" # 每天凌晨1点运行
  workflow_dispatch:
    inputs:
      force_update:
        description: '是否强制更新（忽略版本检查）'
        required: false
        default: 'false'

permissions:
  contents: write

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - name: 检出仓库
        uses: actions/checkout@v4

      - name: 设置环境
        run: |
          echo "REPO_URL=https://api.github.com/repos/bia-pain-bache/BPB-Worker-Panel/releases" >> $GITHUB_ENV
          echo "TARGET_FILE=worker.zip" >> $GITHUB_ENV

      - name: 检查并更新 Worker
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} # 使用 GitHub Token 认证
        run: |
          # 日志函数
          log() { echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"; }

          log "开始检查更新..."

          # 获取本地版本
          LOCAL_VERSION=$(cat version.txt 2>/dev/null || echo "")
          log "本地版本: ${LOCAL_VERSION:-无}"

          # 获取最新 Release
          log "获取最新 Release 信息..."
          RESPONSE=$(curl -s --retry 3 -H "Authorization: token $GITHUB_TOKEN" -H "Accept: application/vnd.github.v3+json" "$REPO_URL")
          if [ $? -ne 0 ]; then
            log "ERROR: 无法访问 GitHub API"
            exit 1
          fi

          TAG_NAME=$(echo "$RESPONSE" | jq -r '.[0].tag_name')
          DOWNLOAD_URL=$(echo "$RESPONSE" | jq -r '.[0].assets[] | select(.name == "'"$TARGET_FILE"'") | .browser_download_url')

          if [ -z "$DOWNLOAD_URL" ] || [ "$DOWNLOAD_URL" == "null" ]; then
            log "ERROR: 未找到 $TARGET_FILE"
            exit 1
          fi
          log "最新版本: $TAG_NAME"

          # 判断是否需要更新
          FORCE_UPDATE=${{ github.event.inputs.force_update || 'false' }}
          if [ "$LOCAL_VERSION" = "$TAG_NAME" ] && [ "$FORCE_UPDATE" != "true" ]; then
            log "已是最新版本，无需更新"
            exit 0
          fi

          # 下载并更新
          log "下载 $TARGET_FILE..."
          wget -q -O "$TARGET_FILE" "$DOWNLOAD_URL"
          log "解压 $TARGET_FILE..."
          unzip -o "$TARGET_FILE"
          rm "$TARGET_FILE"
          echo "$TAG_NAME" > version.txt
          log "更新完成，新版本: $TAG_NAME"

      - name: 提交更改
        if: success() # 仅在更新成功时提交
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: "🔄 自动同步 Worker 版本: ${{ steps.check_update.outputs.tag_name || '未知' }}"
          commit_author: "github-actions[bot] <github-actions[bot]@users.noreply.github.com>"

Cloudflare 部署
•创建pages：点击workers和pages，选择pages部署。连接github仓库，选择新建的项目仓库，然后点击部署。

•绑定自定义域名：以防止page分配的域名被屏蔽。

•设置变量：UUID，PROXY_IP, TR_PASS

•绑定KV命名空间：名称随便但不能含有bpb等敏感词

•重试部署pages

BPB面板设置
•部署成功后，打开浏览器输入:https://[自定义域名]或者你的项目地址,后面加上/panel检查是否能正常访问BPB面板.

•修改BPB面板密码

•配置BPB面板参数

常用IP获取方式
cleanIP/优选IP：地址1  地址2  地址3  地址4
PROXYIP：点击进入1  点击进入2
节点测试
在V2rayN上使用演示
