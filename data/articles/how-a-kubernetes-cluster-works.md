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

$$\text{Key Format: } \texttt{/registry/}\langle\text{resource\_type}\rangle\texttt{/}\langle\text{namespace}\rangle\texttt{/}\langle\text{name}\rangle$$

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
