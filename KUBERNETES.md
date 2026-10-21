## Learning objectives

After studying these notes, you should be able to:

- Explain what Kubernetes is and why teams use it.
- Describe the relationship between Docker, container images, and Kubernetes.
- Explain clusters, nodes, Pods, Deployments, and Services.
- Describe desired state, reconciliation, scaling, and self-healing.
- Read a basic Kubernetes Deployment YAML file.
- Use beginner `kubectl` commands to inspect and manage resources.
- Demonstrate a simple application deployment on a local cluster.

---

# 1. What is Kubernetes?

**Kubernetes (often called K8s) is a platform for orchestrating containerized applications.** It helps deploy, scale, manage, and recover applications running in containers across a cluster of machines.

Imagine you have built a Node.js backend and packaged it into a container image. At first, one running copy may be enough. As the application grows, you may want multiple copies, a stable way to reach them, controlled updates, and recovery when an application instance fails.

Kubernetes helps manage those tasks.

### What Kubernetes helps with

- **Scaling:** Run more or fewer application replicas.
- **Self-healing:** Replace failed Pods when the workload controller is configured to maintain replicas.
- **Service discovery and networking:** Give a group of Pods a stable network endpoint.
- **Rollouts:** Manage application updates through Deployments.
- **Configuration:** Supply configuration separately from container images.
- **Desired-state management:** Continuously work toward the configuration you declared.

Kubernetes does not make every failure disappear automatically. Recovery depends on the workload configuration, cluster capacity, health of the cluster, and the kind of failure.

---

# 2. Docker and Kubernetes

Docker and Kubernetes are related, but they solve different problems.

| Technology        | Main role                                                        |
| ----------------- | ---------------------------------------------------------------- |
| Dockerfile        | Instructions for building a container image                      |
| Container image   | Packaged application and its runtime dependencies                |
| Container runtime | Runs containers                                                  |
| Kubernetes        | Orchestrates containerized applications across a cluster         |
| `kubectl`         | Command-line tool for communicating with a Kubernetes API server |

A common workflow is:

1. Write application code.
2. Create a Dockerfile.
3. Build a container image.
4. Make the image available to the cluster.
5. Configure Kubernetes to run the image in Pods.
6. Use Deployments and Services to manage and access the application.

**Important:** Kubernetes does not require the Docker Engine on every worker node. Kubernetes can use container runtimes such as `containerd`. Docker-built images can generally be used by compatible runtimes.

---

# 3. Kubernetes architecture

A Kubernetes **cluster** is the overall environment. It consists of a **control plane** and one or more **worker nodes**.

## 3.1 Control plane

The control plane manages the cluster.

| Component              | Beginner-friendly explanation                                                             |
| ---------------------- | ----------------------------------------------------------------------------------------- |
| **API server**         | The main entry point for Kubernetes API requests. `kubectl` communicates with it.         |
| **etcd**               | Stores Kubernetes cluster state and configuration data.                                   |
| **Scheduler**          | Chooses a suitable node for newly created Pods that have not yet been assigned to a node. |
| **Controller manager** | Runs controllers that observe the cluster and work toward the desired state.              |

A simplified sequence:

1. You ask Kubernetes to run an application, for example with `kubectl apply`.
2. The request reaches the API server.
3. Kubernetes stores the desired resource state.
4. Controllers observe the requested state.
5. The scheduler assigns Pods to nodes when needed.
6. Node components arrange for the containers to run.

This is a simplified teaching model; actual cluster operations involve asynchronous components and more details.

## 3.2 Worker node

A worker node runs application Pods.

| Component              | Role                                                                                 |
| ---------------------- | ------------------------------------------------------------------------------------ |
| **kubelet**            | Node agent that works to ensure assigned Pods and their containers run as specified. |
| **Container runtime**  | Runs the containers, such as `containerd`.                                           |
| **Network components** | Help implement Pod and Service networking, depending on the cluster setup.           |

For beginner explanations, remember:

- **Control plane:** makes and coordinates cluster-level decisions.
- **Worker node:** runs application workloads.

---

# 4. Core Kubernetes objects

These are the most important beginner concepts.

## 4.1 Cluster

The complete Kubernetes environment, including its control plane and worker nodes.

## 4.2 Node

A machine that participates in a Kubernetes cluster. A node can be a virtual machine or a physical machine.

## 4.3 Pod

A **Pod is the smallest deployable unit in Kubernetes**. It contains one or more closely related containers that share certain resources, such as network identity and configured storage.

Most introductory application workloads use one main application container per Pod.

A Pod is not the same as a Deployment. Pods can be replaced, so you should not rely on a particular Pod's identity or IP address remaining permanent.

## 4.4 Deployment

A **Deployment** manages a set of replicated Pods and supports controlled updates.

For example, setting `replicas: 3` tells Kubernetes that you want three replicas of the application. The Deployment's controller works toward that desired state.

You generally manage stateless application replicas through Deployments rather than creating each Pod manually.

## 4.5 Service

A **Service** provides a stable network endpoint for accessing a selected group of Pods. Since Pods may be replaced and their IP addresses may change, applications should not generally rely on an individual Pod IP as a permanent endpoint.

A Service uses selectors and labels to identify the Pods that should receive traffic.

### Common Service types

| Type             | Basic meaning                                                                                                            |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **ClusterIP**    | Makes the Service reachable from inside the cluster. This is the default type.                                           |
| **NodePort**     | Exposes the Service on a port on each eligible node, subject to cluster networking and configuration.                    |
| **LoadBalancer** | Requests an external load balancer from a supporting environment or provider. Exact behavior depends on the environment. |

**Port forwarding is different from exposing a Service publicly.** `kubectl port-forward` creates a temporary connection from your local machine to a Kubernetes resource.

## 4.6 ConfigMap and Secret

- **ConfigMap:** Stores non-confidential configuration data.
- **Secret:** Intended for sensitive configuration such as credentials or tokens.

Secrets require appropriate access controls and careful handling; the name “Secret” alone does not guarantee that data is secure.

## 4.7 PersistentVolume and PersistentVolumeClaim

These relate to storage:

- **PersistentVolume (PV):** A storage resource made available to the cluster.
- **PersistentVolumeClaim (PVC):** A request for storage by a workload.

They are useful when data must survive beyond the lifecycle of an individual Pod. Storage is a later topic for our learning path.

---

# 5. Desired state, reconciliation, and self-healing

These ideas explain much of Kubernetes' behavior.

## Desired state

The state you ask Kubernetes to maintain. For example:

> “I want three replicas of this application.”

## Actual state

What currently exists in the cluster. For example, only two replicas may be running after one Pod fails.

## Reconciliation

Kubernetes controllers continually observe resources and work toward making the actual state match the desired state.

A simplified example:

1. Desired replica count: **3**.
2. Actual running replicas: **2**.
3. The Deployment controller detects the mismatch.
4. Kubernetes works to create a replacement Pod.
5. The workload should return to three replicas if the cluster can schedule and run the replacement.

## Self-healing demonstration

In our introductory lab, we deleted a Pod managed by a Deployment. The Deployment controller created a replacement because the desired replica count had not changed.

**Precision matters:** Kubernetes usually replaces a failed or deleted Pod rather than repairing that exact Pod. A replacement may have a different name and IP address. Recovery is not guaranteed if, for example, the cluster has insufficient resources or cannot pull the required image.

---

# 6. Scaling

Scaling changes the number of application replicas.

For a Deployment named `k8s-demo`, scale it to three replicas:

```powershell
kubectl scale deployment k8s-demo --replicas=3
```

Check the Pods:

```powershell
kubectl get pods
```

Watch changes:

```powershell
kubectl get pods -w
```

Press `Ctrl+C` to stop watching.

The Deployment controller works toward the replica count you requested. Scaling does not necessarily mean that traffic is automatically exposed outside the cluster; networking and Service configuration are separate concerns.

---

# 7. Labels and selectors

**Labels** are key-value pairs attached to Kubernetes objects. They help organize and identify resources.

Example label:

```yaml
labels:
  app: web-app
```

A Deployment selector can choose Pods with that label:

```yaml
selector:
  matchLabels:
    app: web-app
```

The Pod template must include labels that match the Deployment selector. Otherwise, the Deployment will not manage the intended Pods correctly.

Services also use selectors to identify which Pods should receive traffic.

Useful command:

```powershell
kubectl get pods --show-labels
```

Filter Pods by label:

```powershell
kubectl get pods -l app=web-app
```

---

# 8. Kubernetes YAML and declarative configuration

Kubernetes resources are often described in YAML manifest files.

Instead of expressing every setting in a long command, you declare the desired configuration in a file and apply it.

## Example: Deployment manifest

Save the following as `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: web-app

spec:
  replicas: 3

  selector:
    matchLabels:
      app: web-app

  template:
    metadata:
      labels:
        app: web-app

    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
```

## Explanation of the fields

| Field                      | Meaning                                                                                                      |
| -------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `apiVersion: apps/v1`      | API group/version for the Deployment resource                                                                |
| `kind: Deployment`         | Type of Kubernetes resource                                                                                  |
| `metadata.name`            | Resource name                                                                                                |
| `spec.replicas`            | Desired number of replicas                                                                                   |
| `selector.matchLabels`     | Labels the Deployment uses to identify its Pods                                                              |
| `template`                 | Blueprint used to create Pods                                                                                |
| `template.metadata.labels` | Labels attached to the Pods                                                                                  |
| `containers`               | Container definitions inside each Pod                                                                        |
| `image`                    | Container image to run                                                                                       |
| `containerPort`            | Documents the port the container is expected to use; it does not publish the port to your computer by itself |

## Apply and inspect the manifest

```powershell
kubectl apply -f deployment.yaml
```

```powershell
kubectl get deployments
kubectl get pods -l app=web-app
```

Change `replicas: 3` to `replicas: 5`, save the file, and apply it again:

```powershell
kubectl apply -f deployment.yaml
kubectl get pods -l app=web-app
```

Kubernetes will work toward the new desired replica count.

## Imperative vs. declarative

| Approach    | Example                                                   | Meaning                                               |
| ----------- | --------------------------------------------------------- | ----------------------------------------------------- |
| Imperative  | `kubectl create deployment k8s-demo --image=nginx:alpine` | Directly issue a command to create a resource.        |
| Declarative | `kubectl apply -f deployment.yaml`                        | Apply the resource configuration described in a file. |

Both approaches are useful. Manifest files are a common way to manage and review configuration in real projects.

---

# 9. `kubectl` command reference

`kubectl` is the command-line tool used to communicate with a Kubernetes API server.

## Check cluster access

```powershell
kubectl version --client
kubectl config current-context
kubectl config get-contexts
kubectl cluster-info
kubectl get nodes
```

- `kubectl version --client`: displays client version information.
- `kubectl config current-context`: displays the selected context, if one is set.
- `kubectl config get-contexts`: lists configured contexts.
- `kubectl cluster-info`: displays cluster endpoint information when connected.
- `kubectl get nodes`: lists nodes and their status.

A **context** identifies the cluster, user credentials, and optionally the namespace used by `kubectl`. If no context is set, `kubectl` may try a default endpoint and fail. The fix is to inspect the configuration and select a valid context—not to assume the cluster itself is broken.

## Inspect resources

```powershell
kubectl get deployments
kubectl get pods
kubectl get pods -o wide
kubectl get services
kubectl get endpointslices
kubectl get pods -A
```

- `-o wide`: adds extra details, such as node and IP information where applicable.
- `-A`: lists resources across all namespaces.

## Investigate problems

```powershell
kubectl describe deployment k8s-demo
kubectl describe pod POD_NAME
kubectl logs POD_NAME
kubectl get events
```

Replace `POD_NAME` with the actual Pod name. The `describe` output often includes useful events near the bottom.

## Apply and remove resources

```powershell
kubectl apply -f deployment.yaml
kubectl delete -f deployment.yaml
kubectl delete deployment k8s-demo
```

**Warning:** `delete` removes resources. Check the resource name and namespace before running deletion commands.

---

# 10. The practical lab completed

This lab was performed on Windows using Docker Desktop's local Kubernetes cluster.

## Step 1: Verify the cluster

```powershell
kubectl config current-context
kubectl get nodes
kubectl get pods -A
```

The cluster was initially unavailable because no current context was set and no cluster entries were listed. After Kubernetes was started in Docker Desktop, the learner continued with the lab.

## Step 2: Create a Deployment

```powershell
kubectl create deployment k8s-demo --image=nginx:alpine
kubectl get deployments
kubectl get pods
```

This created a Deployment named `k8s-demo` using the `nginx:alpine` image.

## Step 3: Access the application locally

```powershell
kubectl port-forward deployment/k8s-demo 8080:80
```

Then open `http://localhost:8080` in a browser while the port-forward command is running.

- `8080`: local Windows port.
- `80`: target port in the application Pod.
- Keep the terminal open while using the forwarded connection.
- Press `Ctrl+C` to stop port forwarding.

## Step 4: Scale the Deployment

```powershell
kubectl scale deployment k8s-demo --replicas=3
kubectl get pods -w
```

The learner observed the Deployment working toward three replicas.

## Step 5: Create a Service

```powershell
kubectl expose deployment k8s-demo --name=k8s-demo-svc --port=80 --target-port=80 --type=ClusterIP
```

Inspect the Service:

```powershell
kubectl get services
kubectl describe service k8s-demo-svc
kubectl get endpointslices
```

Access it through local port forwarding:

```powershell
kubectl port-forward service/k8s-demo-svc 8081:80
```

Open `http://localhost:8081` while the command is running.

## Step 6: Demonstrate self-healing

First list the Pods:

```powershell
kubectl get pods
```

Delete one actual Pod name:

```powershell
kubectl delete pod POD_NAME
```

Then watch:

```powershell
kubectl get pods -w
```

The Deployment controller works to restore the desired number of replicas.

## Step 7: Create a YAML-based Deployment

Create `deployment.yaml` using the manifest in Section 8, then run:

```powershell
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods -l app=web-app
```

Changing `replicas` in the file and applying it again demonstrates declarative management.

---

# 11. Common misunderstandings

### “A Pod is the same thing as a Deployment.”

No. A Pod runs one or more containers. A Deployment manages replicated Pods and their updates.

### “A Service is the same as port forwarding.”

No. A Service is a Kubernetes resource that provides a stable endpoint for selected Pods. Port forwarding is a temporary local connection created by `kubectl`.

### “`containerPort: 80` exposes port 80 on my laptop.”

No. It describes the container's intended port. Use a Service and the appropriate access method to reach the application.

### “Kubernetes always fixes every failure.”

No. Controllers work toward the desired state, but recovery can be blocked by insufficient capacity, unavailable images, configuration errors, or cluster problems.

### “Kubernetes requires Docker Engine on every node.”

No. Kubernetes works with compatible container runtimes, including `containerd`.

### “A ClusterIP Service is public on the internet.”

No. ClusterIP is intended for internal cluster access. External access requires an appropriate mechanism and network configuration.

### “A context is the same as a cluster.”

No. A context is a client configuration that points `kubectl` to a cluster and identifies credentials and optionally a namespace.

---

### Questions

1. What problem does Kubernetes solve?
2. How is a Pod different from a Deployment?
3. Why do applications use a Service instead of depending on a Pod IP?
4. What happens if one Pod managed by a Deployment is deleted?
5. What does `replicas: 3` express in a Deployment manifest?
6. Does `containerPort: 80` automatically publish the application to the internet?
7. What is the difference between a ClusterIP Service and `kubectl port-forward`?
8. What does `kubectl apply -f deployment.yaml` do?

### Short answer key

1. It orchestrates containerized applications.
2. A Pod runs containers; a Deployment manages replicated Pods and updates.
3. Pod identities and IP addresses can change; a Service provides a stable endpoint.
4. The Deployment controller works to restore the desired replica count.
5. It declares the desired number of replicas.
6. No.
7. ClusterIP is an in-cluster Service endpoint; port forwarding creates a temporary local connection.
8. It applies the configuration described in the manifest file.

---
