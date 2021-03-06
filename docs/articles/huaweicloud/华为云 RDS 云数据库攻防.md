---
title: 华为云 RDS 云数据库攻防
id: huaweiyuncloud_rds
---

本文所有的演示都购买的Mysql服务
### 0x01 前期侦查
##### 1、访问凭证泄露

<!-- more -->

在进行信息收集时，可以收集目标的数据库账号密码、云服务账号的访问凭证、临时凭证、SDK 等等。
##### 2、备份管理
在备份管理处可以看到数据库的备份情况，可以恢复或者下载，查找数据

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650353815-214172-image.png)

如果我们拿到了ak sk可以登录OBS Browser查看外部桶，看用户是否将备份放到其中

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650353827-693755-image.png)

我们在信息搜集的时候，可以留意如下链接

`https://obs.cn-south-1.myhuaweicloud.com/dbsbucket-cn-south-1-ea28ad0541ce46d1a14cf6affac93242/1471f191bf52426f937b8e9b5f87a5e5_2b51b2c6abdf462893939964b4c612fain01_Snapshot_fccfc85d4767460786a8cdd00b021f87br01_20220328075533310_20220328075534031_dd6d837fe38542e39c939c65cc2eb946no01_.qp?AWSAccessKeyId=GJADEH3HQHGDFA1I9TZ1&Expires=1648455224&response-cache-control=no-cache,no-store&Signature=ZvOnZ1Ggj+d3M3rR+RNWFCLf5Xk=`


`https://obs.cn-south-1.myhuaweicloud.com:443/dbsbucket-cn-south-1-ea28ad0541ce46d1a14cf6affac93242/1471f191bf52426f937b8e9b5f87a5e5_2b51b2c6abdf462893939964b4c612fain01_Db_270e4eee5c0645c491a0256aee83277fbr01_20220328075745865_20220328075807598_b1f9ce13819a4a089c868f408cd66ecdno01_.qp?AWSAccessKeyId=GJADEH3HQHGDFA1I9TZ1&Expires=1648455325&response-cache-control=no-cache,no-store&Signature=cH06dRbBcK/fSX8MX04aM6lntoQ=`

上述链接可以看到是以.qp结尾的文件，该链接就是数据库的备份下载链接，需注意的是其中一个链接的实效只有5分钟

下面介绍下如何使用华为云cli来获取备份下载链接
1、获取 "tenant_id"

`hcloud ECS ListServersDetails --cli-region="cn-south-1"`

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650353946-998365-image.png)

2、查询数据库实例列表获取id

`hcloud RDS ListInstances --cli-region="cn-south-1" --Content-Type="application/json" --project_id="0a8453d1f700250e2f02c00e683b8531" `

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650353982-397947-image.png)

3、获取备份列表，拿到id

`hcloud RDS ListBackups --cli-region="cn-south-1" --project_id="0a8453d1f700250e2f02c00e683b8531" --instance_id="2b51b2c6abdf462893939964b4c612fain01"`

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650353992-77346-image.png)

4、获取备份下载链接

`hcloud RDS ShowBackupDownloadLink --cli-region="cn-south-1" --project_id="0a8453d1f700250e2f02c00e683b8531" --backup_id="270e4eee5c0645c491a0256aee83277fbr01"`

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354003-645191-image.png)

### 0x02 初始访问
##### 1、访问凭证登录
如果前期收集到了数据库的账号密码，则可以直接使用其登录。

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354121-171657-image.png)

如果泄漏了高权限的AK SK，可以新建一个同权限的IAM账号登录控制台
##### 2、弱口令
如果数据库存在弱口令，则可以通过密码爆破，猜解出 RDS 的账号密码。
华为云RDS默认用户名为root
##### 3、绑定公网IP
如果数据库创建在内网中，我们可以绑定公网IP来连接，需要注意的是这个公网IP必须是用户自己购买的

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354173-815492-image.png)

绑定后我们就可以通过远程登录来登录数据库了
##### 4、同VPC下主机登陆
如果RDS不出网，我们可以想办法登录同网段的ECS服务器，然后通过MYSQL连接到不出网的数据库

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354190-679343-image.png)

`mysql -h 192.168.0.12 -uroot -p`

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354206-163895-image.png)

### 0x03 语句执行
##### 1、数据管理服务DAS
华为云控制台需要数据库的密码才可以登录执行语句，我们可以通过泄漏的AK SK来新增账号或修改原账号密码来登录数据库，下面第五节会讲到

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354224-514046-image.png)

进入开发工具就可以登录数据库，执行语句

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354236-117517-image.png)

##### 2、数据库连接
如果知道数据库的账号密码后，可以远程连接试试
### 0x04 权限提升
##### 1、低权限下收集到高权限访问凭证信息
在使用低权限账号登录到数据库后，在数据库中可以尝试搜索访问密钥、账号密码等信息，获得高权限账号信息。
如果当前数据存在 Web 应用上的用户密码信息，则可以通过尝试获得 Web 应用系统权限，在 Web 应用系统权限下，搜索数据库的访问密钥或者云服务的 API 密钥，从而进一步横向到数据库高权限。
##### 2、数据库自身漏洞
比如提权，暂时未发现云服务商有此类漏洞。
##### 3、修改读写权限
控制台修改只读权限为读写

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354294-764973-image.png)

cli修改只读权限为读写

`hcloud RDS AllowDbUserPrivilege --cli-region="cn-south-1" --instance_id="dddddd" --project_id="0a8453d1f700250e2f02c00e683b8531" --db_name="huoxian" --users.1.readonly=false --users.1.name="admin"`

##### 4、修改IAM账号权限
首先看下官方的权限说明

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354352-643652-image.png)

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354377-453228-image.png)

我们可以修改IAM账号的权限使其拥有修改数据库的权限

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354395-145911-image.png)

### 0x05 权限维持
##### 1、添加数据库账号密码
控制台添加账号

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354419-990223-image.png)

华为云cli添加账号

`hcloud RDS CreateDbUser --cli-region="cn-south-1" --instance_id="2b51b2c6abdf462893939964b4c612fain01" --project_id="0a8453d1f700250e2f02c00e683b8531" --password="1234qwer!" --name="stay"`

##### 2、修改数据库账号密码
控制台修改密码

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354433-397349-image.png)

华为云cli修改密码(密码强规则)

`hcloud RDS ResetPwd --cli-region="cn-south-1" --instance_id="2b51b2c6abdf462893939964b4c612fain01" --project_id="0a8453d1f700250e2f02c00e683b8531" --db_user_pwd="1234qwer!"`

##### 3、共享数据库实例
我们可以将对方的数据库实例分享给自己，然后在自己的控制台就可以登录
首先我们需要知道数据库的账号密码，或者可以使用前面讲到的重新添加一个账号

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354462-140106-image.png)

接着新增数据库实例登录，这里选择记住密码，我们自己账号登录时便不需要密码了

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354471-149479-image.png)

然后查看共享用户数，选择手动录入，输入自己的账号ID即可，返回自己的账号看下

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354483-100347-image.png)

直接点击后面的登录

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354490-999734-image.png)

这样就可以在管理员不注意的情况下一直可以登录对方的数据库实例
##### 4、修改现有账号权限
例如账号无法使用UPDATE，CREATE等权限，可以在用户管理-全局权限处设置

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354501-105129-image.png)

### 0x06 防御绕过
可关闭告警相关的所有订阅
### 0x07 信息收集
##### 1、日志下载
可下载错误日志，慢日志和主备切换日志来查找信息
![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354551-943184-image.png)

##### 2、备份数据库
可下载已备份数据库到本地，来搜集信息

![](https://huoxian-community.oss-cn-beijing.aliyuncs.com/2022-04-19/1650354560-703348-image.png)

### 0x08 影响
可导致数据库信息泄漏，数据被篡改盗用等等

参考链接：
https://zone.huoxian.cn/d/1105-aws
