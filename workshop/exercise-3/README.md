# Exercise 3 - Validate Istio Installation

In this exercise, you will validate your Istio installation and launch some debugging containers to experiment with the service mesh.


## Service Mesh Sidecar

![istio architecture](istio-whatisit.svg)

Istio's core functionality is achieved by setting up a proxy, or sidecar, inside every kubernetes pod. Outgoing and incoming traffic bound for the pod is intercepted by iptables and delivered to the `envoy` daemon running in the sidecar. The `envoy` proxy can then inspect, modify, forward, and record all traffic used by the application. East-West traffic goes through two `envoy` proxies, one at source and one at destination. All `envoy` proxies are configured by `pilot`, an Istio component.

## Sidecar injection

In Kubernetes, a sidecar is a utility container in the pod, and its purpose is to support the main container. For Istio to work, Envoy proxies must be deployed as sidecars to each pod of the deployment. There are two ways of injecting the Istio sidecar into a pod: manually using istioctl CLI tool or automatically using the Istio Initializer. In a previous step, we set up our cluster to use automatic injection.

## Terminals

In the upcoming exercises, it is useful to have multiple terminals open. Each will need the `KUBECONFIG` environment variable you exported earlier in the tutorial. If you get errors like:

```bash
kubectl get pod
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

Go back and export that environment variable and try again.


### Launch a debug container

```bash
kubectl run -i --tty poke2-`date +%s` --image=nibalizer/utilities --restart=ver -- sh
```

This container is running with plenty of utilities. A great way to debug issues in your cluster. Full credit to [Paul](https://twitter.com/pczarkowski) for teaching us this one.
Leave this container running, and examine it with a different terminal.

```bash
kubectl get pod
NAME                     READY     STATUS        RESTARTS   AGE
poke2-1534738056         2/2       Running       0          35m
```

Note that the pod is running, and that it has a `2/2` for containers. That's the sidecar!

Lets dig deeper:

```bash
kubectl describe pod/poke2-1534738056
```

There is a ton of output in here! Note that we have an `init-container` referenced, an `init-container` is a contianer that runs before the main application comes up. This is used all over Kubernetes. It's a great place to just run a little intializer script or what-have-you. In this case the `init-container` runs a script that runs a bunch of `iptables` commands that reroute all network traffic through the sidecar. You can view the logs of that like this:

```bash
kubectl logs poke2-1534738056 -c istio-init
```

You can see all the `iptables` magic there. Also note that when we have more than one container in the pod, we have to use the `-c` flag to specify which contianer we're refering to.

Further down the `describe` output, we can see that there is a long running sidecar container called `istio-proxy`. We can see in the args to the container what kind of configuration information it has. Note the `istio-pilot` discovery address. This is how every sidecar checks into the `pilot` system to get configuration. There is more digging to do here, but let's move on.


### See the proxy in action

Create an nginx service

```bash
kubectl apply -f nginx.yaml
```

And inspect

```
kubectl get pod
NAME                     READY     STATUS    RESTARTS   AGE
nginx-645dbd8899-mwnsc   2/2       Running   0          4s
poke2-1534738056         2/2       Running   0          52m
```

```bash
kubectl describe pod/nginx-645dbd8899-mwnsc
```

This looks pretty much like the other pod. Both an `init-container` and an `istio-proxy` sidecar container are present.

Now, lets see the traffic moving between pods via the proxy mesh.

In one terminal, start watching logs on the nginx's istio sidecar.

```bash
kubectl logs nginx-645dbd8899-mwnsc -c istio-proxy -f
```
> Note, your exact pod name will be different.

In the other terminal, the one still at a root shell in the utilities container, make some requests.

```bash
curl nginx
curl nginx
curl nginx:3000
```

In your logs terminal, you should be seeing the requests filter through that `envoy` process.

```
[2018-08-20T05:10:27.314Z] "GET / HTTP/1.1" 200 - 0 612 5 0 "-" "curl/7.47.0" "9891a5fe-910b-912c-978b-2ead37c27528" "nginx" "127.0.0.1:80"
[2018-08-20T05:10:54.802Z] "GET / HTTP/1.1" 200 - 0 612 0 0 "-" "curl/7.47.0" "61fb9a7f-c0bf-9e01-bd01-2b23ba0fe088" "nginx" "127.0.0.1:80"
```

You can find all 3 stops of the request in the logs. First at the sidecar in the source pod, then at the sidecar in the destination pod (that's what we just looked at), and finally in the application logs inside the destination pod.


You've now done some basic Istio poking, and hopefully have a deeper understanding of what is going on when we use Istio as a service mesh.

You can now continue to [Exercise 4](../exercise-4/README.md), where we'll deploy an application in an istio-enabled cluster, and explore more of Istio's features.
