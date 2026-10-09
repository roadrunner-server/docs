# Kubernetes + Argo CD

<img alt="k8s-argo" src="https://github.com/user-attachments/assets/10be64f4-85cd-47ed-8c4c-858b9cbd232c" />


You can deploy RoadRunner to Kubernetes via GitOps using Argo CD with the official example repository: [roadrunner-server/k8s-examples](https://github.com/roadrunner-server/k8s-examples). The chart lives in [`deploy/charts/roadrunner`](https://github.com/roadrunner-server/k8s-examples/tree/master/deploy/charts/roadrunner), and the Argo CD example is in [`deploy/argocd`](https://github.com/roadrunner-server/k8s-examples/tree/master/deploy/argocd).

## Prerequisites

- Kubernetes `>= 1.26`
- Argo CD installed
- `kubectl` access to your cluster
- A RoadRunner v3 application image in a registry that the cluster can access. See [Build the Application Image](docker.md#build-the-application-image).

## Use the Official Example

Use these files from the upstream repository:

- Chart: [`deploy/charts/roadrunner`](https://github.com/roadrunner-server/k8s-examples/tree/master/deploy/charts/roadrunner)
- Argo application: [`deploy/argocd/application.yaml`](https://github.com/roadrunner-server/k8s-examples/blob/master/deploy/argocd/application.yaml)
- Values overrides: [`deploy/argocd/values.yaml`](https://github.com/roadrunner-server/k8s-examples/blob/master/deploy/argocd/values.yaml)

## Recommended Values (MetalLB-Friendly)

Set `image.repository` and `image.tag` to your published v3 application image. The probe paths below require v3.

This profile uses MetalLB for external service access:

{% code title="values.yaml" %}

```yaml
image:
  repository: registry.example.com/your-team/rr-app
  tag: "3.0.0"

service:
  type: LoadBalancer

gateway:
  enabled: false

ingress:
  enabled: false
```

{% endcode %}

## Configure Probes

Set the chart probe paths to the RoadRunner v3 status endpoints:

{% code title="values.yaml" %}

```yaml
probes:
  liveness:
    path: /livez?plugin=http
    port: status
  readiness:
    path: /readyz?plugin=http
    port: status
```

{% endcode %}

The chart binds the status server to port `2114` in the pod. Liveness accepts active HTTP workers, including busy workers. Readiness requires at least one idle HTTP worker. During graceful shutdown, liveness remains successful and readiness fails. See [Health and Readiness checks](../lab/health.md).

## Apply with Argo CD

Commit your values to the repository used by Argo CD. Set `repoURL` and `targetRevision` in `application.yaml` to that repository and revision. Then apply the Argo CD `Application` manifest:

{% code %}

```bash
kubectl apply -f deploy/argocd/application.yaml
```

{% endcode %}

## Verify Sync and Health

{% code %}

```bash
kubectl -n roadrunner get svc roadrunner -w
curl -sS http://<external-ip>/
```

{% endcode %}

To check the status server, forward its pod port:

```bash
kubectl -n roadrunner port-forward deployment/roadrunner 2114:2114
```

In another terminal, request the probes:

```bash
curl -sS 'http://127.0.0.1:2114/livez?plugin=http'
curl -sS 'http://127.0.0.1:2114/readyz?plugin=http'
```

## Troubleshooting

- If Argo CD shows `Progressing` and `Waiting for controller`, Gateway resources are enabled but no Gateway controller/GatewayClass is reconciling them.
- If your cluster has no Gateway controller, keep `gateway.enabled=false` and `ingress.enabled=false`, and use `service.type=LoadBalancer`.
- If no external IP appears, verify MetalLB IP pool allocation and inspect service events.

## Next Steps

- Argo CD deployment guide: [deploy/argocd/README.md](https://github.com/roadrunner-server/k8s-examples/blob/master/deploy/argocd/README.md)
- Health checks and probes in RoadRunner docs: [HealthChecks](../lab/health.md)
