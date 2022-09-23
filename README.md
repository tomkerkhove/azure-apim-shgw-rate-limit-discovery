# Multi-instance rate limiting with Azure API Management's self-hosted gateway

A sample on how to run multiple instances of Azure API Management's self-hosted gateway using rate limiting with cross-instance synchronization.

## How to configure rate limiting synchronization

You can easily configure synchronization of rate limiting across multiple instances of the self-hosted gateway:

1. Get the Kubernetes YAML manifest from the Azure Portal as a starting point
2. Create the Kubernetes secret to authenticate the self-hosted gateway with Azure API Management
3. Change the Kubernetes Deployment so that it exposes additional ports:
  - Port 4290 (UDP) for the rate limiting synchronization
  - Port 4291 (UDP) for sending heartbeats to other instances
4. Create a new headless Kubernetes Service for the self-hosted gateway which will be used to discover the gateway instances. (see `deployment/rate-limit-discovery.yaml`)
  - The Service should expose the same ports that were added in step 1)
5. Introduce a new `neighborhood.host` environment variable on the Kubernetes Deployment that points to the headless Service created in step 2.
  - The value should be the name of the headless Service, e.g. `rate-limit-discovery`

Once everything is deployed, the self-hosted gateway instances will automatically discover each other and synchronize their rate limiting counters.

*> ⚠️ Make sure that your Kubernetes Secret was created and correctly used by the Kubernetes Deployment*

## Getting information about IP addresses per pod

You can easily find the IP addresses of the self-hosted gateway instances by running the following command:

```shell
 ~  k describe endpoints/<service-name>
Name:         <service-name>
Namespace:    <namespace>
Labels:       app=apim-gateway
              service.kubernetes.io/headless=
Annotations:  <none>
Subsets:
  Addresses:          10.240.0.10,10.240.0.30,10.240.0.50
  NotReadyAddresses:  <none>
  Ports:
    Name                  Port  Protocol
    ----                  ----  --------
    rate-limit-discovery  4290  UDP
    discovery-heartbeat   4291  UDP

Events:  <none>
```

Or get more details about which pod is assigned to which IP address:

```shell
 ~  k get endpoints <service-name> -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  creationTimestamp: "2022-09-23T09:34:48Z"
  labels:
    app: apim-gateway
    service.kubernetes.io/headless: ""
  name: <service-name>
  namespace: <namespace>
  resourceVersion: "116516019"
  uid: a26a45c8-3ba4-44e8-9155-ba1f341f3b3c
subsets:
- addresses:
  - ip: 10.240.0.10
    nodeName: aks-agentpool-42421532-vmss000000
    targetRef:
      kind: Pod
      name: apim-gateway-764ff6d88-tzj29
      namespace: <namespace>
      resourceVersion: "116515885"
      uid: 70a8c697-6b6f-4847-88b4-64d9c5e6a3fa
  - ip: 10.240.0.30
    nodeName: aks-agentpool-42421532-vmss000000
    targetRef:
      kind: Pod
      name: apim-gateway-764ff6d88-w85sp
      namespace: <namespace>
      resourceVersion: "116515942"
      uid: bf80ce9d-1c4f-47db-b551-58b8deb20aa9
  - ip: 10.240.0.50
    nodeName: aks-agentpool-42421532-vmss000000
    targetRef:
      kind: Pod
      name: apim-gateway-764ff6d88-n8mmp
      namespace: <namespace>
      resourceVersion: "116516002"
      uid: ec2ba843-0a24-45fb-bb32-deae47eefba8
  ports:
  - name: rate-limit-discovery
    port: 4290
    protocol: UDP
  - name: discovery-heartbeat
    port: 4291
    protocol: UDP
```