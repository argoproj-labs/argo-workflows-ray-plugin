# argo workflows ray plugin


A plugin lets Argo Workflows orchestrate Ray jobs.


## Why argo-workflows-ray-plugin

* Submit tasks using non-string methods. More flexibly control and observe the status of ray jobs.

* Save costs. In scenarios where a large number of Ray jobs are orchestrated, there is no need to generate an equal number of resource pods.

## Getting Started

1. Enable Plugin capability for controller
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workflow-controller
spec:
  template:
    spec:
      containers:
        - name: workflow-controller
          env:
            - name: ARGO_EXECUTOR_PLUGINS
              value: "true"
```
2. Build argo-ray-plugin image

```
git clone https://github.com/argoproj-labs/argo-workflows-ray-plugin.git
cd argo-workflows-ray-plugin
docker build -t argo-ray-plugin:v1 .
```
3. Deploy argo-ray-plugin
```
kubectl apply -f ray-executor-plugin-configmap.yaml
```

4. Permission to create RayJob CRD
```
kubctl apply -f install/role-secret.yaml
```

4. Submit Ray jobs
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: ray-distributed-demo-
  namespace: argo
spec:
  entrypoint: ray-demo
  templates:
    - name: ray-demo
      plugin:
        ray:
          # RayJob definition (Ray Operator must be installed in advance)
          apiVersion: ray.io/v1
          kind: RayJob
          metadata:
            name: ray-example
            namespace: argo
          spec:
            entrypoint: python /app/ray_script.py
            runtimeEnv: |
              pip:
                - requests==2.26.0
            clusterSpec:
              headGroupSpec:
                rayStartParams:
                  dashboard-host: '0.0.0.0'
                template:
                  spec:
                    containers:
                      - name: ray-head
                        image: rayproject/ray:2.9.3
                        ports:
                          - containerPort: 6379
                          - containerPort: 8265  # Dashboard
                        resources:
                          limits:
                            cpu: 2
                            memory: 4Gi
                        volumeMounts:
                          - name: code
                            mountPath: /app
              workerGroupSpecs:
                - replicas: 2
                  minReplicas: 1
                  maxReplicas: 3
                  groupName: worker
                  rayStartParams: {}
                  template:
                    spec:
                      containers:
                        - name: ray-worker
                          image: rayproject/ray:2.9.3
                          resources:
                            limits:
                              cpu: 4
                              memory: 8Gi
                          volumeMounts:
                            - name: code
                              mountPath: /app
            shutdownAfterJobFinishes: true

      volumes:
        - name: code
          persistentVolumeClaim:
            claimName: ray-code-pvc
```