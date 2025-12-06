结论先行  
1. 只要让 containerd（或 Docker）把流量指向你那台开了 Clash-Verge 的“本地代理”，**就能**顺利拉到 gcr.io/docker.io 等被墙仓库的镜像；  
2. 整个流程需要“**给节点级运行时配置代理**”＋“**放行 Clash 的局域网 SOCKS/HTTP 端口**”两步，缺一则失败；  
3. 下面给出一份可直接落地的「验证手册」，按顺序执行即可确认是否生效。

------------------------------------------------
Kubernetes 节点拉镜像走 Clash-Verge 代理 · 验证手册
------------------------------------------------
0. 前置确认  
   a. 运行 Clash-Verge 的电脑（下文称“代理机”）能科学上网，且已打开  
      Settings → System Proxy → Allow LAN（允许局域网连接）。  
   b. 记住代理机局域网 IP 和 Clash 端口，例如：  
      HTTP 192.168.1.100:7890  
      SOCKS5 192.168.1.100:7891  

1. 给 **每个** K8s 节点配置运行时代理  
   1.1 containerd 方案（官方推荐，1.20+ 默认）  
   ```bash
   # 在所有节点执行
   sudo mkdir -p /etc/systemd/system/containerd.service.d
   sudo tee /etc/systemd/system/containerd.service.d/proxy.conf <<EOF
   [Service]
   Environment="HTTP_PROXY=http://192.168.1.100:7890"
   Environment="HTTPS_PROXY=http://192.168.1.100:7890"
   Environment="NO_PROXY=127.0.0.1,localhost,10.0.0.0/8,192.168.0.0/16"
   EOF
   sudo systemctl daemon-reload
   sudo systemctl restart containerd
   # 确认生效
   sudo systemctl show --property=Environment containerd
   ```
   1.2 Docker 方案（若节点仍用 Docker 作为运行时）  
   ```bash
   sudo mkdir -p /etc/systemd/system/docker.service.d
   sudo tee /etc/systemd/system/docker.service.d/proxy.conf <<EOF
   [Service]
   Environment="HTTP_PROXY=http://192.168.1.100:7890"
   Environment="HTTPS_PROXY=http://192.168.1.100:7890"
   Environment="NO_PROXY=127.0.0.1,localhost,10.0.0.0/8,192.168.0.0/16"
   EOF
   sudo systemctl daemon-reload && sudo systemctl restart docker
   ```

2. 在代理机上临时放行 7890/7891 端口（Windows 防火墙或 ufw）  
   PowerShell(管理员) 示例：  
   ```powershell
   New-NetFirewallRule -DisplayName "Clash-HTTP" -Direction Inbound -Protocol TCP -LocalPort 7890 -Action Allow
   New-NetFirewallRule -DisplayName "Clash-SOCKS" -Direction Inbound -Protocol TCP -LocalPort 7891 -Action Allow
   ```

3. 快速拉取验证（任选其一）  
   - 节点上用 crictl（containerd）  
     ```bash
     sudo crictl pull k8s.gcr.io/pause:3.9
     sudo crictl images | grep pause
     ```
   - 节点上用 docker（docker 运行时）  
     ```bash
     docker pull k8s.gcr.io/pause:3.9
     docker images pause:3.9
     ```
   **预期结果**：进度条走完，镜像出现在列表中，无 `Error response from daemon: Get ... i/o timeout` 报错。  
   若成功，说明代理已生效，墙不再影响镜像下载 。

4. 正式让 kubelet 拉镜像  
   编辑 Deployment/DaemonSet，把镜像写成原始地址即可，例如：  
   ```yaml
   image: k8s.gcr.io/pause:3.9
   ```
   之后  
   ```bash
   kubectl apply -f xxx.yaml
   kubectl describe pod <名> | grep -i pull
   ```
   若 Events 里出现 `Successfully pulled image "k8s.gcr.io/pause:3.9"` 即完全 OK。

5. 常见坑排查清单  
   - `systemctl show containerd` 里看不到代理变量 → 配置文件写错或没 daemon-reload。  
   - 节点能 curl 外网，但 crictl 仍失败 → 只配置了 Docker 代理，而节点运行时实际是 containerd（或相反）。  
   - 拉一半卡住 → Clash 规则把“境外 IP”漏掉，确认规则里放行了 gcr.io、docker.io、registry.k8s.io 等域名。  

按以上步骤执行完，即可确认“本地 Clash-Verge 代理”能否真正帮 K8s 绕墙拉镜像。祝一次成功！
