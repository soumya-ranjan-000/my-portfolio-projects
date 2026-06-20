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

---
The control plane nodes run the control plane agents, such as the API Server, Scheduler, Controller Managers, and etcd in addition to the kubelet and kube-proxy node agents, the container runtime, and add-ons for container networking, monitoring, logging, DNS, etc.

Worker nodes run the kubelet and kube-proxy node agents, the container runtime, and add-ons for container networking, monitoring, logging, DNS, etc.

Collectively, the control plane node(s) and the worker node(s) represent the Kubernetes cluster. A cluster’s nodes are systems distributed either on the same private network, across different networks, even across different cloud networks.

# Namespaces

Think of a Kubernetes cluster as a **massive tech company building**.

If everyone in the company worked in one giant, open room without any walls, it would be pure chaos. Marketing might accidentally delete Engineering’s whiteboard drawings, and HR might look at confidential sales data.

To fix this, the company builds **floors and offices** to separate teams. In Kubernetes, those virtual walls are called **Namespaces**.

---

## 🏢 The Real-World Analogy

Imagine a building divided into two main sections: the **Marketing Team Floor** and the **Engineering Team Floor**.

* **Duplicate Names Allowed:** The Marketing team has a printer named `printer-01`. The Engineering team *also* has a printer named `printer-01`. Because they are on completely different floors, there is no confusion. If you are on the Marketing floor and say "print to `printer-01`," everyone knows exactly which one you mean.
* **Resource Quotas:** The building manager might decide that the Marketing floor is only allowed to use 10 reams of paper per week, while Engineering gets 50.

In Kubernetes, the "floors" are **Namespaces**, the "teams" are your developers, the "printers" are your **Pods/Services**, and the "paper limits" are **Resource Quotas**.

---

## 🗺️ How it Looks

Here is a visual map of how a single physical cluster is divided into virtual sub-clusters (Namespaces):

```
+------------------------------------------------------------------------+
|                       KUBERNETES CLUSTER (The Building)                |
|                                                                        |
|  +---------------------------+        +-----------------------------+  |
|  |  NAMESPACE: MARKETING     |        |   NAMESPACE: ENGINEERING    |  |
|  |  (Floor 1)                |        |   (Floor 2)                 |  |
|  |                           |        |                             |  |
|  |  [Pod: web-app]           |        |   [Pod: web-app]            |  |
|  |  (Marketing Version)      |        |   (Engineering Version)     |  |
|  |                           |        |                             |  |
|  |  [Service: database]      |        |   [Service: database]       |  |
|  +---------------------------+        +-----------------------------+  |
+------------------------------------------------------------------------+

```

### Request Flow: How Traffic Finds the Right Resource

When an application wants to talk to a database, Kubernetes uses the Namespace to route the request correctly:

```
[ Your Request ] 
       │
       ▼
Is a Namespace specified?
       │
       ├─► YES ──► Route directly to that Namespace (e.g., Engineering) ──► Finds [database]
       │
       └─► NO  ──► Automatically routes to the [default] Namespace  ──► Finds [database]

```


* **Definition:** Namespaces partition a single physical cluster into multiple **virtual sub-clusters**. They provide unique naming scopes, meaning two different resources can share the exact same name as long as they live in different Namespaces.
* **The Out-of-the-Box Namespaces:** Kubernetes creates four default namespaces automatically:
1. `default`: Where your apps go by default if you don't specify a namespace.
2. `kube-system`: The highly sensitive area reserved for Kubernetes' own internal features and control plane agents.
3. `kube-public`: An unsecured, publicly readable namespace used for cluster-wide public data.
4. `kube-node-lease`: A newer namespace that tracks cluster node heartbeats (node leases) to ensure they are alive.

* **Key Commands:**
* To view namespaces: `kubectl get namespaces`
* To create a namespace: `kubectl create namespace new-namespace-name`

---

**Guardrails (Limits):** To prevent one namespace from hogging all the cluster's CPU or memory, administrators use **Resource Quotas** (to limit total namespace consumption) and **LimitRanges** (to set min/max resource constraints on individual containers).

---

# 📦 Kubernetes Pods


![image.png](https://raw.githubusercontent.com/soumya-ranjan-000/image-hosting/main/articles/kubernetes-building-blocks/1780426719022-image.png)



## 💡 What is a Pod?
* **Definition:** A [Pod](https://trainingportal.linuxfoundation.org/learn/course/introduction-to-kubernetes/kubernetes-building-blocks-1/kubernetes-building-blocks?page=4) is the **smallest, most basic deployable workload object** in Kubernetes. It represents a single instance of a running application.
* **Composition:** A logical collection of **one or more containers** enclosed and isolated together.

---
To understand how a Pod is built and what it looks like under the hood, we have to look at it from two different angles: **how Kubernetes builds it technically**, and **what the structural blueprint looks like**.

---

## 🛠️ How a Pod is Built (The Mechanics)

A Pod isn't a physical object or an actual piece of software; it is a **Linux abstraction**. When you tell Kubernetes to create a Pod, here is exactly what happens behind the scenes:

1. **The Blueprint:** You hand a YAML or JSON file to the Kubernetes API.
2. **The Evaluation:** The `kubelet` (the worker bee agent on a node) looks at your configuration.
3. **The Infra Container (The Secret Ingredient):** Before your actual application container starts, Kubernetes launches a hidden container called the **`pause` container** (or infrastructure container).
* This tiny container does nothing except hold open a Linux network namespace and obtain an IP address.


4. **The App Containers Join In:** Your actual application containers (like Nginx) are then launched inside that exact same network namespace. Because they share the same namespace as the `pause` container, they instantly share the same IP address and can talk to each other via `localhost`.

---

## 🎨 What a Pod Looks Like (The Structure)

Visually, a Pod looks like an isolated bubble containing shared infrastructure that any container inside the bubble can access.

As shown in the course material diagram on [Single- and Multi-Container Pods](https://trainingportal.linuxfoundation.org/learn/course/introduction-to-kubernetes/kubernetes-building-blocks-1/kubernetes-building-blocks?page=4), a Pod acts as a bounding circle enclosing the IP address, storage volumes, and containerized apps. Here is a text-based breakdown of that structure:

```
+-------------------------------------------------------------+
|                     POD BOUNDARY                            |
|  (Has a single Cluster IP Address: e.g., 10.10.10.4)        |
|                                                             |
|   +------------------+             +--------------------+   |
|   |  CONTAINER 1     |             |  CONTAINER 2       |   |
|   |  (e.g., Web App) |             |  (e.g., Log Agent) |   |
|   +--------┬---------+             +---------┬----------+   |
|            │                                 │              |
|            │     Both talk via localhost     │              |
|            └─────────────────────────────────┘              |
|                                                             |
|   +─────────────────────────────────────────────────────+   |
|   |                SHARED STORAGE VOLUME                |   |
|   | (Both containers can read/write to this same space) |   |
|   +─────────────────────────────────────────────────────+   |
+-------------------------------------------------------------+

```

---

## 📄 The Blueprint (What it looks like in Code)

If you look at a Pod in your code editor, it is defined as a declarative manifest block. Every single Pod is built using **four required root fields**:

```yaml
apiVersion: v1        # 1. Ruleset version being used
kind: Pod             # 2. The type of object you are building
metadata:             # 3. Workload accounting (Names, labels, identification)
  name: my-web-pod
  labels:
    env: production
spec:                 # 4. The actual technical specifications for the containers
  containers:
  - name: web-container
    image: nginx:1.22.1
    ports:
    - containerPort: 80

```

---

## ⚙️ Key Architectural Characteristics
* **Co-scheduling:** All containers inside a single Pod are always scheduled together on the **same physical or virtual host node**.
* **Shared Network Namespace:** Containers within a Pod share a **single IP address** assigned to the Pod.
  * They communicate with each other locally via `localhost`.
* **Shared Storage:** Containers can mount the **same external storage volumes** and share common dependencies.
* **Ephemeral Nature:** Pods are temporary and disposable. They **do not have self-healing capabilities** on their own.

---

## 🛠️ Management & Orchestration
Because Pods are ephemeral, they are rarely managed as standalone objects in production. Instead, they are managed by **Controllers or Operators** that handle replication, fault tolerance, and self-healing.

### Common Controllers:
* Deployments
* ReplicaSets
* DaemonSets
* Jobs

> **Note:** When managed by a controller, the Pod's specific configuration is nested inside the controller's definition using a **Pod Template**.

---

## 📄 Manifest Structure (YAML vs. Imperative)

### 1. Declarative Method (YAML Template)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    run: nginx-pod
spec:
  containers:
  - name: nginx-pod
    image: nginx:1.22.1
    ports:
    - containerPort: 80

```

* **Required Fields:** `apiVersion` (always `v1` for Pods), `kind` (Pod), `metadata` (names, labels), and `spec` (defines the desired container state, evaluated by the `kubelet`).


### 2. Imperative Method (Quick Run)

To run a Pod instantly without a file:

```bash
kubectl run nginx-pod --image=nginx:1.22.1 --port=80

```

### 3. Generating a Starter Template (The Lifecycle Lifesaver)

To safely generate a YAML or JSON blueprint file without actually executing it in the cluster, use a dry run:

* **YAML:** `kubectl run nginx-pod --image=nginx:1.22.1 --port=80 --dry-run=client -o yaml > nginx-pod.yaml`
* **JSON:** `kubectl run nginx-pod --image=nginx:1.22.1 --port=80 --dry-run=client -o json > nginx-pod.json`

---

## 📟 Essential Pod Commands

* **Deploy/Update from file:** `kubectl apply -f nginx-pod.yaml` (or `kubectl create -f nginx-pod.yaml`)
* **List all pods:** `kubectl get pods`
* **View running configuration (YAML):** `kubectl get pod nginx-pod -o yaml`
* **View running configuration (JSON):** `kubectl get pod nginx-pod -o json`
* **Inspect / Debug a pod:** `kubectl describe pod nginx-pod`
* **Delete a pod:** `kubectl delete pod nginx-pod`

---
**Containers are created directly on the host machine, not "inside" some magical physical bubble called a Pod.**

A Pod is not a virtual machine, and it is not a physical cage. A Pod is an **illusion** created by the Linux kernel using two features: **Namespaces** (for isolation) and **Cgroups** (for resource limits).

Here is exactly how a Pod looks in memory, what it takes from the host, and how containers are born into it.

---

## 👁️ What a Pod Looks Like to the Host Machine (The Reality)

If you log directly into the worker node (the host Linux machine) and look at the memory and CPU, **you won’t find anything called a "Pod."** To the host operating system, a Pod is just a collection of standard Linux processes that have been tricked into sharing the same sandbox.

```
WHAT KUBERNETES SEES (The Illusion):
+------------------------------------------+
| Pod: my-web-pod                          |
|  ├── [Container: Nginx App]              |
|  └── [Container: Logger App]             |
+------------------------------------------+

WHAT THE HOST OS SEES (The Reality):
[Linux Kernel Memory / Process Tree]
 ├── PID 4012 (pause)        <-- The Pod's anchor
 ├── PID 4055 (nginx)        <-- The App (shares Network Namespace of 4012)
 └── PID 4090 (fluentd)      <-- The Logger (shares Network Namespace of 4012)

```

---

## 🧾 What Exactly Does a Pod Get From the Host System?

When Kubernetes allocates resources for a Pod, it carves out specific items from the host kernel:

1. **A Network Namespace (The IP):** The host allocates **one single IP address** from its network bridge and assigns it to a virtual network interface.
2. **Cgroup Slices (The Resource Limits):** The host limits how much CPU and RAM those processes can collectively hog. If your Pod spec says `limits: memory: "512Mi"`, the host kernel enforces a hard 512MB limit on the total memory used by *all* containers in that group.
3. **Storage Mounts:** The host takes a directory from its own hard drive (or a network drive) and maps it so the specific processes can see it.

---

## 🏗️ Step-by-Step: How a Container is Created "Inside" a Pod

Since containers are created on the host, how do they end up sharing a Pod? This is where the **`pause` container** comes in.

Every time you create a Pod, the Container Runtime (like `containerd` or Docker) performs a three-step dance on the host system:

### Step 1: The Parent (Pause) Process is Born

The host runtime creates a tiny container running a piece of code called `pause`. This process does absolutely nothing—it just goes to sleep. But because it exists, the host Linux kernel creates a clean **Network Namespace** and attaches a unique IP address to it.

* *This `pause` process is the actual physical foundation of your Pod.*

### Step 2: The Application Container is Born on the Host

Next, the host runtime starts your actual application container (e.g., Nginx). It is created as a normal process directly on the host's CPU and memory.

### Step 3: The Join

Instead of giving Nginx its own new IP address, the runtime tells the Linux kernel: *"Take this new Nginx process, and force it to use the **Network Namespace** of the `pause` process we created in Step 1."*

Because they share the exact same network sandbox, the Nginx container instantly inherits the Pod's IP address. If you add a third container, it joins the same namespace.

---

## 🧠 Summary: The Living Room Analogy

Think of the host machine as a giant house.

* A **Container** is just a person.
* A **Pod** isn't a separate room built inside the house; a Pod is simply a rule that says: *"These three specific people must sit on the same couch, share the same plate of food (Storage), and talk using the same phone line (IP Address)."*

They are all standing on the exact same living room floor (the Host OS), but they are bound together by the rules of the cluster.

Does seeing how the `pause` container anchors the network help clear up how multiple containers can share one single IP address?


----

**A Pod is nothing more than a group of standard Linux processes engineered to share specific Namespaces and Cgroups.**



## 🧩 1. Linux Namespaces (The Isolation Walls)

A Linux **Namespace** is a feature of the Linux kernel that isolates system resources. When you place a process in a namespace, it behaves as if it is the *only* process on the computer with access to that resource. It cannot see or interact with anything outside its assigned boundary.

The Linux kernel has several types of namespaces, but three are critical to understanding how Pods work:

### 🌐 Network Namespace (`net`)

* **What it does:** Isolates network devices, IP addresses, routing tables, and port mappings.
* **How a Pod uses it:** When a Pod is created, Kubernetes spins up the `pause` container, creating **one single Network Namespace** for the Pod. Every container that joins the Pod later is forced into this exact same Network Namespace. Because they share it, Container A and Container B share the same IP address, can see the same ports, and can communicate using `localhost`.

### 🗂️ Process ID Namespace (`pid`)

* **What it does:** Isolates the process tree. Inside a PID namespace, a process can think it is **PID 1** (the master system process), while the host system sees it as a random, non-privileged process number like PID 24053.
* **How a Pod uses it:** By default, containers in a Pod get their own isolated PID namespaces so they can't see each other's active processes. However, you can configure a Pod manifest to share a PID namespace (`shareProcessNamespace: true`), allowing Container A to view and signal processes running inside Container B.

### 💾 Mount Namespace (`mnt`)

* **What it does:** Isolates filesystem mount points. A process inside a mount namespace sees a completely different file tree than a process outside it.
* **How a Pod uses it:** This is how your container thinks it is running a pure Ubuntu or Alpine OS filesystem, even though it is just running on a standard RedHat or Ubuntu host system.

---

## 📈 2. Control Groups / Cgroups (The Resource Budget)

While namespaces block a process's *sight*, **Cgroups** block its *appetite*.

Cgroups (Control Groups) are a Linux kernel feature that limits, polices, and accounts for resource usage (CPU, Memory, Disk I/O, Network bandwidth) for a collection of processes.

```
                  [ HOST OS KERNEL ]
                           │
       ┌───────────────────┴───────────────────┐
       ▼                                       ▼
  [ Namespace ]                           [ Cgroup ]
"What can this process                 "How much CPU and RAM
    SEE and TALK to?"                    can this process USE?"
 (Isolates IP, Files, PIDs)              (Enforces 512MB RAM limits)

```

When you define resource limits in your Pod blueprint:

```yaml
resources:
  limits:
    memory: "512Mi"
    cpu: "500m"

```

The host operating system creates a dedicated cgroup directory inside its kernel filesystem (typically under `/sys/fs/cgroup/`). The host tells the kernel: *"Any process associated with this Pod cannot collectively cross 512MB of RAM or use more than half of a CPU core."* If the containers try to allocate 513MB, the host kernel steps in and kills the offending container process with an **OOMKilled** (Out Of Memory) error.

---

## ⚙️ Putting It All Together: How a Container "Joins" a Pod

This is the exact mechanism of how a container engine (like `containerd` or `Docker`) creates a container on the host machine but locks it inside a Pod's configuration.

Linux provides a system call command called **`setns()`**. This command allows a brand new process to leave its own default namespace and **jump directly into an existing namespace** belonging to a different process.

### The Step-by-Step Birth of a Pod:

```
Step 1: Host creates [Pause Process] ──► Generates Network Namespace #402
                                                    │
Step 2: Host spawns [Nginx Process]                 │
          └─► Runtime executes setns() ─────────────┼──► Joins Network Namespace #402
                                                    │
Step 3: Host spawns [Logging Process]               │
          └─► Runtime executes setns() ─────────────┘──► Joins Network Namespace #402

```

1. **The Infrastructure Anchor:** Kubernetes instructs the host container runtime to create the `pause` container process. The host Linux kernel sets up a fresh, unique Network Namespace (let's call it Namespace `#402`) and gives it a dedicated IP address.
2. **The App Injection:** The runtime is told to start your application container (e.g., Nginx). The host OS launches the Nginx process directly onto the host's CPU and RAM.
3. **The `setns()` Magic:** Before the Nginx process fully awakens, the container runtime invokes the Linux `setns()` system call. It essentially says to the Nginx process: *"Do not build your own network configuration. Instead, target file path `/proc/<pause-PID>/ns/net` and merge into Network Namespace #402."*
4. **Enforcing the Limits:** The runtime drops all of these running host PIDs into the same directory within the host's `/sys/fs/cgroup/kubepods/` pathway, cementing their resource budgets.

### 💡 The Takeaway

A Pod is a collection of standard host processes that have been organized into a single family by the container runtime. They are isolated from the rest of the host by **Namespaces**, throttled by **Cgroups**, and welded together because they have all used `setns()` to step into the exact same network sandbox environment.

# Labels


### 1. Grouping and Organizing Objects

Kubernetes clusters can run thousands of Pods, Services, and Nodes. Labels give you a way to categorize them into meaningful structures based on your real-world architecture. Common examples include:

* **Environment:** `env: dev`, `env: qa`, `env: prod`
* **Application Tier:** `app: frontend`, `app: backend`, `app: database`
* **Release Track:** `release: stable`, `release: canary`

### 2. Loose Coupling (The Core Reason)

In Kubernetes, components like **ReplicaSets**, **Deployments**, and **Services** don't hardcode the specific names of the Pods they manage. Instead, they use **Label Selectors**.

As shown in your image, if a Service wants to send traffic only to the QA frontend, it uses a selector to look for Pods that match both `app: frontend` AND `env: qa`. If a Pod dies and a new one is created with a completely different name, as long as it has those same labels, the Service will automatically find it.

### 3. Bulk Operations and Queries

Labels make it incredibly easy for administrators to filter and troubleshoot resources via the command line. For example, if you want to see all your development pods across the entire cluster, you don't have to check them individually; you can just run a single command filtering for `env=dev`.


![image.png](https://raw.githubusercontent.com/soumya-ranjan-000/image-hosting/main/articles/kubernetes-building-blocks/1781026459990-image.png)

In the image above, we have used two Label keys: app and env. Based on our requirements, we have given different values to our four Pods. The Label env=dev logically selects and groups the top two Pods, while the Label app=frontend logically selects and groups the left two Pods. We can select one of the four Pods - bottom left, by selecting two Labels: app=frontend AND env=qa.



Here are three common real-world scenarios showing how labels orchestrate infrastructure flows.

---

### Scenario 1: Routing Traffic to the Right Pods (Service-to-Pod)

Imagine you have a frontend app. You just deployed a new version (`v2.0.0`) to test alongside the old version (`v1.0.0`). You want your external **Service** (the load balancer) to only route live customer traffic to the stable version.

```
       [ Internet Traffic ]
                │
                ▼
        [ Service Selector ] ─── (Looks for: app=frontend, status=stable)
                │
        ┌───────┴───────┐
        ▼               ▼
   ┌───────────┐   ┌───────────┐
   │  Pod A    │   │  Pod B    │
   │ app=front │   │ app=front │
   │ status=st │   │ status=st │
   └───────────┘   └───────────┘
     (Traffic)       (Traffic)

```

* **The Setup:** Pod A & B have labels: `app: frontend`, `status: stable`, `version: v1.0.0`
* Pod C & D (canary) have labels: `app: frontend`, `status: canary`, `version: v2.0.0`
* **The Flow:** Your Kubernetes Service is configured with a selector looking for `app: frontend` AND `status: stable`.
* **The Useful Result:** Even though all four pods are running the frontend application, the Service automatically ignores the canary pods and only sends user traffic to Pods A and B. When you are ready to promote version 2, you simply change Pod C & D's label to `status: stable`, and the Service instantly starts sending them traffic without needing a restart.

---

### Scenario 2: Maintaining App Availability (Deployment/ReplicaSet Flow)

A **Deployment** is responsible for making sure a specific number of Pods (e.g., 3 replicas) are always running. It uses labels to keep count.

```
[ Deployment Selector ] ─── (Ensures 3 copies of: app=backend, env=prod)
         │
         ├──► [ Pod 1 ] (app=backend, env=prod)  ─── State: Healthy
         ├──► [ Pod 2 ] (app=backend, env=prod)  ─── State: Healthy
         ▼
       XXXXX  ─── [ Pod 3 Crashed! ]
         │
         └─► (Selector sees only 2 matches. Instantly spins up Pod 4 with same labels)

```

* **The Setup:** Your deployment creates 3 Pods, all tagged with `app: backend` and `env: prod`.
* **The Flow:** The deployment constantly scans the cluster saying: *"How many pods match `app=backend` and `env=prod`?"* 
* **The Useful Result:** If a server dies and takes one of your pods with it, the count drops to 2. The deployment notices the missing label match and immediately commands the cluster to spin up a new Pod with those exact same labels to restore balance. It doesn't care *which* pod died; it just cares about the label tally.

---

### Scenario 3: Dedicating Specific Hardware (Pod-to-Node Flow)

Sometimes you have specialized workloads—like a Machine Learning model that requires an expensive GPU, or a database that needs an SSD. You can label the **Nodes** (the physical or virtual machines) themselves.

```
[ Incoming ML Pod ] ─── (Specifies nodeSelector: hardware=gpu)
        │
        ├──► [ Node 1 ] (hardware=ssd) ─────────── Skip (No match)
        │
        └──► [ Node 2 ] (hardware=gpu) ─────────── Match! (Pod scheduled here)

```

* **The Setup:** You tag your heavy-duty cluster nodes with the label `hardware: gpu`.
* **The Flow:** When you deploy your machine learning Pod, you add a `nodeSelector` in its configuration file telling Kubernetes: *"Only place me on a machine that has the label `hardware: gpu`."*
* **The Useful Result:** The Kubernetes Scheduler filters out standard CPU servers and guarantees your database or AI model lands exactly on the high-performance hardware it needs to run efficiently.

---

### Summary of Benefits

As you can see across these flows, labels provide:

* **Zero Downtime Updates:** Switch traffic between environments just by swapping a label value.
* **Self-Healing Automation:** Controllers use labels to monitor system health and replace dead components seamlessly.
* **Smart Scheduling:** Easily match software resource requirements to physical hardware constraints.

# Label Selectors

Think of **Label Selectors** as the search engine or the filter system of Kubernetes.

If **Labels** are the tags you stick onto things (like a sticky note that says `env: dev` or `app: frontend`), then **Label Selectors** are the queries you run to find and group those things.

Without selectors, labels would just be useless sticky notes that no one is looking at.

Here is an easy breakdown of how they work.

---

### 1. Equality-Based Selectors (The Exact Match)

This is like shopping online and filtering for an exact match. It uses `=`, `==` (equals), or `!=` (not equals).

* **How it thinks:** "Show me objects that have *exactly* this tag, or exclude objects with *exactly* this tag."
* **Real-World Analogy:** You go to a clothing store and ask the clerk, *"Show me only shirts that are size Medium."*
* **Kubernetes Example:** `env == prod`
*(This will find every single component matching the production environment, completely ignoring `dev` or `qa`).*

### 2. Set-Based Selectors (The Flexible Group)

This is a more powerful filter. It allows you to look for a whole bunch of different values at the same time using keywords like `in`, `notin`, or checking if a tag simply exists.

* **How it thinks:** "Show me objects if their tag belongs to this list of options."
* **Real-World Analogy:** You tell the clerk, *"Show me shirts that are either Small OR Medium, but definitely not Large."*
* **Kubernetes Example:** `env in (dev, qa)`
*(This instantly selects components belonging to both the development and QA environments at the same time).*

---

### How a Service Uses a Selector (A Quick Visual)

Imagine you have 4 Pods running in your cluster. You create a Kubernetes **Service** (a traffic router) and give it a **Label Selector** of `app == backend`.

1. The Service constantly scans the cluster looking for that exact tag.
2. It hooks up to **Pod 1** (`app: backend`) and **View Pod 2** (`app: backend`).
3. It completely ignores **Pod 3** (`app: frontend`), because the selector doesn't match.

### Why is this separation beautiful?

Because it makes Kubernetes dynamic. If Pod 1 crashes and dies, a new pod will be born with a totally different name (like `backend-xyz789`). As long as that new pod spins up with the label `app: backend`, the Selector will instantly spot it and start sending it work.

![image.png](https://d36ai2hkxl16us.cloudfront.net/course-uploads/e0df7fbf-a057-42af-8a1f-590912be5460/jex0jopg5sxf-Selectors2023.png)

# ReplicationController
In Kubernetes, a **ReplicationController (RC)** is an older, legacy workload resource that ensures a specified number of identical pod replicas are running at any given time.

If there are too many pods, the ReplicationController kills the extra ones. If there are too few, it starts more. It acts as a supervisor to guarantee application availability and scaling.

Here is a breakdown of how it works, its core components, and why it has mostly been replaced today.

---

## How a ReplicationController Works

A ReplicationController uses a simple **lifecycle loop** to continuously monitor the state of the cluster. It compares the *actual* number of running pods against the *desired* number of pods you specified.

It relies on three key pieces of information:

1. **Desired Replicas:** The number of pods it should keep running (e.g., 3).
2. **Pod Template:** The blueprint used to create a new pod if the current count falls short.
3. **Selector:** A label selector that tells the controller which pods it is responsible for managing.

### The Loose Coupling (Labels & Selectors)

The ReplicationController doesn't actually "own" the pods it manages. Instead, it looks for pods that match its defined `spec.selector`.

* If you manually create a pod with labels that match the ReplicationController’s selector, the controller will count it toward the total. If that puts the count over the desired number, the controller will delete one of its pods.
* If you change the labels on an existing pod managed by an RC, that pod moves outside the RC's umbrella. The RC will notice it is now short one pod and spin up a brand new one from its template.

---

## A Typical ReplicationController Manifest (YAML)

Here is what an RC definition looks like in YAML format:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc
spec:
  replicas: 3
  selector:
    app: nginx-web
  template:
    metadata:
      labels:
        app: nginx-web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80

```

### Key Fields Explained:

* **`spec.replicas`**: Tells Kubernetes to ensure exactly 3 pods are running.
* **`spec.selector`**: Specifies that this controller manages any pod with the label `app: nginx-web`.
* **`spec.template`**: This is the exact pod definition used to create new pods. Notice that the `metadata.labels` here **must** match the selector above.

---

## ReplicationController vs. ReplicaSet (The Modern Standard)

While ReplicationControllers are still supported, they are largely considered **legacy**. In modern Kubernetes environments, you should use **ReplicaSets** or **Deployments**.

The difference comes down to how they select pods:

| Feature | ReplicationController | ReplicaSet |
| --- | --- | --- |
| **Selector Capability** | **Equality-based** only (`=`, `!=`). It can only match exact key-value pairs (e.g., `app: nginx-web`). | **Set-based** filtering. It supports complex operators like `In`, `NotIn`, `Exists`, and `DoesNotExist`. |
| **Example Usage** | Matches *only* pods labeled exactly `environment: production`. | Can match pods labeled `environment` in `[production, staging]` or any pod that simply *has* a `tier` key. |
| **Deployment Integration** | Cannot be managed directly by a modern Deployment object. | Automatically managed underneath by the higher-level **Deployment** resource. |

---

## Summary of Key Use Cases

Even though ReplicaSets are preferred, understanding the ReplicationController highlights the core self-healing philosophies of Kubernetes:

* **Resilience:** If a node hosting your pods crashes, the controller detects the drop in pod count and schedules new pods on a healthy node.
* **Scaling:** You can easily scale your application up or down by updating the `replicas` count via the CLI (`kubectl scale`) or by updating the YAML file.
* **Load Balancing:** While the RC doesn't route traffic itself, it ensures the steady pool of pods exists behind a Kubernetes **Service**, which handles the actual load balancing.

To understand exactly how a ReplicationController (RC) detects changes, reacts, and triggers downstream actions across the cluster, we have to look under the hood at the **Kubernetes Control Plane**.

This entire process relies on an asynchronous, event-driven pattern called the **Control Loop** (or Reconciliation Loop) and the **Observer Pattern** via `etcd`.

Here is the granular, step-by-step breakdown of how the RC knows about a change, how it acts, and how the rest of the system responds.

---

## 1. How the RC Knows About a Change (The "Watch" Mechanism)

The ReplicationController does not constantly ping or poll every pod in the cluster. That would destroy performance at scale. Instead, it relies on a **Watch API** provided by the `kube-apiserver`.

```
[etcd (Database)] <---> [kube-apiserver] <---(Long Poll/Watch)--- [ReplicationController]

```

* **The Registration:** When the ReplicationController process starts up within the `kube-controller-manager`, it establishes a long-lived HTTP connection to the API server, specifically asking to "watch" events related to Pods and ReplicationControllers.
* **The Trigger:** A change occurs. For example, a Node crashes (killing a pod), a user runs `kubectl scale`, or a developer manually deletes/re-labels a pod.
* **The Notification:** The `kube-apiserver` writes this change into `etcd` (the cluster's source of truth). Instantly, `etcd` fires a change event back to the API server, which immediately streams that event over the established "watch" channel directly to the RC's internal cache (called an **Informer**).

---

## 2. What the RC Does Next (The Reconciliation Loop)

Once the event lands in the RC's queue, it triggers the **Reconciliation Loop**. The RC’s sole job is to enforce this simple math equation:

$$\text{Current State} = \text{Desired State}$$

### Step A: Filtering by Selector

The RC queries its local cache for all pods that match its exact `spec.selector` labels (e.g., `app: nginx-web`). It filters out pods that are actively dying (`DeletionTimestamp` is set).

### Step B: The Math Check

It counts the matching pods. Let’s say `spec.replicas` is **3**, but it only counts **2** active pods.

### Step C: The Action (State Mutation)

The RC realizes it is short by 1 pod. However, **the RC does not create pods itself.** It cannot talk to Docker or containerd, and it doesn't know anything about server nodes.

Instead, it sends a `POST` request to the `kube-apiserver` saying: *"Please create a new Pod object using the template definition I have in my manifest."* Once the API server validates this and writes the new Pod object into `etcd`, the RC’s job for this cycle is completely finished. It goes back to waiting.

---

## 3. How the Rest of the System Reacts (The Ripple Effect)

The creation of that raw Pod object in `etcd` kicks off a chain reaction across other independent components in the cluster.

### Phase 1: The `kube-scheduler` Steps In

1. The `kube-scheduler` also has a "Watch" connection open with the API server. It listens specifically for **unassigned pods** (Pods where `spec.nodeName` is blank).
2. The scheduler detects the new pod created by the RC.
3. It runs its filtering and scoring algorithms to determine which worker node has the CPU, memory, and network resources available to host this pod.
4. Once it chooses a node (e.g., `node-02`), it sends a binding request to the API server, updating the pod's manifest to set `spec.nodeName: node-02`.

### Phase 2: The Worker Node (`kubelet`) Takes Action

1. On `node-02`, a background agent called the **kubelet** is constantly watching the API server for any pods explicitly assigned to *its* node name.
2. The kubelet discovers that the new pod has been assigned to it.
3. It talks to the local **Container Runtime Interface (CRI)** (like `containerd` or `CRI-O`) to pull the required container images and spin up the actual hardware-isolated containers.
4. It talks to the **Container Network Interface (CNI)** plugin to assign a unique cluster IP address to the pod.
5. It reports the status back to the API server: `PodStatus: Running`.

### Phase 3: The Networking Updates (`kube-proxy` & Services)

If you have a Kubernetes **Service** pointing to these pods:

1. The **EndpointSlice Controller** (another background loop) notices a new pod with the label `app: nginx-web` has transitioned to `Running` and has a valid IP address.
2. It updates the corresponding Service’s endpoints list.
3. **kube-proxy** daemons running on *every* single node in the cluster detect this endpoint change via their own API watches.
4. They instantly update their local networking rules (`iptables` or `IPVS`).

Traffic hitting the cluster will now immediately begin routing to the brand-new pod that the ReplicationController requested just moments prior.

# ReplicaSet
A **ReplicaSet** is the direct successor to the ReplicationController. Its primary purpose is to maintain a stable set of replica Pods running at any given time, ensuring high availability.

While it functions almost identically to a ReplicationController under the hood, it introduces more powerful **label selectors** that allow for complex grouping.

---

## 1. How a ReplicaSet Works

Like the ReplicationController, a ReplicaSet runs a continuous **Reconciliation Loop** ($Current State = Desired State$) via the Kubernetes Control Plane. It watches the cluster state, counts existing pods matching its criteria, and creates or deletes pods to hit its target.

The critical difference is *how* it identifies its pods. ReplicaSets use **Set-Based Label Selectors**, whereas ReplicationControllers only support **Equality-Based Selectors**.

### Equality-Based vs. Set-Based Selectors

* **ReplicationController (Old):** Can only match exact, single key-value pairs.
```yaml
selector:
  environment: production

```


*(Matches ONLY pods where environment equals production).*
* **ReplicaSet (Modern):** Can evaluate expressions, allowing you to match pods across multiple environments, tiers, or versions simultaneously.
```yaml
selector:
  matchExpressions:
    - {key: environment, operator: In, values: [production, staging]}
    - {key: tier, operator: Exists}

```


*(Matches any pod where environment is either production OR staging, AND the label 'tier' exists, regardless of its value).*

This advanced filtering makes it significantly easier to manage complex, multi-tenant, or multi-tiered architectures without having to precisely align every single individual label.

---

## 2. How ReplicaSets Work with Deployments

In real-world Kubernetes production, **you almost never create or manage ReplicaSets directly.** Instead, you use a higher-level resource called a **Deployment**.

A Deployment is an abstraction layer *above* the ReplicaSet. It manages ReplicaSets, which in turn manage the Pods.

```
  [ Deployment ]
        |
        v  (Manages / Rolls out)
  [ ReplicaSet ]
        |
        v  (Ensures replica count)
    [ Pods ]

```

When you define a Deployment, it automatically generates a ReplicaSet underneath. The relationship shines during **application updates and rollouts**. Here is exactly how they orchestrate a deployment update (like changing an image version from `v1` to `v2`):

### Step-by-Step: The Rollout Mechanics

1. **The Trigger:** You update the container image in your Deployment manifest from `nginx:v1` to `nginx:v2` and apply it.
2. **Creation of a New ReplicaSet:** The Deployment Controller detects this change. Because the Pod template changed, it creates a *brand new, second ReplicaSet* (let's call it `RS-v2`).
3. **The Rolling Update Dance:** * The Deployment tells `RS-v2` to scale up to **1 replica**. `RS-v2` creates a new `v2` pod.
* Once that `v2` pod is healthy, the Deployment tells the old `RS-v1` to scale down from **3 replicas to 2**. `RS-v1` terminates a `v1` pod.
* This step-up, step-down process repeats incrementally (controlled by parameters like `maxSurge` and `maxUnavailable`).


4. **The Final State:** Eventually, `RS-v2` reaches **3 replicas**, and `RS-v1` drops to **0 replicas**.

### Why keep the old ReplicaSet around? (Rollbacks)

Even though the old `RS-v1` has 0 active pods, **the Deployment does not delete it.** It keeps it in the cluster history.

If you discover that your `v2` software has a critical bug, you can run:

```bash
kubectl rollout undo deployment/my-deployment

```

The Deployment will instantly reverse the logic: it will scale `RS-v2` down to 0 and scale the preserved `RS-v1` back up to 3. This gives you a near-instantaneous application rollback mechanism with zero downtime.

---

## Summary of the Hierarchy

* **Pod:** The smallest deployable unit (the actual container runtime).
* **ReplicaSet:** The muscle that ensures the exact right number of Pods are healthy and alive using powerful label matching.
* **Deployment:** The brain that orchestrates declarative updates, handles rolling versions, and manages multiple ReplicaSets over time.

Would you like to see a comparison of how the Deployment YAML structure encapsulates the ReplicaSet and Pod templates into a single file?

# Deployment

To achieve zero-downtime updates, a Kubernetes Deployment coordinates a precise choreography between its own controllers, the networking layer, and the worker nodes.

Here is the granular breakdown of how a Deployment updates your application without interrupting traffic, and how the rest of the cluster components react in real-time.

---

## 1. The Strategy: Rolling Update Mechanics

By default, a Deployment uses the `RollingUpdate` strategy. This ensures that a fraction of old pods remain alive to serve traffic while new pods are being spun up and verified.

The safety boundaries of this update are controlled by two critical parameters in the Deployment configuration:

* **`maxSurge`**: How many extra pods can be created above the desired replica count during the update (e.g., `25%`).
* **`maxUnavailable`**: How many pods can be taken offline simultaneously during the update (e.g., `25%`).

### The Rolling Update Process

If you have a 4-replica deployment of Version 1 (`v1`), and you update the image to Version 2 (`v2`):

```
[ Deployment Controller ]
      |
      +---> [ ReplicaSet v1 ] ---> [ Pod v1 ] [ Pod v1 ] [ Pod v1 ] [ Pod v1 ]
      |
      +---> [ ReplicaSet v2 ] ---> [ Pod v2 ] (Spun up incrementally)

```

1. The Deployment creates a new ReplicaSet (`RS-v2`).
2. `RS-v2` creates the first `v2` pod.
3. **Crucial Step:** The Deployment **waits** until the new `v2` pod passes its **Readiness Probe**.
4. Once `v2` is confirmed healthy, `RS-v1` is ordered to terminate one `v1` pod.
5. This shift continues step-by-step until `RS-v2` has 4 pods and `RS-v1` has 0.

---

## 2. How the Rest of the K8s Components React (The Granular Flow)

To understand why traffic isn't dropped, we have to look at the exact timeline of how the Control Plane and Data Plane execute these changes simultaneously.

### Phase A: The Spawning of Version 2

When `RS-v2` creates a new pod object via the `kube-apiserver`:

* **`kube-scheduler`** detects the unassigned pod, evaluates node capacities, and binds it to a healthy node.
* **`kubelet`** on that node talks to the container runtime (e.g., `containerd`) to pull the new image and start the container. It also triggers the network plugin (CNI) to allocate a new internal cluster IP to this pod.
* **The Readiness Probe Check:** The `kubelet` begins executing the defined `readinessProbe` (e.g., hitting an HTTP `/healthz` endpoint inside the container). **Until this probe returns a `200 OK`, the pod is considered "Unready" and is kept completely isolated from production traffic.**

### Phase B: Safely Adding V2 to the Traffic Pool

Once the `kubelet` confirms the readiness probe has passed:

* It updates the pod's status to `Ready: True` via the API server.
* The **EndpointSlice Controller** (watching the API server) sees a new `Ready` pod matching the Service's selector. It appends the new pod's IP address to the active **EndpointSlice** pool.
* **`kube-proxy`** running on every node watches for EndpointSlice changes. It instantly rewrites the local node network rules (`iptables` or `IPVS`).
* **The Result:** Live traffic hitting the Kubernetes Service or Ingress controller is now automatically distributed across the remaining `v1` pods *and* the newly ready `v2` pod.

### Phase C: The Graceful Destruction of Version 1

Simultaneously, when the Deployment controller tells `RS-v1` to scale down by one pod:

* The API Server marks that specific `v1` pod as `Terminating` and sets a `DeletionTimestamp`.
* **The Endpoint Removal:** The EndpointSlice Controller sees this timestamp and **instantly removes** the pod's IP from the active traffic pool. `kube-proxy` updates `iptables` across the cluster. **No new traffic will be sent to this pod.**
* **The Graceful Shutdown (`SIGTERM`):** Concurrently, the `kubelet` hosting that `v1` pod receives the termination event. It sends a `SIGTERM` signal to the container process.
* **The Application Reacts:** The application catches the `SIGTERM`, stops accepting new connections, finishes processing any flights/requests currently in progress, and gracefully closes database pools.
* **The Hard Kill (`SIGKILL`):** Kubernetes waits for a configurable period (default is 30 seconds via `terminationGracePeriodSeconds`). If the container is still alive after this window, the `kubelet` sends a `SIGKILL` to forcefully stop it.

---

## 3. Why Traffic is Never Interrupted

The secret to zero-downtime lies in the strict synchronization of these independent components:

1. **Isolation of the Unprepared:** New pods are never handed traffic until they explicitly prove they are ready via Readiness Probes.
2. **Immediate Removal on Death:** Old pods are removed from the network routing table *before* or *at the exact same fraction of a second* that they receive the shutdown signal.
3. **Grace Period Processing:** The old pods are given a countdown window to finish servicing requests that were already mid-flight before they completely disappear.

To truly zoom in on the granular level, we need to map out the entire Kubernetes architecture and trace exactly how the **Control Plane** (the brains) and the **Data Plane** (the muscle) communicate during a Deployment update.

The magic of Kubernetes is that these components don't actually talk to each other directly. They all look at a single source of truth—the **kube-apiserver**—and react independently based on what they see.

Here is the exact, deep-dive choreography of how every major Kubernetes component is involved when you trigger a Deployment update.

---

## The Complete Architecture Flow

### 1. The Core Database: `etcd`

* **Role:** The cluster's distributed, highly available key-value store.
* **Involvement:** `etcd` is the only place where the state of the cluster is actually saved. When you update a Deployment, or when a pod changes status, that data is permanently written to `etcd`. No other component talks to `etcd` directly except the API Server.

### 2. The Gatekeeper: `kube-apiserver`

* **Role:** The central hub and exposure point for the Kubernetes API.
* **Involvement:** Think of this as the nervous system. When you execute a deployment update, the API server validates your request, stores it in `etcd`, and streams the change notification out to all the other components that are "watching" for updates.

### 3. The Orchestrator: `kube-controller-manager`

This is a single binary that runs multiple distinct "controller loops" in the background. During an update, three specific internal controllers swing into action:

* **Deployment Controller:** It notices the Deployment manifest changed. It calculates the `maxSurge` and `maxUnavailable` limits, and creates a brand-new ReplicaSet object via the API Server.
* **ReplicaSet Controller:** It watches the new ReplicaSet. Realizing that the desired pod count is higher than the actual pod count, it creates raw Pod objects (with no node assigned yet) via the API Server.
* **EndpointSlice Controller:** It continuously matches pods to Services. The moment a new pod becomes healthy, or an old pod begins to terminate, this controller instantly updates the Service’s network endpoint lists.

### 4. The Matchmaker: `kube-scheduler`

* **Role:** Assigns unassigned pods to optimal worker nodes.
* **Involvement:** The scheduler watches the API Server specifically for new pods that have a blank `spec.nodeName`.
* It filters through all available worker nodes to see which ones have enough CPU/Memory, checks affinity rules, and then scores the nodes. Once it picks the best node, it writes a binding back to the API Server, setting the pod's `spec.nodeName` to that node.

### 5. The Captain of the Node: `kubelet`

* **Role:** The primary agent running on every worker node.
* **Involvement:** The `kubelet` on the selected node notices that a pod has been assigned to it. It acts as the executioner on the machine:
* It communicates with the **Container Runtime (CRI)** (like `containerd`) to pull the container image and execute the container.
* It actively monitors the container's **Liveness and Readiness probes**.
* It reports the pod’s status (e.g., `ContainerCreating`, `Running`, `Ready`) back to the API Server.



### 6. The Cluster Operator: `CoreDNS`

* **Role:** Handles internal cluster name resolution.
* **Involvement:** When a new pod is assigned an IP address, or when a Service endpoint changes, CoreDNS updates its internal DNS mappings. This ensures that if another microservice is trying to talk to your deployment using a domain name (like `http://my-backend-service`), CoreDNS correctly resolves that name to the changing pool of backend pods without a single millisecond of cache frustration.

### 7. The Network Router: `kube-proxy`

* **Role:** Manages network routing rules on every single node.
* **Involvement:** `kube-proxy` watches the API Server for changes to `Services` and `EndpointSlices`.
* The moment the EndpointSlice Controller alters the active backend IP addresses, `kube-proxy` captures that event and instantly updates the host node's underlying packet-filtering rules (using **iptables** or **IPVS**). This is what ensures that external or internal traffic bypasses terminating pods and routes straight to the newly ready ones.

---

## Chronological System Reaction Timeline

To tie it all together, here is exactly how the ball is passed between components during a single rolling-update step:

```
[User] ──(Updates Image)──> [kube-apiserver] <───> [etcd]
                                  │
                       (Notifies Deployment Ctrl)
                                  │
                                  ▼
                     [Deployment Controller]
                        (Creates New RS)
                                  │
                                  ▼
                     [ReplicaSet Controller]
                       (Creates Raw Pod)
                                  │
                                  ▼
                        [kube-scheduler]
                      (Assigns Pod to Node)
                                  │
                                  ▼
                        [Node's kubelet]
                   (Starts Container via CRI)
                                  │
                       (Probes return Healthy)
                                  │
                                  ▼
                    [EndpointSlice Controller]
                     (Updates Service Pod IPs)
                                  │
         ┌────────────────────────┴────────────────────────┐
         ▼                                                 ▼
   [CoreDNS]                                         [kube-proxy]
(Updates Internal DNS)                       (Rewrites Network iptables)
         │                                                 │
         └────────────────────────┬────────────────────────┘
                                  ▼
                        [Zero-Downtime Traffic]

```

By decoupling these steps, Kubernetes ensures that if the scheduler is slow, or a node's container runtime takes time to pull a massive image, the networking layer (`kube-proxy` and `CoreDNS`) remains completely unaffected. They only alter the traffic flow when the `kubelet` completely verifies that the new pod is fully functional.

Let's trace the lifecycle of a Deployment update dynamically. Instead of looking at components as a static list, let’s follow the **event-driven chain reaction** that occurs across the cluster the exact moment you trigger a rolling update.

Assume we have an application running with 3 replicas of Version 1 (`v1`), and we update the manifest to use Version 2 (`v2`).

---

## Stage 1: The Trigger & State Record

The process begins when you apply the updated Deployment manifest (e.g., executing `kubectl apply -f deployment.yaml` or changing an image via the CLI).

1. **`kube-apiserver` receives the request:** It acts as the cluster's secure API entry point. It validates that your YAML syntax is correct and that you have permissions to modify the Deployment.
2. **`etcd` persists the intent:** The API Server writes the new desired state (e.g., *"Deployment `my-app` should now use image `v2`"*) into `etcd`. `etcd` is strictly a database; it does not take action, but saving this data triggers an immediate notification to any component "watching" the API Server for Deployment changes.

---

## Stage 2: Orchestrating the Strategy

The **Deployment Controller** (a background process loop inside the `kube-controller-manager`) is constantly watching for changes to Deployment objects.

1. **The Controller reacts:** It detects that the Pod template inside your Deployment has changed.
2. **Calculating the math:** It checks your update strategy (e.g., `RollingUpdate` with a `maxSurge` of 1). It computes that it needs to spin up a new version without exceeding the maximum allowed pods.
3. **Spawning the manager:** The Deployment Controller does not create pods directly. Instead, it sends a request back to the API Server to create a brand-new **ReplicaSet** object (`RS-v2`), passing along the `v2` pod template.
4. **The ReplicaSet Controller steps in:** Another loop inside the controller manager notices this new `RS-v2` object. It calculates: *"My desired count is 1, but my current count is 0."* It immediately posts a request to the API Server to create a new **Pod object** with the `v2` configuration.

---

## Stage 3: The Race to Schedule and Provision

At this exact moment, the new `v2` pod exists *only as a data record* in `etcd`. It has no physical home; it is an unassigned pod.

1. **`kube-scheduler` makes a match:** The scheduler is constantly watching the API Server for pods where `spec.nodeName` is empty. It catches the new `v2` pod event.
2. **Evaluating the cluster topology:** The scheduler analyzes all worker nodes in the cluster, filtering out nodes that lack CPU/Memory or violate affinity rules. It assigns a score to the remaining healthy nodes and selects the best fit (e.g., `Worker-Node-B`). It writes this assignment back to the API Server.
3. **`kubelet` executes locally:** On `Worker-Node-B`, the local `kubelet` agent watches the API Server specifically for pods assigned to *its* node name. It detects the `v2` pod assignment.
4. **Creating the environment:** The `kubelet` calls the **Container Runtime Interface (CRI)** (like `containerd`) to pull the `v2` container image and execute it. Simultaneously, it interfaces with the **Container Network Interface (CNI)** plugin to assign the pod a distinct, internal cluster IP address.

---

## Stage 4: Traffic Isolation via Health Probes

The container is now technically running on the server, but it is not yet ready to handle real user traffic. This is the most critical stage for preventing downtime.

1. **`kubelet` monitors the Probes:** The `kubelet` actively begins executing the **Readiness Probe** defined in your deployment manifest (e.g., running a script or hitting `GET /healthz` inside the container).
2. **The Isolation Period:** While these probes run, the pod's status remains `Ready: False`. Because it is not ready, it is completely ignored by the networking plane.
3. **The Success Event:** Once the application fully boots up, hooks into its database, and the readiness probe returns successful responses, the `kubelet` posts an update back to the API Server changing the status to `Ready: True`.

---

## Stage 5: The Networking Shift

The transition of the pod to a `Ready` state triggers a massive reaction across the cluster's data plane.

1. **The EndpointSlice Controller routes traffic:** This controller watches for `Ready` pods matching specific labels. It sees the new `v2` pod is online and healthy. It instantly updates the cluster's **EndpointSlice** object, adding the new pod's internal IP address to the active pool for that application's Service.
2. **`kube-proxy` rewrites the network:** On *every single node* in the cluster, the `kube-proxy` agent catches the EndpointSlice update. It instantly rewrites the host node's packet routing rules (**iptables** or **IPVS**).
3. **Live Traffic hits V2:** If an external user hits your load balancer or Ingress, the underlying node network rules will now seamlessly direct a portion of that traffic to the new `v2` pod.

---

## Stage 6: Graceful Demolition of Version 1

Now that Version 2 is safely handling traffic, the Deployment Controller can begin scaling down the old version.

1. **Ordering the scale-down:** The Deployment Controller sends a request to the old ReplicaSet (`RS-v1`) to reduce its count. `RS-v1` selects one of its old `v1` pods and sends a deletion request to the API Server.
2. **Immediate Network Eviction:** The API Server marks that `v1` pod as `Terminating` and attaches a timestamp. The **EndpointSlice Controller** catches this instantly and **removes** the pod's IP address from the routing pool. `kube-proxy` updates network rules cluster-wide. **No new user traffic will hit this pod.**
3. **Graceful Shutdown execution:** On the node hosting that old pod, the `kubelet` notices the termination event. It sends a `SIGTERM` signal directly into the container process.
4. **Fulfilling mid-flight requests:** The application inside the container catches the `SIGTERM`, stops taking new work, and uses its remaining time (default 30-second `terminationGracePeriod`) to finish processing any HTTP requests or background jobs that were already mid-flight.
5. **The Final Kill:** Once the active connections drain (or the grace period timer runs out), the `kubelet` issues a `SIGKILL` to clean up the container resources entirely.

---

## Stage 7: The Loop Repeats

The Deployment Controller looks at the cluster state again. If your total target was 3 replicas, it repeats this exact chain reaction—spawning another `v2` pod, waiting for readiness, updating the endpoints, and terminating a `v1` pod—until `RS-v2` successfully holds all 3 healthy replicas and `RS-v1` drops to 0.

When we focus purely on the interplay between **old pods** and **new pods** during a rolling update, the Deployment acts like a cautious traffic controller. It uses a "step-up, step-down" dance governed by two safety parameters: `maxSurge` (how many extra pods you can temporarily create) and `maxUnavailable` (how many old pods you can temporarily take offline).

Let's look at the exact mechanics of how old pods are phased out and new pods are brought in, step-by-step, without losing traffic capacity.

---

## The Starting Line

* **Desired State:** 3 replicas of **Old Pods (v1)**.
* **The Goal:** 3 replicas of **New Pods (v2)** with zero downtime.
* **The Configuration:** `maxSurge: 1` and `maxUnavailable: 0` (this ensures we never drop below our 3-replica capacity).

```
Traffic Pool: ──► [ Old Pod A ]   [ Old Pod B ]   [ Old Pod C ]

```

---

## Step 1: The New Pod is Created (But Isolated)

The Deployment controller tells the new ReplicaSet to create one **New Pod (v2)**.

* **The Action:** The control plane schedules and boots up `New Pod 1`.
* **The Traffic State:** At this moment, `New Pod 1` is running, but it is **not** added to the network routing tables yet. It is completely isolated.
* **Why?** It must pass its `readinessProbe` first. If the new code crashes or fails to connect to the database on startup, the system stalls here. Your users are entirely unaffected because 100% of the traffic is still going to the 3 healthy old pods.

```
Traffic Pool: ──► [ Old Pod A ]   [ Old Pod B ]   [ Old Pod C ]
                    
Isolated Pool:    [ New Pod 1 (Booting/Testing) ]

```

---

## Step 2: The New Pod Joins the Pool

Once `New Pod 1` passes its health checks, the `kubelet` marks it as `Ready`.

* **The Action:** The EndpointSlice Controller detects this health status and injects `New Pod 1`'s IP address into the Service's network pool. `kube-proxy` rewrites the routing rules on the nodes.
* **The Traffic State:** Traffic is now actively split across four endpoints: the 3 old pods and the 1 new pod.

```
Traffic Pool: ──► [ Old Pod A ]   [ Old Pod B ]   [ Old Pod C ]   [ New Pod 1 ]

```

---

## Step 3: Evicting the First Old Pod

Now that the cluster has 4 running pods (3 old + 1 new), the Deployment can safely remove one old pod to get back down to the target capacity of 3.

* **The Action:** The old ReplicaSet selects `Old Pod C` for termination. The API Server changes its state to `Terminating`.
* **Immediate Network Removal:** The EndpointSlice Controller instantly pulls `Old Pod C` out of the traffic pool. `kube-proxy` updates node networking rules within milliseconds. **No new incoming traffic requests will be routed to this pod.**
* **The Grace Period:** The node's `kubelet` sends a `SIGTERM` signal to `Old Pod C`. The pod doesn't instantly die; it stops accepting new work and uses its remaining time to finish processing any user requests that were already actively mid-flight right before the eviction notice.

```
Traffic Pool: ──► [ Old Pod A ]   [ Old Pod B ]                   [ New Pod 1 ]
                    
Draining Pool:    [ Old Pod C (Finishing mid-flight work...) ]

```

---

## Step 4: The Cycle Repeats

Once `Old Pod C` completely shuts down and disappears, the cluster is back to exactly 3 active traffic-serving pods (2 old + 1 new). The Deployment Controller evaluates the state and repeats the entire cycle:

1. It spins up `New Pod 2` in isolation.
2. `New Pod 2` passes health checks and is added to the traffic pool.
3. The controller targets `Old Pod B` for termination, pulling it from network routing and letting its active requests drain gracefully.

```
Traffic Pool: ──► [ Old Pod A ]                                   [ New Pod 1 ]   [ New Pod 2 ]
                    
Draining Pool:    [ Old Pod B (Finishing mid-flight work...) ]

```

---

## The Finish Line

The loop triggers one last time for the final pod. Once `Old Pod A` is safely drained and removed, and `New Pod 3` is verified healthy:

* The old ReplicaSet scales to 0.
* The new ReplicaSet holds all 3 active replicas.
* The rolling update is complete.

```
Traffic Pool: ──►  [ New Pod 1 ]   [ New Pod 2 ]   [ New Pod 3 ]

```

## Summary of the Hand-Off Strategy

| Phase | What happens to New Pods? | What happens to Old Pods? |
| --- | --- | --- |
| **Ingress/Traffic** | Only receive traffic **after** passing the Readiness Probe. | Cut off from **new** traffic the instant they are marked `Terminating`. |
| **Lifecycle** | Brought up incrementally based on `maxSurge` rules. | Given a `SIGTERM` grace period to cleanly finish existing jobs before being forcefully killed (`SIGKILL`). |

# Services

In Kubernetes, **Services** are the ultimate solution to a major problem: **Pods are ephemeral (temporary).** When a Pod dies (due to a crash, a node failure, or a rolling update), it is replaced by a brand-new Pod with a completely new, unpredictable IP address. If your frontend app is trying to talk to a backend Pod at `10.10.10.4`, and that backend Pod dies, its replacement might be born at `10.10.10.9`. Hardcoding IP addresses would cause your application to break constantly.

A **Service** acts as a permanent, stable gatekeeper. It provides a single static IP address and DNS name that never changes, automatically load-balancing traffic across a dynamic pool of backend Pods.

---

## 🛠️ How a Service Works under the Hood

Instead of talking directly to the Pods, your application talks to the Service. The Service handles the mapping using three core elements:

1. **The Selector (The Search Query):** Just like Deployments, a Service uses labels to find its target Pods. If a Service has a selector for `app: backend`, it continuously tracks all Pods matching that label.
2. **Endpoints / EndpointSlices:** Behind the scenes, a background controller watches the cluster. Every time a Pod matching the label is born or dies, its live IP address is added to or removed from a list called an **EndpointSlice**.
3. **kube-proxy:** This agent runs on every worker node. It watches the EndpointSlice list and instantly configures the node's local networking rules (`iptables` or `IPVS`). When traffic hits the Service's stable IP, the node handles routing it straight to a healthy backend Pod.

```
[ Incoming Traffic ] ──► [ Stable Service IP ]
                                │
                 ┌──────────────┴──────────────┐
                 ▼ (Load Balanced)             ▼
           ┌───────────┐                 ┌───────────┐
           │   Pod 1   │                 │   Pod 2   │
           │ backend-A │                 │ backend-B │
           └───────────┘                 └───────────┘

```

---

## 🚦 The 4 Core Service Types

Depending on where your traffic is coming from (inside the cluster vs. the outside internet), Kubernetes provides four primary types of Services:

### 1. ClusterIP (Default)

* **What it does:** Exposes the Service on an **internal-only** cluster IP.
* **Use Case:** Perfect for internal communication between your own microservices (e.g., your frontend app talking to your backend database). It cannot be reached from outside the cluster.

### 2. NodePort

* **What it does:** Opens a specific port (usually between `30000-32767`) on the actual physical/virtual IP address of **every single worker node** in your cluster.
* **Use Case:** A quick, basic way to expose an application to external traffic. If you hit `http://<Any-Node-IP>:<NodePort>`, the cluster will automatically route that traffic to your internal Pods.

### 3. LoadBalancer

* **What it does:** Integrates with cloud providers (like AWS, GCP, Azure) to automatically spin up a native, physical cloud load balancer.
* **Use Case:** The standard production method for exposing services directly to the internet. The cloud load balancer gets a public IP, which routes traffic into your cluster's `NodePort` and down to your `ClusterIP`.

### 4. ExternalName

* **What it does:** Acts as an internal alias/shortcut that maps a Kubernetes Service to an external DNS name (like `my-database.amazonaws.com`).
* **Use Case:** Useful when your applications running inside the cluster need a clean way to talk to a database or API hosted *outside* the cluster.

---

## 📄 What a Service Looks Like in Code

Here is a standard declarative blueprint for a Kubernetes Service mapping to a set of backend web applications:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service     # The permanent DNS name for your cluster
spec:
  type: ClusterIP           # The type of service
  selector:
    app: backend            # Looks for Pods labeled 'app: backend'
  ports:
  - protocol: TCP
    port: 80                # The port the Service listens on
    targetPort: 8080        # The actual port running inside the container

```

With this configuration, any other Pod in the cluster can simply hit `http://backend-service` to safely communicate with your backend, completely insulated from the chaos of individual Pod lifecycles.

---
In Kubernetes, Pods are ephemeral — they get created and destroyed constantly, and their IP addresses change. A Service is a stable abstraction that sits in front of a group of Pods and gives you a consistent endpoint to reach them, regardless of which Pods are alive at any moment.
Services use label selectors to find the Pods they should route traffic to. As long as a Pod has the right labels, the Service will include it automatically.
Let me show you the four main Service types:

![image.png](https://raw.githubusercontent.com/soumya-ranjan-000/image-hosting/main/articles/kubernetes-building-blocks/1781804708110-image.png)

One Service exists across the entire cluster — it's not per-node. The Service is a cluster-level object with a single stable IP (ClusterIP). What runs on each node is kube-proxy, which just programs local routing rules so that node knows how to handle traffic destined for the Service.


![image.png](https://raw.githubusercontent.com/soumya-ranjan-000/image-hosting/main/articles/kubernetes-building-blocks/1781805325454-image.png)


**One Service exists across the entire cluster** — it's not per-node. The Service is a cluster-level object with a single stable IP (ClusterIP). What runs on each node is **kube-proxy**, which just programs local routing rules so that node knows how to handle traffic destined for the Service.To make it crystal clear:

**There is only ONE Service object** — it lives in the control plane (stored in etcd). It has one ClusterIP. It doesn't belong to any node.

**What runs on every node is `kube-proxy`** — a daemon process that watches the API Server for Service and Endpoint changes. Whenever a new Service is created or a Pod comes/goes, `kube-proxy` updates the local `iptables` (or `ipvs`) rules on that node so it knows: "if traffic is heading for `10.96.0.1:80`, redirect it to one of these real Pod IPs."

So the flow is:

1. You create one Service → stored once in the cluster
2. `kube-proxy` on Node 1 reads it → writes iptables rules on Node 1
3. `kube-proxy` on Node 2 reads the same Service → writes the same iptables rules on Node 2
4. Now any Pod on any node that hits `10.96.0.1:80` gets routed correctly — the iptables rule on that node intercepts and redirects the packet

The Service itself never touches the nodes. It's the `kube-proxy` agents that do the legwork of translating the Service's virtual IP into actual Pod routing on each machine.