#　kubernetes 使用configmap记录

**需求**：有些镜像启动需要外挂配置文件，所以在kubernetes中为了保证在任意机器中都能正常启动，要共享配置文件。使用kubernetes configmap可以实现。

## 配置文件

以创建etcd的web端e3w为例，文件名称：e3w-alert.ini，配置内容：

```ini
[app]
port=8080
auth=false

[etcd]
root_key=/AlertConfig
addr=10.13.56.79:2379
```

创建configmap命令：　`kubectl create configmap e3w-alert --from-file=config.default.ini=e3w-alert.ini`

查看configmap命令：　`kubectl describe configmap e3w-alert`，`kubectl get configmaps e3w-alert -o yaml`

更新configmap命令：　`kubectl edit configmap e3w-alert`

删除configmap命令：`kubectl delete configmap e3w-alert`

有两种方式让pod使用，第一种是环境变量或参数，第二种是文件挂载。

使用以下配置，创建e3w的Deployment，同时会在容器内部/app/conf目录下创建config.default.ini文件，因为在创建时指定文件的名称为config.default.ini，所以会按此名称生成，如果不指定，默认使用原始文件名称。


```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: e3w-alert
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: alert-manager
    spec:
      containers:
      - name: e3w-alert
        image: soyking/e3w:latest
        ports:
        - containerPort: 8080
          hostPort: 8089
        volumeMounts:
          - name: e3w-alert-ini
            mountPath: /app/conf
      volumes:
      - name: e3w-alert-ini
        configMap:
          name: e3w-alert  
```

挂载文件的configmap支持热更新，`kubectl apply -f X.yaml`



## 配置目录　

从同一目录下的一组文件中创建ConfigMap，命令：`kubectl create configmap config-name --from-file=dir`

也可以多次传递--from-file参数使用不同的数据源来创建ConfigMap，命令：`kubectl create configmap e3w-alert --from-file=e3w-alert.ini --from-file=e3w-config.ini`


## 配置项

使用--from-literal参数在命令中定义字面值，命令：`kubectl create configmap config-literal --from-literal=literal.a=a --from-literal=literal.b=b`



**参考：**

- https://k8smeetup.github.io/docs/tasks/configure-pod-container/configmap/
- http://blog.51cto.com/devingeng/2156864
