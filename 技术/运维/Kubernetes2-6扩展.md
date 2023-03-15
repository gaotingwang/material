# Custom Resource

自定义资源（Custom Resource） 是对 Kubernetes API 的扩展。定制资源可以通过动态注册的方式在运行中的集群内或出现或消失，集群管理员可以独立于集群 更新定制资源。一旦某定制资源被安装，用户可以使用 kubectl 来创建和访问其中的对象，就像他们为 pods 这种内置资源所做的一样。

什么时候应该使用自定义资源

- 希望使用 Kubernetes 客户端库和 CLI 来创建和更改新的资源。
- 希望 kubectl 能够直接支持自定义的资源；例如，kubectl get my-object object-name。
- 希望构造新的自动化机制，监测新对象上的更新事件，并对其他对象执行 CRUD 操作，或者监测后者更新前者。
- 希望编写自动化组件来处理对对象的更新。
- 希望使用 Kubernetes API 对诸如 .spec、.status 和 .metadata 等字段的约定。
- 希望对象是对一组受控资源的抽象，或者对其他资源的归纳提炼  

## 创建 CRD

当创建新的 CustomResourceDefinition（CRD）时，Kubernetes API 服务器会为你所指定的每一个版本生成一个 RESTful 的资源路径。CRD 可以是名字空间作用域的，也可以是集群作用域的，取决于CRD 的 scope 字段设置。和其他现有的内置对象一样，删除一个名字空间时，该名字空间下的所有定制对象也会被删除。CustomResourceDefinition 本身是不受名字空间限制的，对所有名字空间可用

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        # openAPIV3Schema is the schema for validating custom objects.
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                  # 合法性检验
                  pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
                image:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

```shell
$ kubectl apply -f resourcedefinition.yaml
$ kubectl get crd
```

## 创建定制对象

创建定制对象（Custom Objects）。定制对象可以包含定制字段。这些字段可以包含任意的 JSON 数据。 在下面的例子中，在类别为 CrontTab 的定制
对象中，设置了cronSpec 和 image 定制字段  

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
  replicas: 5
```

```shell
$ kubectl apply -f my-crontab.yaml
$ kubectl get crontab
```

