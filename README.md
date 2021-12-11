```


Ques. What is a loadbalancer ?

Ans.

Load balancer is meant for Layer-4 traffic load balancing

https://cloud.google.com/load-balancing/docs/network


Ques. What is a ingress  ?

Ans.

Ingress is a set of rules to deal with how to route the traffic based on incoming HTTP/HTTPS header:

https://kubernetes.io/docs/concepts/services-networking/ingress/


Ques. What is a ingress-controller ?

Ans.

An Ingress controller is a specialized load balancer for Kubernetes (and other containerized)
 environments. Kubernetes is the de facto standard for managing containerized applications. 
 For many enterprises, moving production workloads into Kubernetes brings additional challenges
 and complexities around application traffic management. An Ingress controller abstracts
 away the complexity of Kubernetes application traffic routing and provides a bridge
 between Kubernetes services and external ones.
 Kubernetes as a project supports and maintains AWS, GCE, and nginx ingress controllers.
 
https://www.nginx.com/resources/glossary/kubernetes-ingress-controller/
https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/


My Setup
===========================
[+] My cluster is a baremetal setup. I used kubeadm to install it.

kmaster2@kmaster2:~/ingress-testing$ kubectl version --short
Client Version: v1.22.2
Server Version: v1.22.2
kmaster2@kmaster2:~/ingress-testing$
kmaster2@kmaster2:~/ingress-testing$ kubeadm version -o short
v1.22.2
kmaster2@kmaster2:~/ingress-testing$
kmaster2@kmaster2:~/ingress-testing$ kubectl get nodes -o wide
NAME       STATUS   ROLES                  AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE           KERNEL-VERSION       CONTAINER-RUNTIME
kmaster2   Ready    control-plane,master   58d   v1.22.2   192.168.1.86   <none>        Ubuntu 18.04 LTS   4.15.0-162-generic   docker://20.10.9
knode3     Ready    <none>                 58d   v1.22.2   192.168.1.87   <none>        Ubuntu 18.04 LTS   4.15.0-162-generic   docker://20.10.9
knode4     Ready    <none>                 58d   v1.22.2   192.168.1.88   <none>        Ubuntu 18.04 LTS   4.15.0-162-generic   docker://20.10.9
kmaster2@kmaster2:~/ingress-testing$

[+] For loadbalancer I am going to use Metallb.
[+] For ingress-controller I am going to use nginx.

Metallb Install
================
[+] Metallb is a loadbalancer
[+] Install steps are mostly here :
    https://metallb.universe.tf/installation/
[+] Copy paste the steps below into the cluster:

kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl diff -f - -n kube-system


kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml

[+] create a configmap for metallb in layer-2 mode to give some IP in the same range as my cluster nodes :

kmaster2@kmaster2:~/ingress-testing$ cat metallb-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.96-192.168.1.99
kmaster2@kmaster2:~/ingress-testing$
kmaster2@kmaster2:~/ingress-testing$ kubectl apply -f metallb-configmap.yaml
configmap/config created
kmaster2@kmaster2:~/ingress-testing$
kmaster2@kmaster2:~/ingress-testing$ kubectl get all -n metallb-system
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-7dcc8764f4-2qblb   1/1     Running   0          23h
pod/speaker-4gtlp                 1/1     Running   0          23h
pod/speaker-689hw                 1/1     Running   0          23h
pod/speaker-6tjsz                 1/1     Running   0          23h

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/speaker   3         3         3       3            3           kubernetes.io/os=linux   23h

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           23h

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-7dcc8764f4   1         1         1       23h
kmaster2@kmaster2:~/ingress-testing$


Nginx-Controller
====================================
[+] Steps are listed here: 
    https://kubernetes.github.io/ingress-nginx/deploy/#bare-metal-clusters
[+] Using helm for the setup as its easier. Steps are listed here.
    Below the "helm upgrade" command will create the namespace as well

kmaster2@kmaster2:~/ingress-testing$ helm repo list
NAME                    URL
prometheus-community    https://prometheus-community.github.io/helm-charts
kmaster2@kmaster2:~/ingress-testing$ helm version --short
v3.7.1+g1d11fcb
kmaster2@kmaster2:~/ingress-testing$

kmaster2@kmaster2:~/ingress-testing$ sudo helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
[sudo] password for kmaster2:
"ingress-nginx" has been added to your repositories
kmaster2@kmaster2:~/ingress-testing$

kmaster2@kmaster2:~/ingress-testing$ sudo helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ingress-nginx" chart repository
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈
kmaster2@kmaster2:~/ingress-testing$

kmaster2@kmaster2:~/ingress-testing$ helm repo list
NAME                    URL
prometheus-community    https://prometheus-community.github.io/helm-charts
ingress-nginx           https://kubernetes.github.io/ingress-nginx
kmaster2@kmaster2:~/ingress-testing$ sudo helm search repo ingress
NAME                            CHART VERSION   APP VERSION     DESCRIPTION
ingress-nginx/ingress-nginx     4.0.13          1.1.0           Ingress controller for Kubernetes using NGINX a...
kmaster2@kmaster2:~/ingress-testing$

kmaster2@kmaster2:~/ingress-testing$ sudo helm upgrade --install ingress-nginx ingress-nginx   --repo https://kubernetes.github.io/ingress-nginx   --namespace ingress-nginx --create-namespace
Release "ingress-nginx" does not exist. Installing it now.
NAME: ingress-nginx
LAST DEPLOYED: Fri Dec 10 14:07:36 2021
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running 'kubectl --namespace ingress-nginx get services -o wide -w ingress-nginx-controller'

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx #IMPORTANT
    rules:
      - host: www.example.com
        http:
          paths:
            - backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
kmaster2@kmaster2:~/ingress-testing$

[+] As the metallb is already there, nginx-controller got a loadbalancer IP:
kmaster2@kmaster2:~/ingress-testing$ kubectl get all -n ingress-nginx
NAME                                          READY   STATUS    RESTARTS   AGE
pod/ingress-nginx-controller-54bfb9bb-z4wvg   1/1     Running   0          2m45s

NAME                                         TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.109.89.191   192.168.1.96   80:31322/TCP,443:30448/TCP   2m45s
service/ingress-nginx-controller-admission   ClusterIP      10.105.137.40   <none>         443/TCP                      2m45s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           2m45s

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-54bfb9bb   1         1         1       2m45s
kmaster2@kmaster2:~/ingress-testing$


Testing
==========================
[+] Already have a service and a deployment. All the application running in 
   the pod, does is to return the IP address of the POD & I have 2 POD's for now.
   YAML files for the application are here:
   https://github.com/upinder-sujlana/HXDEMO

kmaster2@kmaster2:~/ingress-testing$ kubectl get deployment,svc,pod
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hxdemo   2/2     2            2           2d23h

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/hxdemo-svc     ClusterIP   10.96.159.246   <none>        8080/TCP         2d22h
service/kubernetes     ClusterIP   10.96.0.1       <none>        443/TCP          6d23h
service/postgres-svc   NodePort    10.98.220.198   <none>        5432:31336/TCP   5d3h

NAME                          READY   STATUS    RESTARTS   AGE
pod/hxdemo-8695769664-c57d2   1/1     Running   0          2d23h
pod/hxdemo-8695769664-pgc7v   1/1     Running   0          2d23h
pod/postgres-0                1/1     Running   0          5d3h
kmaster2@kmaster2:~/ingress-testing$
kmaster2@kmaster2:~/ingress-testing$ kubectl get pods -o wide | grep hxdemo
hxdemo-8695769664-c57d2   1/1     Running   0          2d23h   10.44.0.1   knode3   <none>           <none>
hxdemo-8695769664-pgc7v   1/1     Running   0          2d23h   10.36.0.5   knode4   <none>           <none>
kmaster2@kmaster2:~/ingress-testing$
kmaster2@kmaster2:~/ingress-testing$ kubectl describe svc hxdemo-svc
Name:              hxdemo-svc
Namespace:         default
Labels:            app=hxdemo
                   snack=pizza
Annotations:       <none>
Selector:          app=hxdemo,snack=pizza
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.96.159.246
IPs:               10.96.159.246
Port:              <unset>  8080/TCP
TargetPort:        5000/TCP
Endpoints:         10.36.0.5:5000,10.44.0.1:5000
Session Affinity:  None
Events:            <none>
kmaster2@kmaster2:~/ingress-testing$


[+] lets create the ingress (rules) using a sample from below,
    as my cluster is 1.22 I am using the one specified for 1.19 and above:
https://kubernetes.github.io/ingress-nginx/user-guide/basic-usage/
https://docs.nginx.com/nginx-ingress-controller/configuration/ingress-resources/basic-configuration/

kmaster2@kmaster2:~/ingress-testing$ cat ingress-resource-1.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-1
spec:
  rules:
  - host: hxdemo-svc.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hxdemo-svc
            port:
              number: 8080
  ingressClassName: nginx #IMPORTANT
kmaster2@kmaster2:~/ingress-testing$ 

kmaster2@kmaster2:~/ingress-testing$ kubectl get ingress
NAME                 CLASS   HOSTS                    ADDRESS        PORTS   AGE
ingress-resource-1   nginx   hxdemo-svc.example.com   192.168.1.96   80      64m
kmaster2@kmaster2:~/ingress-testing$

kmaster2@kmaster2:~/ingress-testing$ kubectl describe ingress ingress-resource-1
Name:             ingress-resource-1
Namespace:        default
Address:          192.168.1.96
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host                    Path  Backends
  ----                    ----  --------
  hxdemo-svc.example.com
                          /   hxdemo-svc:8080 (10.36.0.5:5000,10.44.0.1:5000)
Annotations:              <none>
Events:                   <none>
kmaster2@kmaster2:~/ingress-testing$

[+] Using CURL to test, adding the http header:

kmaster2@kmaster2:~/ingress-testing$ curl -H "Host: hxdemo-svc.example.com" "http://192.168.1.96/"
{
  "pod_hostname": "hxdemo-8695769664-c57d2",
  "pod_podip": "10.44.0.1"
}
kmaster2@kmaster2:~/ingress-testing$ curl -H "Host: hxdemo-svc.example.com" "http://192.168.1.96/"
{
  "pod_hostname": "hxdemo-8695769664-pgc7v",
  "pod_podip": "10.36.0.5"
}
kmaster2@kmaster2:~/ingress-testing$ 

[+] lets configure the default backend as well, theory here:
https://kubernetes.io/docs/concepts/services-networking/ingress/

[+] Lets create a POD for testing to which I shall point the "defaultBackend" to:

kmaster2@kmaster2:~/ingress-testing$ cat default-backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: default-backend
  name: default-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      run: default-backend
  template:
    metadata:
      labels:
        run: default-backend
    spec:
      volumes:
      - name: data
        emptyDir: {}
      initContainers:
      - name: init-container
        image: busybox
        volumeMounts:
        - name: data
          mountPath: "/data"
        command: ["/bin/sh", "-c", 'echo "<h1><font color=blue>Oops.... this is the default backend </font></h1>" > /data/index.html']
      containers:
      - image: nginx
        name: default-backend
        volumeMounts:
        - name: data
          mountPath: "/usr/share/nginx/html"
kmaster2@kmaster2:~/ingress-testing$
kmaster2@kmaster2:~/ingress-testing$ kubectl apply -f default-backend.yaml
deployment.apps/default-backend created
kmaster2@kmaster2:~/ingress-testing$ 
kmaster2@kmaster2:~/ingress-testing$ kubectl get deployment
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
default-backend   1/1     1            1           81s
hxdemo            2/2     2            2           3d15h
kmaster2@kmaster2:~/ingress-testing$ 

[+] Expose the default-backend deployment to create a service 

kmaster2@kmaster2:~/ingress-testing$ kubectl expose deployment default-backend --port 80 --name default-backend-svc
service/default-backend-svc exposed
kmaster2@kmaster2:~/ingress-testing$ kubectl describe svc default-backend-svc
Name:              default-backend-svc
Namespace:         default
Labels:            run=default-backend
Annotations:       <none>
Selector:          run=default-backend
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.109.189.109
IPs:               10.109.189.109
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.44.0.6:80
Session Affinity:  None
Events:            <none>
kmaster2@kmaster2:~/ingress-testing$

[+] Modify the ingress rule to add the default backend

kmaster2@kmaster2:~/ingress-testing$ vi ingress-resource-1.yaml
kmaster2@kmaster2:~/ingress-testing$ cat ingress-resource-1.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-1
spec:
  defaultBackend:
    service:
      name: default-backend-svc
      port:
        number: 80
  rules:
  - host: hxdemo-svc.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hxdemo-svc
            port:
              number: 8080
  ingressClassName: nginx #IMPORTANT
kmaster2@kmaster2:~/ingress-testing$

kmaster2@kmaster2:~/ingress-testing$
kmaster2@kmaster2:~/ingress-testing$ kubectl apply -f ingress-resource-1.yaml
ingress.networking.k8s.io/ingress-resource-1 configured
kmaster2@kmaster2:~/ingress-testing$
kmaster2@kmaster2:~/ingress-testing$ kubectl describe ingress ingress-resource-1
Name:             ingress-resource-1
Namespace:        default
Address:          192.168.1.96
Default backend:  default-backend-svc:80 (10.44.0.6:80)
Rules:
  Host                    Path  Backends
  ----                    ----  --------
  hxdemo-svc.example.com
                          /   hxdemo-svc:8080 (10.36.0.5:5000,10.44.0.1:5000)
Annotations:              <none>
Events:
  Type    Reason  Age               From                      Message
  ----    ------  ----              ----                      -------
  Normal  Sync    7s (x4 over 16h)  nginx-ingress-controller  Scheduled for sync
kmaster2@kmaster2:~/ingress-testing$

[+] Checking of indeed the default backend is working 
    and if the demo service is working still :

kmaster2@kmaster2:~/ingress-testing$ curl  "http://192.168.1.96/"
<h1><font color=blue>Oops.... this is the default backend </font></h1>
kmaster2@kmaster2:~/ingress-testing$

kmaster2@kmaster2:~/ingress-testing$ curl  -H "Host: hxdemo-svc.example.com" "http://192.168.1.96/"
{
  "pod_hostname": "hxdemo-8695769664-c57d2",
  "pod_podip": "10.44.0.1"
}
kmaster2@kmaster2:~/ingress-testing$ curl  -H "Host: hxdemo-svc.example.com" "http://192.168.1.96/"
{
  "pod_hostname": "hxdemo-8695769664-pgc7v",
  "pod_podip": "10.36.0.5"
}
kmaster2@kmaster2:~/ingress-testing$

[+] Expanding the testing a bit, creating the below deployment & service:

kmaster2@kmaster2:~/ingress-testing$ kubectl create deployment hello-server-deployment --image=gcr.io/google-samples/hello-app:1.0
deployment.apps/hello-server-deployment created
kmaster2@kmaster2:~/ingress-testing$
kmaster2@kmaster2:~/ingress-testing$ kubectl get deployment
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
default-backend           1/1     1            1           44m
hello-server-deployment   1/1     1            1           62s
hxdemo                    2/2     2            2           3d16h
kmaster2@kmaster2:~/ingress-testing$
kmaster2@kmaster2:~/ingress-testing$ kubectl get pods
NAME                                      READY   STATUS    RESTARTS   AGE
default-backend-59cd64d6cb-7zgdr          1/1     Running   0          44m
hello-server-deployment-b6dc8856d-qn74p   1/1     Running   0          30s
hxdemo-8695769664-c57d2                   1/1     Running   0          3d16h
hxdemo-8695769664-pgc7v                   1/1     Running   0          3d16h
postgres-0                                1/1     Running   0          5d19h
kmaster2@kmaster2:~/ingress-testing$
kmaster2@kmaster2:~/ingress-testing$ kubectl expose deployment hello-server-deployment --name hello-server-svc --port 80 --target-port 8080
service/hello-server-svc exposed
kmaster2@kmaster2:~/ingress-testing$ kubectl describe svc hello-server-svc
Name:              hello-server-svc
Namespace:         default
Labels:            app=hello-server-deployment
Annotations:       <none>
Selector:          app=hello-server-deployment
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.107.255.67
IPs:               10.107.255.67
Port:              <unset>  80/TCP
TargetPort:        8080/TCP
Endpoints:         10.44.0.7:8080
Session Affinity:  None
Events:            <none>
kmaster2@kmaster2:~/ingress-testing$

[+] Editing the service to include the hello-server-svc :

kmaster2@kmaster2:~/ingress-testing$ vi ingress-resource-1.yaml
kmaster2@kmaster2:~/ingress-testing$ cat ingress-resource-1.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-1
spec:
  defaultBackend:
    service:
      name: default-backend-svc
      port:
        number: 80
  rules:
  - host: hxdemo-svc.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hxdemo-svc
            port:
              number: 8080
      - path: /hello
        pathType: Prefix
        backend:
          service:
            name: hello-server-svc
            port:
              number: 80
  ingressClassName: nginx #IMPORTANT
kmaster2@kmaster2:~/ingress-testing$ kubectl apply -f ingress-resource-1.yaml
ingress.networking.k8s.io/ingress-resource-1 configured
kmaster2@kmaster2:~/ingress-testing$ kubectl get ingress
NAME                 CLASS   HOSTS                    ADDRESS        PORTS   AGE
ingress-resource-1   nginx   hxdemo-svc.example.com   192.168.1.96   80      17h
kmaster2@kmaster2:~/ingress-testing$ kubectl describe ingress ingress-resource-1
Name:             ingress-resource-1
Namespace:        default
Address:          192.168.1.96
Default backend:  default-backend-svc:80 (10.44.0.6:80)
Rules:
  Host                    Path  Backends
  ----                    ----  --------
  hxdemo-svc.example.com
                          /        hxdemo-svc:8080 (10.36.0.5:5000,10.44.0.1:5000)
                          /hello   hello-server-svc:80 (10.44.0.7:8080)
Annotations:              <none>
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    19s (x5 over 17h)  nginx-ingress-controller  Scheduled for sync
kmaster2@kmaster2:~/ingress-testing$

[+] Testing again to confirm the default backend, / and /hello path are working
kmaster2@kmaster2:~/ingress-testing$
kmaster2@kmaster2:~/ingress-testing$ curl  "http://192.168.1.96/"
<h1><font color=blue>Oops.... this is the default backend </font></h1>
kmaster2@kmaster2:~/ingress-testing$
kmaster2@kmaster2:~/ingress-testing$ curl  -H "Host: hxdemo-svc.example.com" "http://192.168.1.96/"
{
  "pod_hostname": "hxdemo-8695769664-pgc7v",
  "pod_podip": "10.36.0.5"
}
kmaster2@kmaster2:~/ingress-testing$ curl  -H "Host: hxdemo-svc.example.com" "http://192.168.1.96/"
{
  "pod_hostname": "hxdemo-8695769664-c57d2",
  "pod_podip": "10.44.0.1"
}
kmaster2@kmaster2:~/ingress-testing$
kmaster2@kmaster2:~/ingress-testing$
kmaster2@kmaster2:~/ingress-testing$ curl  -H "Host: hxdemo-svc.example.com" "http://192.168.1.96/hello"
Hello, world!
Version: 1.0.0
Hostname: hello-server-deployment-b6dc8856d-qn74p
kmaster2@kmaster2:~/ingress-testing$




```
