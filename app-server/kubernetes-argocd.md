# Kubernetes + Argo CD

<img alt="k8s-argo" src="https://github.com/user-attachments/assets/10be64f4-85cd-47ed-8c4c-858b9cbd232c" />


You can deploy RoadRunner to Kubernetes via GitOps using Argo CD with the official example repository: [roadrunner-server/k8s-examples](https://github.com/roadrunner-server/k8s-examples). The chart lives in [`deploy/charts/roadrunner`](https://github.com/roadrunner-server/k8s-examples/tree/master/deploy/charts/roadrunner), and the Argo CD example is in [`deploy/argocd`](https://github.com/roadrunner-server/k8s-examples/tree/master/deploy/argocd).

## Prerequisites

- Kubernetes `>= 1.26`
- Argo CD installed
- `kubectl` access to your cluster
- A runnable RoadRunner container image (or the example image from `k8s-examples`)

## Use the Official Example

Use these files from the upstream repository:

- Chart: [`deploy/charts/roadrunner`](https://github.com/roadrunner-server/k8s-examples/tree/master/deploy/charts/roadrunner)
- Argo application: [`deploy/argocd/application.yaml`](https://github.com/roadrunner-server/k8s-examples/blob/master/deploy/argocd/application.yaml)
- Values overrides: [`deploy/argocd/values.yaml`](https://github.com/roadrunner-server/k8s-examples/blob/master/deploy/argocd/values.yaml)

## Recommended Values (MetalLB-Friendly)

This profile avoids dependency on a Gateway controller and works well on clusters using MetalLB for external service exposure:

{% code title="values.yaml" %}

```yaml
service:
  type: LoadBalancer

gateway:
  enabled: false

ingress:
  enabled: false
```

{% endcode %}

## Apply with Argo CD

Apply the Argo CD `Application` manifest:

```bash
kubectl apply -f deploy/argocd/application.yaml
```

If needed, customize `repoURL` and `targetRevision` in `application.yaml` before applying.

## Verify Sync and Health

![Argo CD application in Synced and Healthy state](images/argocd-roadrunner-synced-healthy.png)

```bash
kubectl -n roadrunner get svc roadrunner -w
curl -sS http://<external-ip>/
curl -sS http://<external-ip>/health
```

## Troubleshooting

- If Argo CD shows `Progressing` and `Waiting for controller`, Gateway resources are enabled but no Gateway controller/GatewayClass is reconciling them.
- If your cluster has no Gateway controller, keep `gateway.enabled=false` and `ingress.enabled=false`, and use `service.type=LoadBalancer`.
- If no external IP appears, verify MetalLB IP pool allocation and inspect service events.

## Next Steps

- Argo CD deployment guide: [deploy/argocd/README.md](https://github.com/roadrunner-server/k8s-examples/blob/master/deploy/argocd/README.md)
- Health checks and probes in RoadRunner docs: [HealthChecks](../lab/health.md)
