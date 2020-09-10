# kubernetes-servicemesh

## Do you need a service mesh ?  

few years ago i started to evaluate this feature fitting in an existing infrastructure  

There are many concept to consider and many mistake the people usually think  
Better to start with what is *NOT* a work for a service mesh   

- is not an apigw (even if could share some components)  
- is not the place to put firewall rules  
- is not something magic that boost the applications
- is something that if not used with a know scope could generate a mess


So what is ...   
well short answer   
- is the missing link in the infrastructure observability  
- is a way to handle in a structured way the application routing  
- is an internal ratelimit / anti ddos / infrastructure layer (be careful)
- could be a clever way to improve some application limits (expanded next)


Anyway is this something that we can add in our infrastructure ?  

There no YES/NO , however we can evaluate the company and the maturity of microservices  
internal rate limit , is usually a feature that could safe the infrastructure during snowball effects   
however means that if the infrastructure is SYNC (no decoupling) have the rate limit can just   
stop the application to serve requests , and this will propagated to the others below.  

result: no answer  
threads safe  

Sometimes it's better to have a strategic business login using the a circuit breaker that bring a huge complexity in configuration.  

The other point related to rate limit is .. who will maintain those values ?  
Should be part of deployment pipeline and directly correlated with the application scope  
In a 200+ micro services infrastructure this could be a huge problem:  

- project that lost ownership  
- projects not well maintained  
- new legacy projects  

So my idea about rate limit is to use it in a specific "strategic" applications
and should not indiscriminately added to the whole infrastructure


About the routing feature, we can consider as a more detailed and customized
blue green deployment , this specific case it's really useful when we have to deploy new features in production and *canary* deployment is not enough to cover
the business measurement we need.

This feature could be used to keep a specific affinity within the microservices
and this is the real feature that some of you can consider, imagine a strict dependencies between application an cache (as usual)  
So the application *Pippo* is using the cache *Paperino*  

Pippo is a namespace composed by 10 pods  
Paperino is a cache composed by 6 pods

Imagine that we have the cache as a replication/sharded and we have 2 availability zones

With service mesh we can use labels to say to Pippo to use the cache Paperino only in the availability zone where the call start from Pippo av, this will reduce dramatically the roundtrip and the answer  


I played a bit with service mesh in order to answer some questions,  
however related to firewall rules the right answer is Cilium :-)

With this, I'd like to say that service mash give you a great value only   
if your infrastructure is able to embrace it and only if you know what you are doing with this infrastructure.

- observability
- routing purpose (this is strictly related to the microservice architecture)
- rate limitng


## Service Mesh sample lab

This lab is provided to discover and test the functionality w'd like to implement
in our environment


### Basic setup

- minikube v1.6.2
- Kubernetes v1.17.0 on Docker '19.03.5'

`curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64`
`chmod +x minikube `  
`sudo install minikube /usr/local/bin/`  
`minikube start --memory=3000 --cpus=3`

- network (since in production we are using flannel that is not able to manage network policy we can start using no CNI to have the environment as much as close to lmn environment)


#### Namespaces

a → traefik (ingress)  
b → apache  
c → application  
d → redis  

b is a namespaces that manage an apache used for some rewrite rules  
c is a python application that is connected to a redis database, provide some function to set a time, get a time with /set context and /get context  
d is the redis database

c code   
```
"""
    Example app that integrates with redis and save/get timing
"""
from os import environ
from datetime import datetime
import json
import redis
from flask import Flask, redirect

VERSION = "1.1.1"
REDIS_ENDPOINT = environ.get("REDIS_ENDPOINT", "redis-svc.d-redis.svc.cluster.local")
REDIS_PORT = int(environ.get("REDIS_PORT", "6379"))


APP = Flask(__name__)


@APP.route("/")
def redisapp():
    """Main redirect"""
    return redirect("/get", code=302)


@APP.route("/set")
def set_var():
    """Set the time"""
    red = redis.StrictRedis(host=REDIS_ENDPOINT, port=REDIS_PORT, db=0)
    red.set("time", str(datetime.now()))
    return json.dumps({"time": str(red.get("time"))})


@APP.route("/get")
def get_var():
    """Get the time"""
    red = redis.StrictRedis(host=REDIS_ENDPOINT, port=REDIS_PORT, db=0)
    return json.dumps({"time": str(red.get("time"))})


@APP.route("/reset")
def reset():
    """Reset the time"""
    red = redis.StrictRedis(host=REDIS_ENDPOINT, port=REDIS_PORT, db=0)
    red.delete("time")
    return json.dumps({"time": str(red.get("time"))})


@APP.route("/version")
def version():
    """Get the app version"""
    return json.dumps({"version": VERSION})


@APP.route("/healthz")
def health():
    """Check the app health"""
    try:
        red = redis.StrictRedis(host=REDIS_ENDPOINT, port=REDIS_PORT, db=0)
        red.ping()
    except redis.exceptions.ConnectionError:
        return json.dumps({"ping": "FAIL"})

    return json.dumps({"ping": red.ping()})


@APP.route("/readyz")
def ready():
    """Check the app readiness"""
    return health()


if __name__ == "__main__":
    APP.run(debug=True, host="0.0.0.0")
```

Dokerfile  
```
FROM python:3.6-alpine
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
ENTRYPOINT ["python"]
CMD ["app.py"]
```

requirements.txt  
```
Flask
redis
pytest
pytest-flask
```

### Folders structure

```
kubernetes/
├── 00-traefik
│   ├── A-00-traefik-ns.yaml
│   ├── A-01-traefik-rbac.yaml
│   └── A-02-traefik-ds.yaml
├── 01-apache
│   ├── B-00-k8s-apacherr-ns.yaml
│   ├── B-01-k8s-apacherr-svc.yaml
│   ├── B-02-k8s-apacherr-ing.yaml
│   ├── B-03-k8s-apacherr-dpl.yaml
│   └── B-04-k8s-apacherr-cfm.yaml
├── 02-redis
│   ├── D-00-lab-redis-ns.yaml
│   ├── D-01-lab-redis-svc.yaml
│   └── D-02-lab-redis-dpl.yaml
└── 03-app
    ├── C-00-app-ns.yaml
    ├── C-01-app-svc.yaml
    └── C-02-app-dpl.yaml
```

### Startup

kubernetes$ kubectl apply -f 00-traefik/  
```
namespace/a-ingress-traefik created  
clusterrole.rbac.authorization.k8s.io/traefik-ingress-controller created    
serviceaccount/traefik-ingress-controller created  
clusterrolebinding.rbac.authorization.k8s.io/traefik-ingress-controller created
serviceaccount/traefik-ingress-controller created
daemonset.apps/traefik-ingress-controller created
service/traefik-ingress-service created
```  

kubernetes$ kubectl apply -f 01-apache/  
```
namespace/b-apacherr created
service/apacherr-svc created
ingress.extensions/apacherr-ingress created
deployment.apps/apacherr created
configmap/apacherr-80-config created
```  

kubernetes$ kubectl apply -f 02-redis/  
```
namespace/d-redis created
service/redis-svc created
deployment.apps/redis created
```  

kubernetes$ kubectl apply -f 03-app/  
```
namespace/c-app-count created    
service/app-count-svc created  
deployment.apps/pythonapp created  
```

kubectl get po --all-namespaces
```
NAMESPACE           NAME                               READY   STATUS    RESTARTS   AGE
a-ingress-traefik   traefik-ingress-controller-jkppg   1/1     Running   0          5m29s
b-apacherr          apacherr-8b786b45d-g9vcl           1/1     Running   0          5m19s
c-app-count         pythonapp-555d6d88cd-slhfb         1/1     Running   0          4m55s
d-redis             redis-b869b89d-pf6ms               1/1     Running   0          5m12s
kube-system         coredns-6955765f44-6nrdr           1/1     Running   1          74m
kube-system         coredns-6955765f44-9fbgt           1/1     Running   1          74m
kube-system         etcd-minikube                      1/1     Running   1          74m
kube-system         kube-addon-manager-minikube        1/1     Running   1          74m
kube-system         kube-apiserver-minikube            1/1     Running   1          74m
kube-system         kube-controller-manager-minikube   1/1     Running   1          74m
kube-system         kube-proxy-cchls                   1/1     Running   1          74m
kube-system         kube-scheduler-minikube            1/1     Running   1          74m
kube-system         storage-provisioner                1/1     Running   2          74m
```

make sure virtualbox 8081 port should be available

![Virtualbox port forwarding](https://res.cloudinary.com/ethzero/image/upload/v1594573715/misc/img_virtualbox-portforwarding.png "Virtualbox port forwarding")




## flow

![flow](https://res.cloudinary.com/ethzero/image/upload/v1594573714/misc/img_flow.png "flow")


##### inizialize the redis database

`$ curl http://pippo.lan/count/set`   
{"time": "b'2019-12-28 20:06:33.919059'"}

##### test from apache to application (case 1)
`$ curl http://pippo.lan/count/get`  
{"time": "b'2019-12-28 20:06:33.919059'"}

##### test from apache to redis (case 2)
`$ curl http://pippo.lan/redis/GET/time`  
{"GET":"2019-12-28 20:06:33.919059"}


##### network rule example

cilium labels  
```
ENDPOINT   POLICY (ingress)   POLICY (egress)   IDENTITY   LABELS (source:key[=value])                           IPv6   IPv4            STATUS   
           ENFORCEMENT        ENFORCEMENT                                                                                               
201        Disabled           Disabled          32580      k8s:app=redis                                                10.15.182.193   ready   
                                                           k8s:io.cilium.k8s.namespace.labels.name=d-redis                                      
                                                           k8s:io.cilium.k8s.policy.cluster=default                                             
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                      
                                                           k8s:io.kubernetes.pod.namespace=d-redis                                              
                                                           k8s:track=redis                                                                      
1257       Disabled           Disabled          4          reserved:health                                              10.15.197.106   ready   
1663       Disabled           Disabled          54130      k8s:app=apacherr                                             10.15.192.41    ready   
                                                           k8s:io.cilium.k8s.namespace.labels.name=b-apacherr                                   
                                                           k8s:io.cilium.k8s.policy.cluster=default                                             
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                      
                                                           k8s:io.kubernetes.pod.namespace=b-apacherr                                           
3167       Disabled           Disabled          33702      k8s:app=pythonapp                                            10.15.247.186   ready   
                                                           k8s:io.cilium.k8s.namespace.labels.name=c-app-count                                  
                                                           k8s:io.cilium.k8s.namespace.labels.purpose=app                                       
                                                           k8s:io.cilium.k8s.policy.cluster=default                                             
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                      
                                                           k8s:io.kubernetes.pod.namespace=c-app-count                                          
                                                           k8s:track=pythonapp-stable         
```

network rule  
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-namespace
  namespace: d-redis
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: c-app-count
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: c-app-count

```

&nbsp;
&nbsp;

cilium/hubble https://github.com/cilium/hubble
![hubble](https://res.cloudinary.com/ethzero/image/upload/v1594574069/misc/hubble-drop.png "hubble")

&nbsp;

istio/kiali
![kiali](https://res.cloudinary.com/ethzero/image/upload/v1594574069/misc/istio-kiali.png "kiali")

&nbsp;


video Cilium example --> [img/cilium.mkv](https://res.cloudinary.com/ethzero/video/upload/v1594574074/misc/cilium.mkv)  

video Istio + Cilium example --> [img/istio.mkv](https://res.cloudinary.com/ethzero/video/upload/v1594574090/misc/istio.mkv)  
&nbsp;  

## requirements

- use service mesh to segregate redis "d" to accept connections only from application "c"  
expected "case 1" still working, "case 2" stop working and receive an error
