### 谷歌云VPS快速搭建（邮箱通知预警）
1. 一键自动创建并重置密码命令

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

# ========================================================
# 配置区
# ========================================================
LIMIT=170                      # 本月限额 (GB)
INTERFACE="ens4"               # 网卡名称
EMAIL="zhuchengshuo0709@gmail.com" # 接收通知的邮箱

# 1. 动态获取当前服务器公网 IP (不使用固定 IP)
SERVER_IP=$(curl -s http://checkip.amazonaws.com || curl -s ifconfig.me || echo "未知IP")

# 2. 提取流量数据 (单位：字节 Bytes)
# 使用 vnstat --oneline 模式确保数据提取不因流量大小而报错
STATS=$(vnstat -i $INTERFACE --oneline b)

# 提取本月累计流量 (第 11 列) 和 自安装起总流量 (第 15 列)
MONTH_BYTES=$(echo "$STATS" | cut -d';' -f11)
TOTAL_BYTES=$(echo "$STATS" | cut -d';' -f15)

# 3. 换算为 GB (整数运算)
# 1GB = 1073741824 字节
MONTH_GB=$((MONTH_BYTES / 1073741824))
TOTAL_GB=$((TOTAL_BYTES / 1073741824))

# ========================================================
# 输出结果 (用于日志记录)
# ========================================================
echo "------------------------------------------"
echo "检测时间: $(date '+%Y-%m-%d %H:%M:%S')"
echo "服务器公网 IP: $SERVER_IP"
echo "本月已使用流量: ${MONTH_GB} GB"
echo "自安装起总流量: ${TOTAL_GB} GB"
echo "设定关机阈值: ${LIMIT} GB"
echo "------------------------------------------"

# ========================================================
# 逻辑判断与执行
# ========================================================
if [ "$MONTH_GB" -ge "$LIMIT" ]; then
    echo "警告：本月流量已达 ${MONTH_GB} GB，执行关机保护！"
    
    # 4. 发送邮件通知 (必须配置好 ssmtp 才能成功发送)
    # 使用 Base64 编码防止中文标题乱码
    SUBJECT_ENCODED=$(echo -n "【流量预警】服务器 $SERVER_IP 自动关机" | base64)
    MESSAGE="您的服务器 ($SERVER_IP) 本月已使用 ${MONTH_GB} GB，总累计使用 ${TOTAL_GB} GB，超过阈值 ${LIMIT} GB，系统已执行关机保护以防止额外扣费。"
    
    echo "$MESSAGE" | mail -s "=?UTF-8?B?${SUBJECT_ENCODED}?=" -a "From: $EMAIL" $EMAIL 2>/dev/null
    
    sleep 10
    # 5. 执行关机
    /sbin/shutdown -h now
else
    echo "状态：流量在安全范围内。"
fi
```

```bash
chmod +x traffic_limit.sh
```

邮箱配置

### 第一步：获取 Gmail “应用专用密码”

由于安全限制，你不能在脚本里直接填 Gmail 的登录密码。

![image.png](attachment:3e3f8bfd-9d88-4240-b31d-cf0edeed2e35:image.png)

1. 登录你的 [Google 账号管理页面](https://myaccount.google.com/)。
2. 确保已开启 **“两步验证” (2-Step Verification)**。
3. 在搜索框输入 **“应用专用密码” (App Passwords)**。
4. 创建一个名称（如 `GCP-Monitor`），系统会生成一个 **16 位** 的随机代码。**请务必记下它**，这是配置脚本的关键。

---

### 第二步：安装必要的轻量组件

在服务器终端运行，确保系统具备发信能力：

Bash

```bash
sudo apt-get update && sudo apt-get install -y ssmtp mailutils
```

---

### 第三步：配置 ssmtp 发信通道

编辑配置文件，让服务器学会使用 Gmail 的通道。

1. 执行：

```bash
sudo nano /etc/ssmtp/ssmtp.conf
```

1. 删除原有内容，直接粘贴以下配置：

代码段

```bash
root=zhuchengshuo0709@gmail.com
mailhub=smtp.gmail.com:587
hostname=35.220.219.7
FromLineOverride=YES
AuthUser=zhuchengshuo0709@gmail.com
AuthPass=你的16位应用专用密码
UseSTARTTLS=YES
```

*注：`AuthPass` 处填入第一步生成的 16 位代码，**中间不要带空格。***

---

### 第四步：发送测试邮件

使用你刚才创建的测试脚本或直接运行这一行：

```bash
echo "测试内容：流量监控系统配置成功" | mail -s "测试邮件" zhuchengshuo0709@gmail.com
```

## 设置定时任务（必须执行）

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
