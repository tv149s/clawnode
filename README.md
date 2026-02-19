OpenClaw Node 快速部署指南 - Ubuntu VM 实战经验
> 作者：Lei  
> 发布日期：2024-02-19  
> 适用环境：Ubuntu 22.04 LTS  
> OpenClaw 版本：2026.1.29
>
> 
***前言
最近在搭建 OpenClaw 分布式节点环境，需要在多台 Ubuntu 虚拟机上快速部署 Node 节点。经过一番摸索和踩坑，最终形成了一套稳定可靠的自动化部署方案，**成功率达到 100%**，平均每台机器部署时间约 4 分钟。
本文分享完整的部署流程和自动化脚本，供大家参考。

***环境说明
测试环境
Gateway 主机： 运行 OpenClaw Gateway 服务
Node 主机： 多台 Ubuntu 22.04 LTS 虚拟机（全新安装）
网络： 局域网环境（192.168.1.0/24）
认证方式： Token 认证（无 TLS）
版本选择
经过测试，以下版本组合最稳定：
Node.js: v24.13.1（支持顶层 await）
OpenClaw: 2026.1.29（稳定版，无强制 TLS）
⚠️ 注意： 不要使用 npm install -g openclaw@latest，新版本（2026.2.19+）强制要求 TLS，会导致局域网明文连接失败。

***架构理解（重要！）
部署前必须理解 OpenClaw 的架构：
┌─────────────────┐          WebSocket (ws://)          ┌──────────────┐
│  Gateway Host   │ ◄─────────────────────────────────  │  Node Host   │
│  (服务端)        │                                     │  (客户端)     │
│  监听 18789      │ ────────────────────────────────►   │  主动连接     │
└─────────────────┘          Token 认证                  └──────────────┘
关键点：
Node 是客户端，Gateway 是服务端
Node 通过 WebSocket 主动连接到 Gateway
--host 和 --port 参数是 Gateway 的地址，不是 Node 本地监听地址
Node 不需要开放端口，只需要能访问 Gateway

***踩坑记录
在成功之前，我遇到了以下问题：
问题 1: Node.js 版本过旧
现象：
SyntaxError: Unexpected reserved word
at Loader.moduleStrategy
原因： Node.js v22 不支持顶层 await
解决： 升级到 v24+

***问题 2: systemd 找不到 Node
现象：
systemd: Failed to locate executable
原因： systemd 不继承 shell 环境，#!/usr/bin/env node 找不到 nvm 管理的 Node
解决： 在 systemd 服务中使用绝对路径
# ❌ 错误（依赖 PATH）
ExecStart=/path/to/openclaw node run ...
# ✅ 正确（明确指定 node 路径）
ExecStart=/home/user/.nvm/versions/node/v24.13.1/bin/node \
          /home/user/.nvm/versions/node/v24.13.1/lib/node_modules/openclaw/openclaw.mjs \
          node run --host 192.168.1.100 --port 18789 --display-name my-node

***问题 3: 缺少 Gateway Token
现象：
unauthorized: gateway token missing
原因： Gateway 启用了 token 认证，但 Node 没有提供
解决： 在 systemd 服务中添加环境变量
Environment="OPENCLAW_GATEWAY_TOKEN=your-token-here"

***问题 4: 版本不兼容 - 强制 TLS
现象：
SECURITY ERROR: Cannot connect over plaintext ws://
原因： 安装了过新的版本（2026.2.19+），强制要求 TLS
解决： 指定安装稳定版本
npm install -g openclaw@2026.1.29

***最终方案
自动化部署脚本
经过多次迭代，我编写了一个完全自动化的部署脚本，特点：
✅ 自动安装所有依赖
✅ 自动配置 nvm + Node.js
✅ 自动生成 systemd 服务
✅ 无交互、无卡顿
✅ 详细的错误诊断

脚本文件： openclaw-node-installer.sh

#!/bin/bash
################################################################################
# OpenClaw Node 自动安装脚本
# 版本: 3.0
# 适用: Ubuntu 22.04 LTS
################################################################################
set -e  # 遇到错误立即退出
################################################################################
# 【需要修改】配置参数
################################################################################
GATEWAY_IP="192.168.1.100"              # 改成你的 Gateway IP
GATEWAY_PORT="18789"                     # 改成你的 Gateway 端口
GATEWAY_TOKEN="your-gateway-token-here"  # 改成你的 Gateway token
NODE_NAME="node-vm-01"                   # 改成有意义的节点名称
# 【无需修改】固定参数
OPENCLAW_VERSION="2026.1.29"  # 已验证稳定版本
NODE_VERSION="24"
################################################################################
# 颜色输出
################################################################################
G='\033[0;32m'; Y='\033[1;33m'; R='\033[0;31m'; NC='\033[0m'
info() { echo -e "${G}[✓]${NC} $1"; }
warn() { echo -e "${Y}[!]${NC} $1"; }
fail() { echo -e "${R}[✗]${NC} $1"; exit 1; }
step() { echo -e "\n${Y}▶ $1${NC}"; }
################################################################################
# 1. 安装系统依赖
################################################################################
step "安装系统依赖"
sudo apt update -qq || fail "apt update 失败"
sudo apt install -y curl wget git build-essential python3 ca-certificates >/dev/null 2>&1 || fail "依赖安装失败"
info "系统依赖安装完成"
################################################################################
# 2. 安装 NVM
################################################################################
step "安装 NVM"
if [ ! -d "$HOME/.nvm" ]; then
    curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash >/dev/null 2>&1
fi
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
info "NVM 已就绪"
################################################################################
# 3. 安装 Node.js
################################################################################
step "安装 Node.js v$NODE_VERSION"
nvm install $NODE_VERSION >/dev/null 2>&1
nvm use $NODE_VERSION >/dev/null 2>&1
nvm alias default $NODE_VERSION >/dev/null 2>&1
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm use $NODE_VERSION >/dev/null 2>&1
info "Node.js $(node --version) 安装完成"
################################################################################
# 4. 安装 OpenClaw
################################################################################
step "安装 OpenClaw $OPENCLAW_VERSION"
npm install -g openclaw@$OPENCLAW_VERSION >/dev/null 2>&1 || fail "OpenClaw 安装失败"
info "OpenClaw $(openclaw --version 2>/dev/null || echo $OPENCLAW_VERSION) 安装完成"
################################################################################
# 5. 获取安装路径
################################################################################
step "确定安装路径"
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm use $NODE_VERSION >/dev/null 2>&1
NODE_BIN=$(command -v node)
[ -z "$NODE_BIN" ] && fail "无法找到 node 可执行文件"
NODE_BASE=$(dirname $(dirname $NODE_BIN))
OPENCLAW_MJS="$NODE_BASE/lib/node_modules/openclaw/openclaw.mjs"
[ ! -f "$OPENCLAW_MJS" ] && fail "OpenClaw 主文件不存在: $OPENCLAW_MJS"
info "Node: $NODE_BIN"
info "OpenClaw: $OPENCLAW_MJS"
################################################################################
# 6. 创建 systemd 服务
################################################################################
step "创建 systemd 服务"
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/openclaw-node.service <<EOF
[Unit]
Description=OpenClaw Node Host
After=network-online.target
Wants=network-online.target
[Service]
Type=simple
Restart=always
RestartSec=5
Environment="OPENCLAW_GATEWAY_TOKEN=$GATEWAY_TOKEN"
ExecStart=$NODE_BIN $OPENCLAW_MJS node run --host $GATEWAY_IP --port $GATEWAY_PORT --display-name $NODE_NAME
[Install]
WantedBy=default.target
EOF
info "服务文件已创建"
################################################################################
# 7. 启动服务
################################################################################
step "启动服务"
systemctl --user daemon-reload
systemctl --user enable openclaw-node >/dev/null 2>&1
systemctl --user start openclaw-node
sleep 3
################################################################################
# 8. 验证
################################################################################
step "验证服务状态"
if systemctl --user is-active --quiet openclaw-node; then
    echo -e "\n${G}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
    echo -e "${G}✅ 安装成功！${NC}"
    echo -e "${G}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
    echo -e "节点: ${Y}$NODE_NAME${NC}"
    echo -e "Gateway: ${Y}$GATEWAY_IP:$GATEWAY_PORT${NC}"
    echo ""
    systemctl --user status openclaw-node --no-pager -l | head -15
    echo ""
    echo -e "${Y}下一步：在 Gateway Web 界面批准配对${NC}"
    echo -e "  访问: http://$GATEWAY_IP:$GATEWAY_PORT"
    exit 0
else
    echo -e "\n${R}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
    echo -e "${R}✗ 服务启动失败${NC}"
    echo -e "${R}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
    journalctl --user -u openclaw-node -n 20 --no-pager
    exit 1
fi

***部署步骤
第一步：获取 Gateway 配置
在 Gateway 主机上执行：
# 查看 Gateway 配置
openclaw gateway config.get
记录以下信息：
{
  "gateway": {
    "port": 18789,
    "auth": {
      "mode": "token",
      "token": "your-gateway-token-here"
    }
  }
}

你需要：
Gateway IP: Gateway 主机的 IP 地址
Gateway Port: 通常是 18789
Gateway Token: 从 gateway.auth.token 字段获取

***第二步：修改脚本配置
编辑 openclaw-node-installer.sh，修改开头的配置参数：
# 【需要修改】配置参数
GATEWAY_IP="192.168.1.100"              # 改成你的 Gateway IP
GATEWAY_PORT="18789"                     # 改成你的 Gateway 端口
GATEWAY_TOKEN="your-gateway-token-here"  # 改成你的 Gateway token
NODE_NAME="node-vm-01"                   # 改成有意义的节点名称
⚠️ 注意事项：
GATEWAY_IP 和 GATEWAY_PORT 是 Gateway 的地址（不是本机地址）
GATEWAY_TOKEN 必须与 Gateway 配置一致
NODE_NAME 建议使用有意义的名称（如 web-server-01、worker-node-02）

***第三步：传输脚本到目标机器
# 从你的工作机（能访问 Gateway 的机器）传输脚本
scp openclaw-node-installer.sh user@192.168.1.10:~/
示例：
scp openclaw-node-installer.sh ubuntu@192.168.1.10:~/

***第四步：SSH 登录并执行
# 1. 登录目标机器
ssh user@192.168.1.10
# 2. 赋予执行权限
chmod +x openclaw-node-installer.sh
# 3. 执行安装
./openclaw-node-installer.sh
预期输出：
▶ 安装系统依赖
[✓] 系统依赖安装完成
▶ 安装 NVM
[✓] NVM 已就绪
▶ 安装 Node.js v24
[✓] Node.js v24.13.1 安装完成
▶ 安装 OpenClaw 2026.1.29
[✓] OpenClaw 2026.1.29 安装完成
▶ 确定安装路径
[✓] Node: /home/ubuntu/.nvm/versions/node/v24.13.1/bin/node
[✓] OpenClaw: /home/ubuntu/.nvm/versions/node/v24.13.1/lib/node_modules/openclaw/openclaw.mjs
▶ 创建 systemd 服务
[✓] 服务文件已创建
▶ 启动服务
▶ 验证服务状态
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ 安装成功！
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
节点: node-vm-01
Gateway: 192.168.1.100:18789
● openclaw-node.service - OpenClaw Node Host
   Loaded: loaded
   Active: active (running)
下一步：在 Gateway Web 界面批准配对
  访问: http://192.168.1.100:18789
安装时间： 约 3-5 分钟
***第五步：批准配对
Node 首次连接需要在 Gateway 批准配对。
方式 1：Web 界面（推荐）
打开浏览器访问：http://GATEWAY_IP:GATEWAY_PORT
登录管理界面
找到待批准的节点（Pending Nodes）
点击 "Approve" 批准
方式 2：命令行
在 Gateway 主机上：
# 查看待批准节点
openclaw nodes pending
# 批准节点
openclaw nodes approve <node-id>
***第六步：验证连接
在 Gateway 主机上：
openclaw nodes status
应该看到新节点，状态为 connected: true：
{
  "displayName": "node-vm-01",
  "platform": "linux",
  "version": "2026.1.29",
  "remoteIp": "192.168.1.10",
  "paired": true,
  "connected": true
}

***批量部署
当需要部署多台机器时，可以使用以下方法：
方法一：为每台机器生成定制脚本
#!/bin/bash
# generate-installers.sh
GATEWAY_IP="192.168.1.100"
GATEWAY_PORT="18789"
GATEWAY_TOKEN="your-gateway-token-here"
# 节点列表（格式：用户@IP 节点名称）
cat > nodes.txt <<EOF
ubuntu@192.168.1.10 web-server-01
ubuntu@192.168.1.11 web-server-02
ubuntu@192.168.1.12 worker-node-01
ubuntu@192.168.1.13 worker-node-02
EOF
# 为每个节点生成定制脚本
while read line; do
    [[ "$line" =~ ^#.*$ ]] && continue
    [[ -z "$line" ]] && continue
    
    SSH_TARGET=$(echo $line | awk '{print $1}')
    NODE_NAME=$(echo $line | awk '{print $2}')
    IP=$(echo $SSH_TARGET | cut -d'@' -f2)
    
    SCRIPT_NAME="openclaw-node-installer-${IP}.sh"
    cp openclaw-node-installer.sh "$SCRIPT_NAME"
    
    # 替换配置
    sed -i "s/NODE_NAME=\"node-vm-01\"/NODE_NAME=\"$NODE_NAME\"/" "$SCRIPT_NAME"
    
    echo "✓ 已生成: $SCRIPT_NAME"
done < nodes.txt

方法二：批量传输和执行
#!/bin/bash
# deploy-all.sh
while read line; do
    [[ "$line" =~ ^#.*$ ]] && continue
    [[ -z "$line" ]] && continue
    
    SSH_TARGET=$(echo $line | awk '{print $1}')
    NODE_NAME=$(echo $line | awk '{print $2}')
    IP=$(echo $SSH_TARGET | cut -d'@' -f2)
    
    SCRIPT_NAME="openclaw-node-installer-${IP}.sh"
    
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "部署节点: $NODE_NAME ($SSH_TARGET)"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    
    # 传输脚本
    scp "$SCRIPT_NAME" "$SSH_TARGET:~/openclaw-node-installer.sh"
    
    # 执行安装
    ssh "$SSH_TARGET" "chmod +x openclaw-node-installer.sh && ./openclaw-node-installer.sh"
    
    echo "✓ $NODE_NAME 部署完成"
    echo ""
    sleep 5
done < nodes.txt
echo "所有节点部署完成！请在 Web 界面批准配对"
***服务管理
常用命令
# 查看服务状态
systemctl --user status openclaw-node
# 查看实时日志
journalctl --user -u openclaw-node -f
# 重启服务
systemctl --user restart openclaw-node
# 停止服务
systemctl --user stop openclaw-node
# 启动服务
systemctl --user start openclaw-node
# 禁用开机自启
systemctl --user disable openclaw-node
# 启用开机自启
systemctl --user enable openclaw-node

***故障排查

问题 1: 服务启动失败
查看详细日志：
journalctl --user -u openclaw-node -n 50
常见错误和解决方案：
| 错误信息 | 原因 | 解决方案 |
|---------|------|---------|
| Failed to locate executable | 路径错误 | 检查服务文件中的路径是否正确 |
| unauthorized: gateway token missing | Token 错误 | 检查服务文件中的 OPENCLAW_GATEWAY_TOKEN |
| SECURITY ERROR: Cannot connect over plaintext | 版本过新 | 卸载后安装 2026.1.29 版本 |
| ECONNREFUSED | Gateway 不可达 | 检查网络和 Gateway 状态 |

***问题 2: 无法连接到 Gateway
排查步骤：
# 1. 测试网络连通性
ping 192.168.1.100
# 2. 测试端口是否开放
nc -zv 192.168.1.100 18789
# 3. 测试 HTTP 访问
curl -v http://192.168.1.100:18789
# 4. 在 Gateway 主机上检查服务
openclaw status
***问题 3: 手动测试连接
如果 systemd 服务有问题，可以手动测试：
# 1. 加载 nvm 环境
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm use 24
# 2. 手动启动（前台运行）
OPENCLAW_GATEWAY_TOKEN="your-token-here" \
openclaw node run \
  --host 192.168.1.100 \
  --port 18789 \
  --display-name test
# 观察输出，按 Ctrl+C 停止
如果手动能成功连接，说明问题在 systemd 配置。
***实战数据
我使用这套方案部署了 5 个节点，统计数据：
| 节点名称 | 系统 | 部署时间 | 一次成功 |
|---------|------|---------|---------|
| node-vm-01 | Ubuntu 22.04 | 4 分钟 | ✅ |
| node-vm-02 | Ubuntu 22.04 | 4 分钟 | ✅ |
| node-vm-03 | Ubuntu 22.04 | 3 分钟 | ✅ |
| node-vm-04 | Ubuntu 22.04 | 4 分钟 | ✅ |
| node-vm-05 | Ubuntu 22.04 | 4 分钟 | ✅ |
成功率： 100%  
平均部署时间： 3.8 分钟

***技术要点总结
1. 版本控制
✅ 推荐：
Node.js: v24.13.1
OpenClaw: 2026.1.29
❌ 避免：
使用 npm install -g openclaw@latest
Node.js v22 或更低版本
2. systemd 路径
✅ 正确： 使用绝对路径
ExecStart=/home/user/.nvm/versions/node/v24.13.1/bin/node /home/user/.nvm/versions/node/v24.13.1/lib/node_modules/openclaw/openclaw.mjs node run ...
❌ 错误： 依赖环境变量
ExecStart=openclaw node run ...
3. 认证配置
✅ 正确： 在服务文件中配置
Environment="OPENCLAW_GATEWAY_TOKEN=your-token-here"
❌ 错误： 依赖 shell 环境变量
4. 参数理解
✅ 正确： --host 和 --port 是 Gateway 的地址
openclaw node run --host 192.168.1.100 --port 18789
❌ 错误： 以为是本地监听地址

***结语
通过这套自动化方案，OpenClaw Node 的部署变得非常简单，只需要：
修改脚本配置（3 个参数）
传输脚本到目标机器
执行脚本
Web 界面批准配对
整个过程约 5 分钟，无需手动配置任何环境。
希望这篇分享对大家有帮助。如果有问题或改进建议，欢迎交流！
*
**附录：完整文件清单
1. openclaw-node-installer.sh - 主安装脚本（上文已提供）
2. generate-installers.sh - 批量生成脚本
#!/bin/bash
# 为多台机器生成定制脚本
GATEWAY_IP="192.168.1.100"              # 修改为你的 Gateway IP
GATEWAY_PORT="18789"                     # 修改为你的 Gateway 端口
GATEWAY_TOKEN="your-gateway-token-here"  # 修改为你的 Gateway token
# 创建节点列表（格式：用户@IP 节点名称）
cat > nodes.txt <<EOF
ubuntu@192.168.1.10 web-server-01
ubuntu@192.168.1.11 web-server-02
ubuntu@192.168.1.12 worker-node-01
ubuntu@192.168.1.13 worker-node-02
EOF
# 为每个节点生成定制脚本
while read line; do
    [[ "$line" =~ ^#.*$ ]] && continue
    [[ -z "$line" ]] && continue
    
    SSH_TARGET=$(echo $line | awk '{print $1}')
    NODE_NAME=$(echo $line | awk '{print $2}')
    IP=$(echo $SSH_TARGET | cut -d'@' -f2)
    
    SCRIPT_NAME="openclaw-node-installer-${IP}.sh"
    cp openclaw-node-installer.sh "$SCRIPT_NAME"
    
    # 替换 NODE_NAME
    sed -i "s/NODE_NAME=\"node-vm-01\"/NODE_NAME=\"$NODE_NAME\"/" "$SCRIPT_NAME"
    
    echo "✓ 已生成: $SCRIPT_NAME"
done < nodes.txt
echo "完成！生成了 $(ls openclaw-node-installer-*.sh 2>/dev/null | wc -l) 个脚本"
3. deploy-all.sh - 批量部署脚本
#!/bin/bash
# 批量部署所有节点
while read line; do
    [[ "$line" =~ ^#.*$ ]] && continue
    [[ -z "$line" ]] && continue
    
    SSH_TARGET=$(echo $line | awk '{print $1}')
    NODE_NAME=$(echo $line | awk '{print $2}')
    IP=$(echo $SSH_TARGET | cut -d'@' -f2)
    
    SCRIPT_NAME="openclaw-node-installer-${IP}.sh"
    
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "部署节点: $NODE_NAME ($SSH_TARGET)"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    
    # 传输脚本
    echo "  1. 传输脚本..."
    scp "$SCRIPT_NAME" "$SSH_TARGET:~/openclaw-node-installer.sh" || {
        echo "  ✗ 传输失败，跳过"
        continue
    }
    
    # 执行安装
    echo "  2. 执行安装..."
    ssh "$SSH_TARGET" "chmod +x openclaw-node-installer.sh && ./openclaw-node-installer.sh" || {
        echo "  ✗ 安装失败"
        continue
    }
    
    echo "  ✓ $NODE_NAME 部署完成"
    echo ""
    
    sleep 5
done < nodes.txt
echo "所有节点部署完成！"
echo "下一步：在 Gateway Web 界面批准配对"
echo "  访问: http://192.168.1.100:18789"
4. nodes.txt - 节点清单模板
# 格式：用户@IP 节点名称
# 示例：
ubuntu@192.168.1.10 web-server-01
ubuntu@192.168.1.11 web-server-02
ubuntu@192.168.1.12 worker-node-01
ubuntu@192.168.1.13 worker-node-02
