## Manage Secrets In Kubernetes Using Vault And External Secrets

### Prepare node 
* NFS-Provisioner Install In Kubernetes
* Vault Install In Kubernetes
* External Secret Install In Kubernetes

#Install NFS-Provisioner In Kubernetes

#Install Vault In Kubernetes
- Clone the Vault Repository
```
git clone 
```
- Traverse To Vault Repo
```
cd vault
```
-Install the RBAC for the Vault
 _**ClusterRoles**: Kubernetes ClusterRoles are entities that have been assigned certain special permissions.
 _**ServiceAccounts**: Kubernetes ServiceAccounts are identities assigned to entities such as pods to enable their interaction with the Kubernetes APIs using the role’s permissions.
 _**ClusterRoleBindings**: ClusterRoleBindings are entities that provide roles to accounts i.e. they grant permissions to service accounts.

```
#kubectl create ns vault

#cat rbac.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault
  namespace: vault

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vault-server-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: vault
  namespace: vault

# kubectl create -f rbac.yaml
```

- Create Vault ConfigMaps
```
#cat configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: vault-config
  namespace: vault
data:
  extraconfig-from-values.hcl: |-
    disable_mlock = true
    ui = true
    
    listener "tcp" {
      tls_disable = 1
      address = "[::]:8200"
      cluster_address = "[::]:8201"
    }
    storage "file" {
      path = "/vault/data"
    }

#kubectl apply -f configmap.yaml
```
_**disable_mlock**: Executing mlock syscall prevents memory from being swapped to
_**disk**: This option disables the server from executing the mlock syscall. 
_**ui**: Enables the built-in web UI.
_**listener**: Configures how Vault is listening for API requests.
_**storage**: Configures the storage backend where Vault data is stored. 

- Deploy Vault Services

**Services in Kubernetes are the objects that pods use to communicate with each other. ClusterIP type services are usually used for inter-pod communication.**

> There are two types of ClusterIP services
_**Headless Services**
_**Services**

Normal Kubernetes services act as load balancers and follow round-robin logic to distribute loads. Headless services don’t act like load balancers.
Also, normal services are assigned IPs by Kubernetes whereas Headless services are not.
For the vault server, we will create a headless service for internal usage. It will be very useful when we scale the vault to multiple replicas.
A non-headless service will be created for UI as we want to load balance requests to the replicas when accessing the UI.
Vault exposes its UI at port 8200. We will use a non-headless service of type NodePort as we want to access this endpoint from outside Kubernetes Cluster.

```
#cat services.yaml
---
# Service for Vault Server
apiVersion: v1
kind: Service
metadata:
  name: vault
  namespace: vault
  labels:
    app.kubernetes.io/name: vault
    app.kubernetes.io/instance: vault
  annotations:
spec:
  type: LoadBalancer  
  publishNotReadyAddresses: true
  ports:
    - name: http
      port: 8200
      targetPort: 8200
      nodePort: 32000
    - name: https-internal
      port: 8201
      targetPort: 8201
  selector:
    app.kubernetes.io/name: vault
    app.kubernetes.io/instance: vault
    component: server

---
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: vault-internal
  namespace: vault
  labels:
    app.kubernetes.io/name: vault
    app.kubernetes.io/instance: vault
  annotations:
spec:
  clusterIP: None
  publishNotReadyAddresses: true
  ports:
    - name: "http"
      port: 8200
      targetPort: 8200
    - name: https-internal
      port: 8201
      targetPort: 8201
  selector:
    app.kubernetes.io/name: vault
    app.kubernetes.io/instance: vault
    component: server

#kubectl apply -f services.yaml
```

- Deploy Vault StatefulSet
```
#cat statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: vault
  namespace: vault
  labels:
    app.kubernetes.io/name: vault
    app.kubernetes.io/instance: vault
spec:
  serviceName: vault-internal
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: vault
      app.kubernetes.io/instance: vault
      component: server
  template:
    metadata:
      labels:
        app.kubernetes.io/name: vault
        app.kubernetes.io/instance: vault
        component: server
    spec:
      serviceAccountName: vault
      securityContext:
        runAsNonRoot: true
        runAsGroup: 1000
        runAsUser: 100
        fsGroup: 1000
      volumes:
        - name: config
          configMap:
            name: vault-config
        - name: home
          emptyDir: {}
      containers:
        - name: vault          
          image: hashicorp/vault:1.8.0
          imagePullPolicy: IfNotPresent
          command:
          - "/bin/sh"
          - "-ec"
          args: 
          - |
            cp /vault/config/extraconfig-from-values.hcl /tmp/storageconfig.hcl;
            [ -n "${HOST_IP}" ] && sed -Ei "s|HOST_IP|${HOST_IP?}|g" /tmp/storageconfig.hcl;
            [ -n "${POD_IP}" ] && sed -Ei "s|POD_IP|${POD_IP?}|g" /tmp/storageconfig.hcl;
            [ -n "${HOSTNAME}" ] && sed -Ei "s|HOSTNAME|${HOSTNAME?}|g" /tmp/storageconfig.hcl;
            [ -n "${API_ADDR}" ] && sed -Ei "s|API_ADDR|${API_ADDR?}|g" /tmp/storageconfig.hcl;
            [ -n "${TRANSIT_ADDR}" ] && sed -Ei "s|TRANSIT_ADDR|${TRANSIT_ADDR?}|g" /tmp/storageconfig.hcl;
            [ -n "${RAFT_ADDR}" ] && sed -Ei "s|RAFT_ADDR|${RAFT_ADDR?}|g" /tmp/storageconfig.hcl;
            /usr/local/bin/docker-entrypoint.sh vault server -config=/tmp/storageconfig.hcl     
          securityContext:
            allowPrivilegeEscalation: false
          env:
            - name: HOSTNAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: VAULT_ADDR
              value: "http://127.0.0.1:8200"
            - name: VAULT_API_ADDR
              value: "http://$(POD_IP):8200"
            - name: SKIP_CHOWN
              value: "true"
            - name: SKIP_SETCAP
              value: "true"
            - name: VAULT_CLUSTER_ADDR
              value: "https://$(HOSTNAME).vault-internal:8201"
            - name: HOME
              value: "/home/vault"
          volumeMounts:
            - name: data
              mountPath: /vault/data  
            - name: config
              mountPath: /vault/config
            - name: home
              mountPath: /home/vault
          ports:
            - containerPort: 8200
              name: http
            - containerPort: 8201
              name: https-internal
            - containerPort: 8202
              name: http-rep
          readinessProbe:
            exec:
              command: ["/bin/sh", "-ec", "vault status -tls-skip-verify"]
            failureThreshold: 2
            initialDelaySeconds: 5
            periodSeconds: 5
            successThreshold: 1
            timeoutSeconds: 3
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
             storage: 1Gi


#kubectl apply -f statefulset.yaml
```

- Unseal & Initialise Vault
```
#kubectl exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > keys.json

#VAULT_UNSEAL_KEY=$(cat keys.json | jq -r ".unseal_keys_b64[]")
#echo $VAULT_UNSEAL_KEY

#VAULT_ROOT_KEY=$(cat keys.json | jq -r ".root_token")
#echo $VAULT_ROOT_KEY

#kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
```

- Login & Access Vault UI
```
#kubectl exec vault-0 -- vault login $VAULT_ROOT_KEY
```

- Creating Vault Secrets
```
#kubectl exec -it vault-0 -- /bin/sh
```

- Create secrets
```
#vault secrets enable -version=2 -path="demo-app" kv
#vault kv put demo-app/user01 name=devopscube
#vault kv get demo-app/user01
```

- Create a Policy
```
#vault policy write demo-policy - <<EOH
path "demo-app/*" {
  capabilities = ["read"]
}
EOH

#vault policy list
```

- Enable Vault Kubernetes Authentication Method
```
#vault auth enable kubernetes
#vault write auth/kubernetes/config token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
#exit
```
- Create ServiceAccount 
```
#kubectl create serviceaccount vault-auth
```
- Login Back to Vault Pod
```
#kubectl exec vault-0 -- vault login $VAULT_ROOT_KEY
#vault write auth/kubernetes/role/webapp \
        bound_service_account_names=vault-auth \
        bound_service_account_namespaces=default \
        policies=demo-policy \
        ttl=72h
```
- Deploy A Pod
```
#cat pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: vault-client
  namespace: default
spec:
  containers:
  - image: nginx:latest
    name: nginx
  serviceAccountName: vault-auth

#kubectl create -f pod.yaml

#kubectl exec -it vault-client /bin/bash

#jwt_token=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

#echo $jwt_token
```

- Install External Secret Operator
```
#helm repo add external-secrets https://charts.external-secrets.io
#helm install external-secrets \
   external-secrets/external-secrets \
    -n external-secrets \
    --create-namespace 

#kubectl get po -n external-secrets
```

- Clone The Repository
```
#git clone

# cd ExternalSecrets

#cat cluster-secret-store.yaml
apiVersion: external-secrets.io/v1alpha1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://vault-server-internal.vault:8200"
      path: "secret"
      version: "v1"
      auth:
        tokenSecretRef:
          name: "vault-token"
          key: "token"
          namespace: vault

#echo -n "Vault Secret Token" |base64

#cat vault-token-secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: vault-token
  namespace: vault
data:
  token: <base64-generated-token"

```
- Apply The Files
```
#kubectl apply -f cluster-secret-store.yaml
#kubectl apply -f vault-token-secret.yaml
```

- Create External Secret
```
#cat external-pullsecret-cluster.yaml
apiVersion: external-secrets.io/v1alpha1
kind: ExternalSecret
metadata:
  name: pullsecret-cluster-sno01
  namespace: sno01
spec:
  refreshInterval: "15s"
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: pullsecret-cluster-sno01
  data:
  - secretKey: .dockerconfigjson
    remoteRef:
      key: secret/pullsecret
      property: dockerconfigjson

#kubectl apply -f external-pullsecret-cluster.yaml
```
- Login to Vault Pod
```
#kubectl exec vault-0 -- vault login $VAULT_ROOT_KEY
#vault secrets enable -path=secret/ kv
#vault kv put secret/pullsecret dockerconfigjson='{"auths":{"cloud.openshift.com":{"auth":"3BlbnNoaWZ0LXJl==","email":"example@redhat.com"},"quay.io":{"auth":"ZZMVhJRUJUR1I3WUwxN05VMQ==","email":"example@redhat.com"},"registry.connect.redhat.com":{"auth":"3BlbnNoaWZ0LXJl==","email":"example@redhat.com"},"registry.redhat.io":{"auth":"==","email":"example@redhat.com"}}}'
```
- Apply The Pull-Secret
```
#kubectl apply -f apply -f external-pullsecret-cluster.yaml
```

- Validation
```
#kubectl get externalsecret -n default pullsecret-cluster-sno01
#kubectl get secrets -n default pullsecret-cluster-sno01
#kubectl secrets -n default pullsecret-cluster-sno01 -o yaml 
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJjbG91ZC5vcGVuc2hpZnQuY29tIjp7ImF1dGgiOiIzQmxibk5vYVdaMExYSmw9PSIsImVtYWlsIjoiZXhhbXBsZUByZWRoYXQuY29tIn0sInF1YXkuaW8iOnsiYXV0aCI6IlpaTVZoSlJVSlVSMUkzV1V3eE4wNVZNUT09IiwiZW1haWwiOiJleGFtcGxlQHJlZGhhdC5jb20ifSwicmVnaXN0cnkuY29ubmVjdC5yZWRoYXQuY29tIjp7ImF1dGgiOiIzQmxibk5vYVdaMExYSmw9PSIsImVtYWlsIjoiZXhhbXBsZUByZWRoYXQuY29tIn0sInJlZ2lzdHJ5LnJlZGhhdC5pbyI6eyJhdXRoIjoiPT0iLCJlbWFpbCI6ImV4YW1wbGVAcmVkaGF0LmNvbSJ9fX0=

#echo -n "eyJhdXRocyI6eyJjbG91ZC5vcGVuc2hpZnQuY29tIjp7ImF1dGgiOiIzQmxibk5vYVdaMExYSmw9PSIsImVtYWlsIjoiZXhhbXBsZUByZWRoYXQuY29tIn0sInF1YXkuaW8iOnsiYXV0aCI6IlpaTVZoSlJVSlVSMUkzV1V3eE4wNVZNUT09IiwiZW1haWwiOiJleGFtcGxlQHJlZGhhdC5jb20ifSwicmVnaXN0cnkuY29ubmVjdC5yZWRoYXQuY29tIjp7ImF1dGgiOiIzQmxibk5vYVdaMExYSmw9PSIsImVtYWlsIjoiZXhhbXBsZUByZWRoYXQuY29tIn0sInJlZ2lzdHJ5LnJlZGhhdC5pbyI6eyJhdXRoIjoiPT0iLCJlbWFpbCI6ImV4YW1wbGVAcmVkaGF0LmNvbSJ9fX0=" | base64 --decode
```

