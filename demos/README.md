## Demo web application to show load balancing.

### Create a namespace to deploy the web application.
######
    kubectl create namespace demo

### Deploy the web application .
######
    kubectl apply -f ip-viewer/deployment.yaml

### Start a service to make the web application accessible from public.
######
    kubectl apply -f ip-viewer/service.yaml

### Apply an ingress resource
to tell the ingress controller when someone is accessing the ingress port (80), on the ingress IP (10.0.2.20), forward that request to a k8s service (ip-viewer-service) on the service port (80).
######
    kubectl apply -f ingress.yaml

### Test if the service is running.
Use kubectl command with filters.
######
    kubectl get pods -l app=ip-viewer -n demo

    NAME                         READY   STATUS    RESTARTS   AGE
    ip-viewer-5855bbddc9-ctwjl   1/1     Running   0          20m
    ip-viewer-5855bbddc9-lzrzh   1/1     Running   0          20m
    ip-viewer-5855bbddc9-t7729   1/1     Running   0          20m

Or the curl command on the ingress IP and port
######
    curl 10.0.2.20:80/ip | grep "Pod IP:"

    <h1>Pod IP: 192.168.160.203</h1>

Note that the IP address changes when you enter another curl command, which indicates that the ingress controller is acting as a load balancer between the pods.

### Try deleting the pods
Before deleting the pods, lets see what we currently have.
######
    kubectl get pods -l app=ip-viewer -n demo

    NAME                         READY   STATUS    RESTARTS   AGE
    ip-viewer-5855bbddc9-ctwjl   1/1     Running   0          25m
    ip-viewer-5855bbddc9-lzrzh   1/1     Running   0          25m
    ip-viewer-5855bbddc9-t7729   1/1     Running   0          25m

Now delete the pods.
######
    kubectl delete pods -l app=ip-viewer -n demo

######
    kubectl get pods -l app=ip-viewer -n demo

    NAME                         READY   STATUS        RESTARTS   AGE
    ip-viewer-5855bbddc9-c5l8x   1/1     Running       0          16s
    ip-viewer-5855bbddc9-ctwjl   1/1     Terminating   0          26m
    ip-viewer-5855bbddc9-l628x   1/1     Running       0          15s
    ip-viewer-5855bbddc9-lzrzh   1/1     Terminating   0          26m
    ip-viewer-5855bbddc9-mxshp   1/1     Running       0          16s
    ip-viewer-5855bbddc9-t7729   1/1     Terminating   0          26m

Previous pods are being deleted and k8s automatically deploys new pods to fulfill the definition.

### Add a pod by editing the replica set.
Open deploymeny.yaml and edit .
######
    spec:
        replicas: 3

to

######
    spec:
        replicas: 6

Then apply the file.
######
    kubectl apply -f deployment.yaml

Get the pods again.
######
    kubectl get pods -l app=ip-viewer -n demo

    NAME                         READY   STATUS    RESTARTS   AGE
    ip-viewer-5855bbddc9-bb7l5   1/1     Running   0          20s
    ip-viewer-5855bbddc9-c5l8x   1/1     Running   0          3m57s
    ip-viewer-5855bbddc9-l628x   1/1     Running   0          3m56s
    ip-viewer-5855bbddc9-mcjrs   1/1     Running   0          20s
    ip-viewer-5855bbddc9-mxshp   1/1     Running   0          3m57s
    ip-viewer-5855bbddc9-wr5sf   1/1     Running   0          20s

We can see that there are 3 new pods.

### Add a new web service
    kubectl apply -f hostname-viewer/deployment.yaml 
    kubectl apply -f hostname-viewer/service.yaml

######
    kubectl get pods -l app=hostname-viewer -n demo

    NAME                              READY   STATUS    RESTARTS   AGE
    hostname-viewer-d7db866ff-4l76d   1/1     Running   0          7m2s
    hostname-viewer-d7db866ff-nldk2   1/1     Running   0          7m2s
    hostname-viewer-d7db866ff-pff57   1/1     Running   0          7m2s

######
    curl 10.0.2.20:80/hostname | grep "Pod Host:"

    <h1>Pod Host: hostname-viewer-d7db866ff-8fv88</h1>