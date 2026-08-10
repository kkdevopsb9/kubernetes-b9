# Kubernetes Probes: Liveness, Readiness, and Startup

Kubernetes probes continuously check the health and availability of containers running inside Pods.

## 1. Liveness Probe

A **liveness probe** checks whether the application is still alive and functioning.

When the liveness probe fails continuously, Kubernetes restarts the container.

### Real-time example

Your Java application process is running, but it is stuck because of:

* Deadlock
* Infinite loop
* Memory issue
* Application hang

The container is running, but the application is not responding. The liveness probe detects this and restarts the container.

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3
```

### Meaning

* Wait 30 seconds after the container starts.
* Check `/health/live` every 10 seconds.
* Wait a maximum of 3 seconds for the response.
* Restart the container after 3 consecutive failures.

---

## 2. Readiness Probe

A **readiness probe** checks whether the application is ready to receive user traffic.

When the readiness probe fails:

* Kubernetes does not restart the container.
* Kubernetes removes the Pod IP from the Service endpoints.
* The Pod stops receiving traffic.
* The probe continues checking.
* Once the probe succeeds, the Pod is added back to the Service.

### Real-time example

The application is running, but:

* Database connection is unavailable.
* Cache connection is unavailable.
* Application initialization is incomplete.
* Required dependency is temporarily down.

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
  successThreshold: 1
```

### Important point

A Pod can show:

```text
STATUS: Running
READY: 0/1
```

This means the container is running, but it is not ready to accept traffic.

---

## 3. Startup Probe

A **startup probe** checks whether the application has completed its startup process.

It is mainly used for slow-starting applications.

Until the startup probe succeeds:

* Liveness checks are disabled.
* Readiness checks are disabled.
* Kubernetes gives the application enough time to start.

If the startup probe keeps failing beyond the configured limit, Kubernetes restarts the container.

### Real-time example

A Java application may require several minutes to:

* Load configuration
* Run database migrations
* Establish database connections
* Load cache
* Start the JVM
* Initialize Spring Boot components

```yaml
startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
  periodSeconds: 10
  failureThreshold: 30
```

The application gets:

```text
30 × 10 seconds = 300 seconds
```

That is **5 minutes** to start.

---

# Complete Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-demo
spec:
  replicas: 3

  selector:
    matchLabels:
      app: probe-demo

  template:
    metadata:
      labels:
        app: probe-demo

    spec:
      containers:
        - name: probe-demo
          image: nginx:latest

          ports:
            - containerPort: 80

          startupProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 5
            failureThreshold: 30

          readinessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
            successThreshold: 1

          livenessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
```

---

# Main Difference

| Probe     | Main Question                        | What happens when it fails?                    |
| --------- | ------------------------------------ | ---------------------------------------------- |
| Startup   | Has the application started?         | Container is restarted after the failure limit |
| Readiness | Can the application receive traffic? | Pod is removed from Service endpoints          |
| Liveness  | Is the application still healthy?    | Container is restarted                         |

---

# Easy Interview Explanation

```text
Startup probe checks whether the application has started.

Readiness probe checks whether the application is ready to receive traffic.

Liveness probe checks whether the application is still alive and healthy.
```

## Simple memory trick

```text
Startup  = Start completed?
Readiness = Ready for traffic?
Liveness  = Alive or dead?
```

---

# Probe Execution Order

When a startup probe is configured:

```text
Container starts
      ↓
Startup probe runs
      ↓
Startup probe succeeds
      ↓
Readiness and liveness probes begin
```

Without a startup probe:

```text
Container starts
      ↓
Readiness and liveness probes begin based on their configuration
```

---

# Probe Types

Kubernetes supports four common probe mechanisms.

## HTTP Probe

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
```

Kubernetes sends an HTTP request. HTTP status codes from `200` to `399` are generally treated as successful.

## TCP Probe

```yaml
livenessProbe:
  tcpSocket:
    port: 3306
```

Kubernetes checks whether it can establish a TCP connection.

Useful for:

* MySQL
* PostgreSQL
* Redis
* TCP-based applications

## Command Probe

```yaml
livenessProbe:
  exec:
    command:
      - sh
      - -c
      - test -f /tmp/app-healthy
```

Kubernetes runs a command inside the container.

* Exit code `0` means success.
* Non-zero exit code means failure.

## gRPC Probe

```yaml
livenessProbe:
  grpc:
    port: 50051
```

Used for applications that expose gRPC health-check endpoints.

---

# Important Probe Parameters

```yaml
initialDelaySeconds: 20
periodSeconds: 10
timeoutSeconds: 3
failureThreshold: 3
successThreshold: 1
```

| Parameter             | Meaning                                        |
| --------------------- | ---------------------------------------------- |
| `initialDelaySeconds` | Delay before the first probe                   |
| `periodSeconds`       | How frequently the probe runs                  |
| `timeoutSeconds`      | Maximum time to wait for a response            |
| `failureThreshold`    | Consecutive failures required to mark failure  |
| `successThreshold`    | Consecutive successes required to mark success |

---

# Common Mistakes

## Using the same dependency check for liveness

Avoid making the liveness endpoint fail just because the database is temporarily unavailable.

Otherwise:

```text
Database unavailable
      ↓
Liveness probe fails
      ↓
Container restarts
      ↓
Database is still unavailable
      ↓
Container restarts repeatedly
```

This can create `CrashLoopBackOff`.

A better design is:

```text
Liveness → Check the application process itself
Readiness → Check whether the application can serve traffic
```

## Starting the liveness probe too early

If a slow application takes 2 minutes to start but liveness begins after 10 seconds, Kubernetes may repeatedly restart it.

Use a startup probe for slow applications.

## Making health endpoints expensive

Health endpoints should be:

* Lightweight
* Fast
* Reliable
* Independent of heavy business logic

---

# Useful Troubleshooting Commands

```bash
kubectl get pods
```

```bash
kubectl describe pod <pod-name>
```

Look for events such as:

```text
Liveness probe failed
Readiness probe failed
Startup probe failed
Back-off restarting failed container
```

Check container logs:

```bash
kubectl logs <pod-name>
```

Check previous container logs after a restart:

```bash
kubectl logs <pod-name> --previous
```

View Service endpoints:

```bash
kubectl get endpoints
```

When readiness fails, the Pod IP may disappear from the Service endpoints.

---

# Best-Practice Example

```yaml
startupProbe:
  httpGet:
    path: /actuator/health/startup
    port: 8080
  periodSeconds: 10
  failureThreshold: 30

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3
```

**One-line summary:**

> Liveness restarts an unhealthy container, readiness controls traffic to the Pod, and startup gives slow applications enough time to initialize.
