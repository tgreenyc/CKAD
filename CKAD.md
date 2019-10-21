
### Core Concepts

deploy to a namespace using -n argument. no yaml attribute available for this.

`kubectl run hazelcast --image=hazelcast --env="DNS_DOMAIN=cluster" --env="POD_NAMESPACE=default”`  

`kubectl create quota myrq --hard=cpu=1,memory=1G,pods=2`

`kubectl replace -f nginx-deploy.yaml`

```yaml
command: ["nginx"]  
args: ["-g", "daemon off;"]
```

### Labels and Annotations

`kubectl run hazelcast --image=hazelcast --labels="app=hazelcast,env=prod”`

`kubectl get po --show-labels`

`kubectl label po foo unhealthy=true`

`kubectl label po --all status=unhealthy` (add `--overwrite` if label already exists)

`kubectl label po foo bar-`

`kubectl get po -L app`

`kubectl get po -L app -l app=v2`

```yaml
# Selector which must match a node's labels for the pod to be scheduled on that node.
spec:
  containers:
  - name: nvidia
    image: tesla
  nodeSelector:
	accelerator: nvidia-tesla-p100
```

Taints – they allow a node to repel a set of pods.

`kubectl taint nodes node1 key=value:NoSchedule`

Values: `NoSchedule`, `PreferNoSchedule`, `NoExecute`

You specify a toleration for a pod in the PodSpec. The following toleration matches the taint created by the kubectl taint line above, and thus a pod with either toleration would be able to schedule onto node1:

```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

**Note:** Taint tells a node to only accept specific Tolerations.  If you want to restrict a Pod to a specific Node, then use Pod Affinity.

**Note:**  Node Selectors don’t support more complex logic like Large OR Medium, NOT Small, etc.  Next section on Node Affinity and Anti Affinity provides a solution to this.

If `nodeAffinity` is on the exam, just look it up in the docs (very verbose).

Annotations are key/value pairs that can be larger than labels and include arbitrary string values such as structured JSON.
(view them using `kubectl describe po`)

`kubectl annotate po nginx1 nginx2 nginx3 description='my description’`

### Deployments

`kubectl rollout status deploy/nginx`

Set a deployment's nginx container image to 'nginx:1.9.1'
`kubectl set image deploy/nginx nginx=nginx:1.9.1` (note, `kubectl edit` works here also)

`kubectl rollout history deploy/nginx` (`--revision=3` to check revision 3)

Rollback to the previous deployment
`kubectl rollout undo deploy/nginx` (`--to-revision=3` to specify version 3)

`kubectl autoscale deploy nginx --min=5 --max=10 --cpu-percent=80`

`kubectl rollout pause deploy/nginx`

`kubectl rollout resume deploy/nginx`

`kubectl get deploy foo -o yaml --export`

```yaml
# Important Job properties
spec:
	completions: 5
	parallelism: 5
```
  
`kubectl run cronjob --image=busybox --schedule="*/1 * * * *" --restart=OnFailure --command -- /bin/sh -c 'date;echo Hello from the Kubernetes Cluster'`

### Configuration

`kubectl create cm config --from-literal=foo=lala --from-literal=foo2=lolo` (or `--from-file` or `--from-env-file`)

`--from-file=special=config.txt` (change the key)

Using `--from-file` makes sense if you are going to load the configuration directly from a file on a volume.

Using `--from-env-file` is useful if you want to inject configuration directly into environment vars.

```yaml
# set variables per container
    env:
    - name: option # name of the env variable
      valueFrom:
        configMapKeyRef:
          name: options # name of config map
          key: var5 # name of the entity in config map
```

Use `envFrom` to define all of the ConfigMap’s data as container environment variables. The key from the ConfigMap becomes the environment variable name in the Pod.
  
```yaml
# again, set variables per container
    envFrom: # different than previous one, that was 'env'
    - configMapRef: # different from the previous one, was 'configMapKeyRef'
        name: anotherone # the name of the config map
```

```yaml
    envFrom:
      - secretRef:
          name: test-secret
```
  
```yaml
  volumes: # add a volumes list
  - name: myvolume # just a name, you'll reference this in the pods
    configMap:
      name: cmvolume # name of your configmap
  containers:
  - name: nginx
    volumeMounts: # your volume mounts are listed here
    - name: myvolume # the name that you specified in pod.spec.volumes.name
      mountPath: /etc/lala # the path inside your container
```
  
```yaml
# apply to container or pod
  securityContext: # insert this line
    runAsUser: 101 # UID for the user
```
  
```yaml
# apply to a container ONLY
    securityContext: # insert this line
      capabilities: # and this
        add: ["NET_ADMIN", "SYS_TIME"] # this as well
```
  

`kubectl run busybox --image=busybox —restart=Never --limits='cpu=200m,memory=512Mi' --requests='cpu=100m,memory=256Mi' --dry-run -oyaml`

`kubectl create secret generic mysecret --from-literal=password=mypass (or --from-file=filename)`

```yaml
  volumes: # specify the volumes
  - name: foo # this name will be used for reference inside the container
    secret: # we want a secret
      secretName: mysecret2 # name of the secret - this must already exist on pod creation
  containers:
  - name: nginx
    volumeMounts: # our volume mounts
    - name: foo # name on pod.spec.volumes
      mountPath: /etc/foo #our mount path
```
  
```yaml
    env: # our env variables
    - name: USERNAME # asked name
      valueFrom:
        secretKeyRef: # secret reference
          name: mysecret2 # our secret's name
          key: username # the key of the data in the secret
```
  
```yaml
# use ‘myuser’ service account for Pod
spec:
	serviceAccountName: myuser # we use pod.spec.serviceAccountName
```
  

### Observability:

  
```yaml
    livenessProbe: # our probe
      exec: # add this line
        command: # command definition
        - ls # ls command
```
 
```yaml
    livenessProbe: 
      initialDelaySeconds: 5 # add this line
      periodSeconds: 10 # add this line as well
```

```yaml
    readinessProbe: # declare the readiness probe
      httpGet: # add this line
        path: / #
        port: 80 #
```
events are also useful for debugging
`kubectl get events`

force kill a pod
`kubectl delete po busybox --force --grace-period=0`
  

### Services and Networking

A Service is backed by a group of Pods. These Pods are exposed through endpoints.

For `Ingress` resources, look it up in the documentation.  It's very verbose syntax.

Note: `kubectl expose` does not allow you to specify a NodePort.  Open up the yaml and add:

```yaml
  - port: 30080
    protocol: TCP
    targetPort: 80
    nodePort: 30080 # This MUST be added separately
``` 

Show pods that back specific service endpoints:
`kubectl get ep # endpoints`

`kubectl expose deploy foo --port=6262 --target-port=8080`

`--name=foo-service` is optional, but allows you to override the default service name, which in the above command would be `foo`.

For a NodePort, use the following:
`kubectl expose deploy foo --port=6262 --type=NodePort`
Then add `nodePort: <port>` to the `ports:` section of the yaml service definition.

Also, `ClusterIP` is the default `type` when expose is run.

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: access-nginx # pick a name
spec:
  podSelector:
    matchLabels:
      run: nginx # selector for the pods
  ingress: # allow ingress traffic
  - from:
    - podSelector: # from pods
        matchLabels: # with this label
          access: 'true' # 'true' *needs* quotes in YAML, apparently
```

### State Persistence

```yaml
  volumes: #
  - name: myvolume #
    emptyDir: {} #
```

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: myvolume
spec:
  storageClassName: normal
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
    - ReadWriteMany
  hostPath:
    path: /etc/foo
    type: DirectoryOrCreate # if you also need to create the directory on the host
```

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mypvc
spec:
  storageClassName: normal
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
```

```yaml
  volumes:
  - name: myvolume
    persistentVolumeClaim:
      claimName: mypvc
```

`kubectl cp busybox:/etc/passwd ./passwd # kubectl cp command`

### Other Tips

```yaml
# Run before the app containers are started.
spec:
  initContainers:
  - name: init
	image: drupal:8.6
```

you can target multiple objects in a single command (e.g. `kubectl delete po nginx1 nginx2 nginx3` or `nginx{1..3}`)

aliases

-   quota - resourcequota
-   hpa - horizontal pod autoscaler* (`kubectl get hpa`)
-   cj - cronjobs
-   cm - configmap
-   sa - serviceaccount
-   pv - persistentvolume (`kubectl get pv`)
-   netpol - networkpolicy

You can inject environment variables and secrets **into** a volume.
