# IAM


## 1 IAM Identity
IAM Identity 是 AWS 中权限管理的对象，通过将 Identity 与 Policy 关联，可以控制 Identity 访问 AWS 资源的权限。

IAM Identity 是一个抽象概念，实际可以代表：
* Root User
* IAM User
* IAM User Group
* IAM Role

### 1.1 Root User
创建 AWS 账户时，自动创建唯一的 Root User，有着所有的权限。

因为 Root User 的权限过大，不要将其用于日常操作。

### 1.2 IAM User
你可以在 AWS 账户下创建多个 IAM User，提供给他人或者程序使用。

IAM User 也有着独立的用户名与密码，可以登陆到 AWS 控制台。也可以包含两个访问密钥，用于 API 或者 CLI 来访问 AWS。

通过为 IAM User 附加合适的 Policy，来控制其用户的权限。

### 1.3 IAM User Group
多个 IAM User 可以组成 IAM User Group，用于批量管理一组 IAM User 的权限。同一组下的 IAM User 会自动被分配 User Group 配置的权限。

### 1.4 IAM Role
IAM Role 与 IAM User 类似，但是 Role 不与特定的人员关联，也就是没有关联任何永久性的密码或访问密钥。

使用时，User 可以担任某个 Role，这样就会为其创建 Temporary Credentials，用于一定时间内的访问权限。

### 1.5 使用场景
除了最初创建 AWS 账户时，都不要使用 Root User。可以为自己创建一个 IAM User，使用其来使用 AWS。

为受信任的人员创建 IAM User，通过 IAM User Group 分组进行管理。

创建 Role，提供给应用程序使用。因为 Role 没有关联永久性的凭证，所以适合在应用程序使用。


## 3 权限检查流程
权限检查原则：
* 默认所有请求都是被拒绝；
* 通过检查
