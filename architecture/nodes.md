# Nodes

[Services on Node](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/architecture.md#the-kubernetes-node):
- Container runtime
- Kubelet
- Kube-proxy

## Node Status

`kubectl describe node <NAME>`

- **Addresses**
    - The usage of these fields vary depending on your cloud provider or bare metal config
    - HostName
        - hostname reported by the node's kernel
        - can be overriden via the kubelet `--hostname-override` param
    - ExternalIP
        - typically the IP address of the node that is externally routable (available from outside the cluster)
    - InternalIP
        - typically the IP address of the node that is routable only within the cluster
- **Conditions**
    - Describes the status of all `Running` nodes
    - Ready
        - True: node is healthy and ready to accept pods
        - False: node is not healthy and not accepting pods
        - Unknown: node controller has not heard from the node in the last `node-monitor-grace-period` (default 40s)
        - If not True for longer than `pod-eviction-timeout` (passed to kube-controller-manager) all Pods on the node are scheduled for deletion by the Node Controller.
            - Default eviction timeout is 5 minutes.
            - In some cases when the node is unreachable, the apiserver is unable to communicate with kubelet on the node.
                - The decision to delete pods cannot be communicated to the kubelet until communication with the apiserver is re-established.
                - The pods scheduled for deletion may continue to run on the node until then.
            - Pods that might be running on an unreachable node may be in the `Terminating` or `Unknown` state.
            - In cases where Kubernetes cannot deduce from the underlying infrastructure if a node has permanently left a cluster, the cluster admin may need to delete the object by hand.
        - The node lifecycle controller automatically creates [taints](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) that represent these conditions.
            - The scheduler takes the Node's taints into consideration when assigning a Pod to a Node
            - Pods can have tolerations which allow them to tolerate a Node's taints
    - MemoryPressure
    - PIDPressure
        - True: if pressure exists on the processes (i.e. too many processes on the node)
    - DiskPressure
    - NetworkUnavailable
        - True: if network of node is not correctly configured
- **Capacity and Allocatable**
- **Info**

## Management


## Node Topology


## API Object
