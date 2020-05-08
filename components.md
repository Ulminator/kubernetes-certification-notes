# Components

## Control Plane Components

- Make global decisions about the cluster.
- Can be run on any machine in the cluster
    - Setup scripts typically place them all on the same machine and do not run user contianers on this machine

### kube-apiserver

- Exposes the Kubernetes API
- The front-end for the control plane
- Scales horizontally

### [etcd](https://etcd.io/docs/)

- Consistent and highly-available KV store used as Kubernetes' backing store for all cluster data
- If you use it, make sure to have a [back up](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster) plan for the data

### kube-scheduler

- Watches for newly created Pods with no assigned node and selects a node for them to run on.
- Determining factors for scheduling
    - Inidividual and collective resource requirements
    - Hardware/software/policy constraints
    - Affinity and anti-affinity specifciations
    - Data locality
    - Inter-workload interference
    - Deadlines

### kube-controller-manager

- Runs controller processes
- Logically, each controller is a separate process, but to reduce complexity, they are all compiled into a single binary and run in a single process
- These include:
    - Node Controller
        - Responsible for noticing and responding when nodes go down
    - Replication Controller
        - Responsible for maintaining the current number of pods for every replication controller object in the system
    - Endpoints Controller
        - Populates the Endpoints object (that is, joins Services and Pods)
    - Service Account and Token Controllers
        - Create default accounts and API access tokens for new namespaces

### cloud-controller-manager

- Embeds cloud-specific control logic
- Lets you link your cluster into your cloud provider's API
- Separates components that interact with that cloud platform from components that just interact with your cluster
- Only runs controllers that are specific to your cloud provider.
    - Not present with Kubernetes running on your own premises or own PC
- The following controllers can have cloud provider dependencies
    - Node Controller: For checking the cloud provider to determine if a node has been deleted in the cloud after it stops responding
    - Route Controller: For setting up routes in the underlying infrastructure
    - Service Controller: For creating, updating, and deleting cloud provider load balancers
    - NOT replication controller

## Node Components


### kubelet

- An agent that runs on each node
- Makes sure that containers are running in a Pod
- Takes a set of PodSpecs provided and ensures the containers described in those PodSpecs are running and healthy
- Does not manage containers not created by Kubernetes

### kube-proxy

- A network proxy that runs on each node in your cluster (implements part of Service concept)
- Mantains network rules on nodes.
    - These network rules allow network communication to your Pods network sessions inside or outside your cluster.
- Uses the OS packet filtering layer if there is one and it's available. (otherwise it forwards traffic itself)

### Container Runtime

- Software responsible for running containers
- Several are supported:
    - Docker
    - containerd
    - CRI-O
    - Any implementation of the Kubernetes CRI (Container Runtime Interface)

## Addons

- Use Kubernetes resources (DaemonSet, Deployment, etc) to implement cluster features.
- Providing cluster level features -> kube-system namespace
- DNS
- Web UI
- Container Resource Monitoring
- Cluster-level logging
