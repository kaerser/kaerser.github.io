---
layout:     post
title:      nginx-ingress配置会话保持
subtitle:   
date:       2019-04-30
author:     Kaerser
header-img: img/bg-04.jpg
catalog: false
tags:
    - kubernetes
	- ingress
---

使用nginx-ingress-controller镜像：quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.24.1

``` yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/affinity: cookie
spec:
  rules:
  - host: test.paas.dc.servyou-it.com
    http:
      paths:
      - path: /
        backend:
          serviceName: test-deployment
          servicePort: 8888
```
其中主要需要配置的是annotations，nginx.ingress.kubernetes.io/affinity: cookie

```shell
# kubectl get pods -o wide | grep test-
test-deployment-d7cf5ccbd-kfskl   1/1     Running   0          2d2h    172.30.42.4   10.199.139.105   <none>
test-deployment-d7cf5ccbd-tkn9g   1/1     Running   0          2d2h    172.30.65.5   10.199.139.104   <none>

# kubectl describe svc test-deployment 
Name:              test-deployment
Namespace:         default
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"test-deployment","namespace":"default"},"spec":{"ports":[{"name":...
Selector:          app=test-pod
Type:              ClusterIP
IP:                10.254.133.122
Port:              http  8888/TCP
TargetPort:        80/TCP
Endpoints:         172.30.42.4:80,172.30.65.5:80
Session Affinity:  None
Events:            <none>

# kubectl describe ingresses.extensions test-ingress 
Name:             test-ingress
Namespace:        default
Address:          
Default backend:  default-http-backend:80 (172.30.42.6:8080,172.30.46.4:8080)
Rules:
  Host                         Path  Backends
  ----                         ----  --------
  test.paas.dc.servyou-it.com  
                               /   test-deployment:8888 (<none>)
Annotations:
  kubectl.kubernetes.io/last-applied-configuration:  {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{"nginx.ingress.kubernetes.io/affinity":"cookie"},"name":"test-ingress","namespace":"default"},"spec":{"rules":[{"host":"test.paas.dc.servyou-it.com","http":{"paths":[{"backend":{"serviceName":"test-deployment","servicePort":8888},"path":"/"}]}}]}}

  nginx.ingress.kubernetes.io/affinity:  cookie
Events:
  Type    Reason  Age                  From                      Message
  ----    ------  ----                 ----                      -------
  Normal  CREATE  38m                  ingress-controller        Ingress default/test-ingress
  Normal  CREATE  38m                  ingress-controller        Ingress default/test-ingress
  Normal  UPDATE  37m (x2 over 38m)    ingress-controller        Ingress default/test-ingress
  Normal  UPDATE  37m                  ingress-controller        Ingress default/test-ingress
  Normal  CREATE  32m                  nginx-ingress-controller  Ingress default/test-ingress
  Normal  CREATE  32m                  nginx-ingress-controller  Ingress default/test-ingress
  Normal  UPDATE  4m48s (x4 over 32m)  nginx-ingress-controller  Ingress default/test-ingress
  Normal  UPDATE  4m48s (x3 over 32m)  nginx-ingress-controller  Ingress default/test-ingress
```


测试结果：
![https://kaerser.github.io/img/screenshot/2019-04-30-nginx-ingress-01.png](en-resource://database/1943:0)


结果显示，配置会话保持的请求一直请求至同一pods服务，未配置会话保持的请求会使用轮询的方式至后端pods服务

有关[nginx load balancer详解](http://nginx.org/en/docs/http/load_balancing.html)可点击查看。

nginx ingress官方关于[会话保持详细配置](https://kubernetes.github.io/ingress-nginx/examples/affinity/cookie/)。
