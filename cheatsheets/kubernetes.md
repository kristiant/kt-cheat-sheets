# Kubernetes on AWS (EKS) + Helm

**What it is:** A container orchestrator that schedules, scales, and heals containerized apps across a cluster. On AWS it's run as **EKS** (managed control plane); **Helm** packages your manifests into versioned, parameterized releases.

**Why people use it:** Declarative deploys, self-healing, horizontal autoscaling, and zero-downtime rollouts — you describe desired state, the cluster reconciles to it.

**Typically used for:** Running stateless TS/Node services (an LLM-agent API, a tool/MCP server, a worker pulling jobs off a queue) that need to scale on load, hold secrets safely, and roll out without downtime.

> `kubectl` talks to the cluster API; `helm` templates manifests. AWS specifics here assume EKS + IRSA (IAM Roles for Service Accounts).

---

## Mental model

```
Pod         smallest unit — one or more containers sharing a network/volume
Deployment  manages a ReplicaSet of identical Pods; handles rollouts
Service     stable virtual IP/DNS in front of Pods (ClusterIP / LoadBalancer)
Ingress     HTTP(S) routing into Services (ALB on EKS)
ConfigMap   non-secret config (env, files)
Secret      base64 config — pair with AWS Secrets Manager for real secrets
Namespace   logical partition (e.g. agents-staging, agents-prod)
HPA         HorizontalPodAutoscaler — scales replicas on CPU / custom metrics
```

Everything is a declarative object you `apply`; the controller reconciles actual → desired.

---

## Connect to an EKS cluster

```bash
# write kubeconfig for an EKS cluster
aws eks update-kubeconfig --name my-cluster --region us-east-1

kubectl config current-context        # which cluster am I pointed at?
kubectl config use-context <ctx>      # switch cluster
kubectl get nodes                     # confirm connectivity
```

---

## Most common commands

```bash
kubectl get pods -n agents            # list pods in a namespace
kubectl get pods -A                   # all namespaces
kubectl get deploy,svc,ingress -n agents

kubectl describe pod <pod> -n agents  # events + why it's not starting
kubectl logs <pod> -n agents          # logs
kubectl logs -f deploy/agent-api -n agents          # follow a deployment's logs
kubectl logs <pod> -n agents --previous             # logs from the crashed prior container

kubectl exec -it <pod> -n agents -- sh              # shell into a container
kubectl port-forward svc/agent-api 8080:80 -n agents  # test a service locally

kubectl apply -f manifest.yaml        # create/update from file
kubectl rollout status deploy/agent-api -n agents   # watch a rollout
kubectl rollout restart deploy/agent-api -n agents  # restart (re-pull secrets/config)
kubectl rollout undo deploy/agent-api -n agents     # roll back to previous revision

kubectl get events -n agents --sort-by=.lastTimestamp | tail -20  # recent cluster events
kubectl top pods -n agents            # live CPU/memory (needs metrics-server)
```

---

## Deploying a TS/Node agent service

A typical stateless agent API: a Deployment + Service + HPA. Note the **probes** (Node apps must expose `/healthz`/`/readyz`) and **resource requests** (the scheduler and HPA need them).

```yaml
# agent-api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agent-api
  namespace: agents
spec:
  replicas: 2
  selector:
    matchLabels: { app: agent-api }
  template:
    metadata:
      labels: { app: agent-api }
    spec:
      serviceAccountName: agent-api      # bound to an IAM role via IRSA (below)
      containers:
        - name: agent-api
          image: 1234.dkr.ecr.us-east-1.amazonaws.com/agent-api:1.4.0
          ports: [{ containerPort: 3000 }]
          env:
            - name: NODE_ENV
              value: production
            - name: ANTHROPIC_API_KEY
              valueFrom:
                secretKeyRef: { name: agent-secrets, key: anthropic-api-key }
          resources:
            requests: { cpu: 250m, memory: 512Mi }   # guaranteed (HPA baseline)
            limits:   { cpu: "1",  memory: 1Gi }      # ceiling (OOMKilled if exceeded)
          readinessProbe:                              # gate traffic until ready
            httpGet: { path: /readyz, port: 3000 }
            initialDelaySeconds: 5
          livenessProbe:                               # restart if wedged
            httpGet: { path: /healthz, port: 3000 }
            initialDelaySeconds: 15
---
apiVersion: v1
kind: Service
metadata: { name: agent-api, namespace: agents }
spec:
  selector: { app: agent-api }
  ports: [{ port: 80, targetPort: 3000 }]
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: agent-api, namespace: agents }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: agent-api }
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

```bash
kubectl apply -f agent-api.yaml
kubectl rollout status deploy/agent-api -n agents
```

> **LLM workloads scale on concurrency, not CPU.** Agent requests are I/O-bound (waiting on the model), so CPU stays low while pods saturate. Scale on a custom metric (in-flight requests / queue depth via KEDA or a Prometheus adapter) rather than CPU alone.

---

## AWS-specific patterns

### IRSA — give a pod an IAM role (no static keys)

Pods assume an IAM role via a ServiceAccount annotation — the AWS SDK in your Node app picks up temporary creds automatically. No access keys in env vars.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: agent-api
  namespace: agents
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::1234:role/agent-api-irsa
```

```ts
// In the Node app — SDK resolves IRSA creds with zero config
import { BedrockRuntimeClient } from "@aws-sdk/client-bedrock-runtime";
const client = new BedrockRuntimeClient({ region: "us-east-1" }); // creds from IRSA
```

### Secrets from AWS Secrets Manager

Don't hand-maintain k8s Secrets. Sync them with the **External Secrets Operator** so rotation in AWS propagates into the cluster.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: agent-secrets, namespace: agents }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: aws-secrets, kind: ClusterSecretStore }
  target: { name: agent-secrets }          # creates the k8s Secret of this name
  data:
    - secretKey: anthropic-api-key
      remoteRef: { key: prod/agent/anthropic, property: apiKey }
```

### Ingress via ALB

The AWS Load Balancer Controller turns an Ingress into a real ALB.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: agent-api
  namespace: agents
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:1234:certificate/abc
spec:
  rules:
    - host: agents.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: agent-api, port: { number: 80 } } }
```

---

## Helm

Helm packages the manifests above into a **chart** — one templated, versioned, parameterized unit you install/upgrade as a release.

```
agent-api/
  Chart.yaml          name + version
  values.yaml         default config (image tag, replicas, env)
  templates/
    deployment.yaml   manifests with {{ .Values.x }} placeholders
    service.yaml
    ingress.yaml
    _helpers.tpl       reusable template snippets
```

### Common commands

```bash
helm install agent-api ./agent-api -n agents                 # first install
helm upgrade --install agent-api ./agent-api -n agents \      # idempotent deploy
  --set image.tag=1.4.0 -f values.prod.yaml
helm upgrade agent-api ./agent-api -n agents --atomic --wait  # auto-rollback on failure

helm list -n agents                  # releases + revisions
helm history agent-api -n agents     # revision history
helm rollback agent-api 3 -n agents  # roll back to revision 3
helm template ./agent-api -f values.prod.yaml   # render manifests locally (no apply)
helm diff upgrade agent-api ./agent-api -f values.prod.yaml   # preview (helm-diff plugin)
helm uninstall agent-api -n agents   # remove the release
```

### Templating

```yaml
# templates/deployment.yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources: {{- toYaml .Values.resources | nindent 12 }}
```

```yaml
# values.yaml — overridden per environment with -f values.prod.yaml or --set
replicaCount: 2
image: { repository: 1234.dkr.ecr.us-east-1.amazonaws.com/agent-api, tag: latest }
resources:
  requests: { cpu: 250m, memory: 512Mi }
  limits:   { cpu: "1", memory: 1Gi }
```

> **Deploy a new image in CI:** `helm upgrade --install agent-api ./agent-api -n agents --set image.tag=$GIT_SHA --atomic --wait`. `--atomic` rolls back automatically if pods don't become healthy — safe for unattended pipelines.

---

## Practical recipes

Find why a pod won't start (pending / crashlooping):
```bash
kubectl describe pod <pod> -n agents | sed -n '/Events/,$p'
kubectl logs <pod> -n agents --previous
```

Pods `OOMKilled` — check the limit vs actual:
```bash
kubectl top pod <pod> -n agents
kubectl get pod <pod> -n agents -o jsonpath='{.status.containerStatuses[0].lastState}'
```

Set the image without editing files (quick hotfix):
```bash
kubectl set image deploy/agent-api agent-api=...ecr.../agent-api:1.4.1 -n agents
```

Scale manually (e.g. pre-warm before a load test):
```bash
kubectl scale deploy/agent-api --replicas=10 -n agents
```

Tail logs across all pods of a deployment:
```bash
kubectl logs -f -l app=agent-api -n agents --max-log-requests=20
```

Drain a node before maintenance:
```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

**Destructive — deletes running workloads:**
```bash
kubectl delete deploy agent-api -n agents          # remove a deployment + its pods
kubectl delete namespace agents                    # ⚠️ removes EVERYTHING in the namespace
helm uninstall agent-api -n agents                 # remove a Helm release
kubectl delete pods --all -n agents                # ⚠️ kills all pods (they reschedule if owned)
```

Force-delete a stuck pod:
```bash
kubectl delete pod <pod> -n agents --grace-period=0 --force
```

---

## Tips

- **Always set resource `requests`** — without them the scheduler bin-packs blindly and the HPA can't compute utilization.
- **`readinessProbe` ≠ `livenessProbe`** — readiness gates traffic (startup, warm-up); liveness restarts a wedged container. A slow agent boot needs a generous readiness `initialDelaySeconds`.
- **Use `--atomic --wait`** in CI so a bad deploy auto-rolls-back instead of leaving half-updated pods.
- **One namespace per environment**, not per app — simpler RBAC and quota boundaries.
- **Never bake secrets into images or `values.yaml`** — use IRSA for AWS APIs and External Secrets for the rest.
- **Pin image tags** (the git SHA), never `latest` in prod — `latest` makes rollbacks meaningless.

> Related: [docker.md](docker.md), [aws-secrets-manager.md](aws-secrets-manager.md), [aws-bedrock.md](aws-bedrock.md), [terraform.md](terraform.md) (provision the EKS cluster itself).
