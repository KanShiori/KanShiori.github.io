# 跨云访问资源


跨云访问指的是：在一个云厂商的环境中，访问另一个云厂商的服务。最简单的方式就是将静态密钥（AWS AK/SK 与 GCP ServiceAccount Key）注入到应用中，但是显而易见这种方式非常的不安全。

下面通过各个云的 Service 来提供一种安全的跨云访问的方式。

## 1 背景知识

### 1.1 GCP Workload Identity Federation

在 GCP 环境中应用都是都过 Service Account Key 访问 GCP 资源。为了能够在其他 Cloud 上安全地使用 Service Account，GCP 提供了 [**Workload Identity Federation**](https://cloud.google.com/iam/docs/workload-identity-federation) 服务，支持外部安全的访问 GCP。

简单点说，Workload Identity Federation 服务遵循标准的 OAuth 2 Token Exchange 协议，<important>允许 Identity Provider 颁发的 ID Token 来交换得到 Service Account Token</important>，进而 GCP Client 可以使用该 Token 得到 Service Account 的凭证后访问 GCP。

GCP 通过 **`Workload Identity Pool`** 来维护 Identity Provider 与 Service Account 的映射关系。一个 Workload Identity Pool 包括：

* **Connected Service Accounts** - 允许访问的 Service Account
  
  为了能够让 Workload Identity Pool 允许访问某个 Service Account，需要配置 Server Account 的 Principals 加上 Pool Name。

  不同的 Principals 格式会限制 Pool 中哪些 Identity Provider 能够访问：

  {{< image src="img1.png" height=200 >}}

* **Identity Providers** - 指定哪些 Identity Providers 颁发的 Token 能够交换 Token

  目前支持的 Identity Provider 类型有：

  * AWS - 允许特定 AWS 账户的 Role 来获取 Service Account；
  * OIDC - 允许特定 OIDC Identity Provider 颁发的 ID Token 来交换 Service Account；
  * SAML

在下面的示例可以看到，基本所有外部访问 GCP 都是使用的 Workload Identity Federation。

### 1.2 AWS IAM Role

在 [**AWS Role Trust Relationship**](../../aws_learning/iam/#41-trust-relationship) 中提到，Role 可以配置哪些 Identity 可以 Assume 改 Role。在跨云访问 AWS 场景下，就是为 IAM Role 配置映射的 OIDC Identity Provider，使得带着 ID Token 就可以 Assume Role（即获得 Role 的访问凭证）。

对应 IAM Policy 类似如下：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Principal": {
                "Federated": "accounts.google.com"
            },
            "Condition": {
                "StringEquals": {
                    "accounts.google.com:aud": [
                        "123"
                    ]
                }
            }
        }
    ]
}
```

* **Principal.Federated** 表明支持的 OIDC Identity Provider
* **Condition.StringEquals** 根据 ID Token 的 Audience 与 Issuer 进行判定

## 2 AWS Instance 访问 GCP

在当前场景下，应用程序中的 GCP SDK 会访问 EC2 metadata 来获取 AWS 凭证。接着，SDK 会拿着 AWS 凭证去请求 Workload Identity Federation 以获取 Service Account Token。获取成功后，SDK 就能够以对应的 Service Account 身份访问 GCP 服务了。

{{< image src="img2.png" height=330 >}}

### 2.1 具体步骤

1. 创建 EC2 Instance，配置好 Instance Profile 以使用指定的 Role

2. 创建一个 Workload Identity Pool，并配置支持 OIDC Identity Provider
   
   ```bash
   # 创建 Workload Identity Pool
   gcloud iam workload-identity-pools create ${pool_id} \
       --location="global" \
       --description="My AWS Identity Pool" \
       --display-name=${pool_id}
   
   # 配置 AWS Identity Provider
   gcloud iam workload-identity-pools providers create-aws ${provider_id} \
        --location="global" \
        --workload-identity-pool=${pool_id} \
        --account-id=${aws_account}
   ``` 

   参考文档 [**Configuring workload identity federation**](https://cloud.google.com/iam/docs/configuring-workload-identity-federation)。

3. 允许 Workload Identity Pool 访问我们需要的 Service Account
   
   ```bash
   gcloud iam service-accounts add-iam-policy-binding ${sa_email} \
        --role=roles/iam.workloadIdentityUser \  # 必须提供的权限
        --member="principalSet://iam.googleapis.com/projects/${project_number}/locations/global/workloadIdentityPools/${pool_id}/attribute.aws_role/arn:aws:sts::${aws_account}:assumed-role/${aws_role}"      # pool principal，其中 project number 必须是 project 的数字编号
   ```

4. 创建 Credential 配置文件，用以 SDK 或者 CLI 使用。

   目前，相关的权限配置已经完成了。GCP SDK 与 CLI 内部都实现了使用 Workload Identity Pool 获取 Token 的方式，我们只需要创建好 Credential 配置文件，让 SDK 或者 CLI 加载即可。

   Credential 配置文件中的字段都是固定的，可以自己拼接，也可以使用 CLI 帮助生成：

   ```bash
   gcloud iam workload-identity-pools create-cred-config \
        projects/${project_number}/locations/global/workloadIdentityPools/${pool_id}/providers/${provider_id} \
        --service-account=${sa_email} \  # 使用的 Service Account
        --aws \ 
        --output-file=credentials.json
   ```

   生成的 Credentials 配置文件如下：

   ```json
   {
     "type": "external_account",
     "audience": "//iam.googleapis.com/projects/${project_number}/locations/global/workloadIdentityPools/${pool_id}/providers/${provider_id}",
     "subject_token_type": "urn:ietf:params:aws:token-type:aws4_request",
     "token_url": "https://sts.googleapis.com/v1/token",
     "credential_source": {
       "environment_id": "aws1",
       "region_url": "http://169.254.169.254/latest/meta-data/placement/availability-zone",
       "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials",
       "regional_cred_verification_url": "https://sts.{region}.amazonaws.com?Action=GetCallerIdentity&   Version=2011-06-15"
     },
     "service_account_impersonation_url": "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/${sa_name}@${project_id}.iam.gserviceaccount.com:generateAccessToken"
   }
   ```

5. 在 Instance 中，通过 SDK 或者 CLI 使用 Credentials 配置文件，进而访问 GCP

   以 CLI 为例，使用 `gcloud auth login --cred-file=${file}` 就可以以 Service Account 方式登陆。

   ```bash
   gcloud auth login --cred-file=credentials.json

   Authenticated with external account credentials for: [${sa_email}].
   ``` 

## 3 GCP 访问 AWS

GCP Service Account 支持为 [**Service Account 生成 OIDC ID Token**](https://cloud.google.com/iam/docs/create-short-lived-credentials-direct#sa-credentials-oidc)。所以我们只需要创建一个 Service Account，然后配置 AWS IAM Role Trust Relationship，使得能够使用 Service Account ID Token 来执行 AssumeRoleWithIdentity，获取临时凭证后 GCP 环境访问 AWS。

{{< image src="img2-1.png" height=300 >}}

### 3.1 具体步骤

1. 创建 VM Instance，配置好使用的 Service Account
   
2. 创建一个 AWS Role，并配置 Trust Relationship，使得允许使用 Service Account Token 执行 AssumeRoleWithIdentity 操作

   添加的配置如下：

   ```json
   {
       "Version": "2012-10-17",
       "Statement": {
           "Sid": "RoleForGoogle",
           "Effect": "Allow",
           "Principal": {"Federated": "accounts.google.com"},
           "Action": "sts:AssumeRoleWithWebIdentity",
           "Condition": {"StringEquals": {"accounts.google.com:aud": "${sa_client_id}"}}
       }
   }
   ```

3. 在 VM Instance 上，通过 Service Account Token 来获取 AWS 访问凭证
   
   1. 访问 Metadata Server，获取 Instance 信息与 Service Account 的 ID Token
   
      ```bash
      instance_name=$(curl -H Metadata-Flavor:Google \ 
        'http://metadata.google.internal/computeMetadata/v1/instance/name')

      project_id=$(curl -H Metadata-Flavor:Google \ 
        'http://metadata.google.internal/computeMetadata/v1/project/project-id')

      sa_token=$(curl -H Metadata-Flavor:Google \ 
        'http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?format=standard&audience=gcp')
      ```

      其中，Service Account ID Token 解析后如下格式：

      ```json
      {
        "aud": "gcp",
        "azp": "${sa_client_id}",
        "exp": 1667312309,
        "iat": 1667308709,
        "iss": "https://accounts.google.com",
        "sub": "${sa_client_id}"
      }
      ```

   3. 调用 AssumeRoleWithIdentity 获取 Role 的临时访问凭证
   
      ```bash
      aws sts assume-role-with-web-identity \ 
        --role-arn arn:aws:iam::${account_id}:role/${role_name} \ 
        --role-session-name ${project_id}.${instance_name} \ 
        --web-identity-token ${sa_token}
      ```

      输出的结果如下，可以看到成功获取了临时的 AK/SK：

      ```json
      {
          "Credentials": {
              "AccessKeyId": "${access_key}",
              "SecretAccessKey": "${secret_key}",
              "SessionToken": "${session_token}",
              "Expiration": "2022-11-01T14:31:14+00:00"
          },
          "SubjectFromWebIdentityToken": "${sa_client_id}",
          "AssumedRoleUser": {
              "AssumedRoleId": "xxxx",
              "Arn": "arn:aws:sts::${account_id}:assumed-role/${role_name}/${role_session_name}"
          },
          "Provider": "accounts.google.com",
          "Audience": "${sa_client_id}"
      }
      ```

4. 在 Instance 上，使用临时的 AK 与 SK 访问 AWS 服务
   
   ```bash
   export AWS_ACCESS_KEY_ID="${access_key}"
   export AWS_SECRET_ACCESS_KEY="${secret_key}"
   export AWS_SESSION_TOKEN="${session_token}"

   # 访问 AWS
   ```

## 4 AWS EKS 访问 GCP

在 EKS 中，Pod 想访问 AWS 是都通过 [**IRSA**](../../aws_learning/eks/#63-irsa) 的方式。其本质上时使用的 OIDC 方式：由 EKS OIDC Identity Provider 给 Pod 中注入 ID Token，Pod 中应用使用 ID Token 访问 AWS STS 服务得到 Role 的访问凭证，进而访问 AWS 服务。

既然 Pod 有了 ID Token，那么我们就可以使用 GCP **Workload Identity Federation** 服务，支持使用 EKS OIDC Identity Provider 颁发的 ID Token，来交换得到 GCP Service Account Token。

{{< image src="img3.png" height=330 >}}

### 4.1 具体步骤

1. 参考 [**IRSA 使用流程**](../../aws_learning/eks/#631-使用流程) 创建 EKS 并使用 IRSA 提供给 Pod 权限。 
   
   这时候，Pod 里面就注入了 ID Token，解析后可以看到 EKS OIDC Identity Provider URL 标识：

   ```json
    {
        "aud": [
            "sts.amazonaws.com"
        ],
        "exp": 1666828971,
        "iat": 1666742571,
        "iss": "https://oidc.eks.us-west-2.amazonaws.com/id/12312415", // IdP URL
        "kubernetes.io": {
            "namespace": "default",
            "pod": {
                "name": "controller-l6x5p",
                "uid": "20a40844-12f8-44bc-b3c3-1b9b245a62ab"
            },
            "serviceaccount": {
                "name": "controller",
                "uid": "a2d72939-1234-4045-83d8-00e8854651e5"
            }
        },
        "nbf": 1666742571,
        "sub": "system:serviceaccount:default:controller"
    }
   ```
   
   我们也可以通过获取 EKS 信息来得到 Identity Provider URL 标识：

   ```json
   // aws eks describe-cluster --name ${cluster-name} --region us-west-2
   {
       "cluster": {
           // ...
           "identity": {
               "oidc": {
                   "issuer": "https://oidc.eks.us-west-2.amazonaws.com/id/3TY232GFASDFSDG"
               }
           }
       }
   }
   ```

2. 创建一个 Workload Identity Pool，并配置支持 OIDC Identity Provider
   
   ```bash
   # 创建 Workload Identity Pool
   gcloud iam workload-identity-pools create ${pool_id} \ 
       --location="global" \ 
       --description="My AWS Identity Pool" \ 
       --display-name=${pool_id}
   
   # 配置 OIDC Identity Provider
   gcloud iam workload-identity-pools providers create-oidc ${provider_id} \ 
        --location="global" \ 
        --workload-identity-pool=${pool_id} \ 
        --issuer-uri=${issuer} \                           # 信任的 issuer，即第 1 步中的 issuer url
        --allowed-audiences="sts.amazonaws.com" \          # 信任的 audiences，即第 1 步中的 aud
        --attribute-mapping="google.subject=assertion.sub" # 属性映射
        # --attribute-condition=""                         # 可选的条件判断
   ``` 

   参考文档 [**Configuring workload identity federation**](https://cloud.google.com/iam/docs/configuring-workload-identity-federation)。

3. 允许 Workload Identity Pool 访问我们需要的 Service Account
   
   也就是给 Service Account Principal 加上 Workload Identity Pool，允许 Workload Identity Pool 下的所有 Identity Provider 得到 Service Account 的 Access Token：

   ```bash
   gcloud iam service-accounts add-iam-policy-binding ${sa_email} \ 
        --role=roles/iam.workloadIdentityUser \  # 必须提供的权限
        --member="principalSet://iam.googleapis.com/projects/${project_number}/locations/global/workloadIdentityPools/${pool_id}/*"      # pool principal，其中 project number 必须是 project 的数字编号
   ```

   更细粒度可以按照 Identity 的属性授予权限，参考文档 [**Service account impersonation**](https://cloud.google.com/iam/docs/workload-identity-federation?authuser=1#impersonation)。

4. 创建 Credential 配置文件，用以 SDK 或者 CLI 使用。
   
   目前，相关的权限配置已经完成了。GCP SDK 与 CLI 内部都实现了使用 Workload Identity Pool 获取 Token 的方式，我们只需要创建好 Credential 配置文件，让 SDK 或者 CLI 加载即可。

   Credential 配置文件中的字段都是固定的，可以自己拼接，也可以使用 CLI 帮助生成：

   ```bash
   gcloud iam workload-identity-pools create-cred-config \ 
        projects/${project_number}/locations/global/workloadIdentityPools/${pool_id}/providers/${provider_id} \
        --service-account=${sa_email} \  # 使用的 Service Account
        --credential-source-file="/var/run/secrets/eks.amazonaws.com/serviceaccount/token" \   # 用于交换的源 Token 文件，即 EKS 注入 Pod 的 ID Token 文件
        --output-file=credentials.json
   ```

   生成的 Credentials 配置文件如下：

   ```json
   {
     "type": "external_account",
     "audience": "//iam.googleapis.com/projects/${project_number}/locations/global/workloadIdentityPools/${pool_id}/providers/${provider_id}",
     "subject_token_type": "urn:ietf:params:oauth:token-type:jwt",
     "token_url": "https://sts.googleapis.com/v1/token",
     "credential_source": {
       "file": "/var/run/secrets/eks.amazonaws.com/serviceaccount/token"
     },
     "service_account_impersonation_url": "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/${sa_name}@${project_id}.iam.gserviceaccount.com:generateAccessToken"
   }
   ```

5. 在 Pod 中，通过 SDK 或者 CLI 使用 Credentials 配置文件，进而访问 GCP

   以 CLI 为例，使用 `gcloud auth login --cred-file=${file}` 就可以以 Service Account 方式登陆。

   ```bash
   gcloud auth login --cred-file=credentials.json

   Authenticated with external account credentials for: [${sa_email}].
   ``` 

## 参考

* Blog: [**Securely access cloud resources between AWS and GCP**](https://lingxiankong.github.io/2022-02-13-access-resource-multi-clouds.html)

