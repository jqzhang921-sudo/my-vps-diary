# AWS Lightsail 自建节点经验总结

> 2026-11-07 晚 — 2026-11-08 下午
> 作者：Cleo & 噗噗
> 一句话：从 Oracle 屡次被拒，到自己搭起一个能用的悉尼节点，中间踩的坑比路还多。

## 背景

- Oracle Cloud 免费档一直被风控拒，同一套信息被标记了，冷静一周、不再硬试。
- 工单 261102-002095 还在排队，不要重复提交、不要联系其他团队。
- 备用方案：AWS Lightsail，$100 额度、182 天（到 2027-05-07）。
- 目标：一个干净 IP 的节点，解决 IP 太脏、Antigravity 老断的问题。

## 成果

- 实例：Lightsail，悉尼 ap-southeast-2，Ubuntu 22.04 LTS，$5/月，512MB / 2 vCPU / 20GB SSD
- 节点：sing-box 1.14.4，VLESS + Reality，443 端口，SNI www.microsoft.com
- 客户端：Clash Mi，GLOBAL → AWS-Sydney
- 2026-11-08 14:21 确认：Google / Gemini / ChatGPT / muse.ai 都能打开，而且很快。

> ⚠️ 公开版请不要贴真实的 UUID、私钥、公钥、VLESS 链接和服务器公网 IP。
> 下面所有密钥都用占位符代替，照着自己的实际值填就行。

## 时间线

### 2026-11-07 晚：注册

- 支付页先选 China UnionPay 通道，实际用 Mastercard，Send code 一度报服务器错误，陷入「没梯子加载不出 / 有梯子发码报错」的死循环。
- **转机：改选 Other credit or debit card 后，绑卡验证通过，注册完成。**
- 控制台确认 Credits remaining $100、Days remaining 182 天。
- 区域被锁在悉尼 ap-southeast-2，改地址栏、挂新加坡梯子刷新都没用。AWS 提示要「激活高级功能」才能自由选区域，那个流程不可逆（升级账号、设 root 密码、管组织、消费上限移除等），**不要激活**。
- 悉尼是流量减半区域：$5 套餐在悉尼是 512GB，在新加坡是 1TB。先跑起来再说。

### 2026-11-07 23:22：建实例

- 实例名 Ubuntu-1，Ubuntu 22.04.5 LTS，General purpose、Dual-stack，自动快照**没勾**（勾了额外花钱）。
- 账单提醒已创建（Budgets，收件填 QQ 邮箱）。

### 2026-11-07 23:23：升级系统

- 浏览器 SSH 连入，`sudo apt update`、`sudo apt upgrade -y`。
- 中途两个正常弹窗（Pending kernel upgrade、Daemons using outdated libraries），回车 / Ok 通过。
- `sudo reboot` 后立即重连会报 515 / instance error，等 1–2 分钟再连。
- 重启后内核 6.8.0-1069-aws，`0 updates can be applied`。

### 2026-11-07 23:53：装 sing-box

```bash
bash <(curl -fsSL https://sing-box.app/deb-install.sh)
sing-box generate uuid
sing-box generate reality-keypair
```

- 配置文件：`/etc/sing-box/config.json`
- VLESS inbound，listen 0.0.0.0:443，TLS Reality，server_name / handshake server 均为 `www.microsoft.com:443`，short_id 自定。
- `sing-box check` 通过后，`sudo systemctl enable --now sing-box`，确认 Active: active (running)。
- 防火墙：Lightsail 实例页 → 联网，添加 HTTPS / TCP / 443，来源全部 IPv4。重复添加会提示已存在规则，不用再加。

### 2026-11-08 00:09–00:25：Clash Mi 导入

- Clash Mi 的「添加配置链接」和「从剪贴板导入」只认 http/https 订阅链接，**不认单个 vless:// 节点链接**，会报 `Unsupported scheme 'vless'`。
- 改走本地 YAML：proxies 里写 AWS-Sydney，proxy-groups 用 PROXY select，rules 用 `MATCH,PROXY`。
- 保存时编辑器自动加后缀，文件实际是 `aws.yaml.txt`，Clash 按 *.yaml/*.yml 过滤时看不到，最后在文件名框手动输入 aws.yaml.txt 才导入成功。
- 第一次导入后报 `REALITY is based on uTLS...`。
- **修复：在 YAML 里加 `client-fingerprint: chrome`，并删掉客户端的 `flow: xtls-rprx-vision` 行**，重导入后报错消失。
- 00:25 Clash Mi 显示「已连接」，GLOBAL → AWS-Sydney。
- GLOBAL 不能选 DIRECT，选 DIRECT 等于直连、没走代理。

### 2026-11-08 00:25–00:43：第一次测试，不稳

- Anthropic 能开、很流畅；tympanus.net 能开但慢。
- GitHub 先报 ERR_CONNECTION_CLOSED，切 DIRECT 能开（GitHub 本来就能直连，不能证明节点好坏）。
- Gemini、muse.ai 经节点打不开，切 DIRECT 也打不开 / 很慢。
- 结论：节点通了但不稳、慢，不是单个网站的问题。先切回 DIRECT 睡觉，白天再修。

## 2026-11-08 白天排查：真正的问题在哪

### 第一步：服务器自己能不能上网？能

在服务器 SSH 里跑：

```bash
curl -I https://www.google.com
curl -I https://www.anthropic.com
curl -I https://chatgpt.com
curl -I https://muse.ai
```

全部 HTTP/2 200。所以服务器出口没问题，悉尼 IP 也没被这些站拦，问题在「电脑 → 服务器」这一段。

### 第二步：日志里那些 ERROR 是什么？

```bash
sudo journalctl -u sing-box --since "5 minutes ago" --no-pager
```

日志里大量 `REALITY: processed invalid connection`，来源 IP 各不相同。

**别慌：那是全网扫 443 端口的机器人在乱敲门，Reality 本来就是要把它们挡在外面的，是正常噪音。**

真正有用的是：你刚开 Google 那一秒，日志里有没有出现你自己的那条连接。如果出现了，而且也是 invalid，那就是你的 Clash 握手被拒了。

### 第三步：flow 两边对不上

- 服务器 config 里用户那行还写着 `"flow": "xtls-rprx-vision"`。
- 客户端 yaml 里 flow 已经删了。
- 两边不一致，握手时好时坏。

去删服务器那行：

```bash
sudo nano /etc/sing-box/config.json
```

### 第四步：删 flow 删坏了配置文件

删完 flow 后，sing-box 直接起不来了：

```
Active: activating (auto-restart) (Result: exit-code)
Main PID: (code=exited, status=1/FAILURE)
```

检查：

```bash
sudo sing-box check -C /etc/sing-box
```

报错：

```
decode config at /etc/sing-box/config.json: invalid character ']' looking for beginning of object key string: row 10, column 7
```

原因：删 flow 那行的时候，把用户对象的大括号 `}` 一起删掉了，uuid 后面那个逗号悬空了。

用 `cat -n` 看行号，确认第 9、10 行是：

```
9    { "uuid": "你的UUID",
10   ],
```

修复（只改第 9 行，把行尾逗号换成 ` },`）：

```bash
sudo sed -i '9s/,/ },/' /etc/sing-box/config.json
```

再跑：

```bash
cat -n /etc/sing-box/config.json
sudo sing-box check -C /etc/sing-box
sudo systemctl restart sing-box
sudo systemctl status sing-box
```

check 不报错、status 回到 Active: active (running)、日志里有 `tcp server started at 0.0.0.0:443`，才算修好。

**教训：改 JSON 别只删一行，要看逗号和括号是不是还成对。改完先 check，再 restart。**

### 第五步：Clash 里的配置是旧的

Clash 导入文件是复制一份存自己那儿的，桌面上的 yaml 改了，它里面还是旧版，不会自动跟上。

**改了 yaml，一定要删掉 Clash 里的旧配置，重新导入一次。**

### 第六步：public-key 差一个字

yaml 里的 public-key 结尾，大写 I 和小写 l 在编辑器字体里长得太像了，差一个字 Reality 握手就全军覆没。

干脆换一对新的，全程复制粘贴、不手打：

```bash
sing-box generate reality-keypair
```

- 服务器 config 里的 `private_key` 换成新的 PrivateKey。
- 客户端 yaml 里的 `public-key` 换成新的 PublicKey。
- `sing-box check` 通过后 restart。

### 第七步：yaml 文件本身被编辑器改坏了

第一次重导新密钥后，Clash 报：

```
yaml: line 1: did not find expected key
```

原因是之前用的那个编辑器是 Markdown 编辑器（带 H1、B、I 按钮的那种），存出来的文件带了看不见的格式，Clash 第 1 行就读不懂。

**改 yaml 用记事本，不要用 Markdown 编辑器。**

用记事本新建 `aws2.yaml`，完整内容（把占位符换成自己的实际值）：

```yaml
proxies:
  - name: AWS-Sydney
    type: vless
    server: 你的服务器公网IP
    port: 443
    uuid: 你的UUID
    tls: true
    servername: www.microsoft.com
    reality-opts:
      public-key: 你的新PublicKey
      short-id: a1b2c3d4
    client-fingerprint: chrome
    network: tcp
    udp: true
proxy-groups:
  - name: PROXY
    type: select
    proxies:
      - AWS-Sydney
rules:
  - MATCH,PROXY
```

Clash 里删掉旧配置，导入 aws2.yaml，GLOBAL 选 AWS-Sydney，再开 google.com。

### 结果

2026-11-08 14:21，Google / Gemini / ChatGPT / muse.ai 都能打开，而且很快。节点正式上岗。

## 经验清单（下次直接照着做）

1. **先定性，再动手**：服务器上 `curl -I` 先跑一圈，能通就不是服务器出口的问题，别在服务器上乱改。
2. **日志要分清噪音和信号**：Reality 挡扫描器是正常的，只看你自己操作那几秒有没有新记录。
3. **flow 两边必须一致**：服务器和客户端，要么都有 flow，要么都没有。
4. **改 JSON 先 check 再 restart**：`sing-box check -C /etc/sing-box` 不报错再重启，不然服务直接起不来。
5. **密钥全程复制粘贴**：I / l、0 / O 这种字，手打必错，换密钥就整对换。
6. **Clash 改配置要删了重导**：它存的是导入时的副本，改桌面文件不会自动同步。
7. **yaml 用记事本改**：别用 Markdown 编辑器，存出来的格式会把 Clash 搞懵。
8. **浏览器 SSH 掉线很正常**：sing-box 是后台服务，SSH 断了它还在跑。别用 `journalctl -f` 一直挂着，容易把浏览器 SSH 卡断，改用 `--since "5 minutes ago"` 看最近的日志。
9. **测试网站要选对**：GitHub 本来就能直连，DIRECT 能开不能证明节点好坏；测节点用 google.com 判断。

## 备忘

- Lightsail $5 套餐在悉尼只有 512GB 流量，记得看用量。
- 只停实例不停收费，要删才停；删后记得释放静态 IP，不然闲置也要收费。
- 节点链接是钥匙，别发给别人、别发群里。
- Oracle 工单还在排队，Lightsail 只是备用，不要因为节点通了就去动 Oracle 的工单。

---

*本报由 Cleo 与噗噗共同编辑。日子再小，登报为大。*
