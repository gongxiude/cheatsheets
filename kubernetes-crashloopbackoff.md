---
title: Kubernetes CrashloopBackOff
layout: 2017/sheet
prism_languages: [bash,yaml]
weight: -3
tags: [Featured]
updated: 2020-07-20
category: Kubernetes
---

## 如何在Kubernetes中调试CrashloopBackOff pod
{: .-one-column}

很多时候pod报错CrashloopBackOff都表示为pod中的程序存在了某些问题，并且一直在重复的启动 通过查看logs的方式又不能获取到有用的信息。 这个时候我们需要在失败的容器中运行一个shell，对失败的原因进行排查， 但是没有容器可以连接。 

在这种情况下我们可以参考[Run a command in a Shell documentation](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/#run-a-command-in-a-shell)中的介绍， 覆盖docker中的命令


修改deployment 

```
$ kubectl edit deploy mydeployment -n mynamespace
```

在容器中找到定义command的位置， 如果你的启动脚本为`bootstrap.sh`： 


```
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        command: bootstrap.sh
```

如果在deployment中没有找到相关的配置， 我们依据可以添加一个command， 因为他会覆盖在image的默认配置。 

修改会添加如下的参数到资源文件中， 如下所示： 

```
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        command: ["/bin/sh"]
        args: ["-c", "while true; do echo hello; sleep 10;done"]
```


**注意: **  `**args**`需要定义为一个数组，当我们保存后他会加载为如下所示内容： 

```
spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        command:
        - /bin/sh
        args:
        - -c
        - while true; do echo hello; sleep 60;done
```

同样的道理我们也可以添加环境变量到pod中， 如下所示： 


```
spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        command:
        - /bin/sh
        args:
        - -c
        - while true; do echo hello; sleep 60;done
        env:
        - name: MYAPP_DEBUG
          value: "true"
```

保存资源的定义后，你会看到pod变为了running状态， 我们可以[Get a shell to the running container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#getting-a-shell-to-a-container):

```
$ kubectl exec -it mydeployment-65gvm -n mynamespace -- sh
```

这个时候我们就可以进入running的container中调试程序了

```
/ # echo $MYAPP_DEBUG
true
/ #
```
