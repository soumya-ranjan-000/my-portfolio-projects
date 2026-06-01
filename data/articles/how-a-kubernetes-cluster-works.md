# To understand how a Kubernetes cluster works, it helps to look at it through three core high-level flows:

1. The Deployment Flow (Creating a new application)
2. The Scheduling Flow (Deciding where the app runs)
3. The Healing Flow (Fixing things when they break)

## Here is a step-by-step breakdown of how these components interact in each scenario.

### 1. The Deployment Flow (How an App gets Created)
This flow triggers when you run a command like kubectl apply -f deployment.yaml. It is all about validating and saving your intent.

**Step 1:** The Request: You (or a CI/CD pipeline) send a YAML file to the API Server.

**Step 2:** Authentication & Validation: The API Server checks your credentials (Are you allowed to deploy this?) and validates the YAML syntax.

**Step 3:** Persisting State: Once validated, the API Server writes the deployment configuration into etcd.

**Step 4:** The Controller Takes Over: The Kube-Controller-Manager (specifically the Deployment Controller) is constantly watching the API Server. It notices the new deployment in etcd and says, "Wait, the user wants 2 replicas of this app, but 0 exist right now!"

**Step 5:** Pod Creation: The Controller Manager creates Pod definitions (the blueprints) and sends them back to the API Server, which saves them to etcd. At this stage, the pods are in a Pending state because they don't have a node assigned yet.

---
## *To understand exactly what goes into a deployment.yaml file and why, we have to look at it through the lens of the Kubernetes control plane you were just exploring.*

A deployment.yaml isn't just a configuration file; it is a declarative statement of your desired state. When you hand this to the API Server, you are telling the Controller Manager exactly how to build and maintain your application.

Here is a breakdown of a production-ready, standard deployment.yaml file, split into its structural components, followed by an explanation of why each piece is mandatory.

``` YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: production
  labels:
    app: auth-service
    tier: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
      - name: auth-container
        image: nginx:1.25.3
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
```

# Anatomy of the Blueprint: What is there and Why
A deployment file is strictly split into three main logical layers: The API Type, The Deployment Configuration, and The Pod Blueprint (Template).

A deployment file is strictly split into three main logical layers: **The API Type**, **The Deployment Configuration**, and **The Pod Blueprint (Template)**.

## 1. The API Infrastructure (The "Who Am I" Layer)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: production

```

### 1.1) apiVersion: apps/v1 & kind: Deployment

**What it is:** Identifies the exact schema validation endpoint in the API Server.

**Why it's there:** In Phase 1 of your flow, the API Server reads this to know *how* to parse 

the rest of the file. `apps/v1` tells Kubernetes to hand this request over to the **Deployment Controller** inside the Controller Manager.

### 1.2) metadata
* **What it is:** The unique identity of the Deployment itself.
* **Why it's there:** The `name` and `namespace` dictate the precise key location where this object will be saved inside **etcd** (e.g., `/registry/apps/deployments/production/auth-service`).



## 2. The Controller Spec (The "Desired State" Layer)

```yaml
spec:
  replicas: 3
  selector:
    matchLabels:
      app: auth-service

```
### 2.1) replicas: 3

* **What it is:** The exact number of concurrent instances you want running.
* **Why it's there:** The **ReplicaSet Controller** continuously counts the actual pods in the cluster. If it sees 0 running, and this file says `3`, it calculates a delta of `+3` and instantly fires off 3 individual Pod creation requests to the API Server.
### 2.2) selector / matchLabels
* **What it is:** The loose glue that connects the Deployment to its children.
* **Why it's there:** **CRITICAL COMPONENT.** Kubernetes doesn't track pods by an explicit list; it tracks them via query tagging. The ReplicaSet controller continuously queries the cluster: *"Give me all pods labeled `app: auth-service`"*. If it counts 3, it does nothing. If a node crashes and a pod dies, the count drops to 2, and the loop triggers a replacement.



## 3. The Pod Template (The "What to Run" Layer)

Everything nested under `template:` is literally a **blueprint for a Pod**. When the ReplicaSet decides to spin up a new instance, it copies this section entirely and submits it as a new Pod spec to the API Server.

```yaml
    metadata:
      labels:
        app: auth-service

```

### 3.1) template.metadata.labels
* **Why it's there:** This **must** match the `selector.matchLabels` above. If they don't match, the Deployment will create a Pod, fail to recognize that it created it, and recursively spin up infinite pods until the cluster runs out of memory. (Modern API Servers will reject the file during validation if these don't match).



```yaml
    spec:
      containers:
      - name: auth-container
        image: nginx:1.25.3
        ports:
        - containerPort: 80

```

### 3.2) containers array
* **Why it's there:** This is the execution contract passed ultimately down to the **Kubelet** and the **Container Runtime (Docker/containerd)** on the worker node. It tells the runtime exactly which image registry to pull from (`nginx:1.25.3`) and which internal port to open.



```yaml
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"

```

### 3.3) resources
* **Why it's there:** This is the **Scheduler's** cheat sheet. When the Pod is in a *Pending* state, the `kube-scheduler` reads the `requests` block (e.g., *I need 250m CPU and 64Mi RAM free*). It filters out any worker nodes that don't have that capacity available. The `limits` block tells the node's Linux kernel (via cgroups) to kill or throttle the container if it tries to consume more than its allowed share.



```yaml
        readinessProbe:
          httpGet:
            path: /healthz
            port: 80

```

### 3.3) readinessProbe
* **Why it's there:** Dictates traffic safety. The Kubelet on the worker node will constantly ping `/healthz`. If your app returns a `200 OK`, the Kubelet tells the API Server the pod is healthy, and traffic starts flowing. If your app crashes or locks up, the probe fails, traffic stops routing to it automatically, and the deployment self-heals.
---

## Phase 1: Authentication & Validation
***Phase 1:*** The Request — kubectl apply
When you run kubectl apply -f deployment.yaml, your intent travels through a chain of trust before anything is scheduled. Let's visualize what happens at the very first gate.

![image.png](https://raw.githubusercontent.com/soumya-ranjan-000/image-hosting/main/articles/how-a-kubernetes-cluster-works/1779468340502-image.png)
The API Server is Kubernetes' front door and only front door — no component talks to etcd directly. Every request must pass all four gates above. A bad YAML, wrong permissions, or a rejected admission policy kills the request before anything is persisted.



When you run `kubectl apply`, you aren't just sending YAML; your client is sending an HTTP request with cryptographic proof of who you are. The API Server itself is completely stateless, meaning it doesn't "log you in" or keep a session. Instead, it evaluates your identity on **every single request** using a chain of authenticators.

Here is what goes down under the hood during the Authentication phase:

### 1. The "First Match Wins" Stripping Request

When the HTTP request hits the API Server, it goes through a list of enabled authentication modules (configured as flags on the API Server). The server runs through them sequentially.

* As soon as **one** module successfully authenticates the request, the checking stops.
* It extracts your **Username**, **UID**, and **Groups**.
* If a module fails, it just passes the request to the next module. If all modules pass it along without a match, the server rejects it with a `401 Unauthorized` error.

---

### 2. The 4 Main Ways You Prove Who You Are

Kubernetes support several authentication strategies, but in production and cloud environments, you'll almost always see these two dominant players:

* **X.509 Client Certificates:** This is typically how `minikube` or bootstrap cluster admins authenticate. `kubectl` uses a local client certificate (stored in your `~/.kube/config`). The API Server validates this certificate against the cluster's Root Certificate Authority (CA). Your username is pulled directly from the Common Name (`CN`) field, and your groups are pulled from the Organization (`O`) fields of the certificate.
* **OpenID Connect (OIDC) Tokens (Bearer Tokens):** This is how enterprise clusters (like EKS, GKS, or Azure AKS) connect to identity providers (Okta, Entra ID, Google). You log into your provider, which hands `kubectl` a signed JSON Web Token (JWT). `kubectl` sends this in the HTTP header: `Authorization: Bearer <token>`. The API Server doesn't even need to contact your identity provider to verify it; it just decrypts and verifies the JWT's cryptographic signature using the provider's public keys.
* **Service Account Tokens:** What about pods talking to the API Server? They use Service Accounts. Kubernetes automatically mounts a signed JWT inside the pod at `/var/run/secrets/kubernetes.io/serviceaccount/token`. The API Server validates these tokens using its own private signing key.
* **Webhook Token Authentication:** Sometimes, a cluster needs to verify tokens against a custom external system. The API Server will bundle the incoming token into a JSON payload and `POST` it to a remote webhook service to ask, *"Is this token valid, and who does it belong to?"*

---

### 3. The Big Misconception: "User" Objects Do Not Exist

One of the most surprising things to learn when digging deep into Kubernetes architecture is that **Kubernetes does not have a `User` database or a `User` resource type.** If you run `kubectl get users`, you will get an error. You cannot create a user via YAML.

> **How it works instead:** Kubernetes expects an *external* system (like a certificate authority or an OIDC provider) to manage users. The Authentication phase simply acts as a translator. It looks at your certificate or token, extracts a string like `"alice@company.com"`, and hands that string over to the next gate (**Authorization / RBAC**). The API Server only cares if "Alice" is allowed to do what she's asking, not how Alice's account was created.

---

### 4. Anonymous Requests

If you haven't configured authentication tightly, or if a request comes in with no credentials, the API Server may evaluate it as an **Anonymous Request**. It assigns the user a username of `system:anonymous` and puts them in the `system:unauthenticated` group. In modern Kubernetes setups, RBAC blocks this anonymous user from doing anything useful by default, ensuring your cluster isn't open to the public internet.

Once the API Server successfully maps your request to a username and a list of groups, the Authentication gate swings open, and you are instantly handed off to the **Authorization (RBAC)** gate to see if you actually have permission to touch the cluster!

## Phase 2: Persisting Intent — Writing to etcd


![image.png](https://raw.githubusercontent.com/soumya-ranjan-000/image-hosting/main/articles/how-a-kubernetes-cluster-works/1779469361559-image.png)

>`etcd` is the ground truth of the entire cluster. The moment the write is confirmed, kubectl gets its 201 Created — your job is done. Everything from here on is asynchronous. The dashed green lines show the Watch stream: controllers maintain long-lived HTTP connections to the API Server and get notified the instant an object changes.

In **Phase 2: Persisting Intent**, the API Server transitions from a stateless gatekeeper into a stateful writer. This phase is entirely about establishing the cluster's **"Ground Truth."** The moment this phase successfully completes, your deployment is officially real as far as Kubernetes is concerned, even though not a single container has actually started running.

Here is the microscopic breakdown of exactly how the API Server prepares, serializes, and locks down your configuration into `etcd`.

---

### 1. In-Memory Conversion (The Internal Version)

When your request passes Phase 1 (Authentication, Authorization, and Admission Control), the resource exists as a versioned object—for example, `apps/v1.Deployment`.

Before doing anything else, the API Server converts this object into an **Internal Version** (an unversioned, in-memory Go struct).

* **Why?** Kubernetes APIs evolve over time (e.g., `extensions/v1beta1` transitioned to `apps/v1`). By converting everything into a single, canonical internal representation, the API Server ensures it can seamlessly handle older or newer API versions without maintaining separate database logic for each.

---

### 2. Storage Serialization (Protocol Buffers vs. JSON)

The API Server cannot write a raw Go memory struct to disk. It needs to flatten (serialize) the data.

```
[Internal Go Struct] ──(Serialize)──> [Protobuf / JSON Payload] ──(HTTPS PUT)──> [etcd]

```

* **The Format:** By default, when communicating with `etcd`, the API Server serializes the data into **Protocol Buffers (Protobuf)** because it is incredibly fast, compact, and efficient. If you explicitly configure it to or run older versions, it might fall back to JSON, but Protobuf is the production standard. [protocol-buffer-protobuf-in-system-design](https://www.geeksforgeeks.org/system-design/protocol-buffer-protobuf-in-system-design/)
* **Encryption at Rest:** If you have configured EncryptionProviders in your cluster, this is the exact micro-second it happens. The API Server takes the serialized Protobuf byte array and passes it through an encryption transformer (like AES-GCM or a KMS plugin) *before* sending it over the wire. The data traveling to `etcd` becomes completely encrypted ciphertext.

---

### 3. The `etcd` Write Path (The Raft Consensus)

The API Server acts as an `etcd` client. It initiates an HTTP/2 gRPC `PUT` request to the `etcd` cluster. The data is stored under a highly structured key path matching the resource hierarchy:


![image.png](https://raw.githubusercontent.com/soumya-ranjan-000/image-hosting/main/articles/how-a-kubernetes-cluster-works/1780330383397-image.png)


For your deployment, the exact key looks like this:
` /registry/deployments/default/my-app`

#### The Raft Quorum Check:

`etcd` is a distributed, consistent key-value store using the **Raft Consensus Algorithm**.

1. The API Server talks to the `etcd` **Leader** node.
2. The Leader receives the `PUT` request and logs it as a "provisional" entry in its write-ahead log (WAL).
3. The Leader replicates this log entry to all **Follower** nodes in the cluster.
4. The Leader waits for a **Quorum** (a strict majority, meaning $\lfloor N/2 \rfloor + 1$ nodes, where $N$ is the total cluster size) to reply saying they have safely appended the entry to their local WAL.
5. Once quorum is achieved, the Leader **commits** the change to its state machine and signals the followers to commit it too.

---

### 4. Generation of the `resourceVersion` (The Concurrency Guard)

When `etcd` commits the write, it assigns a critical piece of metadata to your resource: the **`resourceVersion`** (tied to `etcd`'s internal data revision counter).

> **Why this matters:** The `resourceVersion` is an explicit string indicator of a specific point in time. It prevents race conditions (Optimistic Concurrency Control). If two operators try to update your deployment at the exact same millisecond, the one with the outdated `resourceVersion` will be strictly rejected by the API Server with a `Conflict` error (`409 Conflict`), forcing it to read the new state before trying again.

---

### 5. The Asynchronous Trigger: The Watch System

Once the data is committed to `etcd`, the API Server returns an HTTP `201 Created` status code back to `kubectl`. Your terminal prints `deployment.apps/my-app created`. As far as you are concerned, the command is over.

However, behind the scenes, this write instantly triggers the **Watch System**:

* The `etcd` cluster maintains a continuous, low-latency streaming connection with the API Server.
* The moment the key `/registry/deployments/default/my-app` is committed, `etcd` pushes a **Watch Event** (`ADDED`) up to the API Server.
* The API Server receives this raw event, deserializes it back, and multiplexes it out to any component holding an open HTTP `WATCH` connection to that resource endpoint.

This is the exact handshake that wakes up the **Kube-Controller-Manager** in Phase 3, notifying it that a brand-new desired state has officially been written to the cluster's memory.

The **Watch System** is the central nervous system of Kubernetes. It is the architectural mechanism that allows Kubernetes to be entirely event-driven, highly scalable, and capable of reconciling changes in seconds without melting down the control plane.

Instead of controllers constantly asking the API Server, *"Are we there yet? Any new changes?"* (polling), the API Server streams changes to the controllers the exact millisecond they happen (streaming).

Here is a deep look into how the watch system functions under the hood.

---

### 1. The Core Problem: Why Polling Fails at Scale

Imagine a cluster with 5,000 Pods and 50 different controller loops running inside the control plane.

If every controller had to poll the API Server every 2 seconds to see if anything changed, the API Server would be hit with thousands of heavy `LIST` requests per minute. The server would spend all its CPU cycles reading from `etcd`, serializing massive JSON payloads, and sending them over the network. The cluster would grind to a halt.

The Watch System solves this by establishing a **long-lived, streaming HTTP connection** using standard HTTP chunked transfer encoding.

---

### 2. The Micro-Flow of a Watch Connection

The watch system operates like a subscription model. When a controller (like the ReplicaSet controller) starts up, it initiates a special HTTP request:

```http
GET /apis/apps/v1/namespaces/default/deployments?watch=true&resourceVersion=10245

```

```
[Controller] ──(HTTP GET ?watch=true)──> [API Server] ──(gRPC Watch)──> [etcd]
                                              │
[Controller] <──(Permanent HTTP Stream)───────┴<──(Stream Events)─────── [etcd]

```

1. **The Request:** The controller tells the API Server it wants to `watch` a specific resource type and passes a specific `resourceVersion`. This version represents the last known state the controller saw.
2. **The Connection Lock:** The API Server opens an HTTP connection but **does not close it**. It keeps the response channel open indefinitely using an HTTP chunked transfer encoding stream.
3. **The `etcd` Binding:** The API Server uses `etcd`'s native gRPC `Watch` API to subscribe to the storage keyspace matching that resource (e.g., `/registry/deployments/default/`).
4. **The Event Stream:** Whenever a resource is created, modified, or deleted, `etcd` pushes a small mutation event to the API Server. The API Server transforms this into a structured Kubernetes API Event and writes it straight into the open HTTP connection to the controller.

---

### 3. The Anatomy of an Event

The data traveling down the watch stream isn't just the raw object. It is wrapped in a special `WatchEvent` envelope that tells the controller exactly *what* happened:

```json
{
  "type": "ADDED", 
  "object": {
    "kind": "Pod",
    "metadata": { "name": "my-pod", "resourceVersion": "10567" },
    "spec": { ... }
  }
}

```

There are four primary event types sent down the wire:

* **`ADDED`:** A brand new resource was created.
* **`MODIFIED`:** An existing resource was changed (labels, specs, status, etc.).
* **`DELETED`:** A resource was removed from the cluster.
* **`BOOKMARK`:** A special, low-overhead sync event used to update the client on the current `resourceVersion` even if no data has changed (preventing timeouts).

---

### 4. Resiliency & The "Too Old Resource Version" Error

What happens if the network blinks and a controller loses its connection to the API Server?

When the controller reconnects, it sends a `GET ...?watch=true&resourceVersion=10567`, effectively saying, *"Hey, I dropped off at point 10567. Send me everything I missed."*

* **The Cache Hit:** The API Server keeps an in-memory **Watch Cache** (sliding window) of recent historical events. If version `10567` is still in the cache, the API Server catches the controller up by streaming the missed events, and the watch continues smoothly.
* **The Cache Miss (`410 Gone`):** `etcd` and the API Server cache only retain historical events for a limited window. If the controller was disconnected for too long and asks for a version that has already been purged, the API Server responds with an HTTP status code `410 Gone` (`Too old resource version`).

#### The Recovery:

When a controller hits a `410 Gone`, it knows its history is broken. It triggers a **List-Watch** cycle:

1. It performs a clean, single `LIST` request to fetch the entire current state of the cluster from scratch.
2. It notes the new, fresh `resourceVersion` returned by that list.
3. It immediately opens a brand-new `WATCH` stream starting exactly from that new version.

---

### 5. How Controllers Safely Process the Stream: Informers

Because raw watch streams can be finicky to manage, Kubernetes client libraries abstract this entire logic into a component called an **Informer**.

An Informer acts as a local cache inside the controller's memory. When a watch event arrives, the Informer updates its local cache and drops a lightweight key (like `namespace/pod-name`) into a **WorkQueue**. The controller's workers pull keys out of the queue and run their reconciliation logic against the local cache, ensuring the controller almost never needs to make direct, expensive read calls back to the API Server.

# Phase 3: The Controller Manager — Reconciliation Loop

This is where Kubernetes' real power lives. The Deployment Controller continuously compares desired state (what you wrote) vs. actual state (what exists).

![image.png](https://raw.githubusercontent.com/soumya-ranjan-000/image-hosting/main/articles/how-a-kubernetes-cluster-works/1780331768591-image.png)
Notice that controllers never act directly — they only write new objects back through the API Server. The Deployment Controller doesn't create pods; it creates a ReplicaSet. The ReplicaSet Controller then creates Pod specs. This layered ownership is deliberate: it makes rollbacks, rolling updates, and scaling each a clean, auditable operation.
At the end of Phase 3, pods exist as records in etcd — but they are Pending. They have no node. No container is running anywhere. They're just blueprints waiting for a home.

---
Think of the Kube-Controller-Manager as a dedicated team within the control plane. Inside this one box, you have specialized managers (or robots) that all work independently but read from the same central bulletin board: the API Server.
![Gemini_Generated_Image_1s5aki1s5aki1s5a.png](https://raw.githubusercontent.com/soumya-ranjan-000/image-hosting/main/articles/how-a-kubernetes-cluster-works/1780331867124-Gemini_Generated_Image_1s5aki1s5aki1s5a.png)


This diagram illustrates the continuous flow: Watch, Match, Correct.
1. The *Deployment Controller* is watching the API Server. It detects that you created a Deployment that requires 10 replicas of an image.
2. The Deployment Controller realizes this means it needs a new ReplicaSet for this specific version. It creates a ReplicaSet Definition and hands it to the API Server.
3. Now, the *ReplicaSet Controller* sees this brand new ReplicaSet Definition via the API Server. It performs the continuous math equation: 
Desired State (10 pods) not equal to Actual State (0 pods).
4. To correct this, the ReplicaSet Controller generates 10 individual Pod Definitions (which are essentially just blueprint paperwork in etcd) and tells the API Server to save them.

---
The crucial point is on the right: NOTHING IS RUNNING YET! 
The controller manager has done all its work, creating the paperwork inside etcd. The pods are now sitting in a Pending state, waiting for the Kube-Scheduler to find them a home.

## 😎 How does the ReplicaSet Controller create 10 individual Pods based on the single Pod template defined in its spec?

**The Deployment Controller did *not* save individual Pod definitions.** It only saved a single, high-level blueprint called a **ReplicaSet Spec**.

Here is exactly how it breaks down step-by-step.

---

### Step 1: What is inside a ReplicaSet Definition?

When the Deployment Controller creates a ReplicaSet, it writes **one single object** into `etcd`. Think of a ReplicaSet definition like a factory order form. It contains two main things:

1. **The Target Number:** `replicas: 10`
2. **The Pod Template:** A single copy of the blueprint for what a pod *should* look like (the image, ports, labels, etc.).

> 📝 **The Key Realization:** At this exact moment, `etcd` does **not** contain 10 pods. It only contains **one** ReplicaSet object that *specifies* it wants 10 pods.

---

### Step 2: The ReplicaSet Controller Wakes Up

The ReplicaSet Controller watches the API Server. It reads this new ReplicaSet object and executes its core logic loop:

1. **It Counts:** It queries the API Server: *"How many active pods currently exist in the cluster that match the label `app: my-web-app`?"*
2. **The Answer:** The API Server checks `etcd` and responds: *"Zero."*
3. **The Math:** 

![image.png](https://raw.githubusercontent.com/soumya-ranjan-000/image-hosting/main/articles/how-a-kubernetes-cluster-works/1780333620314-image.png)



---

### Step 3: How the 10 Individual Pod Definitions are Generated

Because the math tells it that 10 pods are missing, the ReplicaSet Controller enters a loop that runs exactly 10 times.

In each iteration of the loop, it performs a **copy-and-paste** operation:

1. **Copies the Template:** It takes the single "Pod Template" embedded inside the ReplicaSet object.
2. **Stamps Unique Identifiers:** A generic template cannot just be thrown into `etcd`. It needs to become a distinct entity. The controller dynamically generates unique metadata for each one:
* **Unique Name:** It takes the ReplicaSet's name and appends a random string (e.g., `my-web-app-79f8b`, `my-web-app-x2k4p`).
* **Owner Reference:** It attaches a hidden tag to the pod saying, *"My parent is this specific ReplicaSet."* (This is how it claims ownership of the pod later).


3. **Sends to API Server:** It fires off a standalone `POST /api/v1/namespaces/default/pods` request to the API Server.

The ReplicaSet Controller repeats this 10 times.

---

### Summary of the State Change in `etcd`

* **Before the ReplicaSet Controller acts:** `etcd` holds **1 object** (The ReplicaSet, which contains the number `10` and `1` template).
* **After the ReplicaSet Controller acts:** `etcd` now holds **11 objects** (The 1 original ReplicaSet + 10 distinct, individually named Pod definitions).

Now that those 10 distinct Pod definitions exist on "paper" inside `etcd`, the **Kube-Scheduler** finally notices them because their `nodeName` field is blank, kicking off the scheduling phase.
To make this statement technically accurate, we need to clarify a core Kubernetes concept: **The ReplicaSet Controller doesn’t actually generate or store 10 individual Pod definitions in advance.** Instead, it uses a single blueprint to create Pods dynamically.

---

### How It Actually Works (The Reconciliation Loop)

The ReplicaSet Controller doesn't copy-paste 10 separate definition files. It operates on a continuous **Reconciliation Loop** (Desired State vs. Current State).

```
[ Desired: 10 Pods ] <--- ReplicaSet Controller ---> [ Current: 0 Pods ]
                               |
                        (Creates 10 Pods)
                               v
                       [ API Server / etcd ]

```

Here is the step-by-step reality of what happens in `etcd`:

1. **The Single Source of Truth:** You submit a ReplicaSet manifest to the API Server. It contains a `replicas: 10` field and a `template` block (the Pod definition). This *single* ReplicaSet object is stored in `etcd`.
2. **The Headcount:** The ReplicaSet Controller notices the new ReplicaSet. It counts how many existing Pods in the cluster match the ReplicaSet's **label selector**.
3. **The Creation:** If it counts 0 matching Pods, it realizes it is 10 Pods short. It then loops 10 times, sending 10 individual HTTP `POST` requests to the API Server.
4. **Dynamic Generation:** Each request says, *"Please create a Pod using this `template`, and give it a unique name (e.g., `my-app-abcde`)."*
5. **Storage:** The API Server receives these 10 distinct requests and writes 10 individual **Pod objects** into `etcd`.

> 💡 **Key Takeaway:** The ReplicaSet Controller doesn't pre-generate 10 definitions. It uses the *one* definition template it already has, over and over again, to instruct the API Server to spin up individual Pod instances.

## 🔎Why does Kubernetes create distinct Pod instances (with unique names and IPs) for the same Pod type/template?

It can seem counterintuitive at first. If all 10 pods are running the exact same code, the exact same container image, and the exact same configuration, why can't Kubernetes just use one single definition and scale it up?

The reason comes down to how Kubernetes manages infrastructure. Kubernetes is not just a deployment tool; it is a live, granular tracking system. To manage a distributed system reliably, **every single container instance must have its own unique identity and lifecycle.**

Here is exactly why we need distinct Pod definitions for the same pod type:

---

### 1. Individual Tracking and Placement (The Scheduler's Job)

A single blueprint cannot be in two places at once. The **Kube-Scheduler** needs to assign pods to physical or virtual worker nodes based on available resources.

* If you have 10 pods, the scheduler might put 3 on Node A, 4 on Node B, and 3 on Node C.
* To do this, it must update the specific `nodeName` field in the Pod's definition (e.g., `pod-abc` $\rightarrow$ Node A, `pod-xyz` $\rightarrow$ Node B).
* Without distinct definitions, Kubernetes wouldn't know which individual pod lives on which server.

---

### 2. Micro-Managing Health (The "Dead Engine" Analogy)

Think of an airline fleet. An airline might own 50 identical Boeing 737 airplanes. They have the same blueprints, same engines, and same seating charts.

However, the airline still tracks each plane by a distinct tail number. Why?

* If Plane #3 has an engine failure mid-flight, the mechanics need to fix **only Plane #3**. They don't service the entire fleet of 50 planes.
* In Kubernetes, if one container crashes, leaks memory, or fails its health check (Liveness Probe), the **Kubelet** needs to restart *only that specific pod*. Distinct definitions allow Kubernetes to isolate and heal a single failing instance without disturbing the healthy ones.

---

### 3. Unique Networking (IP Addresses)

Every single Pod in a Kubernetes cluster gets its own unique, cluster-internal IP address.

```
                       ┌──► Pod-79f8b (IP: 10.244.1.5) -> Running on Node A
                       │
[ ReplicaSet Template ]├──► Pod-x2k4p (IP: 10.244.2.9) -> Running on Node B
                       │
                       └──► Pod-99a1z (IP: 10.244.1.6) -> Running on Node A

```

When a pod is scheduled and network plugins (like Calico or Flannel) allocate an IP, that IP address is written directly into the Pod's status definition. If there weren't distinct definitions, the cluster network would have no way to route traffic to individual containers.

---

### 4. Granular Logging and Debugging

When something goes wrong in production, you use commands like:
`kubectl logs my-web-app-79f8b`

Because each pod has a distinct name and definition, its standard output (logs) and performance metrics (CPU/Memory usage) are aggregated individually. If all 10 pods shared a single definition, debugging would be a nightmare because the logs of all 10 containers would be tangled together into a single, unreadable stream.

---
 Every single pod instance has its own unique, real-time lifecycle and story, and Kubernetes needs a dedicated "ledger entry" (the Pod definition) to keep track of it.

**Think of it this way:** the Pod template inside the ReplicaSet is the **cookie cutter**, but the individual Pod definitions are the **actual cookies** trackable in real-world space.

Here is the exact real-time data that Kubernetes writes into a distinct Pod definition *after* it gets created from the template:

### What gets added to a distinct Pod definition?


![image.png](https://raw.githubusercontent.com/soumya-ranjan-000/image-hosting/main/articles/how-a-kubernetes-cluster-works/1780334104235-image.png)


---

### A Real-World Scenario: The "CrashLoopBackOff"

Imagine you scale your app to 3 replicas. Pod 1 and Pod 2 are healthy, but Pod 3 lands on a faulty worker node with a corrupted disk, causing that specific container to constantly crash.

```
                  ┌──► Pod-abc (Status: Running, Node: Node A)
                  │
[ ReplicaSet ] ───┼──► Pod-def (Status: Running, Node: Node B)
                  │
                  └──► Pod-xyz (Status: CrashLoopBackOff, Node: Node C) 💥

```

If Kubernetes didn't generate a distinct Pod definition for `Pod-xyz`, the cluster wouldn't be able to isolate the issue. It would look at the shared template and say, *"The template looks fine!"* Because it has a unique definition for `Pod-xyz`, the **Kube-Controller-Manager** can pinpoint the exact failure, see that its `restartCount` is spiking on Node C, and decide to delete *just that one pod* and recreate it somewhere else.

It is all about **state management and individual ownership.**

In the world of systems engineering, this is the difference between a **Class** (the blueprint/template) and an **Object** (the actual live instance running in memory).

By giving every single pod its own distinct definition, Kubernetes can run a continuous, highly precise loop for every single container instance in your cluster.

---

### The 4-Step Life of a Distinct Pod Definition

To see how Kubernetes uses this distinct definition to monitor and make decisions, look at its lifecycle:

```
[ 1. BIRTH ]      ──► ReplicaSet copies the template, gives it a unique name, 
                      and saves it to etcd. (Status: Pending)
      │
      ▼
[ 2. PLACEMENT ]  ──► Scheduler reads the blank "nodeName" field, picks a worker node, 
                      and writes it into the definition.
      │
      ▼
[ 3. MONITOR ]    ──► Kubelet on that worker node boots the container, gets an IP, 
                      and continuously updates the definition with its health status.
      │
      ▼
[ 4. DECISION ]   ──► Controller-Manager monitors the definition. If status changes 
                      to "Failed", it triggers a replacement decision.

```

Throughout a Pod's entire life cycle—from the millisecond it is conceived until the millisecond it is deleted—the **only** source of truth is its dedicated Pod definition in `etcd`. Every milestone, change of health, and resource update is recorded directly into that specific "ledger entry" via the API Server.

---

### The Life Cycle Ledger in Action

To see how this works, let's trace the exact fields that get continuously updated inside a single Pod's `etcd` definition as it lives its life:

```
[ Pod Created ] ──► (State: Pending) ──► [ Scheduled ] ──► (nodeName: Node-3) ──► [ Container Starts ] ──► (podIP: 10.244.1.5) ──► [ Running ]

```

#### 1. Creation Phase (The "Pending" State)

* **What gets written:** The ReplicaSet Controller writes the Pod definition to `etcd`. At this moment, fields like `spec.nodeName` and `status.podIP` are completely empty or set to `null`.
* **The Status:** The API Server marks the `status.phase` as **`Pending`**.

#### 2. Scheduling Phase (The "Assigned" State)

* **The Update:** The Kube-Scheduler picks a physical machine (e.g., `worker-node-02`).
* **What gets written:** The Scheduler sends an update to the API Server, which modifies the definition in `etcd`, changing `spec.nodeName: null` $\rightarrow$ **`spec.nodeName: worker-node-02`**.

#### 3. Initialization Phase (The "Container Loading" State)

* **The Update:** The Kubelet on `worker-node-02` sees its name in the definition and tells the container runtime (like `containerd`) to pull the Docker image.
* **What gets written:** The Kubelet updates `etcd` through the API Server, setting `status.containerStatuses[0].state` to **`Waiting`** with the reason `ContainerCreating`.

#### 4. Execution Phase (The "Running" State)

* **The Update:** The container successfully starts, and the network plugin assigns the pod a unique IP address (e.g., `10.244.2.45`).
* **What gets written:** The Kubelet fires another update to the API Server. In `etcd`, `status.podIP` becomes **`10.244.2.45`**, and `status.phase` transitions to **`Running`**.

#### 5. The Monitoring Loop (Continuous Updates)

* **The Update:** Every few seconds, the Kubelet runs health checks (Liveness/Readiness probes) on the container.
* **What gets written:** If a probe fails or succeeds, or if the container crashes and restarts, the Kubelet continuously updates fields like `status.containerStatuses[0].restartCount` and `status.conditions` inside `etcd`.

---

### The Crucial Concept: The API Server is a "State Machine"

Because of this constant updating, a Pod definition in Kubernetes isn't just a static configuration file—it is a **dynamic state machine**.

Components don't manage the pod by sending direct commands to it. Instead, they **modify the state in `etcd**`, and other components react to that change.

* **To run a pod:** The Scheduler updates the `nodeName` in `etcd`. The Kubelet reacts to that change.
* **To delete a pod:** When you run `kubectl delete pod`, Kubernetes doesn't immediately kill the container. It updates `etcd`, marking the pod's `metadata.deletionTimestamp`. The Kubelet watches `etcd`, sees this timestamp change, and begins gracefully shutting down the actual container.

To make this line accurate, we need to clarify what makes the Pods "distinct."

Even if you are deploying the exact same application type, Kubernetes creates distinct **Pod objects** (or instances) rather than using a single shared record.

Here is the corrected line:

> **Corrected Line:** *Why does Kubernetes create distinct Pod instances (with unique names and IPs) for the same Pod type/template?*

---

### Why Distinct Pods Are Necessary

Even though the underlying container image and configuration are identical, Kubernetes treats each Pod as an independent worker for several critical reasons:

* **Individual IP Addresses:** Every Pod gets its own unique IP address within the cluster network. This allows Kubernetes to load-balance traffic across them and lets the Pods communicate without port conflicts.
* **Independent Lifecycles:** If one container crashes, runs out of memory, or hangs, Kubernetes needs to restart or replace *only that specific instance* without affecting the other healthy ones.
* **Granular Scheduling:** The Kubernetes Scheduler places individual Pods on different physical or virtual nodes across the cluster. This ensures **high availability**—if one node goes down, only the Pods on that node die, while the identical Pods on other nodes keep running.
* **Accurate Health Tracking:** The control plane tracks the individual resource usage (CPU/Memory) and health status (liveness/readiness probes) of each distinct instance.

> 💡 **Analogy:** Think of it like a fleet of identical delivery trucks. They all look the same and do the same job, but they still need unique license plates (IPs/Names) because they drive on different routes, require individual maintenance, and can't all be driven by the same person at the same time.

## The Full Flow So Far — at a Glance

![image.png](https://raw.githubusercontent.com/soumya-ranjan-000/image-hosting/main/articles/how-a-kubernetes-cluster-works/1780335322480-image.png)

# The Scheduling Flow (Deciding where the app runs)

Its primary objective is to solve a classic distributed systems problem: **constraint-satisfaction and optimal placement of workloads across a heterogeneous cluster.**

Let's break down the mathematical logic, architectural mechanics, and the precise two-phase algorithm the scheduler executes.

---

### The Algorithmic Goal

The scheduler's job is to assign a pending Pod to a single optimal Node. Mathematically, for a given Pod $P$ and a set of all Nodes $N$:

1. It filters $N$ to find a subset of **Feasible Nodes** ($V$) that satisfy all constraints.
2. It scores each node in $V$ using a weighted set of priority functions.
3. It selects the node $n \in V$ with the highest cumulative score.

---

## The Two-Phase Scheduling Cycle

Every time a Pod definition is created with an empty placement field (`spec.nodeName: null`), it enters the Active Scheduling Queue. The scheduler evaluates it using a sequential, pipeline architecture consisting of two main phases: **Filtering** and **Scoring**.

```
[Active Queue] ──► [ Phase 1: Filtering ] ──► [ Phase 2: Scoring ] ──► [ Final Selection & Binding ]
                        (Predicates)                  (Priorities)

```

### Phase 1: Filtering (Predicates)

In this phase, the scheduler runs a series of hard-constraint checks called **Predicates**. If a node fails even one predicate, it is immediately disqualified.

Key university-level predicates include:

* **PodFitsResources:** A deterministic check ensuring the node's allocatable CPU and Memory are greater than or equal to the Pod's requested resources:

$$\text{Node}_{\text{Allocatable}} \ge \text{Node}_{\text{Allocated}} + \text{Pod}_{\text{Requested}}$$


* **PodFitsHostPorts:** Checks if the specific network port requested by the Pod (`hostPort`) is already bound on the node.
* **NodeMatchSelector & NodeAffinity:** Evaluates hard labels. If a Pod requires `topology.kubernetes.io/zone=us-east-1a`, any node outside this zone is dropped.
* **PodTopologySpread:** Evaluates failure-domain constraints to ensure high availability (e.g., preventing all replicas from clustering on the same rack or zone).

If the filtering phase results in zero nodes, the Pod remains in a `Pending` state with a `SchedulingFailed` event logged in its state definition.

---

### Phase 2: Scoring (Priorities)

Once the scheduler establishes the list of feasible nodes ($V$), it must rank them. It applies a series of soft-constraint functions called **Priorities**.

Each priority function returns a score from `0` to `10`. The scheduler then multiplies these scores by a configured **Weight** factor and sums them up.

The total score for a Node ($n$) is calculated as:


$$\text{Total Score}(n) = \sum_{i=1}^{k} (\text{Score}_i(n) \times \text{Weight}_i)$$

Prominent priority functions include:

1. **LeastRequestedPriority:** Favors nodes with fewer allocated resources to achieve uniform resource balancing. It calculates the ratio of free capacity to total capacity:

$$\text{Score} = \frac{(\text{Capacity}_{\text{cpu}} - \text{Requested}_{\text{cpu}}) + (\text{Capacity}_{\text{memory}} - \text{Requested}_{\text{memory}})}{\text{Capacity}_{\text{cpu}} + \text{Capacity}_{\text{memory}}} \times 10$$


2. **ImageLocalityPriority:** Checks if the worker node has already cached the required container images locally. Nodes that already contain the image layers score higher because they eliminate network latency during container initialization.
3. **NodeAffinityPriority:** Evaluates *preferred* (soft) affinity rules (e.g., *"I would prefer to run on an ARM64 node, but it's not strictly mandatory"*).

---

## Architectural Climax: Pessimistic Locking & Preemption

What happens if multiple schedulers run concurrently, or if two pods try to claim the exact same remaining resource on a node at the same millisecond?

To prevent race conditions (known as **double-booking**), the scheduling cycle is executed **sequentially** per Pod, utilizing two distinct operational loops:

```
┌────────────────────────────────────────────────────────┐
│               1. SCHEDULING CYCLE (Sequential)        │
│  [Filter Nodes]  ──►  [Score Nodes]  ──►  [Assume Node] │
└───────────────────────────────────────────────┬────────┘
                                                │ (Optimistic Reservation)
                                                ▼
┌────────────────────────────────────────────────────────┐
│               2. BINDING CYCLE (Asynchronous)          │
│               [Asynchronously write state to etcd]     │
└────────────────────────────────────────────────────────┘

```

1. **The Scheduling Cycle (Synchronous):** The scheduler algorithmically selects the best node. Instead of waiting for a slow network write to `etcd`, it immediately executes an **"Assume"** operation. It updates its local, in-memory cache to temporarily reserve those resources. This allows the scheduler to instantly process the next Pod in line without data races.
2. **The Binding Cycle (Asynchronous):** Concurrently, a separate thread sends a `Binding` object payload to the API Server via a `POST /api/v1/namespaces/{ns}/pods/{name}/binding` request.

The API Server performs final atomic validation and mutates the Pod definition ledger entry in `etcd`:

$$\text{spec.nodeName: null} \longrightarrow \text{spec.nodeName: worker-node-B}$$

---
**Let's** see and example of Pessimistic Locking (Concurrency Control) and Preemption using a very simple, real-world analogy: **Booking seats at a packed movie theater**.

---

## 1. Concurrency Control: The Movie Theater Problem

Imagine you and a stranger are both trying to buy the **very last ticket** for a blockbuster movie at the exact same millisecond.

* **The Problem:** If the ticket system isn't careful, it might process both your credit cards at the same time and accidentally sell that single seat to both of you. This is a **race condition** (or double-booking).
* **How the Kube-Scheduler handles it:** To move fast without breaking things, the scheduler uses a clever two-part strategy: **Sequential Logic** and **Optimistic Assumption**.

Instead of locking up the whole database (`etcd`) while it makes a decision, the scheduler handles pods **one by one** in a strict line.

```
[ Pod Queue ] ──► (1. Pick Node) ──► (2. Assume / Cache) ──► (3. Async Bind) ──► [ etcd ]
                                           │
                                           └─► (Blocks next pod from stealing the seat)

```

1. **The Choice:** The scheduler looks at the cluster and decides, *"Pod A gets Node 1."*
2. **The "Assume" Step (Local Lock):** Instead of waiting for `etcd` to slowly write that data down over the network, the scheduler instantly updates its *own local memory notebook* (called a cache): *"Node 1 is now full."* 3. **The Async Binding:** While a slow background process goes off to update the official records in `etcd`, the scheduler instantly jumps to the next pod in line. Because its local notebook already shows Node 1 is full, it won't accidentally try to put Pod B there.

This design ensures that Kubernetes can schedule thousands of pods per second without ever double-booking a server.

---

## 2. Pod Preemption: The VIP Eviction

Now, what happens if the movie theater is completely sold out, but a **VIP celebrity** shows up and demands a seat? The theater manager has to pick a regular ticket holder, refund them, and give their seat to the VIP.

In Kubernetes, this process is called **Preemption**, and it revolves around **Pod Priority**.

### The Scenario

Your cluster is 100% full. Suddenly, a critical database pod (a VIP) needs to be scheduled.

1. **The Panic:** The Scheduler runs its filtering checks and realizes: *"Zero nodes have enough room. This critical pod is stuck."*
2. **The Rescue (Preemption):** Instead of giving up, the scheduler looks at the pods already running on the nodes and checks their priority levels.
3. **The Eviction:** It finds a node running a low-priority pod (like a background testing script) and says, *"You have to leave."*
4. **The Update:** The scheduler triggers a deletion for the low-priority pod and safely assigns the empty slot to the high-priority VIP pod.

---

### Summary in Simple Terms

* **Concurrency/Optimistic Locking** is how the scheduler makes promises to itself in its local notebook so it doesn't accidentally double-book a server while waiting for the main database to update.
* **Preemption** is the security guard kicking out a low-priority "testing" pod to make room for a high-priority "production" pod when the cluster runs out of space.

Now you see how the scheduler balances both speed (locking mechanisms) and urgency (preemption). 

## 🤔 But how do multiple schedulers work if they store node information in a cache? Since the cache is local to each scheduler, other schedulers won't know which nodes have already been reserved by another scheduler?

You have hit on one of the most advanced architectural bottlenecks in distributed systems design! Your logic is 100% correct: if Scheduler A reserves a node in its local cache, Scheduler B has no idea about it because they don't share memory.

If both schedulers simultaneously see 2GB of free RAM on Node 1, they might both greedily assign their pods to Node 1 at the exact same time.

To solve this concurrency nightmare without sacrificing speed, Kubernetes implements **Optimistic Concurrency Control (OCC)** at the **API Server layer**.

Here is exactly how Kubernetes allows multiple schedulers to run without them stepping on each other's toes.

---

## The Solution: The API Server acts as the Ultimate Umpire

While schedulers use their local caches to guess quickly, they do not have the final say. The **API Server** is the single source of truth, and it uses a special tracking number on every single object called a **`resourceVersion`**.

Think of the `resourceVersion` like a **timestamp or a version token** stored inside `etcd` for every Node and Pod.

### The Multi-Scheduler Race Flow

Let’s watch what happens when Scheduler A and Scheduler B try to book the exact same remaining slot on Node 1 at the same millisecond.

```
                  ┌─── [ Scheduler A ] ───(1. Assume & Bind Request)───┐
                  │                                                    ▼
[ Node 1 (v100) ]─┤                                             [ API Server ] ──► [ etcd ]
                  │                                                    ▲
                  └─── [ Scheduler B ] ───(2. Assume & Bind Request)───┘

```

#### Step 1: The Shared Illusion

Both Scheduler A and Scheduler B read the cluster state from the API Server. They both see that **Node 1 is at Version 100 (`resourceVersion: "100"`)** and has enough space for one more pod. They both copy this data into their local caches.

#### Step 2: The Independent Math

* **Scheduler A** processes Pod X $\rightarrow$ decides Node 1 is the best fit $\rightarrow$ marks Node 1 as full in its *local* cache.
* **Scheduler B** processes Pod Y $\rightarrow$ decides Node 1 is the best fit $\rightarrow$ marks Node 1 as full in its *local* cache.

#### Step 3: The Race to the API Server

Both schedulers fire off an asynchronous `Binding` request to the API Server. This request essentially says: *"I want to bind this pod to Node 1, based on the fact that I looked at Node 1 when it was at **Version 100**."*

#### Step 4: The Atomic Verdict (How the conflict is resolved)

The API Server handles these two incoming network requests sequentially (atomically):

1. **Scheduler A's request arrives first:** The API Server checks `etcd`. It sees Node 1 is currently at Version 100. It matches! The API Server says, *"Approved."* It updates the pod's `nodeName`, and **automatically bumps Node 1's version to 101** in `etcd`.
2. **Scheduler B's request arrives a millisecond later:** The API Server checks `etcd`. It sees Node 1 is now at **Version 101**. It looks at Scheduler B's request, which says *"bind based on Version 100"*.
3. **The Rejection:** The API Server shouts: **"Conflict! The data you used to make your decision is stale!"** It rejects Scheduler B's request with a `409 Conflict` HTTP status code.

---

## What happens to the rejected Pod?

Because Scheduler B's binding failed at the API Server gate, nothing is written to `etcd` for Pod Y.

1. Pod Y's `nodeName` remains `null`.
2. Scheduler B receives the error, realizes its local cache is out of date, and flushes its cache to sync back up with the API Server.
3. Pod Y is put back into the scheduling queue to be re-evaluated cleanly in the next cycle.

---

### Summary: Why this design works

Instead of using **Pessimistic Locking** (which would mean Scheduler A locks the entire cluster database while it thinks, making everything painfully slow), Kubernetes uses **Optimistic Locking**.

Schedulers are allowed to freely guess and run calculations in parallel using their local caches. If they happen to clash, the API Server caught them using the `resourceVersion` tag, cleanly rejects the loser, and forces a retry.

# **The Worker Node Execution Flow (Bringing the Pod to Life)**.

Up until this point, everything has just been "paperwork" changing hands inside a secure database (`etcd`). No actual software code is running, and no container exists yet.

---

# The Next Component: The Kubelet

The **Kubelet** is the captain of the Worker Node. It is a tiny, powerful agent that runs on every single machine in your cluster. It constantly watches the API Server, waiting for its specific node's name to appear in a Pod's definition.

When the API Server updates a pod definition to say `spec.nodeName: worker-node-A`, the Kubelet on Worker Node A instantly wakes up and triggers a 3-step physical launch sequence:

### 1. The Container Runtime Interface (CRI)

The Kubelet doesn't actually know how to run a container itself. Instead, it speaks a standardized language called **CRI** to talk to a container runtime engine installed on the machine (like `containerd` or Docker).

* It tells the runtime: *"Hey, go pull the image `my-web-app:v1` from the registry and spin up a container with these resource limits."*

### 2. The Container Network Interface (CNI)

Once the container is created, it needs to talk to the outside world. The Kubelet calls out to a networking plugin (like Calico, Flannel, or Cilium) using the **CNI** protocol.

* The CNI plugin configures a virtual network interface, hooks the container into the cluster's network fabric, and assigns it a unique, dedicated **Pod IP address**.

### 3. The Container Storage Interface (CSI)

If your application needs to save data permanently (like a database or file upload service), the Kubelet calls out to storage drivers via the **CSI** protocol.

* It hooks up cloud disks or local network storage directly into the running container.

---

## The Feedback Loop

Once the container is successfully running and has an IP address, the Kubelet turns around, reports back to the **API Server**, and says: *"Mission accomplished. Pod is up, healthy, and its IP is 10.244.1.45."* The API Server writes this final status into `etcd`, changing the pod's state to **`Running`**. The loop is complete.


