# terraform-google-backend-to-gke

A Terraform module for easily building a Backend Service to a Workload
running in one or more GKE clusters.  Mostly meant to be used by the
[terraform-google-ingress-to-gke](
https://github.com/TyeMcQueen/terraform-google-ingress-to-gke)⧉ module
but can be useful on its own.


## Contents

* [Simplest Example](#simplest-example)
* [Multi-Region Example](#multi-region-example)
* [Output Values](#output-values)
* [Example Workload](#example-workload)
* [Custom Backend](#custom-backend)
* [Backend Service](#backend-service)
* [Health Check](#health-check)
* [Limitations](#limitations)
* [Input Variables](#input-variables)


## Simplest Example

First, let's see how simple this module can be to use.  This invocation
of the module creates a Backend Service for a Kubernetes Workload running in
a GKE Cluster (via zonal Network Endpoint Groups), including generating a
generic Health Check.

    module "my-backend" {
      source            = (
        "github.com/TyeMcQueen/terraform-google-backend-to-gke" )
      cluster-objects   = [ google_container_cluster.my-gke.id ]
      neg-name          = "my-svc"
    }

Before you can `apply` such an invocation, you need to deploy your Workload
to the referenced cluster and it must include a Service object with an
annotation similar to:

    cloud.google.com/neg: '{"exposed_ports": {"80": {"name": "my-svc"}}}'

This step creates the Network Endpoint Groups (one per Compute Zone) that
route requests to any healthy instances of your Workload.  The "name" in
the annotation must match the `neg-name` you pass to this module.

But see [Example Workload](#example-workload) for how you can create a
Backend Service before you have your workload implementation ready.


## Multi-Region Example

Here is an example that configures a Backend Service that can be used
for multi-region ingress to your Workload running in multiple GKE
clusters (3 regional clusters in this case).

    module "my-ingress" {
      source            = (
        "github.com/TyeMcQueen/terraform-google-ingress-to-gke" )
      clusters          = {
      # Location           GKE Cluster Name
        us-central1     = "my-gke-usc1-prd",
        europe-west1    = "my-gke-euw1-prd",
        asia-east1      = "my-gke-ape1-prd",
      }
      neg-name          = "my-svc"
    }

You can use `clusters` and/or `cluster-objects` to specifies your GKE
Clusters.


## Output Values

The resource records for anything created by this module and some other
data are available as output values.

`module.NAME.backend[0]` will be the resource record for the created Backend
Service.  You can use `module.NAME.backend[0].id` to reference this Backend
when creating other resources.

`module.NAME.health[0]` will be the resource record for the Health Check
if the module created one.

`module.NAME.negs` will be a map from each Compute Zone name to the resource
record for a zonal NEG (that was created by the GKE Ingress controller).

These are declared in [outputs.tf](/outputs.tf).


## Example Workload

The file [/examples/workload.yaml] is an example Kubernetes Workload
specification.  Simply download the file, replace each "{SELF}"
with whatever name you want to use and then you can deploy this via
`kubectl apply -f workload.yaml` (after authenticating to your cluster
and setting it as the default for `kubectl`).  This will create the NEGs
so you can set up a full ingress before you have real code that you want
to deploy.

This example workload uses an image that simply always gives a 403
rejection response to every request (except for GCP health checks).

Once you have your ingress set up, you can delete this workload via
`kubectl delete -f workload.yaml` and the NEGs will remain due to the
Backend Service you created.  Then, when you deploy your own workload
using the same NEG name to the same GKE Cluster(s), the Backend Service
will automatically route to this new workload.


## Custom Backend

There are a lot of possible options when configuring a Backend Service.
If you need to set some options that are not supported by this module, then
you can still use this module to find the NEGs that should be added to your
Backend (and possibly to create the simple health check).

    module "my-neg" {
      source            = (
        "github.com/TyeMcQueen/terraform-google-backend-to-gke" )
      cluster-objects   = [ google_container_cluster.my-gke.id ]
      neg-name          = "my-svc"
      lb-scheme         = "" # Don't create the Backend
    }

    resource "google_compute_backend_service" "b" {
      ...
      health_checks             = [ module.my-neg.health[0].id ]
      dynamic "backend" {
        for_each                = module.my-neg.negs
        content {
          group                 = backend.value.id
          balancing_mode        = "RATE"
          max_rate_per_endpoint = 1000
          # Terraform defaults to 0.8 which makes no sense for "RATE" w/ NEGs:
          max_utilization       = 0.0
        }
      }
    }


## Backend Service

This module creates one Backend Service unless you set `lb-scheme` to "".
You must always set `neg-name` to the `name` included in an annotation on
your Kubernetes Service object like:

    cloud.google.com/neg: '{"exposed_ports": {"80": {"name": "my-svc"}}}'

And you must list one or more GKE clusters that you have already
deployed such a Workload to.  You can list GKE cluster resource records
in `cluster-objects`.  You can put `location-name = "cluster-name"` pairs
into the `clusters` map.  You can even list some clusters in the former
and some in the latter.

You can set `lb-scheme = "EXTERNAL"` to use "classic" Global L7 HTTP(S)
Load Balancing.  Note that this value must also be used in the other load
balancing components you connect to the Backend.

`log-sample-rate` defaults to 1.0 which logs all requests for your Backend.
You can set it to 0.0 to disable all request logging.  Or you can set it to
a value between 0.0 and 1.0 to log a sampling of requests.

You can also set `max-rps-per` to specify a different maximum rate of
requests (per second, per pod) that you want load balancing to adhere to.
But exceeding this rate simply causes requests to be rejected; it does not
impact how your Workload is scaled up.  It also does not adapt when the
average latency of responses changes.  So it is better to set this value
too high rather than too low.  It only functions as a worst-case rate limit
that may help to prevent some overload scenarios but using load shedding is
usually a better approach.


## Health Check

By default, this module creates a generic Health Check for the Backend
Service to use.  But you can instead reference a Health Check that you
created elsewhere via `health-ref`.

The generated Health Check will automatically determine which port number to
use.  The requests will use a User-Agent name that starts with "GoogleHC/",
so if you have your Workload detect this and then respond with health
status, then you don't have to have the Health Check and your Workload
agree on a specific URL to use.  But you can specify the URL path to use
in `health-path`.

See [inputs](#inputs) or [variables.tf](variables.tf) for more information
about the other Health Check options: `health-interval-secs`,
`health-timeout-secs`, `unhealthy-threshold`, and `healthy-threshold`.
If you need more customization than those provide, then you can simply
create your own Health Check and use `health-ref`.


## Limitations

* [Google Providers](#google-providers)
* [Error Handling](
    https://github.com/TyeMcQueen/terraform-google-http-ingress/blob/main/docs/Limitations.md#error-handling)⧉
* [Handling Cluster Migration](#handling-cluster-migration)

You should also be aware of types of changes that require special care as
documented in the other module's limitations: [Deletions](
https://github.com/TyeMcQueen/terraform-google-certificate-map-simple/README.md#deletions)⧉.

### Google Providers

This module uses the `google-beta` provider and allows the user to control
which version (via standard Terraform features for such).  We would like
to allow the user to pick between using the `google` and the `google-beta`
provider, but Terraform does not allow such flexibility with provider
usage in modules at this time.

You must use at least Terraform v0.13 as the module uses some features
that were not available in earlier versions.

You must use at least v4.22 of the `google-beta` provider.

### Handling Cluster Migration

The GKE automation that turns the Service Annotation into a Network Endpoint
Group (NEG) in each Compute Zone used by the GKE Cluster has one edge case
that can cause problems if you move your Workload to a new Cluster in the
same Compute Region or Zone.

The Created NEGs contain a reference to the creating Cluster.  When the
Workload is removed from a Cluster, the NEGs will not be destroyed
if the Backend Service created by this module still references them.  If
you then deploy the Workload to the new Cluster, the attempt to create
new NEGs will conflict with these lingering old NEGs.

So to migrate a Workload to a new Cluster that overlaps Zones, you must:

* Either delete the Backend Service (such as by commenting out your
    invocation of this module) or just remove the particular NEGs from the
    Backend Service (by removing the original Cluster from `clusters` or
    `cluster-objects`).

* `apply` the above change.

* Remove the Workload from the old Cluster (or remove the Annotation).  Note
    that it is also okay to do this step first.

* Verify that the NEGs have been garbage collected...

The following command will show you the NEGs so you can verify that any
in Zones of the old Cluster have been removed:

    gcloud --project YOUR-PROJECT compute network-endpoint-groups list | sort

If they fail to be automatically deleted (which we have seen happen if you
do a cluster migration without following these steps and then try to use
these steps to fix things), then you can delete them via:

    gcloud --project YOUR-PROJECT compute network-endpoint-groups \
        delete NEG-NAME --zone ZONE

* Deploy your Workload (with Service Annotation) to the new Cluster.

* Add the new cluster to `clusters` or `cluster-objects` or uncomment the
    invocation of this module.

* `apply` the above change.

If you have the Workload in another Cluster already, then this migration can
happen with no service interruption.


## Input Variables

* [cluster-objects](/variables.tf#L40)
* [clusters](/variables.tf#L27)
* [description](/variables.tf#L79)
* [health-interval-secs](/variables.tf#L116)
* [health-path](/variables.tf#L107)
* [health-ref](/variables.tf#L92)
* [health-timeout-secs](/variables.tf#L126)
* [healthy-threshold](/variables.tf#L147)
* [iap-id](/variables.tf#L247)
* [iap-secret](/variables.tf#L258)
* [lb-scheme](/variables.tf#L162)
* [log-sample-rate](/variables.tf#L178)
* [max-rps-per](/variables.tf#L189)
* [name-prefix](/variables.tf#L67)
* [neg-name](/variables.tf#L5)
* [project](/variables.tf#L56)
* [security-policy](/variables.tf#L222)
* [session-affinity](/variables.tf#L232)
* [timeout-secs](/variables.tf#L206)
* [unhealthy-threshold](/variables.tf#L136)
