# 1. Nodes
Nodes are virtual identities assigned by Kubernetes to the systems part of the cluster - whether Virtual Machines, bare-metal, Containers, etc. These identities are unique to each system, and are used by the cluster for resources accounting and monitoring purposes, which helps with workload management throughout the cluster.

Each node is managed with the help of two Kubernetes node agents - kubelet and kube-proxy, while it also hosts a container runtime. The container runtime is required to run all containerized workload on the node - control plane agents and user workloads. The kubelet and kube-proxy node agents are responsible for executing all local workload management related tasks - interact with the runtime to run containers, monitor containers and node health, report any issues and node state to the API Server, and manage network traffic to containers.

Node identities are created and assigned during the cluster bootstrapping process by the tool responsible to initialize the cluster agents. Minikube is using the default kubeadm bootstrapping tool, to initialize the control plane node during the init phase and grow the cluster by adding worker or control plane nodes with the join phase.

---

To understand how everything starts and connects from scratch, it helps to look at the bootstrapping process in two distinct phases: **Initializing the first node (The Control Plane)** and **Joining a worker node**.

Here is the step-by-step breakdown of how a tool like [kubeadm](https://trainingportal.linuxfoundation.org/learn/course/introduction-to-kubernetes/kubernetes-building-blocks-1/kubernetes-building-blocks?page=2) orchestrates this lifecycle.

---

### Phase 1: Bootstrapping the Control Plane (The Brain)

This is what happens when an administrator runs `kubeadm init` on the very first machine.

#### Step 1: Pre-flight Checks and CA Generation

Before running anything, the bootstrapping tool checks the machine to ensure it has a compatible container runtime (like `containerd`), enough CPU/RAM, and that swap memory is disabled.

* Once cleared, it generates a self-signed **Certificate Authority (CA)**. This CA is crucial because every single component in Kubernetes uses TLS certificates to securely verify who they are talking to.

#### Step 2: Starting the Local Agent (`kubelet`)

The tool writes a configuration file for the `kubelet` and starts it as a system service. At this exact moment, the `kubelet` is in a "waiting" state—it is running, but it doesn't have an API Server to talk to yet.

#### Step 3: Deploying the Control Plane Components

To solve the chicken-and-egg problem (running Kubernetes components before the cluster exists), the tool utilizes **Static Pods**. It drops YAML definition files for the core components directly into a specific folder on the host (usually `/etc/kubernetes/manifests/`).

* The local `kubelet` watches this directory. As soon as it sees the files, it instructs the container runtime to launch them as independent containers.

The core components boot up in this sequence:

1. **`etcd`:** The database starts first so there is a place to store data.
2. **`kube-apiserver`:** The central hub starts and connects to `etcd`.
3. **`kube-controller-manager` & `kube-scheduler`:** These components start and connect directly to the API Server.

#### Step 4: Creating the "Node" Object

Now that the API Server is fully alive, the local `kubelet` connects to it via a secure TLS handshake. The API Server registers this first machine, creating its digital identity inside `etcd`. The Control Plane node is now born.

---

### Phase 2: Bootstrapping and Joining a Worker Node

Once the Control Plane is healthy, it generates a secure **Bootstrap Token**. This token is what you use when you go to a blank VM and run `kubeadm join`.

#### Step 5: The Worker Knocks on the Door

When you run the `join` command on a worker machine, its local `kubelet` uses the token to contact the Control Plane's API Server.

* Because the worker doesn't have permanent security certificates yet, the token acts as a temporary backstage pass.

#### Step 6: Certificate Bootstrapping (TLS Handshake)

The API Server validates the token. If it matches, the worker node submits a **Certificate Signing Request (CSR)** to the Control Plane.

* The Controller Manager automatically approves the request, signs it using the cluster's internal CA, and sends unique, permanent TLS certificates back to the worker node.

#### Step 7: Final Assembly and Cluster Integration

With its new secure identity, the worker's `kubelet` can now fully connect to the API Server.

1. **Node Registration:** The API Server logs the worker's CPU, RAM, and storage capacity into `etcd`.
2. **Deploying `kube-proxy`:** The Control Plane sends a command back to the worker node to spin up `kube-proxy` (usually as a DaemonSet) to handle networking rules.
3. **Workload Acceptance:** The Scheduler sees the new worker node listed in the cluster's inventory and immediately marks it as `Ready` to accept application Pods.

Everything is now securely cross-connected, authenticated, and ready to scale.

---
## Q1. Why do we need kubelet on the control plane during Phase 1?

It sounds completely backward at first glance, right? The `kubelet` is supposed to be the "worker agent," so why do we need it on the machine that runs the brain (the Control Plane)?

The answer comes down to a classic engineering problem: **The Chicken-and-Egg Dilemma.**

To run the Control Plane (the API Server, Scheduler, and Controller Manager), Kubernetes needs to launch them as containers. But to launch containers, you need Kubernetes up and running. How do you start Kubernetes using Kubernetes when Kubernetes doesn't exist yet?

The `kubelet` is the secret weapon that solves this loop using a feature called **Static Pods**. Here is exactly how it works:

### 1. The Kubelet Doesn't Need an API Server to Start

Unlike every other component, the `kubelet` can run entirely on its own as a standard system service (like `systemd` on Linux). When `kubeadm` starts the `kubelet` during Phase 1, it tells the `kubelet`: *"Hey, don't look for a cluster API Server yet. Instead, just watch this specific local folder on the hard drive: `/etc/kubernetes/manifests/`."*

### 2. The Kubelet Acts as the "Bootstrap Engine"

Once the `kubelet` is running and watching that folder, `kubeadm` drops the YAML configuration files for the Control Plane components (`etcd.yaml`, `kube-apiserver.yaml`, etc.) right into that exact directory.

The `kubelet` spots the files and immediately says, *"I know what to do with these."* It directly tells the local container runtime (like `containerd`) to spin up those containers.

### 3. It Hands Over Control Once the Brain is Alive

As soon as those containers are running, the **API Server** comes alive. At that exact moment, the `kubelet` shifts gears. It connects to the brand-new API Server it just helped create, reports that the node is ready, and transitions into its normal job of managing the Control Plane node.

### Summary

Without the `kubelet` in Phase 1, there would be no software running on the host machine capable of talking to the container runtime to launch the API Server and `etcd`. The `kubelet` is essentially the construction worker that builds the management office it will eventually report to.

---
In a modern, standard Kubernetes cluster (especially one set up by `kubeadm`), the API Server, Scheduler, and Controller Manager are not installed as traditional software directly onto the host operating system. Instead, they run as **containers**—and yes, the `kubelet` is the component that creates and manages them.

To make sure it's crystal clear, this setup relies on a clever design loop.

### The "Static Pods" Trick

Normally, if you want to run a container in Kubernetes, you send a YAML file to the API Server, the Scheduler picks a node, and that node's `kubelet` runs the container.

But during the bootstrap phase, there *is* no API Server or Scheduler yet. To bypass this, Kubernetes uses a special feature called **Static Pods**:

1. **The Target Folder:** When you run `kubeadm init`, it writes the standard Kubernetes YAML configuration manifests for the `kube-apiserver`, `kube-scheduler`, and `kube-controller-manager` into a specific directory on the host: `/etc/kubernetes/manifests/`.
2. **The Kubelet's Watch Duty:** The `kubelet` is configured to constantly watch that specific folder.
3. **Local Execution:** As soon as the `kubelet` sees those files, it bypasses the entire cluster control plane. It talks directly to the local container runtime (like `containerd`) and says: *"Hey, spin up these containers immediately."*

### Why do it this way?

It sounds like a giant loop, but doing it this way offers two massive advantages for the cluster:

* **Self-Healing Brain:** Because the `kubelet` acts as a local watchdog, if the API Server container crashes or hangs, the `kubelet` will automatically detect the failure and restart it. The brain of your cluster fixes itself.
* **Unified Management:** It means Kubernetes is managed *by* Kubernetes. Once the cluster is fully online, you can actually see the API Server, Scheduler, and Controller Manager listed as Pods in the system namespace (`kube-system`), making them easy to monitor.

---

The `kubelet` is present on **every single control plane node**, and its primary job after the cluster boots up is to act as a relentless watchdog for those control plane containers.

If the `kube-apiserver`, `kube-scheduler`, or `kube-controller-manager` containers crash, run out of memory, or freeze up, the local `kubelet` immediately detects the failure and restarts them.

### Why this is a brilliant design

This setup means **Kubernetes uses its own core mechanics to keep itself alive**.

Instead of writing complex, custom Linux scripts to monitor the API Server process, the Kubernetes creators realized they already had a tool perfectly designed to monitor containers and keep them running: the `kubelet`.

### The One Exception: External Databases

The only major component that sometimes lives *outside* this `kubelet` container loop in large production environments is **`etcd`** (the database).

While tools like `kubeadm` will default to running `etcd` as a container managed by the `kubelet` (just like the API Server), many large companies choose to install `etcd` directly on separate, dedicated bare-metal servers or VMs using traditional system services (`systemd`). They do this because the database is so critical that they want to isolate it completely from the container runtime.

But for the main brains of the operations—the API Server, Scheduler, and Controller Manager—they are almost always containers watched over by a local `kubelet`.

---

To create these specific control plane containers, the `kubelet` bypasses the entire network and reads the **local YAML files** saved directly on the machine's hard drive.

Here is exactly how that looks on the system:

### 1. The Secret Folder

When you initialize a cluster, the bootstrapping tool writes exactly four YAML files into a specific directory on the host's file system:
📁 `/etc/kubernetes/manifests/`

If you were to log into that machine and list the files in that folder, you would see exactly this:

* `etcd.yaml`
* `kube-apiserver.yaml`
* `kube-controller-manager.yaml`
* `kube-scheduler.yaml`

### 2. How the Kubelet Uses Them

The `kubelet` has a configuration setting that tells it to constantly watch that specific folder.

* It reads the YAML files locally.
* It does **not** ask an API Server for permission.
* It talks directly to the local container runtime (like `containerd`) and says, *"Create containers based on these local files."*

### Why this distinction matters

These are called **Static Pods**.

For a normal application container (like a web server), the YAML file is sent over the network and stored in the database (`etcd`). The `kubelet` waits for the API Server to tell it to run it.

But for the Control Plane, the YAML files live permanently as physical files on that specific machine's disk. If you delete `kube-apiserver.yaml` from that folder, the `kubelet` will instantly delete the API Server container. If you drop it back in, the `kubelet` instantly recreates it.
