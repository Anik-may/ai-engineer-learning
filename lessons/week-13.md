# Week 13: Kubernetes Basics

General knowledge. You need to read and discuss Kubernetes, not operate a
cluster. 22% of the Bangladesh postings mention it.

## Goal

Deploy your API to a local cluster and explain every object you used.

## Why it exists

Kubernetes keeps a declared number of containers running across machines,
replaces failed ones, rolls out new versions gradually and scales on load.
You describe the desired state in YAML; the cluster makes it true.

## Objects to know

| Object | Purpose |
| --- | --- |
| Pod | One or more containers that run together; the smallest unit |
| Deployment | Keeps N identical pods running; handles rolling updates |
| Service | A stable name and address for a set of pods |
| Ingress | HTTP routing from outside the cluster to services |
| ConfigMap | Non-secret configuration |
| Secret | Sensitive configuration |
| Namespace | A named partition of the cluster |
| HorizontalPodAutoscaler | Changes replica count based on load |
| Job / CronJob | Run to completion, once or on a schedule |
| PersistentVolumeClaim | Storage that outlives a pod |

## A minimal deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rag-api
spec:
  replicas: 2
  selector:
    matchLabels: { app: rag-api }
  template:
    metadata:
      labels: { app: rag-api }
    spec:
      containers:
        - name: api
          image: rag-api:1.0.0
          ports: [{ containerPort: 8000 }]
          envFrom:
            - configMapRef: { name: rag-config }
            - secretRef: { name: rag-secrets }
          resources:
            requests: { cpu: "250m", memory: "256Mi" }
            limits:   { cpu: "1",    memory: "512Mi" }
          readinessProbe:
            httpGet: { path: /ready, port: 8000 }
          livenessProbe:
            httpGet: { path: /health, port: 8000 }
---
apiVersion: v1
kind: Service
metadata:
  name: rag-api
spec:
  selector: { app: rag-api }
  ports: [{ port: 80, targetPort: 8000 }]
```

## Ideas that interviewers ask about

- **Readiness probe**: is the pod ready for traffic? If not, it is removed
  from the service but not restarted.
- **Liveness probe**: is the pod alive? If not, it is restarted.
- **Requests and limits**: requests are what the scheduler reserves; limits
  are the cap. Exceeding the memory limit kills the container.
- **Rolling update**: new pods start and pass readiness before old ones stop.
- **Rollback**: `kubectl rollout undo deployment/rag-api`.

## Commands

```bash
kubectl get pods
kubectl describe pod <name>
kubectl logs <name> -f
kubectl apply -f k8s/
kubectl rollout status deployment/rag-api
kubectl rollout undo deployment/rag-api
kubectl port-forward svc/rag-api 8080:80
kubectl exec -it <name> -- sh
```

## AI workload notes

- Pods that only call model APIs are ordinary web pods: scale on CPU or
  request rate.
- Pods that **serve models** need GPU nodes, a GPU resource request, slow
  startup handling (model load) and scaling on queue length or latency, not
  CPU.
- Long streaming connections need generous timeouts on the ingress.
- Use a graceful shutdown period long enough to finish streams.

## Kubernetes or ECS?

| | ECS Fargate | Kubernetes |
| --- | --- | --- |
| Learning curve | Low | High |
| Portability | AWS only | Any cloud |
| Ecosystem | Smaller | Very large |
| Good for | Small teams on AWS | Larger platforms, multi-cloud, GPU serving |

## Do

1. Install `kind` or `minikube` and `kubectl`.
2. Load your image and apply the Deployment and Service above.
3. Port-forward and call the API.
4. Kill a pod; watch it come back.
5. Change the image tag; watch the rolling update; then roll back.
6. Break the readiness probe; observe that traffic stops but the pod stays.
7. Put the manifests in a `k8s/` folder in the repository.

## Checkpoint

- [ ] You can explain Pod, Deployment, Service and Ingress without notes.
- [ ] You performed a rolling update and a rollback.
- [ ] Manifests are in the repository.

## Interview questions

1. What is the difference between a liveness and a readiness probe?
2. What happens when a container exceeds its memory limit?
3. How does a rolling update avoid downtime?
4. How would scaling a model-serving pod differ from scaling an API pod?
