# Nextcloud service for Kubernetes on Wodby

Run Nextcloud as a reusable Kubernetes application service with Wodby.

This repository defines the Wodby service manifest for Nextcloud. It uses the
upstream Nextcloud Helm chart and the Docker Official Image maintained by the
Nextcloud community.

- [Nextcloud service on Wodby](https://wodby.com/services/nextcloud)
- [Browse Wodby services](https://wodby.com/services)
- [Wodby service documentation](https://wodby.com/docs/2.0/services/)
- [Service manifest reference](https://wodby.com/docs/2.0/services/template/)

## Wodby stacks using this service

- [Nextcloud application stack](https://github.com/wodby/stack-nextcloud)

## Service overview

| Property | Manifest configuration |
| --- | --- |
| Service name | `nextcloud` |
| Type | Application service |
| Versions | Nextcloud 34 by default; Nextcloud 33 also supported |
| Workload | `main` Deployment using `Recreate` updates |
| Containers | Nextcloud Apache application and cron sidecar |
| Endpoint | `nextcloud`: HTTP 8080 (main) |
| Service links | PostgreSQL and Redis required; SMTP optional |
| Integrations | Optional S3 primary storage through AWS S3, Cloudflare R2, or Backblaze B2 |
| Storage | Persistent volume for `/var/www/html`; optional S3 bucket for file content |
| Helm | `nextcloud/nextcloud` chart version `9.4.1` |

## Runtime behavior

The service disables the chart's bundled database, Redis, ingress, and
HTTPRoute resources. Wodby supplies PostgreSQL, Redis, routing, TLS, and the
persistent volume instead.

The initial administrator username and password are required secret settings.
Both apply only to the first installation.

The PostgreSQL and Redis passwords reach the chart through the app service's
Kubernetes Secret, not through plain Helm values.

Nextcloud background jobs run from the chart's cron sidecar every five minutes.
The sidecar shares the application image, configuration, and persistent volume.

The generated Nextcloud configuration reads Wodby's `WODBY_HOSTS` environment
variable. All active route and App Access hostnames are therefore accepted as
trusted domains. When a hostname is added or removed, Wodby redeploys Nextcloud
so that it picks up the new list.

The default Wodby route uses one-hour request timeouts and passes uploads
through to Nextcloud as they arrive, without a route size limit. The PHP upload
limit (3 GB by default) caps one upload request; Nextcloud clients split larger
files into smaller requests. Do not set the route's `request_body_size`: it
makes the shared gateway hold every upload in memory until it is complete.

Forwarded client addresses require the ingress proxy addresses or CIDR ranges
to be entered in the optional `Trusted proxies` setting. Do not trust a range
broader than the cluster's actual proxy network.

## S3 primary storage

The optional `S3 primary storage` integration supports Amazon S3, Cloudflare
R2, and Backblaze B2. Create an empty bucket dedicated to the Nextcloud
installation, attach the matching storage integration, and enter its name in
the `S3 bucket` setting before the first deployment. Both the integration and
bucket setting are required to activate object storage.

Primary object storage must not be enabled on an existing installation: doing
so makes files already stored on the persistent volume inaccessible. Nextcloud
keeps file and directory metadata in PostgreSQL and stores opaque file content
in the bucket, so the bucket is not a directly browsable copy of the users'
folder structure and must be reserved for Nextcloud.

## Persistence and upgrades

Without an S3 integration, the persistent volume contains `/var/www/html`,
including user data. With S3 primary storage, user file content is stored in
the bucket while the volume still contains generated configuration, themes,
and installed custom applications. Back up PostgreSQL, the persistent volume,
and, when enabled, the S3 bucket as one consistent set.

Nextcloud upgrades its database when a new version starts, before it serves
requests. A startup probe gives the upgrade about 15 minutes before Kubernetes
restarts the container.

Upgrade one Nextcloud major version at a time. For example, upgrade version 33
to 34 before selecting a future version 35 option.

## Use this service

Use this service through the
[Nextcloud application stack](https://github.com/wodby/stack-nextcloud), or
reference `nextcloud` from a custom Wodby stack.

A service is a reusable component and does not deploy by itself. The stack
defines its links, storage sizes, versions, resources, and relationship to the
rest of the application.

## Maintain a custom version

1. Fork this repository.
2. Edit `service.yml`.
3. Import the repository as a
   [Git-backed service](https://wodby.com/docs/2.0/services/create/#create-a-git-backed-service).
4. Reference the service from a stack manifest.

Keep service, workload, container, endpoint, link, volume, and setting names
stable unless dependent stacks and app-level overrides are updated at the same
time.

Validate the manifest with:

```bash
wodby service validate-manifest service.yml --org <org-id>
```

See the [service manifest reference](https://wodby.com/docs/2.0/services/template/)
and the [managed services index](https://github.com/wodby/services).
