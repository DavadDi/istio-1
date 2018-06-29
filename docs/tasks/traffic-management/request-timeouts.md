


> 这个任务使用了新的API： [v1alpha3 traffic management API](/blog/2018/v1alpha3-routing/). 老版本的API已经被弃用了，并会在Istio的下一个发布版本中被移除掉。如果你需要使用旧版本的API，请参考这篇文档：[here](https://archive.istio.io/v0.7/docs/tasks/traffic-management/).

# 设置请求超时

这个任务用于展示如何使用Istio在Envoy中设置请求超时。

## 开始之前

- 遵循[安装指南](../../setup/) 的指导安装Istio。
- 部署[BookInfo](../../guides/bookinfo.md) 示例程序。
- 执行如下命令，初始化基于应用版本的路由。

    ```bash
    istioctl create -f samples/bookinfo/kube/route-rule-all-v1.yaml
    ```

> 注意：本任务假设在Kubernetes上部署这一应用。所有涉及的命令都使用的是Kubernetes版本的yaml文件（例如`samples/bookinfo/kube/route-rule-all-v1.yaml`）。如果要在不同环境运行这一任务，则需要把路径中的`kube`替换成对应的环境（例如`samples/bookinfo/consul/route-rule-all-v1.yaml`）。

## 请求超时

http请求的超时时间可以在路由规则中的`httpReqTimeout`字段来设置。缺省的超时时间是15秒，但在下面的任务中我们会把`reviews`服务的超时时间设置为一秒钟。为了展示效果，我们还需要在调用`ratings`服务的时候加入两秒钟的延迟。

1. 路由请求到`reviews`服务的v2版本，例如，这个版本的`reviews`服务会调用`ratings`服务

    ```bash
    cat <<EOF | istioctl replace -f -
    apiVersion: config.istio.io/v1alpha2
    kind: RouteRule
    metadata:
     name: reviews-default
    spec:
     destination:
       name: reviews
     route:
     - labels:
         version: v2
    EOF
    ```

2. 对调用`ratings`服务的请求加入两秒钟的延迟

    ```bash
    cat <<EOF | istioctl replace -f -
    apiVersion: config.istio.io/v1alpha2
    kind: RouteRule
    metadata:
     name: ratings-default
    spec:
     destination:
       name: ratings
     route:
     - labels:
         version: v1
     httpFault:
       delay:
         percent: 100
         fixedDelay: 2s
    EOF
    ```

3. 在浏览器中打开 BookInfo 的网址（`http://$GATEWAY_URL/productpage`）。

	会看到BookInfo应用在正常运行（并且显示了评级的星星），但是你也会注意到，每次刷新页面的时候会有两秒钟的延迟。

4. 接下来我们增加一个超时时间为1秒中的调用`reviews`服务的请求

    ```bash
    cat <<EOF | istioctl replace -f -
    apiVersion: config.istio.io/v1alpha2
    kind: RouteRule
    metadata:
     name: reviews-default
    spec:
     destination:
       name: reviews
     route:
     - labels:
         version: v2
     httpReqTimeout:
       simpleTimeout:
         timeout: 1s
    EOF
    ```

5. 刷新BookInfo页面

	现在会看到，它会在1秒中时返回结果（而不是之前的两秒钟），但是`reviews`服务不再可用。

## 幕后故事（理解其中发生了什么）

这一任务中，使用Istio设置了超时时间为1秒中的调用`reviews`服务的请求（而不是缺省的 15 秒）。由于`reviews`服务在处理请求的时候会调用`ratings`服务，而我们利用Istio使得`ratings`服务在处理请求时会有2秒中的延迟，这一变动会让`reviews`服务的处理时间超过一秒，故而引发了请求超时。

你会看到BookInfo productpage页面（这个页面的生成需要调用`reviews`服务）不会显示 review信息，而是显示为：“Sorry, product reviews are currently unavailable for this book. ”，这是由于调用`reviews`服务的请求产生了超时错误。

如果检查一下[fault injection task](../../tasks/traffic-management/fault-injection.md)，会发现`productpage`微服务在调用`reviews`服务时，也在应用级别上设置了超时时间（3秒中）。需要注意的是：在前面的任务中，我们使用了Isito的路由规则，将请求的超时时间设置成了1秒中；但如果我们把后面调用的服务的超时时间设置为大于三秒钟的值（例如四秒），由于三秒钟的限制更为严格，会被优先处理，因此过大的超时时间就不会生效了。参看[错误处理](../../concepts/traffic-management/handling-failures.md#faq)一节会有更详细的信息。

还需要注意的就是，除了在路由规则中设置超时之外，还可以在请求中加入“x-envoy-upstream-rq-timeout-ms”头来设置超时，在这一设置中的时间单位是毫秒而不是秒。

## 清理

- 删除应用路由规则。

    ```bash
    istioctl delete -f samples/bookinfo/kube/route-rule-all-v1.yaml
    ```

- 如果没有计划进一步运行下面的任务，可以参照[ Bookinfo cleanup](../../samples/bookinfo.md#cleanup) 中的介绍来关闭这一应用。

## 延伸阅读

- 更多关于[错误处理](../../concepts/traffic-management/handling-failures.md)
- 更多关于[路由规则](../../concepts/traffic-management//rules-configuration.md)
