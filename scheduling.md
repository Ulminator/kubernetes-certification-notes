# Taints and Tolerations

- Restrict nodes from accepting pods unless they have certain 
- Does not guarantee that pods only prefer those nodes

- Prevent other pods from being placed on our nodes.

- Node level isolation is a safeguard to prevent rogue workloads from penetrating a Node's OS-level and container runtime-level protections for a possible attack on shared memory/persistent storage.

- Taint
	- Set on the node
	- WE just happen to have same label/taint k/v pairs but that is not necessary
	- kubectl taint nodes node key=value:effect
	- effect values:
		- NoSchedule
		- PreferNoSchedule
		- NoExecute
			- If at least one un-ignored taint with this effect, then the pod will be evicted from the node
				- if it is already running on the node

- Tolerations
	- Added to pods

tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"

tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"


# Node Affinity

- Label nodes
- Set nodeSelector on pods to tie pods to correct nodes
- Does not guarantee that other pods are not placed onto these nodes.

- Prevent our pods from being placed on their nodes.

# Pod Affinity/Anti-Affinity

```yaml
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: role
        operator: In
        values:
        - web-server  # only matches pods with label role: web-server
    topologyKey: kubernetes.io/hostname  # additional req that needs to be fulfilled
```

- Collocating pods in same place (node, cloud provider, zone, region)
- topologyKey specifies what constitutes collocation

- topologyKey
    - kubernetes.io/hostname
        - Collocate on same node
    - failure-domain.beta.kubernetes.io/zone
        - Collocate in same zone

```yaml
---
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: consul
              release: <release name>
              component: server
          topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: consul
              release: <release name>
              component: server
          topologyKey: topology.kubernetes.io/zone
---
```

- In the above, a consul server pod will not be scheduled:
    - on a node that already has a consul server pod running
    - in a zone that already has a consul server pod running

```yaml
---
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: vault
              release: <release name>
              component: server
          topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: consul
              release: <release name>
              component: server
          topologyKey: kubernetes.io/hostname
---
```

- In the above, a vault server will not be scheduled on nodes that already have either vault or consul running.