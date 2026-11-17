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
[Kubernetes requirements](requirements/kubernetes.md). The chart's `examples/ingress-alb.yaml` and
`examples/ingress-nginx.yaml` set them up. The `301` redirects to `/ui`, from `/` and from the pages of the UI before
`/ui`, now come from `applications`: the Ingress sends these paths to it. The `404` for
`/frontend/endpoint/actuator/prometheus` and `/frontend/endpoint/proxy`, which nginx also returned, now comes from
`applications` too: it answers `404` to a request for these paths that carries `X-Forwarded-Proto` or
`X-Forwarded-For`, which the Ingress controller adds. Prometheus scrapes the pods directly and still gets the metrics.

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
the Ingress, as it did, set `frontend.metrics.public: "true"` in `applications.files`, in
`/var/lib/genestack/properties/application.yaml`. The chart sets it to `"false"`. The
`Access-Control-Allow-Origin: *` header that `nginx.developmentSettings` also added is not needed: `applications`
answers CORS itself. It returns each `Origin` that matches `genestack.connections.cORSPatterns`, which is `".*"`, every
origin, by default.

The chart does not render values that still contain an `nginx` section, and its error message points to `ingress`.
Values of your own that call `<release>-nginx:8080`, in `envVars` for example, must call
`<release>-applications:8080` instead.

On AWS:

- With an ALB the AWS Load Balancer Controller creates, each Service becomes a target group of its own. The chart
  sets no annotations on the Services. Set `alb.ingress.kubernetes.io/healthcheck-path` in
  `<component>.service.annotations` of `applications`, `ui`, `swagger`, `userDocs`, `mailpit` and `mcp`, as the
  chart's `examples/ingress-alb.yaml` does. Without it the target group checks `/` on the traffic port. The old
  example set it in `nginx.ingress.annotations`: leave it out of `ingress.annotations`, where it applies to every
  Service that sets none. An Ingress without `alb.ingress.kubernetes.io/group.name` has an ALB of its own, so the
  controller replaces the ALB of `<release>-nginx` with a new one, under a new DNS name: update the DNS record, unless
  External DNS does it.
- With an ALB you created yourself, replace the target group and the TargetGroupBinding of `<release>-nginx` with
  one per Service, and add the listener rules and attributes, see [AWS](clouds/aws.md#recommended-steps-for-eks-configuration).

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
