# alertmanager-dingtalk-hook
AlertManager 钉钉报警简单服务示例

## 运行
### 使用`Docker`运行
```shell
$ docker run -p 5000:5000 --name -e ROBOT_TOKEN=<钉钉机器人TOKEN> -e ROBOT_SECRET=<钉钉机器人安全SECRET> -e LOG_LEVEL=debug -e PROME_URL=prometheus.local dingtalk-hook -d uhub.service.ucloud.cn/davidertest/alertmanager-dingtalk-hook:v1
```

![image](https://user-images.githubusercontent.com/64472425/112759125-2f329a80-9024-11eb-910e-5a462ff169a3.png)

环境变量配置：

* ROBOT_TOKEN：钉钉机器人 TOKEN
* PROME_URL：手动指定跳转后的 Promethues 地址，默认会是 Pod 的地址
* LOG_LEVEL：日志级别，设置成 `debug` 可以看到 AlertManager WebHook 发送的数据，方便调试使用，不需调试可以不设置该环境变量
* ROBOT_SECRET：为钉钉机器人的安全设置密钥，机器人安全设置页面，加签一栏下面显示的 SEC 开头的字符串


### 在`Kubernetes`集群中运行
第一步建议将钉钉机器人TOKEN创建成`Secret`资源对象：
```shell
$ kubectl create secret generic dingtalk-secret --from-literal=token=<钉钉群聊的机器人TOKEN> --from-literal=secret=<钉钉群聊机器人的SECRET> -n uk8s-monitor
secret "dingtalk-secret" created
```
```shell
示例：
$ kubectl create secret generic dingtalk-secret --from-literal=token=xxx --from-literal=secret=testdingding -n uk8s-monitor
```

然后定义`Deployment`和`Service`资源对象：(dingtalk-hook.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dingtalk-hook
  namespace: uk8s-monitor
spec:
  selector:
    matchLabels:
      app: dingtalk-hook
  template:
    metadata:
      labels:
        app: dingtalk-hook
    spec:
      containers:
      - name: dingtalk-hook
        image: uhub.service.ucloud.cn/davidertest/alertmanager-dingtalk-hook:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
          name: http
        env:
        - name: PROME_URL
          value: prometheus.local
        - name: LOG_LEVEL
          value: debug
        - name: ROBOT_TOKEN
          valueFrom:
            secretKeyRef:
              name: dingtalk-secret
              key: token
        - name: ROBOT_SECRET
          valueFrom:
            secretKeyRef:
              name: dingtalk-secret
              key: secret
        resources:
          requests:
            cpu: 50m
            memory: 100Mi
          limits:
            cpu: 50m
            memory: 100Mi

---
apiVersion: v1
kind: Service
metadata:
  name: dingtalk-hook
  namespace: uk8s-monitor
spec:
  selector:
    app: dingtalk-hook
  ports:
  - name: hook
    port: 5000
    targetPort: http
```

直接创建上面的资源对象即可：
```shell
$ kubectl create -f dingtalk-hook.yaml
deployment.apps "dingtalk-hook" created
service "dingtalk-hook" created
$ kubectl get pods -n uk8s-monitor
NAME                            READY     STATUS      RESTARTS   AGE
dingtalk-hook-c4fcd8cd6-6r2b6   1/1       Running     0          45m
......
```

最后在`AlertManager`中 webhook 地址直接通过 DNS 形式访问即可：
```yaml
receivers:
- name: 'webhook'
  webhook_configs:
  - url: 'http://dingtalk-hook.uk8s-monitor.svc.cluster.local:5000'
    send_resolved: true
```

## 参考文档
* [钉钉自定义机器人文档](https://open-doc.dingtalk.com/microapp/serverapi2/qf2nxq)
* [AlertManager 的使用](https://www.qikqiak.com/k8s-book/docs/57.AlertManager%E7%9A%84%E4%BD%BF%E7%94%A8.html)

