# DevOps

`CIVO-*: free 250 creditg for learning [https://civo.com/thedigitallife](https://www.youtube.com/redirect?event=video_description&redir_token=QUFFLUhqbDl6WXBET1dIcl9ZVFZlU1JoR3JBMXZwUVRld3xBQ3Jtc0ttYmJ2ZU9kcFpxbkRNYmllcGl1enVRNmM0UDMzdkZZSE45aDlrYmtUV2xLYTlvbjNVYUZmdXRBSkRrN19fWnNSQXJvS21CQ1hidGtZekRQSU5vZEpGQ2xkUnROcnAxQ3ZoUmZ5cFhNUjFvdG9mQ0JBQQ&q=https%3A%2F%2Fcivo.com%2Fthedigitallife&v=pumX2Ds5L0c)`

single server is might be not  enough to run all containers, we deploy all containers on multiple servers, or cluster.

To handle the traffic on all the server we need to use LOAD BALANCER and route to all our server instead of just one. we also should have some kindf of shared storage.

![image.png](DevOps%20da6aebebb693497ab12b304ee585cf05/image.png)

The Solution is KUBERNETES.

- EDCD: Distrubuted key-values pair, that store workerload, namespaces, different configurations. only accesible from api server of kubernetes
- API:  every communication for kubernetes, recieve all the request and act as frontend to cluster. we can use termincal client
- controller manager: Run worker loads and process in the background and it regulates the shared state of the cluster and perform routines. e.g if you change any configuration then contoller mamanger spot changes and instructs other comonennt what to do.
- Schedular: controller manger instructs the schedular and schedular is reposnible for assigning the resources.
- kubelet: take instructions from api server and take care if the all the worker load are healthy and containers running in desired state and reports backs to master, its just taking instruction and executing them.
- kube-proxy:  also runnning on the worker load and deals networking and exposing services to externals. the proxy is handling the trafic and making changes in the OS to reciebes the those reqs.
- Pod: smallest entity of K8s, atleast one container in it.
- 

![image.png](DevOps%20da6aebebb693497ab12b304ee585cf05/image%201.png)

Commands

```yaml
# set default namespace 
k config set-context --current --namespace webserver

# get all pods
k get all pods -o wide
```

Nginx webserver:

```yaml
# create deployment.yml file with following content
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webserver
  namespace: webserver
  labels:
    app: webserver

spec:
  selector:
    matchLabels:
      app: webserver
  replicas: 2
  template:
    metadata:
      labels:
        app: webserver
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
            
# use "kubectl apply -f deployment.yml" to apply this deployment
# now if you want to make available your service to public internet 
# then you have to create service object 

apiVersion: v1
kind: Service
metadata:
  name:  servicename
  namespace: namespace
spec:
  selector:
    app:  appname
  # ---
  # type:  ClusterIP
  # ClusterIP means this service can be accessed by any pod in the cluster
  # ports:
  # - name:  http
  #   port:  8080
  #   targetPort: 80
  #   protocol: TCP  # optional protocol
  # ---
  # type:  NodePort
  # NodePort means this service is only accessible by pods in the same namespace
  # ports:
  # - name:  http
  #   port:  80
  #   nodePort: 30001
  #   protocol: TCP  # optional protocol
  # ---
  type:  LoadBalancer
  # LoadBalancer means this service is load-balanced across all nodes in the cluster
	  ports:
    - name:  http
      port:  80
      targetPort: 30001
      protocol: TCP  # optional protocol

# after you apply service the digital ocean is automatically going
# to create load balancer for you and then just hit the external api. 

ahsan@ahsan-XPS-13-7390:~/k-configurations/nginx-deployment$ **kubectl get service** 
NAME        TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)        AGE
webserver   LoadBalancer   10.245.102.12   206.189.249.236   80:31470/TCP   2m55s

```

Persistent Storage:

mounting specific folder inside the pod running from host drive

```yaml
# yml file with persistent volumes:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webserver
  namespace: webserver
  labels:
    app: webserver

spec:
  selector:
    matchLabels:
      app: webserver
  replicas: 2
  template:
    metadata:
      labels:
        app: webserver
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          volumeMounts:
            - name: local
              mountPath: /usr/share/nginx/html
      volumes:
        - name: local
          hostPath:
            path: /home/ahsan/k-configurations/nginx-deployment/nginx

```

above the shared storage that is not accrose the all pods: for this we need to have some shared storeage available to all pods. for that we can use goodle cloud storage aws s3 or azure but for now we are using nfs service for storage. create file with `nfs-pv.yml` .

```yaml
**apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  capacity:
    storage: 500Mi
  accessModes:
    - ReadWriteMany
  storageClassName: nfs
  nfs:
    server: 192.168.1.7
    path: "/srv/nfs"**
```

`kubectl apply -f nfs-pv.yml`  and use `kubectl get pv` to get persistent volumes 

Config maps in kubernetes:

to store keyvalues secrets and the enviromental variables

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webserver-cm
data:
  # key: value
  # file: |
  #   content ==> this will create file 
  # ---
  nginx.conf: |
    user nginx;
    worker_processes 1;
    events {
      worker_connections  10240;
    }
    http {
      server {
        listen       80;
        server_name  _;
       

        location /test {
            return 200 'This is the content you want to return';
            add_header Content-Type text/plain;
        }

        location / {
            autoindex on;
            root /usr/share/nginx/html;
            index index.html index.htm;
        }
      }
    }

```