# Demo-VCluster

## Summary

In this demo will have a look vcluster with Embedded SQLite without Persistent Volume.

## Why Virtual Clusters?

|                    | TablSeparate Namespace        | vcluster           | Separate Cluster  |
|:------------------ |:-----------------------------:|:------------------:| -----------------:|
| Isolation          | very weak                     | strong             | very strong       |
| Access For Tenants | very restricted               | vcluster admin     | cluster admin     |
| Cost               | very cheap                    | cheap              | expensive         |
| Resource Sharing   | easy                          | easy               | very hard         |
| Overhead           | very low                      | very low           | very high         |


## Install vcluster cli

amd64:
```
curl -L -o vcluster "https://github.com/loft-sh/vcluster/releases/latest/download/vcluster-linux-amd64" && chmod +x vcluster;
sudo mv vcluster /usr/local/bin;
```

arm64:
```
curl -L -o vcluster "https://github.com/loft-sh/vcluster/releases/latest/download/vcluster-linux-arm64" && chmod +x vcluster;
sudo mv vcluster /usr/local/bin;
```

Current version of vcluster:
```
vcluster -v

vcluster version 0.14.2
```

Also can be useful:
```
vcluster -h

vcluster root command

Usage:
  vcluster [command]

Available Commands:
  completion  Generate the autocompletion script for the specified shell
  connect     Connect to a virtual cluster
  create      Create a new virtual cluster
  delete      Deletes a virtual cluster
  disconnect  Disconnects from a virtual cluster
  get         Gets cluster related information
  help        Help about any command
  list        Lists all virtual clusters
  pause       Pauses a virtual cluster
  resume      Resumes a virtual cluster
  upgrade     Upgrade the vcluster CLI to the newest version
  version     Print the version number of vcluster

Flags:
      --context string     The kubernetes config context to use
      --debug              Prints the stack trace if an error occurs
  -h, --help               help for vcluster
  -n, --namespace string   The kubernetes namespace to use
  -s, --silent             Run in silent mode and prevents any vcluster log output except panics & fatals
  -v, --version            version for vcluster

Use "vcluster [command] --help" for more information about a command.
```


## Deal with vcluster

Deploy Loadbalancer service to vcluster-demo namesapce:
```
kubectl apply -f vc-service.yaml
```
the result have to be something like this:
```
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP           PORT(S)         AGE
vcluster-demo   LoadBalancer   10.98.113.19   <YOUR EXTERNAL IP>    443:30641/TCP   13s
```
or check the result and the status with:
```
kubectl get svc -n vcluster-demo --output yaml

apiVersion: v1
items:
- apiVersion: v1
  kind: Service
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"vcluster-demo","namespace":"vcluster-demo"},"spec":{"ports":[{"name":"https","port":443,"protocol":"TCP","targetPort":8443}],"selector":{"app":"vcluster","release":"vcluster-demo"},"type":"LoadBalancer"}}
    creationTimestamp: "2023-04-13T10:38:55Z"
    name: vcluster-demo
    namespace: vcluster-demo
    resourceVersion: "479358"
    uid: 4b799519-e790-4fca-ae7f-0e7c5bfcad9d
  spec:
    allocateLoadBalancerNodePorts: true
    clusterIP: 10.98.113.19
    clusterIPs:
    - 10.98.113.19
    externalTrafficPolicy: Cluster
    internalTrafficPolicy: Cluster
    ipFamilies:
    - IPv4
    ipFamilyPolicy: SingleStack
    ports:
    - name: https
      nodePort: 30641
      port: 443
      protocol: TCP
      targetPort: 8443
    selector:
      app: vcluster
      release: vcluster-demo
    sessionAffinity: None
    type: LoadBalancer
  status:
    loadBalancer:
      ingress:
      - ip: <YOUR EXTERNAL IP>
kind: List
metadata:
  resourceVersion: ""
```

then install the virtual cluster:
```
vcluster create vcluster-demo \
    -n vcluster-demo \
    --connect=false \
    -f values.yaml
```

Check the results with:
```
kubectl get all -n vcluster-demo
NAME                                                         READY   STATUS    RESTARTS   AGE
pod/coredns-56d44fc4b4-hhn8r-x-kube-system-x-vcluster-demo   1/1     Running   0          5m7s
pod/vcluster-demo-0                                          2/2     Running   0          6m

NAME                                             TYPE           CLUSTER-IP       EXTERNAL-IP            PORT(S)                  AGE
service/kube-dns-x-kube-system-x-vcluster-demo   ClusterIP      10.103.208.72    <none>                 53/UDP,53/TCP,9153/TCP   5m7s
service/vcluster-demo                            ClusterIP      10.104.249.234   <none>                 443/TCP,10250/TCP        6m1s
service/vcluster-demo-headless                   ClusterIP      None             <none>                 443/TCP                  6m1s
service/vcluster-loadbalancer                    LoadBalancer   10.98.40.218     <YOUR EXTERNAL IP>     443:16571/TCP            7m8s

NAME                             READY   AGE
statefulset.apps/vcluster-demo   1/1     6m1s
```

Update the current kube config via:
```
vcluster connect vcluster-demo -n vcluster-demo --server=https://x.x.x.x

done √ Switched active kube context to vcluster_vcluster-demo_vcluster-demo_kubernetes-admin@kubernetes
- Use `vcluster disconnect` to return to your previous kube context
- Use `kubectl get namespaces` to access the vcluster

kubectl config get-contexts
CURRENT   NAME                                                               CLUSTER                                                            AUTHINFO                                                           NAMESPACE
          kubernetes-admin@kubernetes                                        kubernetes                                                         kubernetes-admin                                                   default
*         vcluster_vcluster-demo_vcluster-demo_kubernetes-admin@kubernetes   vcluster_vcluster-demo_vcluster-demo_kubernetes-admin@kubernetes   vcluster_vcluster-demo_vcluster-demo_kubernetes-admin@kubernetes
```

Switch back context
```
vcluster disconnect
```

Create a separate kube config to use instead of changing the current context
```
vcluster connect vcluster-demo --update-current=false
```

thern with your tenant, you can export kubeconfig.yaml:
export KUBECONFIG=kubeconfig.yaml