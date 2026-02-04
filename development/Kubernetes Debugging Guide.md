# Kubernetes Debugging Guide

This guide provides a comprehensive workflow for debugging applications running on Kubernetes, tailored for beginners. We'll use the `staging` namespace as our example throughout this guide.

## 1. The Big Picture: What is Kubernetes?

Imagine you have a complex application made of many small services (microservices). Instead of running them all on one giant machine, you want to run them on a group of machines (a cluster) for better reliability and scalability.

Kubernetes is an orchestrator that manages this for you. It takes your applications, packages them into containers (using Docker, for example), and intelligently places them on the machines in the cluster. It handles:

- **Deployment:** Rolling out new versions of your application without downtime.
- **Scaling:** Automatically adding or removing container instances based on traffic.
- **Self-healing:** Restarting containers that fail, replacing them, and rescheduling them on healthy machines.
- **Service Discovery:** Allowing services to find and talk to each other easily.

Your main job as a developer is to package your application and tell Kubernetes how to run it. When things go wrong, your job is to ask Kubernetes what happened.

## 2. Prerequisites: Configuring Your Environment

Before you begin, open the Cloud Shell Terminal in Google Cloud Platform (GCP). This environment comes with `kubectl` pre-installed and configured for your GCP projects.

To talk to a Kubernetes cluster, you need a configuration file called `kubeconfig`. This file contains credentials and connection information.

### Check Your Setup

First, make sure `kubectl` (the Kubernetes command-line tool) is installed and check its version.

```bash
kubectl version

```

This shows both the client (kubectl) and server versions. They should be compatible.

Your `kubeconfig` file lets you connect to one or more clusters. A **context** is a combination of a cluster, a user, and a namespace.

```bash
# See all available contexts
kubectl config get-contexts

# See your current context
kubectl config current-context

# Switch to a different context
kubectl config use-context <context-name>

# See all namespaces in given environment
kubectl get namespaces
```

For this guide, we'll use `staging` as our example namespace. It's good practice to explicitly specify the namespace using the `-n` flag with each command (e.g., `kubectl get pods -n staging`).

*[Image: Screenshot of  all namespaces]*

## 3. Basic Pod Inspection: Is it running?

Pods are where your application containers run. The first step in any debugging is to check their status.

### Check Pod Status

To see all pods in your current namespace, run:

```bash
kubectl get pods
```

You will see a list of pods with their status. Here’s what the statuses mean:

- **Running:** The pod is running successfully.
- **Pending:** The pod is waiting to be scheduled on a machine (node) or the container image is still being pulled.
- **Completed:** The pod has finished its job (e.g., a one-off script) and will not be restarted.
- **Error:** The pod has an error.
- **CrashLoopBackOff:** The pod is starting, crashing, and being restarted in a loop. This is a common and important problem to debug.

### Useful `get pods` Commands

```bash
# Get pods in a different namespace
kubectl get pods -n <other-namespace>

# Get more details, including the IP address and the Node it's running on
kubectl get pods -o wide

# Find pods by a label (e.g., all pods for 'my-app')
# Labels are key-value pairs that help you organize and select resources.
kubectl get pods -l app=my-app

# See all labels for a pod
kubectl get pods --show-labels

# Get the pod's full definition in YAML format. Useful for deep inspection.
kubectl get pod <pod-name> -o yaml
```

*[Image: Screenshot of `kubectl get pods -o wide` output]*

### Describe the Pod: The Most Important Command

If a pod is not `Running`, the `describe` command is your best friend. It gives a detailed, human-readable summary of the pod, including its configuration and, most importantly, a log of recent events.

```bash
kubectl describe pod <pod-name>
```

Replace `<pod-name>` with the name of the pod you are debugging.

**What to look for in the output:**

- **Node:** Which machine in the cluster is the pod scheduled on? Is it scheduled at all?
- **Status:** The current state (e.g., `Pending`, `Running`, `Terminated`).
- **Containers:**
    - **Image:** Is the correct container image and tag being used?
    - **State:** The current state of the container (e.g., `Running`, `Waiting`).
    - **Last State:** If the container previously terminated, what was its state and exit code?
    - **Ready:** Is the container ready to accept traffic?
    - **Restarts:** How many times has this container been restarted? A high number indicates a crash loop.
- **Events:** **This is often the most critical section.** It's a timeline of what Kubernetes has been doing with your pod. Look for errors here.
    - `FailedScheduling`: Kubernetes couldn't find a suitable node. The message will explain why (e.g., "insufficient cpu").
    - `ImagePullBackOff` or `ErrImagePull`: The image couldn't be downloaded. Check for typos or authentication issues.
    - `OOMKilled`: The container was killed for using more memory than its limit.
    - `Unhealthy`: The container failed its liveness or readiness probe.

*[Image: Screenshot of `kubectl describe pod` output showing the Events section]*

## 4. Debugging Running Pods: What is it doing?

If a pod is `Running` but the application isn't working, you need to look inside.

### Check Logs

```bash
# Get the logs from a pod (if it has only one container)
kubectl logs <pod-name>

# Follow logs in real-time (like `tail -f`)
kubectl logs -f <pod-name>

# If the pod has multiple containers, you must specify which one
kubectl logs <pod-name> -c <container-name>

# Get logs from the last 50 lines
kubectl logs --tail=50 <pod-name>

# Get logs from the last 10 minutes
kubectl logs --since=10m <pod-name>
```

### Connect to a Running Container

You can get an interactive shell inside a running container to inspect its environment.

```bash
# Get a bash shell inside the container
kubectl exec -it <pod-name> -- /bin/bash

# If /bin/bash doesn't exist, try sh
kubectl exec -it <pod-name> -- /bin/sh
```

Once inside, you can run commands to check the environment:

- `env`: See all environment variables.
- `ls -l`: List files and check permissions.
- `cat <file-path>`: View configuration files.
- `ps aux`: See running processes.
- `netstat -tuln`: Check for listening ports.

### Forward a Port for Local Debugging

If your application has a web interface or an API, you can forward a port from the pod to your local machine. This lets you access it via `localhost`.

```bash
# Forward localhost:8080 to port 80 on the pod
kubectl port-forward <pod-name> 8080:80

```

Now, you can open `http://localhost:8080` in your browser to interact with the application directly.

## 5. Resource Usage: Is it overloaded?

Performance issues are often related to CPU or memory. Kubernetes requires the "Metrics Server" to be installed for these commands to work.

```bash
# Check CPU and Memory for a specific pod
kubectl top pod <pod-name>

# Check CPU and Memory for all pods
kubectl top pods

# Check the resource utilization of the nodes (the machines) themselves
kubectl top nodes
```

### Check Nodes

Nodes are the worker machines in your Kubernetes cluster.

```bash
# List all nodes
kubectl get nodes

# Get detailed information about a specific node
kubectl describe node <node-name>
```

**Understanding Resource Units:**

- **CPU:** Measured in "millicores" or "millicpus" (m). `1000m` is 1 full CPU core. `500m` is half a core.
- **Memory:** Measured in bytes. You will see suffixes like `Mi` (Mebibytes) and `Gi` (Gibibytes).

If a pod's memory usage is close to its limit (defined in the pod spec), it might be `OOMKilled`. You can see these limits with `kubectl describe pod <pod-name>`.

## 6. Debugging Crashing Pods (CrashLoopBackOff)

A `CrashLoopBackOff` means your container is starting, crashing, and Kubernetes is restarting it. Your goal is to find out *why* it's crashing.

1. **Check the logs of the *previous* container instance.** The currently running container is new, so its logs are likely empty. You need the logs from the one that just crashed. Use the `-previous` (or `p`) flag.
    
    ```bash
    kubectl logs -p <pod-name>
    ```
    
    This is the most important command for a `CrashLoopBackOff`. The error that caused the crash is almost always here.
    
2. **Describe the pod.** Check the `Last State` section for the container. It will show you the `Exit Code`.
    - **Exit Code 1:** A generic application error. The logs are essential here.
    - **Exit Code 137:** (128 + 9) The container was killed with signal 9 (SIGKILL). This almost always means it was `OOMKilled` for exceeding its memory limit.
    - **Exit Code 143:** (128 + 15) The container was killed with signal 15 (SIGTERM). This is a graceful shutdown. It might be a problem if it wasn't supposed to shut down.
3. **Check for configuration errors.** A common reason for crashing is a missing or incorrect configuration.
    - Use `kubectl exec` to get into the pod (if it stays running long enough) and verify that configuration files and environment variables are correct.
    - Double-check your Deployment YAML for typos in ConfigMap or Secret names.

*[Image: Diagram showing the CrashLoopBackOff cycle]*

## 7. Checking Related Objects

A pod is usually part of a larger system. If pods are not being created correctly, the problem may be in the object that manages them.

- **Deployments:** Manage a set of replica pods. You define your desired state in the Deployment, and it creates ReplicaSets to achieve that state.
    
    ```bash
    kubectl get deployments
    kubectl describe deployment <deployment-name>
    ```
    
- **Scaling Deployments:** If your application needs more or fewer instances, you can scale your deployment.
    
    ```bash
    # Scale a deployment to 3 replicas
    kubectl scale deployment <deployment-name> --replicas=3
    
    # Autoscale a deployment (requires Horizontal Pod Autoscaler setup)
    kubectl autoscale deployment <deployment-name> --min=2 --max=10 --cpu-percent=80
    ```
    
- **ReplicaSets:** Ensures that a specified number of pod replicas are running. You usually don't manage these directly; Deployments do.
- **Services:** Provides a stable IP address and DNS name for a set of pods. If you can't reach your application, check if the Service is configured correctly and targeting the right pods.
    
    ```bash
    kubectl get services
    kubectl describe service <service-name>
    ```
    
- **Ingress:** Manages external access to services in the cluster, typically for HTTP(S).
    
    ```bash
    kubectl get ingress
    kubectl describe ingress <ingress-name>
    ```
    

## 8. Debugging Istio Resources

If your application uses Istio, you'll have additional resources to check.

- **Gateways:** Manage inbound and outbound traffic for your mesh.
    
    ```bash
    kubectl get gateway -n <namespace>
    kubectl describe gateway <gateway-name> -n <namespace>
    ```
    
- **VirtualServices:** Define how requests are routed to services within the mesh.
    
    ```bash
    kubectl get virtualservice -n <namespace>
    kubectl describe virtualservice <virtualservice-name> -n <namespace>
    ```
    
- **DestinationRules:** Define policies that apply to traffic intended for a service after routing has occurred.
    
    ```bash
    kubectl get destinationrule -n <namespace>
    kubectl describe destinationrule <destinationrule-name> -n <namespace>
    ```
    
- **ServiceEntries:** Used to add entries into Istio's internal service registry that correspond to services that exist outside of the mesh.
    
    ```bash
    kubectl get serviceentry -n <namespace>
    kubectl describe serviceentry <serviceentry-name> -n <namespace>
    ```
    
- **Envoy proxy logs:** Each pod in an Istio mesh will have an `istio-proxy` sidecar container. Its logs can provide valuable insights into traffic routing and policy enforcement.
    
    ```bash
    kubectl logs <pod-name> -c istio-proxy -n <namespace>
    ```
    

## 9. Official Resources and Further Reading

For more in-depth information and to stay up-to-date, always refer to the official Kubernetes and Istio documentation:

- **Kubernetes Official Documentation:** [https://kubernetes.io/docs/](https://kubernetes.io/docs/)
- **kubectl Cheat Sheet:** [https://kubernetes.io/docs/reference/kubectl/cheatsheet/](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- **Istio Official Documentation:** [https://istio.io/latest/docs/](https://istio.io/latest/docs/)

## 10. Quick Reference Cheatsheet

| Task | Command |
| --- | --- |
| See all namespaces | `kubectl get namespaces` |
| See all pods in a namespace | `kubectl get pods -n <namespace>` |
| See pods with more details | `kubectl get pods -o wide -n <namespace>` |
| Get detailed info and events for a pod | `kubectl describe pod <pod-name> -n <namespace>` |
| See a pod's logs | `kubectl logs <pod-name> -n <namespace>` |
| Follow logs in real-time | `kubectl logs -f <pod-name> -n <namespace>` |
| See logs from a crashed pod | `kubectl logs -p <pod-name> -n <namespace>` |
| Execute a command in a pod | `kubectl exec -it <pod-name> -n <namespace> -- <command>` |
| Check a pod's resource usage | `kubectl top pod <pod-name> -n <namespace>` |
| See all deployments | `kubectl get deployments -n <namespace>` |
| Scale a deployment | `kubectl scale deployment <deployment-name> --replicas=X -n <namespace>` |
| See all services | `kubectl get services -n <namespace>` |
| See all nodes | `kubectl get nodes` |
| Get detailed info for a node | `kubectl describe node <node-name>` |
| See all Istio Gateways | `kubectl get gateway -n <namespace>` |
| See all Istio VirtualServices | `kubectl get virtualservice -n <namespace>` |
| Forward a local port to a pod | `kubectl port-forward <pod-name> <local-port>:<pod-port> -n <namespace>` |
| See your current k8s context | `kubectl config current-context` |

By following these steps, you can solve most common Kubernetes issues.
