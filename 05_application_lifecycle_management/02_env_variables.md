- command: `kubectl run nginx --image=nginx --command -- sleep 1000`  
- arguments: `kubectl run nginx --image=nginx -- 1200`  
- [configuration](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/)  
    ``` yaml
    apiVersion: v1
    kind: Pod
    metadata:
    name: command-demo
    labels:
        purpose: demonstrate-command
    spec:
    containers:
    - name: command-demo-container
        image: debian
        command: ["printenv"]
        args: ["HOSTNAME", "KUBERNETES_PORT"]
    restartPolicy: OnFailure
    ```