# helm

a usefull tool to create instance in k8s like apt in ubuntu

## install helm

`curl https://raw.githubusercontent.com/helm/helm/master/scripts/get | bash`

`helm init`

## add chart

chart is helm's source, like ubuntu's apt repository

this is usefull:
`helm repo add bitnami https://charts.bitnami.com`

## config role

```shell
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule \
--clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

## port foward

need forward tiller port for outside cluster

`kubectl -n kube-system port-forward svc/tiller-deploy 44134:44134`

but this is in forground, you can add it to background:
`nohup kubectl -n kube-system port-forward svc/tiller-deploy 44134:44134 2>&1 &`