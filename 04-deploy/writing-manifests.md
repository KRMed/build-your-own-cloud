## Writing Your First App Manifests

**Required** - even if you are using GitOps, you need to write the Kubernetes manifests that describe your app. This section shows what they actually look like.

A typical app needs three resources: a `Deployment` (what to run and how many copies), a `Service` (a stable internal address for the pods), and an `Ingress` (a routing rule so the ingress controller knows to send requests for a given hostname to this service).

### Deployment

A `Deployment` tells Kubernetes what container to run, how many replicas to keep alive, what resources to allocate, and what environment variables or secrets to inject.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: <YOUR_REGISTRY>/my-app:<TAG>
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          # Optional: inject a secret value as an environment variable
          env:
            - name: API_KEY
              valueFrom:
                secretKeyRef:
                  name: my-app-secrets   # Secret from cluster-secrets.md
                  key: API_KEY
      # Optional: pull secret for private registries (see cluster-secrets.md)
      imagePullSecrets:
        - name: registry-credentials
      # Optional: pin to a specific node architecture in a mixed cluster
      # nodeSelector:
      #   kubernetes.io/arch: amd64   # remove this if your image is multi-arch
```

> **Reference:** [Deployment documentation](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

> **Mixed architecture clusters:** If your image is only built for one architecture (e.g. only amd64), uncomment the `nodeSelector` above to prevent the pod from being scheduled onto a node of the wrong architecture. If your image is a proper multi-arch manifest (built with `docker buildx`), you can omit the `nodeSelector` entirely and the scheduler will handle it. You can check your node architectures with `kubectl get nodes -o wide`.

### Service

A `Service` gives your pods a stable internal DNS name and IP address. Without it, pods can only be reached by their individual pod IPs, which change every time a pod restarts.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: my-app
spec:
  selector:
    app: my-app    # must match the labels on your Deployment pods
  ports:
    - port: 80         # port the Service listens on
      targetPort: 3000  # port your container listens on
```

> **Reference:** [Service documentation](https://kubernetes.io/docs/concepts/services-networking/service/)

### Ingress

An `Ingress` resource tells NGINX ingress which hostname to route to which Service. Without this, your app is only reachable from inside the cluster.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: my-app
spec:
  ingressClassName: nginx
  rules:
    - host: app.your-domain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app    # must match the Service name above
                port:
                  number: 80
```

> **Reference:** [Ingress documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)

### The kustomization.yaml for your app

Put the three files above in `apps/my-app/base/` and add a `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

### Recommended security context

Adding a security context to your containers limits what they can do if compromised. This is a strongly recommended baseline for any app you deploy:

```yaml
securityContext:
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
  capabilities:
    drop: [ALL]
```

If your app needs to bind to port 80 (a privileged port below 1024), add back only that capability:

```yaml
capabilities:
  drop: [ALL]
  add: [NET_BIND_SERVICE]
```

If your app needs to write to its filesystem (e.g. a web server writing cache files), you can either mount a writable `emptyDir` volume at the specific path, or remove `readOnlyRootFilesystem: true` and document why.

> **Reference:** [Configure a Security Context for a Pod](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

**Next:** [Deploy Your First App End-to-End](first-app.md)