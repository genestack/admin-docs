# Release notes

## Version 1.66

!!! tip ""
    Helm chart version 1.66.0

### nginx removed from the Helm chart

The Helm chart no longer deploys nginx in front of ODM. Its Ingress, named after the release, routes each path
straight to the Service that serves it: `/ui`, `/swagger`, `/user-docs`, `/mail` and `/mcp` to their Services, and
every other path to `applications`. The upgrade replaces the Ingress `<release>-nginx` with it.

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
the Ingress, as it did, use `ingress.extraPaths`, see [Health and metrics on port 8081](#health-and-metrics-on-port-8081).
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

### Health and metrics on port 8081

`applications` and `core` serve the actuator, the health checks and the Prometheus metrics, on a management port,
8081, at `/actuator/*`. Their port 8080 no longer serves `/frontend/endpoint/actuator/*` or
`/backend/endpoint/actuator/*`. `/frontend/endpoint/proxy`, which returned the metrics of `core` through
`applications`, is gone.

The chart sets the port in `applications.service.management.port` and `core.service.management.port`, and a path
before `/actuator` in `.basePath` of each, empty by default. Both Services get a port named `management`. The chart's
probes and ServiceMonitors call `<basePath>/actuator/...` on it. Its fluent-bit scrape inputs call
`/actuator/prometheus` on port 8081: with a `basePath`, add it to their `metrics_path` in `fluent-bit.config.inputs`.

Move scrape configs of your own to the management port:

| Service | Before | After |
| --- | --- | --- |
| `<release>-applications` | `/frontend/endpoint/actuator/prometheus` on port 8080 | `/actuator/prometheus` on port 8081 |
| `<release>-core` | `/backend/endpoint/actuator/prometheus` on port 8080 | `/actuator/prometheus` on port 8081 |

With a `basePath`, the path is `<basePath>/actuator/prometheus`. Health checks of your own inside the cluster move the
same way, to `/actuator/health` on port 8081. Uptime checks from outside the cluster that called
`/frontend/endpoint/actuator/health` move to `/frontend/endpoint/isAlive`, which answers `200` with the body `true`,
without sign-in.

`frontend.backend.managementUrl` in the `application.yaml` of `applications` is the management URL of `core`, where
`applications` checks its health: the chart sets it to `http://<release>-core:8081` with the `basePath` of `core`.
Values of your own that override `frontend.backend.url` must set `frontend.backend.managementUrl` too.
`core.service.annotations` is new: annotations for the `core` Service, as the other components have.

The Ingress routes to no management port unless `ingress.extraPaths` names one. Each entry adds a path on every host,
ahead of the chart's `/` path, with `path`, `pathType` (`Prefix` by default), `component`, for the Service
`<release>-<component>`, and `port`, the name of a port of that Service. The Ingress passes the path on unchanged, so
a `basePath` gives the actuators of `applications` and `core` paths of their own. To read the metrics through the
Ingress, as `nginx.developmentSettings` did:

```yaml
applications:
  service:
    management:
      basePath: /applications
core:
  service:
    management:
      basePath: /core
ingress:
  extraPaths:
    - path: /applications/actuator
      component: applications
      port: management
    - path: /core/actuator
      component: core
      port: management
```

These paths serve the health and the metrics of both services to anyone who reaches the Ingress. With
`networkPolicy.enabled`, the chart's network policies keep the management ports closed to the Ingress controller, so
these paths also need a NetworkPolicy of your own. Pods in the `monitoring` namespace still reach every port.

### odm-sdk upgrade

odm-sdk 1.66 finds the server with `/frontend/endpoint/isAlive`, in place of `/frontend/endpoint/actuator/health`.
`applications` answered `/frontend/endpoint/isAlive` before 1.66 too: `200` with the body `true`, to any method,
without sign-in. An older odm-sdk fails to connect with `Could not connect to host '<host>'`: upgrade odm-sdk. An
older odm-sdk skips that check when you give the host as `https://<host>/frontend`.

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
