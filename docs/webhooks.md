# Webhooks

Webhooks allow validation and/or defaulting/mutating logic to be added to the Kubernetes API for the creation, updating and deletion of custom resources (CRs).  This enables us to execute validation/mutation against an incoming CR _before_ it is admitted to the reconcile loop of its associated operator.  In this manner, in the case of validation, we can reject improper CR specifications immediately and report that feedback to the user (as opposed to having the reconcile loop within the associated operator detect the problem and surface the error in the status section of the CR, which is a slower feedback mechanism).  Mutation webhooks can be used to set defaults and keep such considerations out of the associated operator's reconcile loop, which makes for a cleaner division of data and its processing logic.

General information about Kubernetes/OpenShift webhooks can be found here:

* https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/
* https://docs.openshift.com/container-platform/4.12//operators/understanding/olm/olm-webhooks.html

## Why did we introduce webhooks?

The initial injection of webhooks into our operators was done to achieve greater flexibility with container image defaults -- that is, for default OpenStack service images, not for the operators themselves.  By using mutating/defaulting webhooks, we are able to remove hard-coded `kubebuilder` annotation defaults from our CRD Golang types and instead provide them via environment variables ([example](https://github.com/openstack-k8s-operators/cinder-operator/blob/master/config/default/manager_default_images.yaml)).  These environment variables are read by the operator controller-manager during its initialization ([example](https://github.com/openstack-k8s-operators/cinder-operator/blob/feac84f5479a33a708ed94d5deedf9366b328038/main.go#L163-L171)) and are then available to the controller-manager's mutating/defaulting webhooks during CR creation, where they can be applied if the user has not included an explicit image in the CR definition ([example](https://github.com/openstack-k8s-operators/cinder-operator/blob/feac84f5479a33a708ed94d5deedf9366b328038/api/v1beta1/cinder_webhook.go#L66-L94)).

Why do we need this environment-variable-based flexibility?  

1. It allows for OpenStack service container image defaults to be changed in a live deployment _without_ having to respin an operator build and reinstall the updated operator (which would otherwise have to be done to pick up a `kubebuilder` annotation change).  Instead, the CSV for a live deployment of an operator can be edited to adjust the container image environment variables to point to whatever default is necessary.  This is extremely useful for what are known as "offline" or "air-gapped" or "disconnected" OCP clusters.  These types of clusters do not have external Internet access and would be unable to reach either the upstream or downstream (online) registries to pull the default container images.  Instead, after the operator is initially deployed, the CSV's environment variable declarations can be modified by the cluster administrator to point to images available in an internally-accessible private registry.

2. It makes the operators easier to maintain in a downstream context.  Rather than requiring a specific code patch to modify the CRD Golang structs for downstream builds, the downstream build logic can instead just rebase on every upstream import and override the container image environment variables in the CSVs it uses to build the operator bundles.

3. It makes testing for QE/CI easier.  Similar to #1 and #2, the specific container images required for downstream testing can be easily changed in the CSV.  This avoids requiring rebuilds with different defaults or using CRs with particular downstream container images explicitly specified.

## Adding webhooks to an operator

`operator-sdk` provides convenient scaffolding commands to add webhooks to an operator.  For example:

```bash
cd cinder-operator
operator-sdk create webhook --group cinder --version v1beta1 --kind Cinder --programmatic-validation --defaulting
```

`--programmatic-validation` indicates that you want validation webhooks, while `--defaulting` signals that you want mutating ones.  Using both flags is advisable, even if you do not plan to immediately use both types of webhooks.  The reason for this is that using the command again requires a complete wipe and re-scaffolding of any existing webhook content (done via the `--force` flag, by the way).  So it is better to have both -- even if one is empty scaffolding -- in the interest of not having to re-inject your specific webhook logic for one type of webhook if you later decide that you want the other type of webhook.

Once the webhooks have been scaffolded, you will find them in the `api/<version>` directory.  Examine this file to get an idea of what the scaffolding has provided:

```bash
$ ls api/v1beta1
...
-rw-rw-r--. 1 ocp ocp  4024 Mar 14 11:24 cinder_webhook.go
...
```

There are also additions/changes made to the `config` directory and to `main.go`.  Without going into detail in this doc, the `config` directory will now contain additional YAML to bundle the webhook definitions via operator-sdk such that they are installed alongside the operator via OLM.  The `main.go` changes consist of extra code to start a webserver within the operator controller-manager to serve the endpoints for the validating and mutating webhooks.

It is furthermore recommended to add support for running the operator locally with webhooks.  The pattern for doing so can be seen here: https://github.com/openstack-k8s-operators/cinder-operator/pull/155/files.

## Running an operator that has webhooks via OLM

When the associated operator is deployed via OLM, its webhooks are installed in the cluster and cert management is automatically handled as well.  You don't need to do anything differently than you normally would for installing the operator.

## Locally running an operator that has webhooks

If you would like to run the operator locally, extra steps must be taken to either disable the webhooks completely or to enable them to run outside of an OLM context.

Before following either section below, make sure you're pointing to a valid `KUBECONFIG` or have otherwise used `oc` to log into your cluster.

### Disabling webhooks

You can disable the webhooks if you don't need them for your local dev/testing purposes.  The procedure is as follows:

1. If you installed the operator via OLM, remove its webhook definitions from its CSV:

```bash
oc patch csv -n openstack-operators <your operator CSV> --type=json -p="[{'op': 'remove', 'path': '/spec/webhookdefinitions'}]"
```

2. Run the operator locally with the webhook server disabled:

```bash
cd <your operator root dir>
ENABLE_WEBHOOKS=false GOWORK= OPERATOR_TEMPLATES=./templates make run
```

### Using webhooks locally

Webhooks can be used outside of an OLM context if you want or need them.  However, using webhooks locally is non-trivial and is best handled by adding something like [this](https://github.com/openstack-k8s-operators/cinder-operator/pull/155/files) to the operator itself (as mentioned in the [Adding webhooks to an operator](#adding-webhooks-to-an-operator) section above).  This doc will only cover that use case for the time being.  Thus, do the following to use webhooks locally where the aforementioned support is present:

1. First, if you installed the operator via OLM, remove its webhook definitions from its CSV:

```bash
oc patch csv -n openstack-operators <your operator CSV> --type=json -p="[{'op': 'remove', 'path': '/spec/webhookdefinitions'}]"
```

2. Now execute the `make` target to run the operator locally with webhooks enabled:

```bash
cd <your operator root dir>
GOWORK= OPERATOR_TEMPLATES=./templates make run-with-webhook
```

This `make` command will:  

1. Create self-signed certs for the webhooks and create `ValidatingWebhookConfiguration` and `MutatingWebhookConfiguration` resources within the cluster that use the certs.  The operator itself also knows where the find the certs in a default location on your local host (`/tmp/k8s-webhook-server/serving-certs`) and will use them when it starts running.
2. Find the OpenShift SDN gateway IP for your CRC cluster.  If you are not using CRC, you will need to provide `CRC_IP=<OpenShift SDN gateway IP>` to the aforementioned `make` command!
3. Inject the `CRC_IP` into the `ValidatingWebhookConfiguration` and `MutatingWebhookConfiguration` resources so that they point the webhook server running in your local operator controller-manager process.
4. Open the local host's firewall to allow port 9443 traffic (the port used by the webhooks).  This is done in the `libvirt` `firewall-zone`.  _If your local cluster is running in a different zone, you will have to manually add a firewall rule for port 9443 TCP traffic yourself!_
5. Finally, run the actual operator controller-manager and the webhook server.

You should now be able to create CRs for the associated operator and have the webhook logic execute as expected.  
**NOTE:** If you want to switch back to using the OLM-deployed version of your operator, you will need to manually `oc delete` the `ValidatingWebhookConfiguration` and `MutatingWebhookConfiguration` resources created by this `make` command!
