# memory-chart

Helm chart for the **memory UI** (`monochrome-memory-ui`), the browser front end for browsing and
managing the assistant's durable memories. The companion `memory-api-chart` deploys the pgvector
API this UI talks to.

Despite the bare `-chart` name, this is not a legacy or secondary chart: it is the chart that
actually serves the memory UI, and it is reconciled continuously by FluxCD.

## What it deploys

A single stateless Deployment serving static assets, plus a ClusterIP Service. No database, no
cache, and no subchart dependencies.

| Object | Name |
|---|---|
| Deployment | `<release>-memory-chart` |
| Service | `<release>-memory-chart` (port 80 to container port 3000) |
| PodDisruptionBudget | `<release>-memory-chart-pdb` (when `pdb.create`) |
| HorizontalPodAutoscaler | `<release>-memory-chart` (when `autoscaling.enabled`) |
| Ingress | `<release>-memory-chart` (when `ingress.enabled`) |

The container name inside the pod is `{{ .Chart.Name }}`, i.e. `memory-chart`, so use
`-c memory-chart` with `kubectl exec`. `serviceAccount.create` is `false` by default, so pods run
under the namespace default ServiceAccount unless you set a name.

## Values

Values live in three files, matching the convention used across bouc.io charts. There is no plain
`values.yaml`.

| File | Purpose |
|---|---|
| `base.values.yaml` | Values that never change between environments |
| `lcl.values.yaml` | Local cluster overrides |
| `snbx.values.yaml` | Sandbox overrides |

> In the cluster, FluxCD supplies values from generated ConfigMaps via `valuesFrom:`, not from these
> files directly. They are the source the ConfigMaps are generated from.

### Image

`image.registry` and `image.repository` are separate values, joined by the `memory-chart.image`
helper, so a relocating operator overrides only the registry half, per release or through
`global.imageRegistry`. The repository path ends in `memory-mirror-ui` rather than
`monochrome-memory-ui`: that mismatch between the local directory name and the published artifact
name is deliberate, so do not "correct" it.

`image.tag` in `base.values.yaml` is maintained by FluxCD: its ImageUpdateAutomation rewrites the
value in git whenever CI publishes a newer image. Override it explicitly for a manual install.

### Front-end configuration

This is a Vite single-page app, so its configuration arrives as `VITE_*` environment variables read
at container start: `VITE_IDENTITY_PROVIDER`, `VITE_SSO_SERVER_URL`, `VITE_OAUTH_REALM`,
`VITE_OAUTH_CLIENT_ID`, `VITE_API_URL`, `VITE_OLLAMA_API_URL` and `VITE_AVAILABLE_MODELS`.

`VITE_AVAILABLE_MODELS` is a list in values but a single environment variable in the pod: the
template JSON-encodes it. Keep it a list and let the template do the encoding.

The `${CLUSTER_DOMAIN}` placeholders in the environment files are substituted by FluxCD at the
Kustomization step, not by Helm.

## Probes

`livenessProbe` and `readinessProbe` render only when set, and the shipped values leave both empty
(`{}`). Pods count as ready as soon as the container starts. Supply probes per environment if you
want real health gating.

## Local usage

```bash
helm lint . -f base.values.yaml -f lcl.values.yaml
helm template test . -f base.values.yaml -f lcl.values.yaml
helm install memory . -f base.values.yaml -f lcl.values.yaml
```

The values files layer: `base` first, then exactly one environment file. Pass them to `helm lint`
too: a bare `helm lint .` fails, because there is no `values.yaml` for it to fall back on.

> The chart must be published to the chart registry by CI before FluxCD can reconcile it. Pushing
> chart source to git is not enough.

## License

[Elastic License 2.0](./LICENSE) — source-available; not OSI open source.
