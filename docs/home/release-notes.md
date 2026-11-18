# Release notes

## Version 1.66

!!! tip ""
    Helm chart version 1.66.0

### nginx removed from the Helm chart

The Helm chart no longer deploys nginx in front of ODM. Its Ingress, named after the release, routes each path
straight to the Service that serves it: `/ui`, `/swagger`, `/user-docs`, `/mail` and `/mcp` to their Services, and
`/`, `/frontend` and `/api` to `applications`. The upgrade replaces the Ingress `<release>-nginx` with it.

`/frontend` holds the REST API of the UI and the pages of the UI before `/ui`. `/api` holds the public API, SCIM at
`/api/v1/scim` among it. `/` is an exact path, which `applications` redirects to `/ui`. nginx sent every path it did
not route elsewhere to `applications`. The Ingress routes no other path: the Ingress controller answers `404` to it.

Move your `nginx.ingress` values to `ingress`. Its `hosts` is a list of host names, without paths:

```yaml
ingress:
  className: alb
  annotations: {}
  hosts:
    - "{{ .Values.global.hostname }}"
```

The security headers, which nginx added, are now the Ingress controller's to do, see
[Kubernetes requirements](requirements/kubernetes.md). The chart's `examples/ingress-alb.yaml`, for the AWS Load
Balancer Controller, and `examples/ingress-nginx.yaml`, for F5's NGINX Ingress Controller 5.5 or later, set them up.
The nginx example streams uploads with a snippet, so the controller must run with `-enable-snippets`. That flag lets
any Ingress on the controller, from any namespace, add raw nginx configuration. To stream uploads without it, remove
`nginx.org/location-snippets` from the example and set `location-snippets: proxy_request_buffering off;` in the
controller's ConfigMap: the controller then streams uploads for every Ingress it serves. The example does not support
the community ingress-nginx controller, which reached its end of life in March 2026. If your controller in front of
1.65 was ingress-nginx, it ignores the example's `nginx.org/*` annotations and gives ODM its defaults, a 1 MB limit on
request bodies, 60 s timeouts and no security headers: set their equivalents on that controller, or move to F5's
NGINX Ingress Controller.

The `301` redirects to `/ui`, from `/` and from the pages of the UI before `/ui`, now come from `applications`: the
Ingress sends these paths to it.

`applications` redirects the paths nginx did, with the query string kept:

| Path | Redirect to |
| --- | --- |
| `/` | `/ui` |
| `/frontend/endpoint/application/run/genestack/web` and `.../web/<page>` | `/ui` and `/ui/<page>` |
| `/frontend/endpoint/application/run/genestack/signin` and `.../signin/<page>` | `/ui/signin` and `/ui/signin/<page>` |
| `/frontend/endpoint/application/run/genestack/staticpages/privacy-notice.html` | `/ui/privacy-notice` |

Links in emails sent before the upgrade, such as `.../genestack/signin/reset-password?token=...`, keep working. The
emails ODM sends for password recovery, email confirmation and API tokens now link to the `/ui` pages directly.

`nginx.ingressScim` is gone: a separate host for SCIM is now an Ingress in `extraObjects`, see
[SCIM with Azure AD](single-sign-on/scim/azure.md). `nginx.developmentSettings` is gone. To serve the metrics through
the Ingress, as it did, use `ingress.extraPaths`, see [Health and metrics](#health-and-metrics).
The `Access-Control-Allow-Origin: *` header that `nginx.developmentSettings` also added is not needed: `applications`
answers CORS itself. It returns each `Origin` that matches `genestack.connections.cORSPatterns`, which is `".*"`, every
origin, by default.

The chart does not render values that still contain an `nginx` section, and its error message points to `ingress`.
Values of your own that call `<release>-nginx:8080`, in `envVars` for example, must call
`<release>-applications:8080` instead.

On AWS:

- With an ALB the AWS Load Balancer Controller creates, each Service becomes a target group of its own. The chart
  sets no annotations on the Services. Set `alb.ingress.kubernetes.io/healthcheck-path` in
  `<component>.service.annotations` of `applications`, `ui`, `swagger`, `userDocs`, `mailpit` and `mcp`, as the
  chart's `examples/ingress-alb.yaml` does. Without it the target group checks `/` on the traffic port. For
  `applications` the example sets `/frontend/endpoint/isAlive`, on the traffic port, 8080. The old example set the
  path in `nginx.ingress.annotations`: leave it out of `ingress.annotations`, where it applies to every Service that
  sets none. An Ingress without `alb.ingress.kubernetes.io/group.name` has an ALB of its own, so the controller
  replaces the ALB of `<release>-nginx` with a new one, under a new DNS name: update the DNS record, unless External
  DNS does it.
- With an ALB you created yourself, replace the target group and the TargetGroupBinding of `<release>-nginx` with
  one per Service, and add the listener rules and attributes, see [AWS](clouds/aws.md#recommended-steps-for-eks-configuration).
  The target group of `applications` checks `/frontend/endpoint/isAlive` on its traffic port.

### core at the root of its port

`core` drops its context path `/backend` and its servlet path `/endpoint`: its API moves from `/backend/endpoint/...`
to `/...` on port 8080. Only `applications` calls `core`. Values of your own that override `frontend.backend.url` in
the `application.yaml` of `applications` keep `http://<release>-core:8080`: `applications` no longer appends
`/backend`. Other calls of your own to `core` drop `/backend/endpoint`, so
`POST <release>-core:8080/backend/endpoint/clickhouse-flags?read-only=true` becomes
`POST <release>-core:8080/clickhouse-flags?read-only=true`.

While Kubernetes replaces the `applications` and `core` pods during the upgrade, a call between an old pod and a new
one fails, for the minutes of the rollout.

### Health and metrics

`applications` and `core` serve the actuator, its health checks and Prometheus metrics, under `actuatorPath` on port
8080. The chart's `applications.actuatorPath` and `core.actuatorPath`, `/actuator` by default, set Spring Boot's
`management.endpoints.web.base-path`. Before 1.66 the actuator of `applications` was at `/frontend/endpoint/actuator`,
and that of `core` at `/backend/endpoint/actuator`. `/frontend/endpoint/proxy`, which returned the metrics of `core`
through `applications`, is gone.

The chart's probes and ServiceMonitors follow `actuatorPath`. `applications` checks the health of `core` under
`frontend.backend.actuatorPath`, which the chart sets from `core.actuatorPath`. The chart's fluent-bit inputs call
`/actuator/prometheus`: with another `actuatorPath`, change their `metrics_path` in `fluent-bit.config.inputs` too.

Move scrape configs and checks of your own:

| Before | Now |
| --- | --- |
| `/frontend/endpoint/actuator/prometheus` on `applications` | `/actuator/prometheus` on port 8080 |
| `/backend/endpoint/actuator/prometheus` on `core` | `/actuator/prometheus` on port 8080 |
| `/frontend/endpoint/actuator/health` on `applications`, from inside the cluster | `/actuator/health` on port 8080 |
| `/backend/endpoint/actuator/health` on `core` | `/actuator/health` on port 8080 |
| `/frontend/endpoint/actuator/health`, from outside the cluster | `/frontend/endpoint/isAlive`, through the Ingress |

`/frontend/endpoint/isAlive` answers `200` with the body `true`, to any method, without sign-in. `applications`
answered it before 1.66 too.

The Ingress routes neither actuator. Whatever reaches port 8080 of an `applications` or `core` pod reaches its
actuator, `/actuator/prometheus` and the full health details among it. `ingress.extraPaths` adds paths to the Ingress,
on every host. Each entry has `path`, `pathType` (`Prefix` by default), `component`, for the Service
`<release>-<component>`, and `port`, the name of its port (`http` by default). The Ingress passes the path on
unchanged, and both actuators are at `/actuator` by default, so each needs an `actuatorPath` of its own. To read the
metrics through the Ingress, as `nginx.developmentSettings` did:

```yaml
applications:
  actuatorPath: /applications/actuator
core:
  actuatorPath: /core/actuator
ingress:
  extraPaths:
    - path: /applications/actuator
      component: applications
    - path: /core/actuator
      component: core
```

These paths serve the health and the metrics to anyone who reaches the Ingress. `core.service.annotations` is new, as
the other components have: with an ALB, it sets the health check of the target group that an `ingress.extraPaths`
entry gives `core`.

With `networkPolicy.enabled`, the NetworkPolicy `<release>-ingress-routes` opens port 8080 of `applications`, its
actuator included, to any address: it has no `from`. With an ALB of target type `ip`, any host in the VPC that reaches
the pod's IP address reaches the actuator. Port 8080 of `core` stays closed to the Ingress controller, so an
`ingress.extraPaths` entry to `core` needs a NetworkPolicy of your own.

### odm-sdk upgrade

odm-sdk 1.66 finds the server with `/frontend/endpoint/isAlive`, in place of `/frontend/endpoint/actuator/health`,
which `applications` no longer serves. An older odm-sdk fails to connect with `Could not connect to host '<host>'`:
upgrade odm-sdk. An older odm-sdk skips that check when you give the host as `https://<host>/frontend`.

## Version 1.65

!!! tip ""
    Helm chart version 1.65.0

## Version 1.64

!!! danger

    This version must be installed before proceeding with the next update.

!!! tip ""
    Helm chart version 1.64.0

### Genestack audit logs migration from MySQL to Clickhouse

This release migrates MySQL `genestack_audit` schema to ClickHouse `audit.Event` table and
removes `genestack_audit` schema from MySQL upon successful completion. Migration component responsible for it
is planned for a removal in the next release. As a result, this ODM version cannot be skipped during upgrades.

### Excluding audit events from tracking

It is now possible to exclude specific audit events from tracking if they are not needed, which saves disk space.
For example, you can exclude events related to data reads.

A configuration example is available in the `examples` directory of the Helm chart.

### Audit log retention period

!!! danger

    The retention period cannot be changed after installation. Set it before you install or upgrade to this version.

ODM keeps audit events in ClickHouse for a limited number of months and deletes older events automatically.
The default retention period is 12 months. To use a different one, set it in the Helm chart values:

```yaml
core:
  files:
    "/var/lib/genestack/properties/application.yaml":
      backend:
        clickhouse:
          audit:
            retentionMonths: 24
```
