---
title: Qiyan-Tailscale团队内部手册
categories:
  - - 网络
tags:
date: 2025/12/24
updated: 2025/12/24
comments:
published:
---
负责人：邢成功
手册版本：v0.1.0
日期：2025/12/24

# Tailscale 和 Headscale 简介
Tailscale 是基于 WireGuard 的虚拟组网工具。提供免费、收费的托管控制服务。目前客户端开源，控制端暂未开源。

Headscale 是 Tailescale 控制服务器的开源、自托管实现。

我们turing001上有公网，可以基于 Headscale 自托管控制端。

Tailscale 官方安装：[https://tailscale.com/download](https://tailscale.com/download)
# 团队接入说明：Headscale + Tailscale（HTTPS + 自签证书）

> 我这边已经把 Headscale 控制面跑起来了：**HTTPS + 自签证书**。下面是给大家的“从下载到登录成功”的操作手册，以及常用测试/排障命令。  
> **Headscale 控制面端口：8112**；**Remote CLI（gRPC）端口：50443**。

---

## 1）服务端信息（大家用得到的）

- **Control Server（客户端登录用）**：`https://qiyan.fund:8112`
- **Remote CLI（仅管理员/运维用）**：`qiyan.fund:50443`（gRPC over TLS）
- **MagicDNS 域**：`*.tailnet.qiyan.fund`（例如 `turing001.tailnet.qiyan.fund`）

---

## 2）需要管理员/网管开放的端口（重点）

### 必需（否则客户端无法登录/续租）

- **TCP 8112** → `qiyan.fund`（Headscale 控制面 HTTPS）

### 仅管理员需要（远程管理 Headscale）

- **TCP 50443** → `qiyan.fund`（Headscale Remote CLI / gRPC） 
    
    > 如果安全要求更高：这个端口建议只对白名单（办公网/VPN/Tailscale 管理节点）开放。
    

### 建议（提升点对点直连概率/降低 DERP 延迟）

- **UDP 41641（或 tailscaled 实际监听端口）**：建议服务器侧放行入站/出站 UDP（可选，但体验差距很明显：直连会从 100~300ms 变成几毫秒）

---

## 3）为什么要装证书（自签证书是“必须信任”）

我们用的是 **自签 TLS 证书**，所以：

- 客户端如果**不信任证书**，Tailscale 往往会表现为：**超时、context canceled、bad certificate** 等。
    
- Remote CLI 同样依赖 TLS，官方也明确 Remote CLI 需要 TLS，并建议把自签证书加入系统信任链（不建议长期用 insecure）
    

**我会给大家提供：**

1. 证书文件（例如 `qiyan.fund.crt`）
    
2. 各自的 **PreAuthKey（一次性/限时）**：大家用它把设备注册进来即可（用完就失效，设备会保留长期节点密钥）。
    

---

## 4）通用接入流程（所有平台都一样）

1. 安装 Tailscale 客户端
    
2. 把我发的 `qiyan.fund.crt` 导入 **系统信任**（下面分平台写）
    
3. 用我发的 **PreAuthKey** 登录到：`https://qiyan.fund:8112`
    
4. 验证上线（见第 8 节测试命令）
    

> 注意：**登录地址务必用域名 `qiyan.fund`**。如果用 IP（例如 `https://103.91.178.50:8112`），证书校验会因为“域名不匹配”失败。

---

## 5）macOS（命令行最稳）

### A. 安装

- 官网安装包或 Homebrew 均可（大家怎么顺手怎么来）
    

### B. 导入证书（推荐导入到 System Keychain）

1. 打开 **Keychain Access（钥匙串访问）**
    
2. 选择 **System** 钥匙串 → 导入 `qiyan.fund.crt`
    
3. 双击证书 → Trust → **When using this certificate: Always Trust**
    

### C. 登录（建议最小命令）

```bash
sudo tailscale up --reset \
  --login-server=https://qiyan.fund:8112 \
  --auth-key=<我发给大家的KEY> \
  --hostname=<给自己这台机器起个名字，必须全小写，比如cxingmacbook>
```

---

## 6）Windows（证书必须导入“受信任的根证书颁发机构”）

### A. 安装

- 安装官方 Tailscale 客户端（GUI + CLI 都会有）
    

### B. 导入证书（关键）

1. 双击 `qiyan.fund.crt` → **安装证书**
    
2. 选择 **本地计算机（Local Machine）**（需要管理员权限）
    
3. 存储位置选择：**受信任的根证书颁发机构（Trusted Root Certification Authorities）**
    
4. 完成后重新打开命令行再试
    

### C. 登录（管理员 CMD / PowerShell）

```powershell
tailscale up --reset `
  --login-server=https://qiyan.fund:8112 `
  --auth-key=<我发的KEY> `
  --hostname=<机器名>
```

> 如果 `curl https://127.0.0.1:8112` 报 SNI 不支持，这是因为用了 IP；**Windows 上建议始终用域名 `https://qiyan.fund:8112`**。

---

## 7）Android / iOS / 鸿蒙

### Android（官方 App 支持“Alternate Server”）

Headscale 官方文档给了两种方式：交互登录或 Auth Key 登录。我们走 **Auth Key** 最省事： 

**操作路径：**

1. 安装 Tailscale（Play Store / F-Droid）
    
2. 打开 App → 右上角设置 → Accounts → 右上角三点菜单
    
3. 选择 **Use an alternate server**，填：`https://qiyan.fund:8112` 
    
4. 然后在三点菜单里选 **Use an auth key**，粘贴我发的 KEY 
    
5. 成功后会自动连上
    

**Android 证书安装：**

- 设置 → 安全 → 加密与凭据/证书 → 安装证书 → **CA 证书**（不同厂商入口略有差异）
    
- 装完再回 Tailscale 登录
    

---

### iOS / iPadOS（App 内切换自建控制面）

iOS 也支持使用自建控制面（Custom Coordination/Alternate server），按官方说明进入对应菜单填入控制面 URL。

**大致步骤：**

1. 安装 Tailscale（App Store）
    
2. 在 App 的账号/设置里找到 **Custom Coordination Server / Alternate server**
    
3. 填：`https://qiyan.fund:8112`
    
4. 使用我发的 KEY 完成注册（若界面没有明显“auth key”，通常在账户菜单的更多选项里）
    

**iOS 证书安装：**

1. 把 `qiyan.fund.crt` 发到手机（AirDrop/微信文件/邮件均可）并安装描述文件
    
2. 设置 → 通用 → 关于本机 → **证书信任设置** → 打开对该证书的完全信任
    

---

### 鸿蒙（HarmonyOS）

这里要分版本：

- **HarmonyOS 4.x（仍兼容 Android APK）**：通常可以直接装 **Android 版 Tailscale**，步骤按上面的 Android 走。 
    
- **HarmonyOS NEXT / 5.x+（不再支持 Android APK）**：系统层面已移除 Android 兼容，**就无法直接装 Android 版 Tailscale**。 
    
    > 如果大家是 NEXT/5.x，我建议用电脑（Windows/macOS/Linux）先接入；手机端我再单独看看有没有可行替代方案。
    

---

## 8）常用测试命令（大家自查 + 我排障都用得到）

### 客户端（macOS / Linux / Windows 都通用）

```bash
tailscale status
tailscale ip -4
tailscale ping --verbose turing001.tailnet.qiyan.fund
```

- `tailscale ping --verbose` 里如果看到 `via DERP(...)` 说明走中继；出现 `via <内网IP:端口>` 基本就是直连（延迟会很低）。
    
- **系统自带 `ping` 不通**不代表 Tailscale 不通：可能是 OS 防火墙/ICMP 被禁；优先看 `tailscale ping`。
    

### 服务端（turing001 上）

```bash
# 看 headscale 容器是否健康
podman ps
podman logs --tail=200 headscale

# 看端口监听
ss -lntp | egrep ':8112|:50443'

# 看 tailscaled 的直连情况
tailscale status
tailscale ping --verbose <某客户端名>
```

---

## 9）常见问题速查（最容易踩的坑）

1. **tailscale up 超时 / context canceled**
    

- 80% 是：证书未信任 / 代理干扰 / 端口 8112 未通
    
- 先做：`nc -vz qiyan.fund 8112` 或者 `curl -v https://qiyan.fund:8112`（能连通再谈后面）
    

2. **服务端日志出现 `tls: bad certificate`**
    

- 基本就是客户端不信任自签证书，或用 IP 访问导致域名不匹配
    
- 解决：确认客户端用的是 `https://qiyan.fund:8112`，并把 `qiyan.fund.crt` 加到系统信任
    

3. **直连延迟高（一直 DERP）**
    

- 先用：`tailscale ping --verbose <peer>` 看是否能变成直连
    
- 直连长期失败再考虑：服务器放行 UDP、路由/NAT、以及是否被公司网络策略限制
    

---

## 10）我这边的协作方式（大家怎么找我）

- 大家把 **设备名（hostname）** 发我（或我指定命名规则），我给对应人员发 **PreAuthKey**
    
- 新设备加进来后，我会在 Headscale / Web UI 里做必要的标签/权限归类（比如服务器节点的 `tag:server`）
    

如果大家把上面流程走完还卡住：把下面三样发我就能快速定位：

1. `tailscale status` 输出（打码私密信息即可）
    
2. `tailscale ping --verbose turing001.tailnet.qiyan.fund` 输出
    
3. `nc -vz qiyan.fund 8112`或 `curl -v https://qiyan.fund:8112`