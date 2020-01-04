# install hadoop in k8s using helm

## command

helm install --name my-hadoop \
  --set persistence.nameNode.enabled=true \
  --set persistence.nameNode.storageClass=standard \
  --set persistence.nameNode.size=100Gi \
  --set persistence.dataNode.enabled=true \
  --set persistence.dataNode.storageClass=standard \
  --set persistence.dataNode.size=100Gi \
  stable/hadoop