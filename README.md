<p align="center">
  <a href="https://zentinelproxy.io">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/zentinelproxy/design/main/marks/lockup/zentinel-lockup-dark.svg">
      <img alt="Zentinel" src="https://raw.githubusercontent.com/zentinelproxy/design/main/marks/lockup/zentinel-lockup-light.svg" height="52">
    </picture>
  </a>
</p>

<h3 align="center">Zentinel Helm chart</h3>

<p align="center">A Helm chart that runs the Zentinel proxy on Kubernetes from a KDL configuration you supply.</p>

<p align="center">
  <a href="https://docs.zentinelproxy.io/deployment/kubernetes/">Kubernetes guide</a> &middot;
  <a href="https://docs.zentinelproxy.io">Docs</a> &middot;
  <a href="https://github.com/zentinelproxy">All repositories</a>
</p>

The chart deploys [Zentinel](https://github.com/zentinelproxy/zentinel) as a
Deployment with a Service, a ConfigMap holding `zentinel.kdl`, and optional
Ingress, HorizontalPodAutoscaler, PodDisruptionBudget and Prometheus
ServiceMonitor. It runs `ghcr.io/zentinelproxy/zentinel` at the chart's
`appVersion` (`26.08_2`, chart version `0.1.2`) as a non-root user with a
read-only root filesystem.

> **No chart repository yet.** `https://charts.zentinelproxy.io` does not
> resolve: there is no DNS record, no `gh-pages` branch and no chart release
> automation in this repository, so `helm repo add` fails. Install from a clone
> or a tarball.

## Quick start

Always pass a configuration. The built-in default ConfigMap has no `system`
block, which the proxy requires, so a release installed without `config.raw`
or `config.existingConfigMap` does not start.

Save this as `values.yaml`. It is the smallest configuration that fits the
chart: the container ports are fixed at 80 (`http`), 443 (`https`) and 9090
(`metrics`), and the liveness and readiness probes call `GET /health` on 9090,
so the configuration serves `/health` and `/metrics` from a listener on 9090
and turns off the separate metrics server, which would otherwise bind 9090 and
answer `/health` with 404.

```yaml
config:
  raw: |
    system {
        worker-threads 0
    }

    observability {
        metrics {
            enabled #false
        }
    }

    listeners {
        listener "http" {
            address "0.0.0.0:80"
            protocol "http"
        }
        listener "admin" {
            address "0.0.0.0:9090"
            protocol "http"
            default-route "health"
        }
    }

    routes {
        route "health" {
            priority "high"
            matches {
                path "/health"
                path "/ready"
            }
            service-type "builtin"
            builtin-handler "health"
        }
        route "metrics" {
            priority "high"
            matches { path "/metrics" }
            service-type "builtin"
            builtin-handler "metrics"
        }
        route "api" {
            matches { path-prefix "/api" }
            upstream "backend"
        }
    }

    upstreams {
        upstream "backend" {
            target "my-service:8080"
            health-check {
                type "http" {
                    path "/health"
                }
            }
        }
    }
```

Check it locally with `zentinel test -c <file>`, then install:

```bash
git clone https://github.com/zentinelproxy/zentinel-helm.git
helm install zentinel ./zentinel-helm -f values.yaml
```

## Install

From a clone (above), or straight from a tarball of a branch:

```bash
helm install zentinel \
  https://github.com/zentinelproxy/zentinel-helm/archive/refs/heads/main.tar.gz \
  -f values.yaml
```

To run another proxy release, set `--set image.tag=<tag>`. Published tags
follow Zentinel's CalVer scheme (for example `26.08_2`), plus `latest`.

## Configuration

[`values.yaml`](values.yaml) lists every value. The common ones:

| Option | Default | Meaning |
|---|---|---|
| `replicaCount` | `1` | Replicas (ignored when autoscaling is on) |
| `image.repository` | `ghcr.io/zentinelproxy/zentinel` | Image repository |
| `image.tag` | `""` | Image tag; empty uses the chart `appVersion` |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy |
| `config.raw` | `""` | KDL configuration written to the chart's ConfigMap |
| `config.existingConfigMap` | `""` | Use this ConfigMap instead of creating one |
| `config.configKey` | `zentinel.kdl` | Key in the ConfigMap that holds the configuration |
| `env`, `envFrom` | `[]` | Extra environment, e.g. `RUST_LOG` |
| `service.type` | `ClusterIP` | Service type |
| `service.httpPort` / `httpsPort` / `metricsPort` | `80` / `443` / `9090` | Service ports (container ports are fixed at 80, 443, 9090) |
| `service.loadBalancerIP` | `""` | Used when `service.type` is `LoadBalancer` |
| `resources` | requests `100m`/`128Mi`, limits `1000m`/`512Mi` | Container resources |
| `securityContext` | non-root UID `65534`, read-only root, all capabilities dropped | Container security context |
| `livenessProbe`, `readinessProbe` | `GET /health` on `metrics` | Probes; change them if your config serves health elsewhere |
| `autoscaling.enabled` | `false` | HPA, 2 to 10 replicas at 80% CPU and memory |
| `podDisruptionBudget.enabled` | `false` | PDB with `minAvailable: 1` |
| `serviceMonitor.enabled` | `false` | Prometheus Operator ServiceMonitor scraping the `metrics` port every 30 s |
| `ingress.enabled` | `false` | Ingress for the `http` port |
| `extraVolumes`, `extraVolumeMounts` | `[]` | Extra volumes, e.g. TLS secrets |
| `initContainers`, `sidecars` | `[]` | Extra containers in the pod |
| `nodeSelector`, `tolerations`, `affinity`, `topologySpreadConstraints` | empty | Scheduling |

The proxy's own options are documented in the
[configuration reference](https://docs.zentinelproxy.io/configuration/).

### Existing ConfigMap

```yaml
config:
  existingConfigMap: my-zentinel-config
  configKey: zentinel.kdl
```

### Autoscaling, monitoring, ingress

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

serviceMonitor:
  enabled: true
  interval: 30s

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: proxy.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: proxy-tls
      hosts:
        - proxy.example.com
```

### TLS certificates

Mount a Secret with `extraVolumes` and `extraVolumeMounts`:

```yaml
extraVolumes:
  - name: tls-certs
    secret:
      secretName: zentinel-tls

extraVolumeMounts:
  - name: tls-certs
    mountPath: /etc/zentinel/certs
    readOnly: true
```

and point an HTTPS listener at the files (add it to the `listeners` block
of the quick-start configuration):

```kdl
listener "https" {
    address "0.0.0.0:443"
    protocol "https"
    tls {
        cert-file "/etc/zentinel/certs/tls.crt"
        key-file "/etc/zentinel/certs/tls.key"
    }
}
```

## Development

```bash
helm lint .
helm template zentinel . -f values.yaml     # render locally
```

There is no CI workflow in this repository yet.

## Related

- [Zentinel](https://github.com/zentinelproxy/zentinel): the proxy
- [Kubernetes guide](https://docs.zentinelproxy.io/deployment/kubernetes/) in the docs
- [Rust agent SDK](https://github.com/zentinelproxy/zentinel-agent-rust-sdk): build custom agents

## License

Apache-2.0, see [`LICENSE`](LICENSE).

---

<p align="center">
  <sub>Part of <a href="https://zentinelproxy.io">Zentinel</a>, the security-first reverse proxy. The shisa guards the gate.</sub>
</p>
