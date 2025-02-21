# Profiles: Advanced Provider Configuration

> Note: Only the Kubernetes provider currently supports Profiles!

The OpenFaaS design allows it to provide a standard API across several different container ochestration tools: Kubernetes, containerd, and others. These [faas-providers](/docs/architecture/faas-provider.md) generally implement the same core features and allow your to functions to remain portable and be deployed on _any_ certified OpenFaaS installation regardless of the orchestration layer. However, there are certain workloads or deployments that require more advanced features or fine tuning of configuration. To allow maximum flexibility without overloading the OpenFaaS function configuration, we have introduced the concept of Profiles. This is simply a reserved function annotation that the `faas-provider` can detect and use to apply the advanced configuration.

In some cases, there may be a 1:1 mapping between Profiles and Functions, this is to be expected for TopologySpreadConstraints, Affinity rules. We see no issue with performance or scalability.

In other cases, one Profile may serve more than one function, such as when using a toleration or a runtime class.

Multiple Profiles can be composed together for functions, if required.

> Note: The general design is inspired by [StorageClasses](https://kubernetes.io/docs/concepts/storage/storage-classes/)  and [IngressClasses](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class) in Kubernetes. If you are familiar with Kubernetes, these comparisons may be helpful, but they are not required to understand Profiles in OpenFaaS.

## Using Profiles When You Deploy a Function

If you are a function author, using a Profile is a simple as adding an annotation to your function:

```
com.openfaas.profile: <profile_name>
```

You can do this with the `faas-cli` flags:

```sh
faas-cli deploy --annotation com.openfaas.profile=<profile_name>
```

Or in the stack YAML:
```yaml
functions:
  foo:
    image: "..."
    fprocess: "..."
    annotations:
      com.openfaas.profile: <profile_name>
```

If you need multiple profiles, you can use a comma separated value:

```
com.openfaas.profile: <profile_name1>,<profile_name2>
```

Profiles are created in the `openfaas` namespace, so typically will be created and maintained by Cluster Administrators.

## Creating Profiles

Profiles must be pre-created, similar to Secrets, usually by the cluster admin. The OpenFaaS API does not provide a way to create Profiles because they are hyper specific to the orchestration tool.

### Enable Profiles

When installing OpenFaaS on Kubernetes, Profiles use a CRD. This must be installed during or prior to start the OpenFaaS controller. When using the [official Helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas) this will happen automatically. Alternatively, you can apply this [YAML](https://github.com/openfaas/faas-netes/blob/master/yaml/crd.yml) to install the CRD.

### Available Options

Profiles in Kubernetes work by injecting the supplied configuration directly into the correct locations of the Function's Deployment. This allows us to directly expose the underlying API without any additional modifications. Currently, it exposes the following Pod and Container options from the Kubernetes API.

- `podSecurityContext` : (OpenFaaS CE & Pro) https://kubernetes.io/docs/tasks/configure-pod-container/security-context/ for a description and links to any additional documentation about the Pod Security Context.
- `nodeSelectors` : (OpenFaaS CE & Pro)
- `tolerations` : (OpenFaaS CE & Pro) https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/ for a description and links to any additional documentation about Tolerations.
- `runtimeClassName` : (OpenFaaS Pro) https://kubernetes.io/docs/concepts/containers/runtime-class/ for a description and links to any additional documentation about Pod Runtime Class
- `topologySpreadConstraints` : (OpenFaaS Pro) (https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) for a description and links to any additional documentation about the Pod Security Context.
- `affinity` : (OpenFaaS Pro) https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity for a description and links to any additional documentation about Node Affinity.

The configuration use the exact options that you find in the Kubernetes documentation.

### Examples

#### Use an Alternative RuntimeClass

!!! info "OpenFaaS Pro feature"
    This feature is part of the [OpenFaaS Pro](/openfaas-pro/introduction) distribution.

A popular alternative container runtime class is [gVisor](https://gvisor.dev/) that provides additional sandboxing between containers. If you have created a cluster that is using gVisor, you will need to set the `runTimeClass` on the Pods that are created. This is not exposed in the OpenFaaS API, but it can be set via a Profile.

1. Install the latest `faas-netes` release and the CRD. The is most easily done with [`arkade`](https://github.com/alexellis/arkade)
    ```sh
    arkade install openfaas \
      --set openfaasPro=true
    ```
    This default installation will enable Profiles.

2. Create a Profile to apply the runtime class
    ```yaml
    kubectl apply -f- << EOF
    kind: Profile
    apiVersion: openfaas.com/v1
    metadata:
        name: gvisor
        namespace: openfaas
    spec:
        runtimeClassName: gvisor
    EOF
    ```

3. Let your developers know that they need to use this annotation

    ```
    com.openfaas.profile: gvisor
    ```

The following stack file will deploy a SHA512 generating file in a cluster with gVisor

```yaml
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080

functions:
  stronghash:
    skip_build: true
    image: functions/alpine:latest
    fprocess: "sha512sum"
    annotations:
      com.openfaas.profile: gvisor
```


#### Specify a nodeSelector to schedule functions to specific nodes

This example works for OpenFaaS CE. OpenFaaS Pro users should consider using TopologySpreadConstraints or Affinity rules.

> "nodeSelector is the simplest recommended form of node selection constraint. You can add the nodeSelector field to your Pod specification and specify the node labels you want the target node to have. Kubernetes only schedules the Pod onto nodes that have each of the labels you specify."

Read the Kubernetes docs: [Assign Pods to Nodes](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/)

Label a node as follows:

```yaml
cat <<EOF > kind-nodeselectors.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 31112
    hostPort: 31112
- role: worker
- role: worker
EOF

kind create cluster --config kind-nodeselectors.yaml

kubectl label node/kind-worker customer=1
kubectl label node/kind-worker2 customer=2
```

kind-worker will take on functions with a constraint of `customer=1` and kind-worker2 will take on the workloads for customer 2.

Now deploy a function with a nodeselector:

```yaml
cat <<EOF > stack.yml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  customer1-env:
    skip_build: true
    image: ghcr.io/openfaas/alpine:latest
    fprocess: env
    constraints:
    - "customer=1"
  customer2-env:
    skip_build: true
    image: ghcr.io/openfaas/alpine:latest
    fprocess: env
    constraints:
    - "customer=1"
EOF

faas-cli deploy stack.yml
```

Confirm the scheduling:

```
$ kubectl get deploy -o wide -n openfaas-fn

NAME            READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS      IMAGES                           SELECTOR
customer1-env   1/1     1            1           18s    customer1-env   ghcr.io/openfaas/alpine:latest   faas_function=customer1-env
customer2-env   1/1     1            1           18s    customer2-env   ghcr.io/openfaas/alpine:latest   faas_function=customer2-env
```

This will also work if you have several nodes dedicated to a particular customer, just apply the label to each node and add the constraint at deployment time.

You may also want to consider using a taint and toleration to ensure OpenFaaS workload components do not get scheduled to these nodes.


#### Spreading your functions out across different zones for High Availability

!!! info "OpenFaaS Pro feature"
    This feature is part of the [OpenFaaS Pro](/openfaas-pro/introduction) distribution.

The [topologySpreadConstraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) feature of Kubernetes provides a more flexible alternative to Pod Affinity / Anti-Affinity rules for scheduling functions.

> "You can use topology spread constraints to control how Pods are spread across your cluster among failure-domains such as regions, zones, nodes, and other user-defined topology domains. This can help to achieve high availability as well as efficient resource utilization."

Imagine a cluster with two nodes, each in a different availability zone.

Let's simulate that with KinD:

```yaml
cat <<EOF > kind-zones.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 31112
    hostPort: 31112
- role: worker
- role: worker
EOF

kind create cluster --config kind-zones.yaml

kubectl label node/kind-worker topology.kubernetes.io/zone=a
kubectl label node/kind-worker2 topology.kubernetes.io/zone=b
```

Deploy a profile called `env-tsc`

```yaml
kind: Profile
apiVersion: openfaas.com/v1
metadata:
    name: env-tsc
    namespace: openfaas
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        faas_function: env
```

Deploy a function with this profile:

```bash
faas-cli store deploy env \
  --annotation com.openfaas.profile=tsc
```

Scale the function to 6 replicas:

```bash
kubectl scale -n openfaas-fn deploy/env --replicas=6
```

Notice how the pods are spread evenly between the nodes in the two zones:

```bash
kubectl get pod -n openfaas-fn -o wide -w
NAME                   READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
env-5f5f697594-5clh6   1/1     Running   0          11s   10.244.2.13   kind-worker    <none>           <none>
env-5f5f697594-6fxhq   1/1     Running   0          21s   10.244.2.12   kind-worker    <none>           <none>
env-5f5f697594-92ntn   1/1     Running   0          11s   10.244.1.7    kind-worker2   <none>           <none>
env-5f5f697594-bz6sb   1/1     Running   0          11s   10.244.1.8    kind-worker2   <none>           <none>
env-5f5f697594-hsx98   1/1     Running   0          21s   10.244.1.6    kind-worker2   <none>           <none>
env-5f5f697594-nqxbh   1/1     Running   0          24s   10.244.2.11   kind-worker    <none>           <none>
```

A note on whenUnsatisfiable:

The constraint of `whenUnsatisfiable: DoNotSchedule` will mean pods are not scheduled if they cannot be balanced evenly. This may become an issue for you if your nodes are of difference sizes, therefore you may also want to consider changing this value to `ScheduleAnyway`

#### Use Tolerations and Affinity to Separate Workloads

Tolerations are available in the Community Edition, and could be used with NodeSelectors. The OpenFaaS API exposes the Kubernetes `NodeSelector` via [`constraints`](/docs/reference/yaml#function-constraints). This provides a very simple selection based on labels on Nodes.

This example is for OpenFaaS Pro because it uses Affinity.

The Kubernetes API also exposes two features affinity/anti-affinity and taint/tolerations that further expand the types of constraints you can express. OpenFaaS Profiles allow you to set these options, allowing you to more accurately isolate workloads, keep certain workloads together on the same nodes, or to keep certain workloads separate.

For example, a mixture of taints and affinity can put less critical functions on [preemptable vms](https://cloud.google.com/kubernetes-engine/docs/how-to/preemptible-vms) that are cheaper while keeping critical functions on standard nodes with higher availability guarantees.

In this example, we create a Profile using taints and affinity to place functions on the node with a GPU. We will also ensure that _only_ functions that require the GPU are scheduled on these nodes. This ensures that the functions that need to use the GPU are not blocked by other standard functions taking resources on these special nodes.

1. Install the latest `faas-netes` release and the CRD. The is most easily done with [`arkade`](https://github.com/alexellis/arkade)

    ```sh
    arkade install openfaas
    ```

    This default installation will enable Profiles.

2. Label _and_ Taint the node with the GPU

    ```sh
    kubectl label nodes <NODE_NAME> gpu=installed
    kubectl taint nodes <NODE_NAME> gpu:NoSchedule
    ```

3. Create a Profile to that allows functions to run on this node
    ```yaml
    kubectl apply -f- << EOF
    kind: Profile
    apiVersion: openfaas.com/v1
    metadata:
      name: withgpu
      namespace: openfaas
    spec:
        tolerations:
        - key: "gpu"
          operator: "Exists"
          effect: "NoSchedule"
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: gpu
                  operator: In
                  values:
                  - installed
    EOF
    ```

3. Then add this annotation to your stack.yml file:

    ```yaml
    com.openfaas.profile: withgpu
    ```
