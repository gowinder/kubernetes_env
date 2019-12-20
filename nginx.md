# nginx in k8s

before this you need read helm.md

## install nginx

```shell
#vim nginx-ingress-value.yaml
controller:
  extraArgs:
    v: 2
  service:
    externalIPs:
    - "192.168.1.171"
```

**Note: replace `192.168.1.171` to your load balance ip, here I use master node ip, because I just use it in internal network, you may use a publish ip address in your k8s host.**

```shell
helm install stable/nginx-ingress \
--namespace kube-system --name nginx-ingress \
-f nginx-ingress-value.yaml
```

### test

`curl http://192.168.1.171:80 -vv`, you will get 404, that means ingress is OK.

## install dashboard for k8s

I use k8s.dashboard.local as domain in local,

```shell
helm install stable/kubernetes-dashboard --name dashboard \
--set ingress.annotations."kubernetes\.io/ingress\.class"=nginx \
--set ingress.enabled=true \
--set ingress.hosts[0]="k8s\.dashboard\.local" 
```

if you have problem, u can write a yaml file to overide default value for complex value

```yaml
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: "nginx"
  hosts:
    - k8s.dashboard.local
enableInsecureLogin: true
enableSkipLogin: true
service:
  type: NodeType
```
**Note: I just use it in private network, so I did not need tls**



```shell
helm install stable/kubernetes-dashboard --name dashboard -f dashboard.yaml --debug --dry-run
```

test if ok, than remove `--debug --dry-run`

```shell
helm install stable/kubernetes-dashboard --namespace kube-system --name dashboard -f dashboard.yaml 
```


### add domain to your web browser's hosts file

`192.168.1.171 k8s.dashboard.local`

### generate token for dashboard

#### refer:

`https://jimmysong.io/posts/kubernetes-dashboard-upgrade/`



## basic authoriziation

## tls and let's encrypt support

