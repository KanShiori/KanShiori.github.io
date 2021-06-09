# RBAC 实践


Kubernetes 中，通过 RBAC 机制来实现集群 Pod 中对 APIServer 的访问权限控制与授权。

RBAC 机制有三个最基本的概念：
1. **Role**：一组规则，定义了对 API 对象的操作权限；
2. **Subject**：被作用者，集群内部常常使用的是 ServiceAccount；
3. **RoleBinding**：绑定 Role 与 Subject；

## 1 Role 与 ServiceAccount 实践
目的：通过 Role 与 RoleBinding 限制一类 ServiceAccount，并在 Pod 中使用该 ServiceAccount 观察权限控制。
1. 创建需要使用的自定义 namespace [mynamespace]
   {{< find_img "img1.png" >}}
2. 创建需要限制访问权限的 ServiceAccount [example-sa]
   {{< find_img "img2.png" >}}
   
   可以看到，每个 namespace 有个默认的 ServiceAccount default，提供完整的 APIServer 访问权限。
   
   每个 ServiceAccount 在容器维度看到就是 Secret 对象，包含证书内容。
   {{< find_img "img3.png" >}}
3. 创建 Role，定义允许的权限规则。
   {{< find_img "img4.png" >}}
   
   可以看到，rules 指定了该 Role 为：允许对 mynamespace 下的 pod 进行 get、watch、list 操作。
   {{< find_img "img5.png" >}}
4. 创建 RoleBinding，关联刚刚创建的 Role 与 ServiceAccount。
   {{< find_img "img6.png" >}}
   {{< find_img "img7.png" >}}
5. 创建 Pod，指定使用的 ServiceAccount，在 Pod 内观察权限是否被限制了
   {{< find_img "img8.png" >}}
   
   进入容器中，可以看到，k8s 将 ServiceAccount 对应的 Serects 对象挂载到了 `/run/secrets/kubernetes.io/serviceaccount` 目录下，包含 client 需要使用的【证书 ca.crt】、【token】:
   {{< find_img "img9.png" >}}
