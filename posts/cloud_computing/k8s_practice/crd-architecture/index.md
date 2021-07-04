# CR Controller Architecture


## 构思与创建 CRD
构思我们的 CR 资源的定义：
```yaml
apiVersion: kanshiori.cn/v1alpha1
kind: Trouble
metadata:
  name: one-trouble
spec:
  # action 定义执行的行为
  action: delete
  # target 设置操作对象
  target:
  - kind: Pod
    count: 1
    namePattern: "pod-*"
```

从而去编写对应的 CRD 定义：
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: troubles.kanshiori.cn
spec:
  group: kanshiori.cn
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                action:
                  type: string
                target:
                  type: array
                  items:
                    type: object
                    properties:
                      kind:
                        type: string
                      count:
                        type: integer
                      namePattern:
                        type: string
  scope: Namespaced
  names:
    plural: troubles
    singular: trouble
    kind: Trouble
    shortNames:
    - tb
```


编写 apis types 代码

## code-generator
使用 [code-generator](https://github.com/kubernetes/code-generator) 来生成 Controller 骨架代码。
1. 将库在代码中使用
2. 使用 `go mod vendor` 生成 vendor 包
2. 编写脚本（来自于 tidb-opeartor)
3. 执行命令
```shell
$ bash hack/update-codegen.sh
Generating deepcopy funcs
Generating clientset for kanshiori:v1alpha1 at github.com/KanShiori/troublemaker/pkg/client/clientset
Generating listers for kanshiori:v1alpha1 at github.com/KanShiori/troublemaker/pkg/client/listers
Generating informers for kanshiori:v1alpha1 at github.com/KanShiori/troublemaker/pkg/client/informers
```



