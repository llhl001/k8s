# Bookinfo 应用

这个示例部署了一个用于演示多种 Istio 特性的应用，该应用由四个单独的微服务构成。 这个应用模仿在线书店的一个分类，显示一本书的信息。 页面上会显示一本书的描述，书籍的细节（ISBN、页数等），以及关于这本书的一些评论。

Bookinfo 应用分为四个单独的微服务：

- `productpage`. 这个微服务会调用 `details` 和 `reviews` 两个微服务，用来生成页面。
- `details`. 这个微服务中包含了书籍的信息。
- `reviews`. 这个微服务中包含了书籍相关的评论。它还会调用 `ratings` 微服务。
- `ratings`. 这个微服务中包含了由书籍评价组成的评级信息。

`reviews` 微服务有 3 个版本：

- v1 版本不会调用 `ratings` 服务。
- v2 版本会调用 `ratings` 服务，并使用 1 到 5 个黑色星形图标来显示评分信息。
- v3 版本会调用 `ratings` 服务，并使用 1 到 5 个红色星形图标来显示评分信息。

![image-20200612142422748](/static/istio/images/image-20200612142422748.png)

Bookinfo 应用中的几个微服务是由不同的语言编写的。 这些服务对 Istio 并无依赖，但是构成了一个有代表性的服务网格的例子：它由多个服务、多个语言构成，并且 `reviews` 服务具有多个版本。



### 开始之前

如果您还没有开始，请遵循[安装指南](https://istio.io/latest/zh/docs/setup/)完成 Istio 的部署工作。



### 部署应用

要在 Istio 中运行这一应用，无需对应用自身做出任何改变。 您只要简单的在 Istio 环境中对服务进行配置和运行，具体一点说就是把 Envoy sidecar 注入到每个服务之中。 最终的部署结果将如下图所示：

![image-20200612142559240](/static/istio/images/image-20200612142559240.png)

所有的微服务都和 Envoy sidecar 集成在一起，被集成服务所有的出入流量都被 sidecar 所劫持，这样就为外部控制准备了所需的 Hook，然后就可以利用 Istio 控制平面为应用提供服务路由、遥测数据收集以及策略实施等功能。



### 启动应用服务

> 如果运行的是 GKE，请确您的集群具有至少四个标准 GKE 节点。如果使用的是 Minikube，应该有 4G 以上的内存。

1. 进入 Istio 安装目录。

2. Istio 默认[自动注入 Sidecar](https://istio.io/latest/zh/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection). 请为 `default` 命名空间打上标签 `istio-injection=enabled`：

   ```shell
   $ kubectl label namespace default istio-injection=enabled
   ```

3. 使用 `kubectl` 部署应用：

   ``` shell
   $ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
   ```

   > 如果您在安装过程中禁用了 Sidecar 自动注入功能而选择[手动注入 Sidecar](https://istio.io/latest/zh/docs/setup/additional-setup/sidecar-injection/#manual-sidecar-injection)，请在部署应用之前使用 [`istioctl kube-inject`](https://istio.io/latest/zh/docs/reference/commands/istioctl/#istioctl-kube-inject) 命令修改 `bookinfo.yaml` 文件。
   >
   > ```shell
   > $  kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)
   > ```

   上面的命令会启动全部的四个服务，其中也包括了 reviews 服务的三个版本（v1、v2 以及 v3）。

   > 在实际部署中，微服务版本的启动过程需要持续一段时间，并不是同时完成的

4. 确认所有的服务和 Pod 都已经正确的定义和启动：

   ```shell
   $ kubectl get services
   NAME          TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
   details       ClusterIP   10.0.0.31    <none>        9080/TCP   6m
   kubernetes    ClusterIP   10.0.0.1     <none>        443/TCP    7d
   productpage   ClusterIP   10.0.0.120   <none>        9080/TCP   6m
   ratings       ClusterIP   10.0.0.15    <none>        9080/TCP   6m
   reviews       ClusterIP   10.0.0.170   <none>        9080/TCP   6m
   ```

   还有

   ```shell
   $ kubectl get pods
   NAME                             READY     STATUS    RESTARTS   AGE
   details-v1-1520924117-48z17      2/2       Running   0          6m
   productpage-v1-560495357-jk1lz   2/2       Running   0          6m
   ratings-v1-734492171-rnr5l       2/2       Running   0          6m
   reviews-v1-874083890-f0qf0       2/2       Running   0          6m
   reviews-v2-1343845940-b34q5      2/2       Running   0          6m
   reviews-v3-1813607990-8ch52      2/2       Running   0          6m
   ```

5. 要确认 Bookinfo 应用是否正在运行，请在某个 Pod 中用 `curl` 命令对应用发送请求，例如 `ratings`：

   ```shell
   $ kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
   <title>Simple Bookstore App</title>
   ```

### 确定 Ingress 的 IP 和端口

现在 Bookinfo 服务启动并运行中，您需要使应用程序可以从外部访问 Kubernetes 集群，例如使用浏览器。可以用 [Istio Gateway](https://istio.io/latest/zh/docs/concepts/traffic-management/#gateways) 来实现这个目标。

 1. 为应用程序定义 Ingress 网关：

    ```shell
    $ kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
    ```

 2. 确认网关创建完成：

    ```shell
    $ kubectl get gateway
    NAME               AGE
    bookinfo-gateway   32s
    ```

	3. 根据[文档](https://istio.io/latest/zh/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-i-p-and-ports)设置访问网关的 `INGRESS_HOST` 和 `INGRESS_PORT` 变量。确认并设置。

	4. 设置 `GATEWAY_URL`：

    ```shell
    $ export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
    ```

### 确认可以从集群外部访问应用

可以用 `curl` 命令来确认是否能够从集群外部访问 Bookinfo 应用程序：

```shell
$ curl -s http://${GATEWAY_URL}/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>
```

还可以用浏览器打开网址 `http://$GATEWAY_URL/productpage`，来浏览应用的 Web 页面。如果刷新几次应用的页面，就会看到 `productpage` 页面中会随机展示 `reviews` 服务的不同版本的效果（红色、黑色的星形或者没有显示）。`reviews` 服务出现这种情况是因为我们还没有使用 Istio 来控制版本的路由。



### 应用默认目标规则

在使用 Istio 控制 Bookinfo 版本路由之前，您需要在[目标规则](https://istio.io/latest/zh/docs/concepts/traffic-management/#destination-rules)中定义好可用的版本，命名为 *subsets* 。

运行以下命令为 Bookinfo 服务创建的默认的目标规则：

- 如果**没有**启用双向 TLS，请执行以下命令：

  > 如果您是 Istio 的新手，并且使用了 `demo` [配置文件](https://istio.io/latest/zh/docs/setup/additional-setup/config-profiles/)，请选择此步。

  ```shell
  $ kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
  ```

- 如果**启用了**双向 TLS，请执行以下命令：

  ```shell
  $ kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
  ```

等待几秒钟，以使目标规则生效。

您可以使用以下命令查看目标规则：

```shell
$ kubectl get destinationrules -o yaml
```
