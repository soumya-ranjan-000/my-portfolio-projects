# How Does the System Look, and What Performance Do I Need to Evaluate?

I will not be diving deep into how the application works internally. Instead, I will provide a high-level overview.

First, there is a backend application (built using Spring Boot) whose job is to process requests received from clients (in this case, the clients are companies).

Now, the question is: what does a request contain?

A request contains:

* A unique batch ID
* A payload containing 5,000–10,000 records

The payload size can vary between 10 MB and 20 MB.

### What does the application do with these records?

The application processes each record individually. Each record contains steps to retrieve related data from a database or another server. Later, all of this data is accumulated into a single JSON object and sent back to the client server.

### The flow looks like this:
1. Company A sends a request to the application server.
2. The application server immediately sends an initial response to Company A. We can call this a feedback message or acknowledgement, which simply says:
   > "Yes, I have received your request and started processing it."
3. Now the real work begins. The application server processes the 5,000 records present in the payload one by one and keeps adding the responses to a single JSON object.
4. Once all 5,000 records have been processed successfully, the application redirects the final response back to Company A's registered server.

## What Does My Test Environment Look Like?

The batch servers (the actual application servers) are hosted on AWS. They are designed to auto-scale using:

* Auto Scaling Groups (ASG)
* Target Groups
* Load Balancers

The customer server sits nearby in GCP 😁.

## So, What Is My Job Here?

I have been asked to evaluate the performance of the batch servers.

The evaluation should cover the following points when a client sends **1 request per second**, where each request contains a minimum of **5,000 records**.

### Performance Areas to Evaluate

1. **Initial response/acknowledgement time**
   * Measure the time taken between the client sending the request and receiving the acknowledgement from the batch server.
2. **Auto-scaling behavior**
   * Evaluate how effectively the Auto Scaling Group (ASG) rules respond to increasing workload.
3. **End-to-end batch processing time**
   * Measure the total time taken by the application to process the batch and send the final response (redirect) to the client's server hosted on GCP.
4. Data completeness and integrity
   * The client server must receive all processed responses as part of the final request.
   * The process will be considered unsuccessful if information for any of the 5,000 records is missing.
   * Any loss of record data during processing or response transmission will be treated as a request failure.

---

# My Testing Strategy

Initially, I started using the existing test scripts written in Python using Locust.

## Initial Testing Approach

### 1. Test Script Design

I had a Locust test script designed to perform **one task per second**, randomly selected from a predefined set of tasks.

In this context, a **task** represents sending a request to the batch processing server. Each request contains:

* A payload with **5,000 records**
* A randomly generated `batch_id`

### 2. Test Environment Setup

I created several test servers within the same Virtual Private Cloud (VPC) where the batch servers were running.

The reason for this setup was that, due to security restrictions, I was not allowed to create test servers outside the VPC.

**Test Constraint #1:** All load generation servers had to reside within the same VPC as the target batch servers.

### 3. Test Objective

My goal was to perform load testing for:

* 10 users (representing 10 clients)
* Initial target infrastructure consisting of 18 batch servers

### 4. Load Generation

I used one test server to execute the Locust script and generate traffic toward the batch servers.

### 5. First Challenge

During execution, Locust started reporting warnings indicating CPU utilization of approximately 90%.

This warning suggested that:

* The load generator itself was becoming resource-constrained.
* Test accuracy and reporting reliability could be affected.

The reason was straightforward:

* One server was generating all traffic.
* 10 users were each sending 1 request per second.
* Every request contained 5,000 records.

As a result, the single load generator became a bottleneck.

### 6. Distributed Testing Solution

To overcome this limitation, I decided to distribute the load generation across multiple servers.

Instead of one server simulating all 10 users, multiple servers would share the workload.

Benefits:

* Reduced CPU and memory utilization on individual test servers.
* Improved test stability.
* Ability to execute longer-duration tests reliably.

### 7. Locust Master-Worker Architecture

Locust supports distributed execution through a Master-Worker architecture.

#### Master Node Responsibilities

* Controls worker nodes.
* Coordinates test execution.
* Aggregates metrics and statistics.
* Generates final reports.

#### Worker Node Responsibilities

* Generates actual traffic.
* Sends requests to target servers.
* Reports execution metrics back to the master.

### 8. Distributed Setup Requirements

To establish the master-worker setup:

* Locust commands had to be executed on each server.
* All test files needed to be available on every server (master and workers).
* Configuration had to follow Locust's distributed execution guidelines.

### 9. Infrastructure Selection

I selected a total of five test servers:

* 1 Master Server
* 4 Worker Servers

### 10. Test File Distribution

All test files were stored in an S3 bucket.

Although GitHub, GitLab, or Bitbucket could have been used, organizational security policies prevented configuring Git credentials and repository access on the servers.

**Test Constraint #2:** Git-based repository access was not permitted on test servers.

---

# My First Approach

To execute a test, I had to manually perform the following steps:

### Step 1

Log in to all AWS test instances:

* 1 Master
* 4 Workers

### Step 2

Download the latest test files from the S3 bucket.

### Step 3

Start the worker processes on all worker EC2 instances.

### Step 4

Start the master process on the master EC2 instance.

Once started:

* The master automatically connected to all workers.
* Workers began generating traffic toward the target batch servers.
* Metrics were collected and reports were generated.

### Important Limitation

Locust could only measure the initial acknowledgement response.

It could **not** measure:

* Batch processing completion time.
* Internal batch execution duration.
* Time taken to forward results to customer systems.

### Step 5

After test completion, I manually uploaded the generated reports to S3 and downloaded them to my local machine for analysis.

---

## Why I Used CLI Instead of the Locust Web UI

All test execution was performed through the command line.

Although Locust provides a web interface, using it was not feasible because:

* It requires exposing ports.
* It requires public IP accessibility.
* Corporate security policies prohibited exposing internal servers.

None of the servers had public IP addresses:

* Test servers
* Batch servers
* Supporting infrastructure

Communication was only possible because all systems resided within the same VPC.

**Test Constraint #3:** No public IPs or externally accessible ports were allowed.

---

## Challenges with the First Approach

The process was highly repetitive and operationally expensive.

For every test run, I had to:

* Log in to multiple servers.
* Download files.
* Configure environments.
* Start worker processes.
* Start the master process.
* Collect reports manually.

Repeating these steps for every execution became a significant operational burden.

---

# My Second Solution – Automation Using AppleScript

To eliminate repetitive manual work, I developed AppleScripts on my MacBook to automate the entire setup and execution process.

The scripts performed the following actions automatically:

### 1. Connect to All Test EC2 Instances

Automatically established connections to all servers.

### 2. Install Required Dependencies

Ensured all test-related dependencies were installed.

### 3. Create Required Directories

Created directories for:

* Test files
* Logs
* Reports

### 4. Download Test Files

Pulled all required files from the S3 bucket.

### 5. Configure Environment Variables

Set all required runtime configurations.

### 6. Start Test Execution

Triggered both master and worker processes.

### 7. Upload Results Automatically

Once execution completed, reports were automatically uploaded to S3.

---

# Session Timeout Challenge

Initially, I could only run tests for approximately 45–50 minutes.

The reason was that the AWS SAML session used for terminal access expired after one hour.

If a long-running test exceeded the session duration:

* AWS authentication expired.
* SSH sessions disconnected.
* Ongoing tests were often lost.

### What Happens When the Terminal Disconnects?

When an AWS SAML session expires or an SSH connection drops, the terminal session is terminated.

By default, any foreground process (such as Locust) also receives a **SIGHUP (Hangup Signal)** and is terminated.

As a result:

* The load test stops.
* Metrics collection stops.
* Progress is lost.

---

# Final Improvement – Background Execution

To solve this problem, I enhanced the AppleScript automation.

Instead of running Locust in the foreground, the script:

* Started Locust as a background process.
* Redirected logs to files.
* Stored reporting data continuously.
* Captured important execution information for troubleshooting and analysis.

This ensured that tests continued running even if my terminal session disconnected.

---

# Results

This approach enabled significantly longer and more reliable test executions.

Typical execution durations included:

* 8-hour tests
* 12-hour tests
* Full 24-hour tests

Each test execution was assigned a unique Test ID.

Using the Test ID, I could:

* Track execution status on EC2 instances.
* Review logs.
* Retrieve reports from S3.
* Investigate failures without reconnecting to active sessions.

This automation greatly improved reliability, reduced manual effort, and enabled long-duration performance testing at scale.

---

## The New Problem That Arose

Since the application is expected to support **1 request per second (RPS) per user**, I wrote my Locust script in the following way to ensure that each worker sends exactly one request every second to the target server.

```python
from locust import HttpUser, task, between, constant

class OneRequestPerSecondUser(HttpUser):
    # Ensures each user waits exactly 1 second between tasks
    wait_time = constant(1)

    @task
    def send_request(self):
        self.client.get("/")  # Replace "/" with your endpoint path
```

`constant(1)` guarantees that each simulated user waits exactly one second between task executions.


![Observed RPS Decline Due to Increasing Response Time.png](https://raw.githubusercontent.com/soumya-ranjan-000/image-hosting/main/projects/how-i-analyzed-scalability-and-end-to-end-latency-for-aws-hosted-batch-servers-using-k6/1781942570307-Observed_RPS_Decline_Due_to_Increasing_Response_Time.png)
text

After reviewing my performance report, the SRE raised a question:

> Why is the Requests Per Second (RPS) declining over time?
>
> Since we are sending 1 request per second, the RPS graph should ideally be a straight line.

To answer this, I started exploring how Locust sends requests internally.

Eventually, I found the root cause.

The Locust framework is designed in such a way that a user must wait for the current task to complete before moving on to the next task. In other words, task execution is sequential for a single virtual user.

In my case, Locust was not receiving responses from the server within one second. As a result, it had to wait for the previous request to complete before it could schedule the next one.

This behavior is expected and is how Locust is designed to work.

### Example Scenarios

#### Scenario 1: Response Time Less Than 1 Second

```
1st request sent
→ Response received in 30 ms
→ User waits for the remaining 970 ms
→ Next request is sent

Effective rate ≈ 1 request per second
```

This is the ideal scenario and maintains a steady 1 RPS per user.

#### Scenario 2: Response Time Greater Than 1 Second

```
1st request sent
→ Response received in 1.5 seconds
→ Next request is sent immediately after the response is received
```

In this case, the user cannot maintain 1 request per second because the request itself takes longer than one second to complete.

As response times increase, each virtual user becomes blocked waiting for responses, resulting in a gradual decline in the overall RPS. Therefore, even though the script is configured for 1 request per second, the actual throughput becomes dependent on the application's response time.


