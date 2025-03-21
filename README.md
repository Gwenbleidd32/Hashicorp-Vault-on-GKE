# Installing & Using Hashicorp Vault on a GKE Cluster

This configuration guide will walk you through the entire process of installing, configuring, and testing a working Vault instance/instances within your Kubernetes cluster.

You might be wondering: Why integrate Vault with Kubernetes? While Kubernetes provides a built-in way to manage secrets, its default Base64-encoded secrets are not encrypted at rest by default, making them less secure. Deploying your own Vault instance enhances security by encrypting secrets, enforcing strict access controls, and enabling dynamic secret generation.

Vault pods offer the following **key advantages** over Kubernetes Secrets:
- Encryption at rest and transit using AES
- Capable of generating and expiring secrets dynamically. *Including key rotation*
- Supports multiple forms of authentication 
- Granular access policies linked to RBAC and IAM within cluster
- Secret env variables are automatically injected into their associated workloads
- Audit logging of all events

In summary, we are deploying a secure, centralized secrets management solution within our Kubernetes cluster, functioning similarly to an SaaS database for secrets—but with full control over environment variables, credentials, and other sensitive data in a highly secure and automated way.

The documentation I referenced i referenced for this build is in below link: 
https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-sidecar

*Before Beginning*
- Ensure you have a working Kubernetes cluster 
- Ensure you have helm installed on your machine

---
## Installing Vault using Helm

1. After signing into your Kubernetes cluster go ahead and add the hashicorp helm repository to your local helm library. `helm repo add hashicorp https://helm.releases.hashicorp.com`

2. Use `helm repo update` to ensure all of your repositories are up to date.

3. Now use the following command to deploy vault pods to your cluster with the setting for server enabled.
```shell
helm install vault hashicorp/vault --set "server.dev.enabled=true"
```

4. After a Successful installation, Verify all is well using:    

*`kubectl get pods`*
```shell
User@Caranthir MINGW64 /c/terraform/anton/module-practice
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          2m23s
vault-agent-injector-84b987db6f-8kkrh   1/1     Running   0          2m23s
```

*Take note of the vault-agent-injector pod—this is the webhook controller responsible for intercepting pod creation events and mutating them when specific Vault annotations are present. It ensures that secrets stored in Vault are automatically injected into containers that use the corresponding configured service accounts.*

5. Using `kubectl exec vault-0 -- vault status`. We can check the status of any of our vault instances. If you look in the example below we can see that the attributes of `initialized` and `Unseal` both contain negative values. These indicate that our settings for our instances have yet to be configured. In the next group of steps we will ready the vault for usage. 

*Example*
```shell
$ kubectl exec vault-0 -- vault status
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.18.1
Build Date      2024-10-29T14:21:31Z
Storage Type    inmem
Cluster Name    vault-cluster-de590ca9
Cluster ID      cea09880-1a9b-a06f-2a4f-13afd35a2b63
HA Enabled      false
```

---
## Configuring our Vault instance

Now that we've confirmed our vault instance and the injector are running in a healthy state. We can begin the configuration process. We will need to:

- Enable the secret engine within vault
- Create a secret at a specific file path within vault 
- Turn on authentication for Kubernetes
- Create a vault policy with permissions to read the path of our secret 
- Create a Role linking our policy to a Kubernetes service account

### Enabling & creating secrets

1. First we execute or "ssh" into our `vault-0` container

*Command can vary depending on machine type*
```shell 
#Windows
kubectl exec --stdin --tty vault-0 -- sh
# >>>
#Apple
kubectl exec -it vault-0 -- /bin/sh
```

2. Next we create a key/value secret within the vault using our your own choice of file path. We can then specify the key and the value to store in said path. If you are following my example view my terminal below.

*I create a username and password to mimic database credentials.*
```shell
#Input
vault kv put internal/database/config username="pooper" password="scooper"

#Output
======== Secret Path ========
internal/data/database/config

======= Metadata =======
Key                Value
---                -----
created_time       2025-03-21T01:49:21.6230911Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

3. Then we can confirm it's existence: 
```shell
#Input
vault kv get internal/database/config
#Output
======= Metadata =======
Key                Value
---                -----
created_time       2025-03-21T01:49:21.6230911Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

====== Data ======
Key         Value
---         -----
password    scooper
username    pooper
```

### Enabling Kubernetes Authentication & Creating/Linking Vault Policies

1. Next we enable vault authentication with Kubernetes as the method:
```shell
#Input
vault auth enable kubernetes
#Output
Success! Enabled kubernetes auth method at: kubernetes/
```

2. Now we set the **URL of the Kubernetes API server** that Vault will use for token verification.

*I use the default Kubernetes api environment variable from the pod to set my url*
```shell
#Input
vault write auth/kubernetes/config \
      kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
>       kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
#Output
Success! Data written to: auth/kubernetes/config
```

3. Next we create a vault policy with permissions to read our configured secret

*This is akin to a role within IAM*
```shell
#Input
vault policy write internal-app - <<EOF
path "internal/data/database/config" {
   capabilities = ["read"]
}
EOF
#Output
/ $ vault write auth/kubernetes/role/internal-app \
>       bound_service_account_names=internal-app \
>       bound_service_account_namespaces=default \
>       policies=internal-app \
>       ttl=24h
^[[39;3RSuccess! Data written to: auth/kubernetes/role/internal-app
```

### Recap

Now that we've successfully created a Vault secret, enabled the necessary authentication methods, and granted a Kubernetes service account permission to access that secret, we're ready to deploy an application in our cluster. This application can be automatically configured to receive the Vault secret, injected securely into its environment or file system using annotations. *Pretty cool right?*

We need to be aware of a few key factors 
- The service account we tied to our vault policy
- The path of our secret within vault.
 
---
## Deploying an Application with Vault Secret Injection 

Since our Vault instance is configured with a service account that has permissions to access our secret, we can now modify a Kubernetes Deployment or StatefulSet to integrate Vault secret injection.

- We attach a service account matching our vault binding`serviceAccountName: internal-app`
- We define a block of `annotations` that preform the following operations upon container creation:
	- [`agent-inject`](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-sidecar#agent-inject) enables the Vault Agent Injector service
	- [`role`](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-sidecar#role) is the Vault Kubernetes authentication role
	- [`agent-inject-secret-FILEPATH`](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-sidecar#agent-inject-secret-filepath) prefixes the path of the file, `database-config.txt` written to the `/vault/secrets` directory. And sets the value to the Vault secret path.
	
All of these together allow the vault agent to install itself within your deployments pod, Identify and use the role tied to our service account, and find the path of our secret within the vault instance. 

These configurations enable Vault to:

1. Inject a Vault Agent sidecar into the pod.
2. Authenticate using the pod’s service account.
3. Authorize based on the associated Vault role.
4. Retrieve the secret from the specified Vault path.
5. Write the secret into a file within the application container.

With this setup, your application doesn't need to handle secrets directly — Vault securely delivers them into the pod at runtime, adhering to least privilege and secure delivery principles.

1. First we Create our service account. *No Kubernetes based roles are required for this configuration.*  
```shell
kubectl create sa internal-app
```

2. Then we apply our manifest and wait for the container to full initialize
*You can use either the be a man folder or the basic folder to deploy an application to test injection*
```shell
kubectl apply -f be-a-man/
```

3. After a few Moments confirm Health
*Notice how there are 2 containers running per pod.  The second container is the vault agent*
```shell
# Input
kubectl get pods
# Output
NAME                                    READY   STATUS    RESTARTS   AGE
type-a-0                                2/2     Running   0          2m10s
type-a-1                                2/2     Running   0          79s
type-a-2                                2/2     Running   0          40s
vault-0                                 1/1     Running   0          161m
vault-agent-injector-84b987db6f-8kkrh   1/1     Running   0          161m

```
##### When modifying your own applications to use vault secrets...
*Pay attention to the placement of the annotations and the service account. **Remember** the service account token for you deployed pod needs to be attached in order for it to authenticate to vault*

*My be-a-man example*
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: type-a
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: type-a
  updateStrategy:
    type: RollingUpdate # When this configuration is re-applied, pods are gracefully updated with new configuration.
  template:
    metadata:
      labels:
        app: type-a
        istio: monitor
      annotations:
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/role: 'internal-app'
        vault.hashicorp.com/agent-inject-secret-database-config.txt: 'internal/data/database/config'
    spec:
      serviceAccountName: internal-app
      automountServiceAccountToken: true # Token needed for Vault Agent to authenticate
#>>>>>------------------------POD-LEVEL-CONTEXT--------------------------------
# All Volumes bound to this set of pods are owned and only Usable by group or user 5000
#>>>>>-------------------------------------------------------------------------
      securityContext:
        fsGroup: 5000 # Incredible how much this little argument resolves 
        fsGroupChangePolicy: "Always" 
#>>>>>------------------------MAIN-CONTAINER-----------------------------------
      containers:
      - name: giftwrapped-container
        image: europe-central2-docker.pkg.dev/pooper-scooper/run-gmp/milk:v2
        imagePullPolicy: Always
      #>>> PORTS 
        ports:
        - name: http 
          containerPort: 5000  
        - name: http-envoy-prom # prom istio port
          containerPort: 15090 # prom-monitoring default
      #>>> SECURITY-CONTEXT
        securityContext: 
          seccompProfile:
            type: RuntimeDefault 
          readOnlyRootFilesystem: true 
          runAsNonRoot: true # <
          runAsUser: 5000 #
          allowPrivilegeEscalation: false
          capabilities: 
            drop:
            - ALL
      #>>> RESOURCE-REQUESTS
        resources:
          requests: # Minimum resources
            memory: "256Mi"  
            cpu: "250m"            
          limits: # Maximum resources
            memory: "512Mi"      
            cpu: "500m"    
      #>>> STATEFUL-SET MOUNT-PATH
        volumeMounts:          
        - name: orbiter-module
          mountPath: /data
      #>>> OPTIONAL ENVIORNMENT VARIABLE for MOUNT-PATH 
        env:       
        - name: library_path
          value: "/data" 
      #>>> INIT/CONTAINER SCRIPT TO TEST MOUNT PATH AND WRITE ACCCESS.    
        command: ["sh", "-c", "python app.py & while true; do date >> /data/date.txt; sleep 30; done"] 
        # Run this command to test write access: kubectl exec type-a-0 -n lizzoslunch -- sh -c "cat /data/date.txt"
      #>>> LIVENESS AND READINESS PROBES
        readinessProbe:
          httpGet:
            path: /i-am-ready
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 1
        livenessProbe:
          httpGet:
            path: /i-am-alive
            port: 5000
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 1
#>>>>>------------------STATEFULSET-VOLUME-CLAIM-------------------------------
  volumeClaimTemplates:
  - metadata:
      name: orbiter-module
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
      storageClassName: "bordeaux"
```

4. And finally to confirm our secret was injected into our main container we execute on a pod and it's container and read the path we had used to store our secret

*This command can change depending on your machine type*
```shell
# Input
kubectl exec type-a-0 --container giftwrapped-container -- sh -c 'cat /vault/secrets/database-config.txt'
#Output
data: map[password:scooper username:pooper]
metadata: map[created_time:2025-03-21T01:49:21.6230911Z custom_metadata:<nil> deletion_time: destroyed:false version:1]
```

And voilà! 🎉 We've confirmed that our Vault instance is securely injecting configuration data into applications, providing secrets only to those that need them; all in a controlled, authenticated, and secure manner.

Now that we have a way to provide secure data to our applications, you can further process this data into an application specific format of enable other functions such as pod restart upon secret rotation by using additional annotations such as the below. 

*Example of data formatting*
```yaml
template:
   metadata:
      annotations:
      vault.hashicorp.com/agent-inject: 'true'
      vault.hashicorp.com/role: 'internal-app'
      vault.hashicorp.com/agent-inject-secret-database-config.txt: 'internal/data/database/config'
      vault.hashicorp.com/agent-inject-template-database-config.txt: |
         {{- with secret "internal/data/database/config" -}}
         postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard
         {{- end -}}

# Output from exec command:
postgresql://db-readonly-user:db-secret-password@postgres:5432/wizard
```

Feel free to read further into Vault documentation for the different configuration options and operational best practices you can implement within you own environments.

---