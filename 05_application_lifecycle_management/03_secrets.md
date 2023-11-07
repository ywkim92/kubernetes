- encode: `echo -n 'echo' | base64`  
- decode: `echo -n 'ZWNobw==' | base64 --decode`  
- create a secret
  ```shell
  kubectl create secret generic db-secret \
    --from-literal=DB_Host=sql01 \
    --from-literal=DB_User=root \
    --from-literal=DB_Password=password123 \
    --dry-run=client -o yaml > db-secret.yaml
  ```
- `envFrom`
    ```yaml
    apiVersion: v1 
    kind: Pod 
    metadata:
    labels:
        name: webapp-pod
    name: webapp-pod
    namespace: default 
    spec:
    containers:
    - image: kodekloud/simple-webapp-mysql
        imagePullPolicy: Always
        name: webapp
        envFrom:
        - secretRef:
            name: db-secret
    ```