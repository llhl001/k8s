# Kubernetes Replication Controller

1 [ReplicationController 工作原理](#ReplicationController 工作原理)

​		1.1 [示例：](#示例：)

​		1.2 [删除ReplicationController及其Pods](#删除ReplicationController及其Pods)



2 [RC 替代方法](#RC 替代方法)

 		2.1 [ReplicaSet](#ReplicaSet)

​		2.2 [Deployment（推荐）](#Deployment（推荐）)

​		2.3 [Bare Pods](#Bare Pods)





ReplicationController（简称RC）是确保用户定义的Pod副本数保持不变。

注意：建议使用[Deployment](http://docs.kubernetes.org.cn/317.html) 配置 [ReplicaSet](http://docs.kubernetes.org.cn/314.html) （简称RS）方法来控制副本数。



### ReplicationController 工作原理

在用户定义范围内，如果pod增多，则ReplicationController会终止额外的pod，如果减少，RC会创建新的pod，始终保持在定义范围。例如，RC会在Pod维护（例如内核升级）后在节点上重新创建新Pod。

- ReplicationController会替换由于某些原因而被删除或终止的pod，例如在节点故障或中断节点维护（例如内核升级）的情况下。因此，即使应用只需要一个pod，我们也建议使用ReplicationController。
- RC跨多个Node节点监视多个pod。



##### 示例：

- [replication.yaml](./replication.yaml)

  ```yaml
  apiVersion: v1
  kind: ReplicationController
  metadata:
    name: nginx
  spec:
    replicas: 3
    selector:
      app: nginx
    template:
      metadata:
        name: nginx
        labels:
          app: nginx
      spec:
        containers:
          - name: nginx
            image: hub.gerrywen.com/library/nginx:1.7.9
            ports:
              - containerPort: 80
  ```

  示例文件运行：

  ```shell
  kubectl create -f replication.yaml 
  ```

  ```shell
  [root@k8s-master01 02]# kubectl get rc
  NAME    DESIRED   CURRENT   READY   AGE
  nginx   3         3         3       5s
  [root@k8s-master01 02]# kubectl get rc,pod -o wide
  NAME                          DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                                 SELECTOR
  replicationcontroller/nginx   3         3         3       12s   nginx        hub.gerrywen.com/library/nginx:1.7.9   app=nginx
  
  NAME              READY   STATUS      RESTARTS   AGE    IP             NODE         NOMINATED NODE   READINESS GATES
  pod/nginx-5b5nk   1/1     Running     0          12s    10.244.1.124   k8s-node02   <none>           <none>
  pod/nginx-9f4zp   1/1     Running     0          12s    10.244.2.131   k8s-node01   <none>           <none>
  pod/nginx-gq2jl   1/1     Running     0          12s    10.244.2.130   k8s-node01   <none>           <none>
  ```

  检查ReplicationController状态：

  ```shell
  kubectl describe replicationcontrollers/nginx
  ```

  ```shell
  [root@k8s-master01 02]# kubectl describe replicationcontrollers/nginx
  Name:         nginx
  Namespace:    default
  Selector:     app=nginx
  Labels:       app=nginx
  Annotations:  <none>
  Replicas:     3 current / 3 desired
  Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
  Pod Template:
    Labels:  app=nginx
    Containers:
     nginx:
      Image:        hub.gerrywen.com/library/nginx:1.7.9
      Port:         80/TCP
      Host Port:    0/TCP
      Environment:  <none>
      Mounts:       <none>
    Volumes:        <none>
  Events:
    Type    Reason            Age   From                    Message
    ----    ------            ----  ----                    -------
    Normal  SuccessfulCreate  113s  replication-controller  Created pod: nginx-gq2jl
    Normal  SuccessfulCreate  113s  replication-controller  Created pod: nginx-5b5nk
    Normal  SuccessfulCreate  113s  replication-controller  Created pod: nginx-9f4zp
  [root@k8s-master01 02]# 
  ```

  列出属于ReplicationController的所有pod：

  ```shell
  pods=$(kubectl get pods --selector=app=nginx --output=jsonpath={.items..metadata.name}) echo $pods
  ```

  ```shell
  kubectl get pods --selector=app=nginx -o jsonpath="{.items..metadata.name}"; echo
  ```

  

##### 删除ReplicationController及其Pods

使用[kubectl delete](https://www.kubernetes.org.cn/doc-60)命令删除ReplicationController及其所有pod。

当使用REST API或客户端库时，需要明确地执行这些步骤（将副本缩放为0，等待pod删除，然后删除ReplicationController）。

```shell
kubectl delete -f replication.yaml 
```

```shell
[root@k8s-master01 02]# kubectl delete -f replication.yaml 
replicationcontroller "nginx" deleted
[root@k8s-master01 02]# 
[root@k8s-master01 02]# 
[root@k8s-master01 02]# kubectl get pod
NAME          READY   STATUS        RESTARTS   AGE
nginx-5b5nk   0/1     Terminating   0          8m22s
nginx-9f4zp   0/1     Terminating   0          8m22s
nginx-gq2jl   0/1     Terminating   0          8m22s
```



### RC 替代方法

##### ReplicaSet

[ReplicaSet](http://docs.kubernetes.org.cn/314.html)是支持新的set-based选择器要求的下一代ReplicationController 。它主要用作[Deployment](http://docs.kubernetes.org.cn/317.html)协调pod创建、删除和更新。请注意，除非需要自定义更新编排或根本不需要更新，否则建议使用Deployment而不是直接使用ReplicaSets。



##### Deployment（推荐）

[Deployment](http://docs.kubernetes.org.cn/317.html)是一个高级的API对象，以类似的方式更新其底层的副本集和它们的Pods kubectl rolling-update。如果您希望使用这种滚动更新功能，建议您进行部署，因为kubectl rolling-update它们是声明式的，服务器端的，并具有其他功能。



##### Bare Pods

与用户直接创建pod的情况不同，ReplicationController会替换由于某些原因而被删除或终止的pod，例如在节点故障或中断节点维护（例如内核升级）的情况下。因此，即使应用只需要一个pod，我们也建议使用ReplicationController。









