# k8s-http-proxy
A simple [tinyproxy](https://tinyproxy.github.io/) blueprint to forward HTTP(S) traffic from apps running on Kubernetes.
This kind of proxy is also called an HTTP CONNECT proxy, or just an HTTP(s) forward proxy. 
It does not intercept the traffic and instead sets up a tunnel between the client and the target.

# TL;DR
Run the following:
```bash
# (Optional) create a local cluster
kind create cluster --name test-proxy

git clone https://github.com/ofirc/k8s-http-proxy/
cd k8s-http-proxy
kubectl apply -f deploy/
kubectl wait --for=condition=Available deployment/proxy-server --timeout=150s

kubectl run curl --image=curlimages/curl -- sleep infinity
kubectl wait --for=condition=Ready pod/curl --timeout=150s
kubectl exec -it curl -- sh
export https_proxy=http://proxy-server.default.svc:8888
curl https://example.com
```

To ensure the traffic went through the proxy:
```bash
kubectl logs -l app=proxy-server
```

Example output:
```bash
$ kubectl logs -l app=proxy-server
INFO      Mar 28 22:03:56.669 [1]: Setting the various signals.
INFO      Mar 28 22:03:56.669 [1]: Starting main loop. Accepting connections.
CONNECT   Mar 28 22:05:34.891 [1]: Connect (file descriptor 7): 10.244.0.6
CONNECT   Mar 28 22:05:34.893 [1]: Request (file descriptor 7): CONNECT example.com:443 HTTP/1.1
INFO      Mar 28 22:05:34.895 [1]: No upstream proxy for example.com
INFO      Mar 28 22:05:34.895 [1]: opensock: opening connection to example.com:443
INFO      Mar 28 22:05:34.915 [1]: opensock: getaddrinfo returned for example.com:443
CONNECT   Mar 28 22:05:35.129 [1]: Established connection to host "example.com" using file descriptor 8.
INFO      Mar 28 22:05:35.129 [1]: Not sending client headers to remote machine
INFO      Mar 28 22:05:35.618 [1]: Closed connection between local client (fd:7) and remote client (fd:8)
$
```

Optional, tear down the cluster:
```bash
kind delete cluster --name test-proxy
```

# Related projects
* [Man-in-the-middle TLS inspection proxy](https://github.com/ofirc/k8s-sniff-https) - useful for deep packet inspection and reverse engineering HTTPS encrypted traffic

* [mTLS Proxy with client credentials](https://github.com/ofirc/go-mtls-proxy) - useful in zero-trust settings

# Use case
In enterprise settings, all outbound (egress) traffic typically goes through a proxy server of some kind.
As a developer / product owner, your app is required to go through this proxy for communicating with remote targets such as your SaaS backends.
This is done for various reasons, e.g. anonymizing, caching, compliance, governance, etc.

What could possibly go wrong 🤔?
1. **Debugging** - The app fails to communicate through the forward HTTP proxy server
   Is the problem in the HTTP proxy server? Is it a misconfigured client? How do I debug it?
2. **Developing** - I am a developer with not much of an experience with proxies in Kuberntes
   I was assigned a task to support proxies for my app but I'm not sure where to start.

**Famous last words**: big deal, I'll just setup a proxy!
But:
![One does not simply setup a forward proxy in k8s](images/one-does-not-simply.jpg)

Also, LLMs may occassionally emit _interesting_ configurations for proxies in Kubernetes.

Finally, incumbents like OSS [Squid proxy](https://www.squid-cache.org/) might be overwhelming
and overkill for beginners, and paying for a managed cloud proxy is out of the question.

That's where this repo shines, providing you with a simple reference proxy you can run test it.
[Jump to TL;DR](#tldr) for more information.

# Overview
## Prerequisites
- [kubectl](https://kubernetes.io/docs/tasks/tools/) - Kubernetes command-line tool
- [git](https://git-scm.com/downloads) - Version control system
- [kind](https://kind.sigs.k8s.io/) (optional) - Tool for running local Kubernetes clusters

**Step 0: (Optional) create a local cluster**
```sh
kind create cluster --name test-proxy
```

**Step 1: clone this repository**
```sh
git clone https://github.com/ofirc/k8s-http-proxy/
cd k8s-http-proxy
```

**Step 2: deploy the tinyproxy deployment**
```sh
kubectl apply -f deploy/
kubectl wait --for=condition=Available deployment/proxy-server --timeout=150s
```

**Step 3: spin up an ephemeral Pod for experimenting**
```sh
kubectl run curl --image=curlimages/curl -- sleep infinity
kubectl wait --for=condition=Ready pod/curl --timeout=150s
```

**Step 4: verify the running Pods**
If all went well, this is what you should see:
```sh
$ kubectl get pod,svc
NAME                               READY   STATUS    RESTARTS   AGE
pod/curl                           1/1     Running   0          18s
pod/proxy-server-xxxxxxxxx-xxxxx   1/1     Running   0          9m2s

NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP    18d
service/proxy-server  ClusterIP   10.98.161.180   <none>        8888/TCP   8m11s
$
```

## Test HTTP/HTTPS traffic through the proxy
Execute into the ephemeral curl Pod:
```sh
kubectl exec -it curl -- sh
```

Set the proxy environment variable:
```sh
export https_proxy=http://proxy-server.default.svc:8888
```

Trigger an HTTPS request:
```sh
curl https://example.com
```

## Verify traffic went through the proxy
Check the proxy server logs:
```sh
kubectl logs -l app=proxy-server
```

Example output:
```sh
$ kubectl logs -l app=proxy-server
INFO      Mar 28 22:03:56.669 [1]: Setting the various signals.
INFO      Mar 28 22:03:56.669 [1]: Starting main loop. Accepting connections.
CONNECT   Mar 28 22:05:34.891 [1]: Connect (file descriptor 7): 10.244.0.6
CONNECT   Mar 28 22:05:34.893 [1]: Request (file descriptor 7): CONNECT example.com:443 HTTP/1.1
INFO      Mar 28 22:05:34.895 [1]: No upstream proxy for example.com
INFO      Mar 28 22:05:34.895 [1]: opensock: opening connection to example.com:443
INFO      Mar 28 22:05:34.915 [1]: opensock: getaddrinfo returned for example.com:443
CONNECT   Mar 28 22:05:35.129 [1]: Established connection to host "example.com" using file descriptor 8.
INFO      Mar 28 22:05:35.129 [1]: Not sending client headers to remote machine
INFO      Mar 28 22:05:35.618 [1]: Closed connection between local client (fd:7) and remote client (fd:8)
$
```

## Cleanup
```sh
kubectl delete -f deploy/
kubectl delete pod curl --force
```

Optional, tear down the cluster:
```sh
kind delete cluster --name test-proxy
```

# Frequently Asked Questions (FAQ)

### Question: What resources are deployed to my cluster?
- A Kubernetes Deployment of `proxy-server` with one replica running [tinyproxy](https://tinyproxy.github.io/)
- A Kubernetes Service called `proxy-server`
- An ephemeral Pod running the `curlimages/curl` image, used for testing

### Question: What ports are used by this deployment?
- Port 8888 for the tinyproxy port
