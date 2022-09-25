# Service

Each pod gets its own IP address.
Pods die so services allow other Pods inside your cluster to communicate with them despite changing IPs.

Service Resource
- An abstraction which defines a logical set of Pods and a policy by which to access them.
- The set of Pods targeted by a Service is usually determined by a `selector`

Cloud-native Service Discovery
- Can query the API server for Endpoints, that get updated whenever the set of Pods in a Service changes.
- Non-native apps: Can place a network port or load balancer in between your app and the backend Pods.

## Defining a Service

- A Sevice is a REST object, similar to a Pod.
- Like all of the REST objects, you can POST a Service definition to the API server to create a new instance.
- The name of the Service object must be a valid `DNS label name`. [[RFC 1123](https://tools.ietf.org/html/rfc1123)]
    - At most 63 characters
    - Contains only lowercase alphanumeric characters or '-'
    - Start and end with an alphanumeric character

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

- Kubernetes assigns Services an IP address (Cluster IP) which is used by the Service proxies.
- The controller for the Service selector continuously scans for Pods that match its selector, and then POSTs any updates to an Endpoint object also named `my-service`.
- **NOTE**: A Service can map any incoming `port` to a `targetPort`. By default the `targetPort` field is set to the same value as the `port` field.
- Port definitions in Pods have names, and you can reference these names in the `targetPort` attribute of a Service.
    - This works even if there is a mixture of Pods in a Service using a single configurd name, with the same network protocol available via different port numbers.
    - Allows you to change the port numbers that Pods expose without breaking clients.
- Default protocol: `TCP`
    - Other [supported protocols](https://kubernetes.io/docs/concepts/services-networking/service/#protocol-support)
- Supports multiple port definitions on a Service object.
    - Each can have different protocols used.

## Services without Selectors

- You want to have an external database cluster in production, but in your test environment you use your own databases.
- You want to point your Service to a Service in a different Namespace or on another cluster.
- You are migrating a workload to Kubernetes.
    - Only run a proportion of backends in Kubernetes.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

- With no selector, the corresponding Endpoint object is not created automatically.
    - You can manually map the Service to the network address and port where it is running by adding an Endpoint object manually.

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 192.0.2.42
    ports:
      - port: 9376
```

- The name of the Endpoints object must be a valid `DNS subdomain name` [[RFC 1123](https://tools.ietf.org/html/rfc1123)]
    - No more than 253 characters
    - Only lowercase alphanumeric chars, '-', or '.'
    - Start and end with alphanumeric chars
- **NOTE**: Endpoint IPs must not be:
    - Loopback
    - Link-Local
    - Cluster IPs of other Kubernetes Services (kube-proxy doesn't support virtual IPs as a destination)

* EndpointSlices
    - An API resource that can provide a more scalable alternative to Endpoints.
    - Conceptually similar to Endpoints, but allow for distributing network endpoints across multiple resources.
    - By default, considered "full" once it reaches 100 endpoints.  
        - Additional EndpointSlices will be created to store any additional endpoints if this happens.

* Application Protocol
    - Provides a way to specify an application protocol to be used for each Service port.
    - Currently in Alpha.

## Virtual IPs and Service Proxies

- Every node in a cluster runs a `kube-proxy`.
    - Responsible for implementing a form of virtual IP for `Services` of type other than `ExternalName`

- Why not use round-robin DNS instead of proxying for Services?
    - There is a long history of DNS implementations not respecting record TTLs, and caching the results of name lookups after they have expired.
    - Some apps do DNS lookups onlyy once and cache the results indefinitely.
    - Even if app libraries did proper re-resolution, the low or zero TTLs on the DNS records could impose high load on DNS that then becomes difficult to manage.

### User Space Proxy Mode

Client -> cluster IP (iptables) -> kube-proxy -> Backend Pods  
- `kube-proxy` watches Kuberenetes master for the addition and removal of Service and Endpoint objects.
    - Opens a random port for each Service on the local node.
    - Any connections to this 'proxy port' are proxied to one of the Service's backend Pods (as reported via Endpoints)
- `kube-proxy` takes the `SessionAffinity` setting of the Service into account when deciding which backend Pod to use.
- Installs iptables rules which capture traffic to the Service's `clusterIP` (which is virtual) and `port`.
    - The rules redirect the traffic to the proxy port which proxies the backend Pod.
- By default, `kube-proxy` in userspace mode chooses a backend via a round-robin algorithm.
- If the first Pod that's selected does not respond, `kube-proxy` would detect that the connection to the first Pod has failed and would automatically retry with a different backend Pod.

### iptables proxy mode

Client -> cluster IP (iptables) -> Backend Pods  
kube-proxy -> clusterIP (iptables)  

- `kube-proxy` watches the Kubernetes control plane for the addition and removal of Service and Endpoint objects.
    - Installs iptable rules for each Service, which capture traffic to the Service's `clusterIP` and `port`, and then redirects that traffic to one of the Service's backend sets.
    - For each Endpoint object, it installs iptables rules which select a backend Pod
- By default, `kube-proxy` in iptables mode chooses a backend at random.
- Using iptables to handle traffic has a lower system overhead, because the traffic is handled by Linux netfilter without the need to switch between userspace and the kernel space.
    - This approach is also likely to be more reliable.
- If the first Pod that's selected does not respond, the connection fails.
    - Can read Pod `readiness probes` to verify that backend Pods are working OK, so that `kube-proxy` only sees backends that test out as healthy.
        - Doing this means you avoid having traffic sent via `kube-proxy` to a Pod that's known to have failed.

### IPVS proxy mode

Client -> cluster IP (Virtual Server) -> Backend Pods (Real Server)
kube-proxy -> clusterIP (Virtual Server)

- `kube-proxy` watches Services and Endpoints, calls `netlink` interface to create IPVS rules accordingly and synchronizes IPVS rules with Services and Endpoints periodically.
    - This control loop ensures that IPVS status matches the desired state.
    - When accessing a Service, IPVS directs traffic to one of the backend Pods.
- Based on netfilter hook function that is similar to iptables mode, but uses a hash table as the underlying data structure and works in the kernel space.
    - This means `kube-proxy` in IPVS mode redirects traffic with lower latency than `kube-proxy` in iptables mode, with is much better performance when synchronising proxy rules.
- IPVS mode also supports higher throughput of network traffic.
- More options for balancing traffic to backend Pods:
    - rr: round robin
    - lc: least connection (smallest number of open connections)
    - dh: destination hashing
    - sh: source hashing
    - sed: shortest expected delay
    - nq: never queue
- To run `kube-proxy` in IPVS mode, you must make IPVS available on the node before starting `kube-proxy`.
    - When starting in IPVS mode, it verifies whether IPVS kernel modules are available.
    - If not detected, then `kube-proxy` falls back to running in iptables proxy mode.

In this proxy models, traffic bound for the Service's IP:Port are proxied to an appropriate backend without the clients knowing anythign about Kubernetes or Services or Pods.

If you want to make sure that connections from a particular client are passed to the same Pod each time, you can select the session affinity based on the client's IP address by setting `service.spec.sessionAffinity: ClientIP` (the default is None).  

You can also set the maximum session sticky time by setting `service.spec.sessionAffinityConfig.clientIP.timeoutSeconds` appropriately. (The default value is 10800 == 3 hours)

## Choosing your own IP address

- Can specify your own cluster IP address as part of a Service creation request.
    - Set `.spec.clusterIP` field.
- If you alread have an existing DNS entry that you wish to reuse, or legacy systems that are configured for a specific IP address and difficult to re-configure.
- The IP address that you choose must be a valid IPv4 or IPv6 address from within the `service-cluster-ip-range` CIDR range that is configured for the API server.
    - If you try to create a Service with an invalid clusterIP, the API server will return a 422 HTTP status code.

## Discovering Services

### Environment Variables

- When a Pod is run on a Node, the `kubelet` adds a set of environment variables for each active Service.
- It supports both `Docker link compatible` variables (see makeLinkVariables) and simplet `{SVCNAME}_SERVICE_HOST` and `{SVCNAME}_SERVICE_PORT` variabels, where the Service name is upper-cased and dashes are converted to underscores.
- **NOTE**: When you have a pod that needs to access a Service, and you are using the environment variable method to publish the port and cluster IP to the client Pods, you must create the Service before the client Pods come into existance.
    - If you only use DNS to discover the cluster IP for a Service, you don't need to worry about this ordering issue.

### DNS

- You can (and almost always should) set up a DNS service for your cluster using an add-on.
- A cluster-aware DNS server, such as CoreDNS, watches the Kubernetes API for new Services and creates a set of DNS records for each one.
    - If DNS has been enabled throughout your cluster then all Pods should automatically be able to resolve Services by their DNS name.
- If you have a Service called `my-service` in a Kubernetes Namespace `my-ns`, the control plane and the DNS Service acting together create a DNS record for `my-service.my-ns`.
    - Pods in the `my-ns` Namespace should be able to find it by simply doing a name lookup for `my-service` (both work).
    - Pods in other Namespaces must qualify the names as `my-service.my-ns`. These names will resolve to the cluster IP assigned for the Service.
- Also suspports DNS SRV (Service) records for named ports.
    - If the `my-service.my-ns` Service has a port named `http` with the protocol set to `TCP`, you can do a DNS SRV query for `_http._tcp.my-service.my-ns` to discover the port number for `http`, as well as the IP address.
- The Kubernetes DNS server is the only way to access `ExternalNmae` Services. More on [ExternalName resolution](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

## Headless Services

Sometimes you don't need load-balancing and a single Service IP.  
- Can create what are termed "healderss" Services, by explicitely specifying `None` for the cluster IP (`.spec.clusterIP`)
- You can use a headless Service to interface with other service discovery mechanisms, without being tied to Kubernetes' implementation.
- For headless Services, a cluster IP is not allocated, `kube-proxy` does not handle these Services, and there is no load balancing or proxying done by the platform for them.
- How DNS is automatically configured depends on whether the Service has selectors defined:
    - With Selectors:
        - The endpoints controller creates `Endpoints` records in the API, and modifies the DNS configuration to return records (addresses) that point directly to the `Pods` backing the `Service`.
    - Without Selectors:
        - The endpoints controller does not create `Endpoints` records. However, the DNS system looks for and configures either:
            - CNAME records for `ExternalName`-type Services
            - A records for any `Endpoints` that share a name with a Service, for all other types.

## Publishing Services (ServiceTypes)

* ServiceTypes expose your Services
* Default type is `ClusterIP`.
* It is also possible to use an `Ingress` to expose your Service.
    - Ingress is not a Service type, but it acts as the entry point for your cluster.
    - It lets you consolidate your routing rules to a single resource as it can expose multiple services under the same IP address.

### ClusterIP
Exposes the Service on a cluster-internal IP.  
Choosing this value makes the Service only reachable from within the cluster.  

### NodePort
Exposes the Service on each Node's IP at a static port (the `NodePort`).
- A ClusterIP Service, to which the NodePort Service routes, is automatically created.
    - You'll be able to contact the NodePort Service from outside your cluster, by requesting `<NodeIP>:<NodePort>`.

* Kubernetes control plane allocates a port from a range specified by `--service-node-port-range` (default: 30000-32767)
    - Each node proxies that port (the same port number on every Node) into your Service.
    - Your Service reports the allocated port in its `.spec.ports[*].nodePort` field.
* If you want to specify a particular IP(s) to proxy the port, you can set the `--nodeport-address` flag in `kube-proxy` to particular IP block(s).
    - The flag takes a comm-delimites list of IP blocks to specify IP address ranges that `kube-proxy` should consider as local to this node.
    - Default is an empty list, meaning `kube-proxy` should consider all available network interfaces for NodePort.
* Can specify port number in the `nodePort` field.
    - Control plane will either allocate that port or report the API transaction failed.
        - i.e. you must handle possible port collisions yourself.
    - Must also use a valid port number, one that's inside the range configured for NodePort use.
* Using NodePort gives you the freedom to:
    - Set up your own load balancing solution
    - Configure environments that are not fully supported by Kubernetes
    - Expose one or more node's IPs directly
* This Service is visible as `<NodeIP>:spec.ports[*].nodePort` and `.spec.clusterIP:spec.ports[*].port`
    - If the `--nodeport-addresses` flag in `kube-proxy` is set, would be filtered NodeIP(s).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: MyApp
  ports:
      # By default and for convenience, the `targetPort` is set to the same value as the `port` field.
    - port: 80
      targetPort: 80
      # Optional field
      # By default and for convenience, the Kubernetes control plane will allocate a port from a range (default: 30000-32767)
      nodePort: 30007
```

### LoadBalancer
Exposes your Service externally using a cloud provider's load balancer.
- NodePort and ClusterIP Services, to which the external load balancer routes, are automatically created.

* The actual creation of the load balancer happens asynchronously and information about the provisioned balancer is published in the Service's `.status.loadBalancer` field.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  clusterIP: 10.0.171.239
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 192.0.2.127
```

* Traffic from external load balancer is directed at backend Pods.
    - Cloud provider determines how it is load balanced.
* When more than one port is defined, all ports must have the same protocol and be one of (`TCP`, `UDP`, `SCTP`)
* In a mixed envrionment where traffic needs to be routed from both external and internal clients, you can set up an `Internal load balancer` by adding annotations to the service.

### ExternalName
Maps the Service to the contents of the `externalName` field, by returning a `CNAME` record with its value.
- No proxying of any kind is set up.
- **NOTE**: You need either `kube-dns` version 1.7 or `CoreDNS` version 0.0.8 or higher to use this type.
- **NOTE**: ExternalName accepts an IPv4 address string, but as a DNS names comprised of digits, not as an IP address. ExternalNames that resemble IPv4 addresses are not resolved by CoreDNS or ingress-nginx because ExternalName is intended to specify a canonical DNS name. To hardcode an IP address, consider using headless Services.
* Redirection happens at the DNS level rather than via proxying or forwarding
- **WARNING**: You may have trouble using ExternalNmae for common protocols (HTTP/HTTPS).
    - If you use ExternalName, then the hostname used by clients inside your cluster is different from the name that the ExternalName references.
    - For protocols that use hostnames this difference may lead to errors or unexpected responses.
        - HTTP requests will have a `Host:` header that the origin server does not recognize.
        - TLS servers will not be able to provide a certificate matching the hostname that the client connected to.

### External IPs

- If there are external IPs that route to one or more cluster nodes, Kubernetes Services can be exposed on those `externalIPs`.
- Traffic that inngresses into the cluster with the external IP (as destination IP), on the Service port, will be routed to one of the Service endpoints.
- externalIPs are not managed by Kubernetes are the responsibility of the cluster administration.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
  externalIPs:
    - 80.11.12.10
```

## Shortcomings