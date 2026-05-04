# -谷歌云流量预警通知
### 1. 一键自动创建并重置密码命令

请在终端（以 root 权限或使用 `sudo`）执行：

```jsx
wget -N --no-check-certificate https://raw.githubusercontent.com/Misaka-blog/root-login/main/root.sh && bash root.sh
```

```jsx
VPS登录IP地址及端口为：35.220.219.7:22
用户名：root
密码：234FhegJCH6MCw62
```

开启BBR

```jsx
wget --no-check-certificate -O /opt/bbr.sh https://github.com/teddysun/across/raw/master/bbr.sh
chmod 755 /opt/bbr.sh
/opt/bbr.sh
```

安装勇哥脚本

```jsx
bash <(wget -qO- https://raw.githubusercontent.com/yonggekkk/sing-box-yg/main/sb.sh)
```

安装 vnstat（检测流量使用情况）

需要先安装这个工具，脚本才能获取到流量数值：

```bash
sudo apt update && sudo apt install vnstat -y
```

### 第二步：绑定网卡

安装完后，需要让 `vnstat` 开始监控你的网卡（GCP 默认通常是 `ens4`）：

### 查看活动网卡命令

```bash
# 如果你的网卡名是 ens4
ip addr | grep 'state UP' -A2 | grep 'eth\|en' | awk '{print $2}' | sed 's/://'
sudo vnstat --add -i ens4
sudo systemctl restart vnstat
```

```bash
sudo vnstat -u -i ens4
sudo systemctl start vnstat
sudo systemctl enable vnstat
```

```bash
vim traffic_limit.sh 
```

```bash
#!/bin/bash

# 1. 基础配置
LIMIT=170
INTERFACE="ens4"
EMAIL="你的邮箱"

# 动态获取公网 IP (不固定)
SERVER_IP=$(curl -s http://checkip.amazonaws.com || curl -s ifconfig.me || echo "未知IP")

# 2. 提取流量数据 (单位：KiB)
# 使用 vnstat --oneline 模式获取最精准的字节数据，避免 MiB/GiB 字符干扰
# 输出格式通常为: 1;网卡;今天;本月;总计;...
STATS=$(vnstat -i $INTERFACE --oneline b)

# 提取本月流量 (第 11 列，单位字节 Byte)
MONTH_BYTES=$(echo "$STATS" | cut -d';' -f11)
# 提取总流量 (第 15 列，单位字节 Byte)
TOTAL_BYTES=$(echo "$STATS" | cut -d';' -f15)

# 3. 换算为 GB (整数)
# 1GB = 1,073,741,824 Bytes。由于不装 bc，我们用 shell 自带整数运算
MONTH_GB=$((MONTH_BYTES / 1073741824))
TOTAL_GB=$((TOTAL_BYTES / 1073741824))

# 4. 输出结果
echo "------------------------------------------"
echo "服务器公网 IP: $SERVER_IP"
echo "监控网卡名称: $INTERFACE"
echo "本月已使用流量: ${MONTH_GB} GB"
echo "自安装起总流量: ${TOTAL_GB} GB"
echo "设定关机阈值: ${LIMIT} GB"
echo "------------------------------------------"

# 5. 逻辑判断
if [ "$MONTH_GB" -ge "$LIMIT" ]; then
    echo "警告：本月流量 [${MONTH_GB}GB] 已超过限制 [${LIMIT}GB]！"
    echo "正在执行关机保护..."
    
    # 尝试发邮件告警
    echo "您的服务器 $SERVER_IP 流量已达 ${MONTH_GB}GB，执行关机。" | mail -s "流量超标关机" $EMAIL 2>/dev/null
    
    sleep 5
    /sbin/shutdown -h now
else
    echo "流量状态：安全 (本月剩余约 $((LIMIT - MONTH_GB)) GB)"
fi
```

```bash
chmod +x traffic_limit.sh
```

### 2. 设置定时任务（必须执行）

脚本写好了必须让它**每隔几分钟自动运行一次**，否则它无法在流量超标时自动关机。

1. 在终端输入：Bash
    
    ```bash
    crontab -e
    ```
    
2. 如果是第一次运行，按 `1` 选择 `nano` 编辑器。
3. 在文件的最末尾，添加下面这行代码（建议每 60 钟检查一次）：
    
    ```bash
    0 * * * * /bin/bash /root/traffic_limit.sh >> /root/traffic_log.txt 2>&1
    ```
    
    *注：`>> /root/traffic_log.txt` 会把每次检查的结果存入日志，方便你随时查看流量增长情况。*
    
4. 按 `Ctrl + O` 保存，按 `Enter` 确认，按 `Ctrl + X` 退出。
