# knative exploration


### Install k3s on top of firecracker VMs via weaveworks ignite
- Create master VM  
```
$ sudo ignite run --config master.yml
INFO[0001] Created VM with ID "67b46cc4bc3fd496" and name "master" 
INFO[0001] Networking is handled by "cni"               
INFO[0001] Started Firecracker VM "67b46cc4bc3fd496" in a container with ID "ignite-67b46cc4bc3fd496" 
$ 
```
- Attach to the newly created firecracker master VM and install k3s without traefik
```
$ sudo ignite attach master

...

CentOS Linux 7 (Core)
Kernel 4.19.125 on an x86_64

67b46cc4bc3fd496 login: root
Password: 
[root@67b46cc4bc3fd496 ~]# yum install -y which curl

...

Dependency Updated:
  libcurl.x86_64 0:7.29.0-59.el7_9.1                                            

Complete!
[root@67b46cc4bc3fd496 ~]# export INSTALL_K3S_SKIP_START=true
[root@67b46cc4bc3fd496 ~]# curl -sfL https://get.k3s.io | sh -

...

[INFO]  systemd: Enabling k3s unit
Created symlink from /etc/systemd/system/multi-user.target.wants/k3s.service to /etc/systemd/system/k3s.service.
[root@67b46cc4bc3fd496 ~]# 
[root@67b46cc4bc3fd496 ~]# nohup k3s server --disable traefik &

...

root@67b46cc4bc3fd496 ~]# kubectl get nodes
NAME               STATUS   ROLES                  AGE   VERSION
67b46cc4bc3fd496   Ready    control-plane,master   42s   v1.20.0+k3s2
[root@67b46cc4bc3fd496 ~]# 

```
- grab kubeconfig from /etc/rancher/k3s/k3s.yaml and store it in your local workstation, and modify it to use the public IP address of the master VM (use 'ifconfig eth0' in the master VM to get it)
```
$ alias | grep kubectl
alias k='kubectl --kubeconfig ~/k3s-kubeconfig.yaml'
$ cat ~/k3s-kubeconfig.yaml | grep 6443
    server: https://10.61.0.14:6443
$ k get nodes
NAME               STATUS   ROLES                  AGE   VERSION
67b46cc4bc3fd496   Ready    control-plane,master   11m   v1.20.0+k3s2
$ 
```
- note: if k3s cluster was installed with traefik ingress controller installed, remove it with the following commands
```
$ kubectl delete -n kube-system helmcharts traefik
$ sudo rm /var/lib/rancher/k3s/server/manifests/traefik.yaml

```
- create the second VM, node1 as a worker node, install k3s binaries
```
$ sudo ignite run --config node1.yml
$ sudo ignite attach node1
[ in node1]
# yum install -y curl which
# export INSTALL_K3S_SKIP_START=true
# curl -sfL https://get.k3s.io | sh -
```
- grab the k3s token from /var/lib/rancher/k3s/server/node-token on the master VM and set as a variable on node1
```
# export TOKEN=<insert_token_from_master>
# nohup k3s agent --server https://10.61.0.14:6443 --token $TOKEN &
```
- Wait for a little bit, then validate that node1 is also available as a k3s worker node
```
$ k get nodes
NAME               STATUS   ROLES                  AGE   VERSION
67b46cc4bc3fd496   Ready    control-plane,master   23m   v1.20.0+k3s2
657d65c45823af58   Ready    <none>                 56s   v1.20.0+k3s2
$ k get po --all-namespaces
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   metrics-server-86cbb8457f-jv9z4           1/1     Running   0          23m
kube-system   coredns-854c77959c-xr4b8                  1/1     Running   0          23m
kube-system   local-path-provisioner-7c458769fb-xw8hz   1/1     Running   0          23m
$ 
```

### Install istio on top of k3s cluster

```
$ istioctl install --set profile=demo
his will install the Istio demo profile with ["Istio core" "Istiod" "Ingress gateways" "Egress gateways"] components into the cluster. Proceed? (y/N) Y
✔ Istio core installed                                     
- Processing resources for Istiod. Waiting for Deployment/istio-system/istiod                         
✔ Istiod installed 
✔ Egress gateways installed            
✔ Ingress gateways installed    
✔ Installation complete
$
$ k get po -n istio-system
NAME                                   READY   STATUS    RESTARTS   AGE
istiod-575d4664c5-pzvk2                1/1     Running   0          4m28s
svclb-istio-ingressgateway-vgjsv       5/5     Running   0          4m10s
svclb-istio-ingressgateway-tr8mf       5/5     Running   0          4m10s
istio-egressgateway-6b777b7484-bv5mx   1/1     Running   0          4m10s
istio-ingressgateway-9b86859b9-2s6jc   1/1     Running   0          4m10s
$ 

```
- Install kiali, prometheus and jaeger add-ons
```
$ k apply -f ../istio-1.8.1/samples/addons/prometheus.yaml 
$ k apply -f ../istio-1.8.1/samples/addons/jaeger.yaml
$ k apply -f ../istio-1.8.1/samples/addons/kiali.yaml 
$ k config set-context --current --namespace=istio-system
Context "default" modified.
$ k get po
NAME                                   READY   STATUS    RESTARTS   AGE
istiod-575d4664c5-pzvk2                1/1     Running   0          10m
svclb-istio-ingressgateway-vgjsv       5/5     Running   0          10m
svclb-istio-ingressgateway-tr8mf       5/5     Running   0          10m
istio-egressgateway-6b777b7484-bv5mx   1/1     Running   0          10m
istio-ingressgateway-9b86859b9-2s6jc   1/1     Running   0          10m
prometheus-7bfddb8dbf-n86bf            2/2     Running   0          2m27s
jaeger-7f78b6fb65-rc8m9                1/1     Running   0          99s
kiali-7476977cf9-xr227                 1/1     Running   0          68s
$ 
```
- Modify kiali service definition to LoadBalancer
```
$ k edit svc kiali
service/kiali edited
$ 
$ k get svc kiali
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                          AGE
kiali   LoadBalancer   10.43.171.192   10.61.0.14    20001:31974/TCP,9090:32293/TCP   3m16s
$ 
```
- now kiali UI can be accessed using the external IP address at port 20001 (e.g. http://10.61.0.14:20001)
- Validate that istio is working properly by installing the sample hello & world service
```
$ k create ns world
namespace/world created
$ k label namespace world istio-injection=enabled
namespace/world labeled
$ k apply -f world.yml -n world
deployment.apps/world-v1 created
service/world-v1 created
deployment.apps/world-v2 created
service/world-v2 created
service/world created
$ k get po -n world
NAME                        READY   STATUS    RESTARTS   AGE
world-v2-96f7f4578-k66jz    2/2     Running   0          22s
world-v1-57f964597f-mjfxz   2/2     Running   0          22s
$ 
$ k create ns hello
namespace/hello created
$ k label namespace hello istio-injection=enabled
namespace/hello labeled
$ k apply -f hello.yml -n hello
deployment.apps/hello created
service/hello created
$ k apply -f hello_ingress.yml -n hello
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
ingress.extensions/hello created
$ k get po -n hello
NAME                     READY   STATUS    RESTARTS   AGE
hello-79475c85b5-wdfs6   2/2     Running   0          23s
hello-79475c85b5-bttlh   2/2     Running   0          23s
$ k get ing -n hello
NAME    CLASS    HOSTS   ADDRESS   PORTS   AGE
hello   <none>   *                 80      18s
$ 
$ k get svc istio-ingressgateway
NAME                   TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.43.10.0   10.61.0.14    15021:31872/TCP,80:31517/TCP,443:30288/TCP,31400:32764/TCP,15443:32607/TCP   21m
$ 
$ curl http://10.61.0.14/hello
hello (10.42.0.11) earth (10.42.1.9)!!$ 
$ 
```
- Validate that the interaction can be visualized in kiali
![kiali_hello_world.png](kiali_hello_world.png)

- You can also change jaeger tracing service to LoadBalancer and view the interaction (also change the port not to use port 80 since that conflicts with istio-ingressgateway)
```
$ k edit svc tracing
service/tracing edited
$ k get svc tracing
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
tracing   LoadBalancer   10.43.163.189   10.61.0.15    16686:30005/TCP   19m
$ 
```
![jaeger_hello_world.png](jaeger_hello_world.png)

### install knative-serving on top of istio

```
$ k apply --filename https://github.com/knative/serving/releases/download/v0.19.0/serving-crds.yaml

$ k apply --filename https://github.com/knative/serving/releases/download/v0.19.0/serving-core.yaml

$ k config set-context --current --namespace=knative-serving
Context "default" modified.
$ k get po
NAME                          READY   STATUS    RESTARTS   AGE
autoscaler-696d8868f4-s2jtj   1/1     Running   0          2m3s
controller-7f7566fbcb-vc2sz   1/1     Running   0          2m3s
webhook-54d6f984b4-h6h8q      1/1     Running   0          2m3s
activator-845748699b-xqwpt    1/1     Running   0          2m3s
$ 

$ k apply --filename https://github.com/knative/net-istio/releases/download/v0.19.0/release.yaml
$
$ k get po
NAME                                READY   STATUS    RESTARTS   AGE
autoscaler-696d8868f4-s2jtj         1/1     Running   0          3m33s
controller-7f7566fbcb-vc2sz         1/1     Running   0          3m33s
webhook-54d6f984b4-h6h8q            1/1     Running   0          3m33s
activator-845748699b-xqwpt          1/1     Running   0          3m33s
istio-webhook-7dd7c94c7b-zlnbq      1/1     Running   0          31s
networking-istio-85f6b5c894-nv549   1/1     Running   0          31s
$ 

$ k apply --filename https://github.com/knative/serving/releases/download/v0.19.0/serving-default-domain.yaml
```
- validate the installation of knative serving using monkey.yaml
```
$ cat monkey.yml 
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: monkey
  namespace: default
spec:
  template:
    metadata:
      name: monkey-version-1
    spec:
      containers:
        - image: awiradarma/greeting:1.0
          env:
            - name: GREETING_MESSAGE
              value: "Monkey v1"
            - name: QUARKUS_HTTP_PORT
              value: "7778"
          ports:
            - name: http1
              containerPort: 7778
  traffic:
  - tag: current
    revisionName: monkey-version-1
    percent: 100
  - tag: latest
    latestRevision: true
    percent: 0
$ k label namespace default istio-injection=enabled
namespace/default labeled
$ k apply -f monkey.yml 
service.serving.knative.dev/monkey created
$ k config set-context --current --namespace=default
Context "default" modified.
$ k get ksvc
NAME     URL                                       LATESTCREATED      LATESTREADY   READY     REASON
monkey   http://monkey.default.10.61.0.14.xip.io   monkey-version-1                 Unknown   RevisionMissing
$ 
$ curl http://monkey.default.10.61.0.14.xip.io/greeting
Monkey v1 ( monkey-version-1-deployment-6dfcff9bf-pswhz/10.42.1.15 ) $ 
$ k get ksvc
NAME     URL                                       LATESTCREATED      LATESTREADY        READY   REASON
monkey   http://monkey.default.10.61.0.14.xip.io   monkey-version-1   monkey-version-1   True    
$ k get po
NAME                                          READY   STATUS        RESTARTS   AGE
monkey-version-1-deployment-6dfcff9bf-pswhz   3/3     Running       0          12s
monkey-version-1-deployment-6dfcff9bf-hzxvf   2/3     Terminating   0          111s
$ 
```
- The interaction can be seen from jaeger as well as from kiali
![kiali_serving_monkey.png](kiali_serving_monkey.png)
![jaeger_serving_monkey.png](jaeger_serving_monkey.png)


### knative eventing

- Install CRDs, core components and the default in-memory channel and broker
```
$ k apply --filename https://github.com/knative/eventing/releases/download/v0.19.0/eventing-crds.yaml
$ k apply --filename https://github.com/knative/eventing/releases/download/v0.19.0/eventing-core.yaml
$ k apply --filename https://github.com/knative/eventing/releases/download/v0.19.0/in-memory-channel.yaml
$ k apply --filename https://github.com/knative/eventing/releases/download/v0.19.0/mt-channel-broker.yaml
$ k config set-context --current --namespace=knative-eventing
Context "default" modified.
$ k get po
NAME                                    READY   STATUS    RESTARTS   AGE
eventing-controller-66c877b879-qtbcj    1/1     Running   0          60s
eventing-webhook-644c5c7667-sshrs       1/1     Running   0          60s
imc-controller-587f98f97d-4kstm         1/1     Running   0          43s
imc-dispatcher-6db95d7857-vghzx         1/1     Running   0          43s
mt-broker-ingress-7d8595d747-8q2dx      1/1     Running   0          31s
mt-broker-filter-6bd64f8c65-bt4pc       1/1     Running   0          32s
mt-broker-controller-76b65f7c96-rbbsv   1/1     Running   0          31s
$ 

```
- At this point, you can follow the instructions at https://knative.dev/docs/eventing/getting-started/ : 
```
$ k create ns event-example
namespace/event-example created
$ kubectl create -f - <<EOF
apiVersion: eventing.knative.dev/v1
kind: broker
metadata:
 name: default
 namespace: event-example
EOF
broker.eventing.knative.dev/default created
$ 
$ kubectl -n event-example get broker default
NAME      URL                                                                              AGE   READY   REASON
default   http://broker-ingress.knative-eventing.svc.cluster.local/event-example/default   41s   True    
$ 
$ kubectl -n event-example apply -f - << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-display
spec:
  replicas: 1
  selector:
    matchLabels: &labels
      app: hello-display
  template:
    metadata:
      labels: *labels
    spec:
      containers:
        - name: event-display
          image: gcr.io/knative-releases/knative.dev/eventing-contrib/cmd/event_display

---

kind: Service
apiVersion: v1
metadata:
  name: hello-display
spec:
  selector:
    app: hello-display
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
EOF

$ kubectl -n event-example apply -f - << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goodbye-display
spec:
  replicas: 1
  selector:
    matchLabels: &labels
      app: goodbye-display
  template:
    metadata:
      labels: *labels
    spec:
      containers:
        - name: event-display
          # Source code: https://github.com/knative/eventing-contrib/tree/master/cmd/event_display
          image: gcr.io/knative-releases/knative.dev/eventing-contrib/cmd/event_display

---

kind: Service
apiVersion: v1
metadata:
  name: goodbye-display
spec:
  selector:
    app: goodbye-display
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
EOF

$ kubectl -n event-example get deployments hello-display goodbye-display
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
hello-display     1/1     1            1           58s
goodbye-display   1/1     1            1           29s
$ 

$ kubectl -n event-example apply -f - << EOF
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: hello-display
spec:
  broker: default
  filter:
    attributes:
      type: greeting
  subscriber:
    ref:
     apiVersion: v1
     kind: Service
     name: hello-display
EOF

$ kubectl -n event-example apply -f - << EOF
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: goodbye-display
spec:
  broker: default
  filter:
    attributes:
      source: sendoff
  subscriber:
    ref:
     apiVersion: v1
     kind: Service
     name: goodbye-display
EOF

$ kubectl -n event-example get triggers
NAME              BROKER    SUBSCRIBER_URI                                            AGE   READY   REASON
hello-display     default   http://hello-display.event-example.svc.cluster.local/     42s   True    
goodbye-display   default   http://goodbye-display.event-example.svc.cluster.local/   19s   True    
$ 

$ kubectl -n event-example apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: curl
  name: curl
spec:
  containers:
    # This could be any image that we can SSH into and has curl.
  - image: radial/busyboxplus:curl
    imagePullPolicy: IfNotPresent
    name: curl
    resources: {}
    stdin: true
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    tty: true
EOF

$ kubectl -n event-example attach curl -it
Defaulting container name to curl.
Use 'kubectl describe pod/curl -n event-example' to see all of the containers in this pod.
If you don't see a command prompt, try pressing enter.
[ root@curl:/ ]$ curl -v "http://broker-ingress.knative-eventing.svc.cluster.local/event-example/default" \
>   -X POST \
>   -H "Ce-Id: say-hello" \
>   -H "Ce-Specversion: 1.0" \
>   -H "Ce-Type: greeting" \
>   -H "Ce-Source: not-sendoff" \
>   -H "Content-Type: application/json" \
>   -d '{"msg":"Hello Knative!"}'
> POST /event-example/default HTTP/1.1
> User-Agent: curl/7.35.0
> Host: broker-ingress.knative-eventing.svc.cluster.local
> Accept: */*
> Ce-Id: say-hello
> Ce-Specversion: 1.0
> Ce-Type: greeting
> Ce-Source: not-sendoff
> Content-Type: application/json
> Content-Length: 24
> 
< HTTP/1.1 202 Accepted
< Date: Mon, 11 Jan 2021 14:33:47 GMT
< Content-Length: 0
< 
[ root@curl:/ ]$ 
[ root@curl:/ ]$ curl -v "http://broker-ingress.knative-eventing.svc.cluster.local/event-example/default" \
>   -X POST \
>   -H "Ce-Id: say-goodbye" \
>   -H "Ce-Specversion: 1.0" \
>   -H "Ce-Type: not-greeting" \
>   -H "Ce-Source: sendoff" \
>   -H "Content-Type: application/json" \
>   -d '{"msg":"Goodbye Knative!"}'
> POST /event-example/default HTTP/1.1
> User-Agent: curl/7.35.0
> Host: broker-ingress.knative-eventing.svc.cluster.local
> Accept: */*
> Ce-Id: say-goodbye
> Ce-Specversion: 1.0
> Ce-Type: not-greeting
> Ce-Source: sendoff
> Content-Type: application/json
> Content-Length: 26
> 
< HTTP/1.1 202 Accepted
< Date: Mon, 11 Jan 2021 14:34:23 GMT
< Content-Length: 0
< 
[ root@curl:/ ]$ 
[ root@curl:/ ]$ curl -v "http://broker-ingress.knative-eventing.svc.cluster.local/event-example/default" \
>   -X POST \
>   -H "Ce-Id: say-hello-goodbye" \
>   -H "Ce-Specversion: 1.0" \
>   -H "Ce-Type: greeting" \
>   -H "Ce-Source: sendoff" \
>   -H "Content-Type: application/json" \
>   -d '{"msg":"Hello Knative! Goodbye Knative!"}'
> POST /event-example/default HTTP/1.1
> User-Agent: curl/7.35.0
> Host: broker-ingress.knative-eventing.svc.cluster.local
> Accept: */*
> Ce-Id: say-hello-goodbye
> Ce-Specversion: 1.0
> Ce-Type: greeting
> Ce-Source: sendoff
> Content-Type: application/json
> Content-Length: 41
> 
< HTTP/1.1 202 Accepted
< Date: Mon, 11 Jan 2021 14:34:56 GMT
< Content-Length: 0
< 
[ root@curl:/ ]$ exit
Session ended, resume using 'kubectl attach curl -c curl -i -t' command when the pod is running
$ 
$ $ kubectl -n event-example logs -l app=hello-display --tail=100
☁️  cloudevents.Event
Validation: valid
Context Attributes,
  specversion: 1.0
  type: greeting
  source: not-sendoff
  id: say-hello
  datacontenttype: application/json
Extensions,
  knativearrivaltime: 2021-01-11T14:33:47.368317272Z
Data,
  {
    "msg": "Hello Knative!"
  }
☁️  cloudevents.Event
Validation: valid
Context Attributes,
  specversion: 1.0
  type: greeting
  source: sendoff
  id: say-hello-goodbye
  datacontenttype: application/json
Extensions,
  knativearrivaltime: 2021-01-11T14:34:56.191439578Z
Data,
  {
    "msg": "Hello Knative! Goodbye Knative!"
  }
$ 
$ kubectl -n event-example logs -l app=goodbye-display --tail=100
☁️  cloudevents.Event
Validation: valid
Context Attributes,
  specversion: 1.0
  type: not-greeting
  source: sendoff
  id: say-goodbye
  datacontenttype: application/json
Extensions,
  knativearrivaltime: 2021-01-11T14:34:23.887687771Z
Data,
  {
    "msg": "Goodbye Knative!"
  }
☁️  cloudevents.Event
Validation: valid
Context Attributes,
  specversion: 1.0
  type: greeting
  source: sendoff
  id: say-hello-goodbye
  datacontenttype: application/json
Extensions,
  knativearrivaltime: 2021-01-11T14:34:56.191439578Z
Data,
  {
    "msg": "Hello Knative! Goodbye Knative!"
  }
$ 
$ k get all -n event-example
NAME                                   READY   STATUS    RESTARTS   AGE
pod/hello-display-5d7b749f6-c9m2d      1/1     Running   0          8m27s
pod/goodbye-display-66c45d6947-fvm5b   1/1     Running   0          7m58s
pod/curl                               1/1     Running   1          5m49s

NAME                                     TYPE           CLUSTER-IP      EXTERNAL-IP                                         PORT(S)   AGE
service/default-kne-trigger-kn-channel   ExternalName   <none>          imc-dispatcher.knative-eventing.svc.cluster.local   <none>    10m
service/hello-display                    ClusterIP      10.43.145.35    <none>                                              80/TCP    8m27s
service/goodbye-display                  ClusterIP      10.43.191.240   <none>                                              80/TCP    7m58s

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-display     1/1     1            1           8m27s
deployment.apps/goodbye-display   1/1     1            1           7m58s

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-display-5d7b749f6      1         1         1       8m27s
replicaset.apps/goodbye-display-66c45d6947   1         1         1       7m58s

NAME                                           BROKER    SUBSCRIBER_URI                                            AGE     READY   REASON
trigger.eventing.knative.dev/hello-display     default   http://hello-display.event-example.svc.cluster.local/     7m3s    True    
trigger.eventing.knative.dev/goodbye-display   default   http://goodbye-display.event-example.svc.cluster.local/   6m40s   True    

NAME                                  URL                                                                              AGE   READY   REASON
broker.eventing.knative.dev/default   http://broker-ingress.knative-eventing.svc.cluster.local/event-example/default   10m   True    

NAME                                                        URL                                                                     AGE   READY   REASON
inmemorychannel.messaging.knative.dev/default-kne-trigger   http://default-kne-trigger-kn-channel.event-example.svc.cluster.local   10m   True    

NAME                                                                                              AGE     READY   REASON
subscription.messaging.knative.dev/default-hello-display-f386265c-a623-4800-9d08-1cde98740565     7m3s    True    
subscription.messaging.knative.dev/default-goodbye-display-700e9dc2-8670-4d93-a3c0-3c14dffd7859   6m40s   True    
$ 
```

### Configure kafka as knative eventing source

knative eventing kafkasource
https://redhat-developer-demos.github.io/knative-tutorial/knative-tutorial/advanced/eventing-with-kafka.html


https://github.com/knative/eventing-contrib/releases/tag/v0.18.8

https://knative.dev/docs/eventing/samples/kafka/


```

