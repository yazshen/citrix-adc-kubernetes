# LAB04: 两层式负载均衡部署模式(Dual Tier)

## 更新时间

2021.02.28

## 1. 实验拓扑

![](./images/101-lab04-k8s-dual-tier-topology.png)

## 2. Demo(1): 通过Dual Tier模式部署一个Nginx应用

SSH登录到Master节点，然后查看yaml配置文件"citrix101-lab04-demo01.yaml"并部署

```
cat citrix101-lab04-demo01.yaml
kubectl apply -f citrix101-lab04-demo01.yaml
```

![](./images/101-lab04-k8s-dual-tier-01.png)

查看pod和deployment运行状态

```
kubectl get pods -o wide -l app=citrix101-lab04-demo01
kubectl get deployments -o wide -l app=citrix101-lab04-demo01
```

![](./images/101-lab04-k8s-dual-tier-02.png)

部署新的Service，通过ClusterIP方式实现集群内访问

```
kubectl expose deployment citrix101-lab04-demo01 --type=ClusterIP --name=citrix101-lab04-demo01-clusterip
kubectl get svc -o wide
```

![](./images/101-lab04-k8s-dual-tier-03.png)

查看并部署ingress的yaml配置文件"citrix101-lab04-demo01-ingress-cpx.yaml"，注意"kubernetes.io/ingress.class"为"citrix-cic-cpx"，此ingress用于Tier-2的CPX设备实现流量管理

```
cat citrix101-lab04-demo01-ingress-cpx.yaml
kubectl apply -f citrix101-lab04-demo01-ingress-cpx.yaml
kubectl get ingress -o wide
```

![](./images/101-lab04-k8s-dual-tier-04.png)

查看CPX和CIC的yaml配置文件，注意这个CIC使用了sidecar方式部署并且不需要认证信息

```
cat citrix101-lab04-demo01-cic-k8s-ingress-controller-cpx.yaml
```

![](./images/101-lab04-k8s-dual-tier-05.png)

部署这个yaml文件，然后查看pod, deployment, service运行状态

```
kubectl apply -f citrix101-lab04-demo01-cic-k8s-ingress-controller-cpx.yaml
kubectl get deployments -o wide -l app=citrix101-lab04-demo01-cpx-ingress
kubectl get pods -o wide -l app=citrix101-lab04-demo01-cpx-ingress
kubectl get svc -o wide -l app=citrix101-lab04-demo01-cpx-service
```

![](./images/101-lab04-k8s-dual-tier-06.png)

查看并部署ingress的yaml配置文件"citrix101-lab04-demo01-ingress-vpx.yaml"，注意"kubernetes.io/ingress.class"为"citrix-cic-vpx"，此ingress用于Tier-1的VPX设备实现流量管理

```
cat citrix101-lab04-demo01-ingress-vpx.yaml
kubectl apply -f citrix101-lab04-demo01-ingress-vpx.yaml
kubectl get ingress -o wide
```

![](./images/101-lab04-k8s-dual-tier-07.png)

打开Student客户端的C:\Windows\system32\drivers\hosts文件，然后绑定一个静态域名解析"lab04-demo01.example.com"

![](./images/101-lab04-k8s-dual-tier-08.png)

打开Student客户端的浏览器， 然后访问域名: http://lab04-demo01.example.com

![](./images/101-lab04-k8s-dual-tier-09.png)

打开Student客户端的浏览器，登录到Citrix ADC(VPX)管理界面，查看相关配置，发现配置已经自动完成

![](./images/101-lab04-k8s-dual-tier-10.png)

![](./images/101-lab04-k8s-dual-tier-11.png)

![](./images/101-lab04-k8s-dual-tier-12.png)

CPX容器默认使用了随机密码，所以我们先通过访问CPX的Shell方式查询当前默认密码为"d0b9d32700514b99bd4fd023c6942a11"

```
kubectl get pods -o wide -l app=citrix101-lab04-demo01-cpx-ingress
kubectl exec citrix101-lab04-demo01-cpx-ingress-79ff9b8f5c-r77f6 --container citrix101-lab04-demo01-cpx-ingress -- cat /var/deviceinfo/random_id

```

![](./images/101-lab04-k8s-dual-tier-18.png)

通过SSH登录到CPX，然后查看当前配置信息

```
ssh nsroot@10.10.10.198
cli_script.sh "show lb vserver -summary"
cli_script.sh "show cs vserver -summary"
```

![](./images/101-lab04-k8s-dual-tier-19.png)



## 3. Demo(2): 一键配置安全策略 (CRD)

打开Student客户端的浏览器， 然后访问域名: http://lab04-demo01.example.com/test

![](./images/101-lab04-k8s-dual-tier-14.png)

SSH登录到Master节点，然后查看和部署Citrix CRD自定义资源"citrix101-lab04-demo02-crd.yaml"

```
cat citrix101-lab04-demo02-crd.yaml
kubectl apply -f citrix101-lab04-demo02-crd.yaml
kubectl get customresourcedefinitions -o wide
```

![](./images/101-lab04-k8s-dual-tier-13.png)

查看和部署Responder策略“citrix101-lab04-demo02-blacklisturls.yaml"，阻止访问/test目录

```
cat citrix101-lab04-demo02-blacklisturls.yaml
kubectl apply -f citrix101-lab04-demo02-blacklisturls.yaml
```

![](./images/101-lab04-k8s-dual-tier-15.png)

直接使用kubectl命令即可查看自定义资源的运行状态，这也是CRD和ConfigMap最大的区别

```
kubectl get rewritepolicies.citrix.com -o wide
```

![](./images/101-lab04-k8s-dual-tier-16.png)

打开Student客户端的浏览器， 再次访问域名: http://lab04-demo01.example.com/test

![](./images/101-lab04-k8s-dual-tier-17.png)

登录到Citrix ADC(VPX)管理界面，查看Responder相关配置

![](./images/101-lab04-k8s-dual-tier-20.png)

![](./images/101-lab04-k8s-dual-tier-21.png)

