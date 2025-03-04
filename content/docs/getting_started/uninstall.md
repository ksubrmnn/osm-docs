---
title: "Uninstall"
description: "Uninstall"
type: docs
weight: 4
---

# Uninstallation Guide

This guide describes how to uninstall Open Service Mesh (OSM) from a Kubernetes cluster. This guide assumes there is a single OSM control plane (mesh) running. If there are multiple meshes in a cluster, repeat the process described for each control plane in the cluster before uninstalling any cluster wide resources at the end of the guide. Taking into consideration both the control plane and dataplane, this guide aims to walk through uninstalling all remnants of OSM with minimal downtime.

## Prerequisites

- Kubernetes cluster with OSM installed
- The `kubectl` CLI
- The [`osm` CLI](/docs/install/#set-up-the-osm-cli) or the Helm 3 CLI

## Remove Envoy Sidecars from Application Pods and Envoy Secrets

The first step to uninstalling OSM is to remove the Envoy sidecar containers from application pods. The sidecar containers enforce traffic policies. Without them, traffic will flow to and from Pods according in accordance with default Kubernetes networking unless there are [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) applied.

OSM Envoy sidecars and related secrets will be removed in the following steps:

1. [Disable automatic sidecar injection](#disable-automatic-sidecar-injection)
1. [Restart pods](#restart-pods)
1. [Delete Envoy bootsrap secrets](#delete-envoy-bootsrap-secrets)

### Disable Automatic Sidecar Injection

OSM Automatic Sidecar Injection is most commonly enabled by adding namespaces to the mesh via the `osm` CLI. Use the `osm` CLI to see which
namespaces have sidecar injection enabled. If there are multiple control planes installed, be sure to specify the `--mesh-name` flag.

View namespaces in a mesh:

```console
$ osm namespace list --mesh-name=<mesh-name>
NAMESPACE          MESH           SIDECAR-INJECTION
<namespace1>       <mesh-name>    enabled
<namespace2>       <mesh-name>    enabled
```

Remove each namespace from the mesh:

```console
$ osm namespace remove <namespace> --mesh-name=<mesh-name>
Namespace [<namespace>] successfully removed from mesh [<mesh-name>]
```

Alternatively, if sidecar injection is enabled via annotations on pods instead of per namespace, please modify the pod or deployment spec to remove the sidecar injection annotation.

### Restart Pods

Restart all pods running with a sidecar:

```console
# If pods are running as part of a Kubernetes deployment
# Can use this strategy for daemonset as well
$ kubectl rollout restart deployment <deployment-name> -n <namespace>

# If pod is running standalone (not part of a deployment or replica set)
$ kubectl delete pod <pod-name> -n namespace
$ k apply -f <pod-spec> # if pod is not restarted as part of replicaset
```

Now, there should be no OSM Envoy sidecar containers running as part of the applications that were once part of the mesh. Traffic is no
longer managed by the OSM control plane with the `mesh-name` used above. During this process, your applications may experience some downtime
as all the Pods are restarting.

### Delete Envoy Bootsrap Secrets

Once the sidecar is removed, there is no need for the Envoy bootstrap config secrets OSM created. These are stored in the application namespace and can be deleted manually with `kubectl`. These secrets have the prefix `envoy-bootstrap-config` followed by some unique ID: `envoy-bootstrap-config-<some-id-here>`.

## Resource Management

## Uninstall OSM Control Plane and Remove User Provided Resources

The OSM control plane and related components will be uninstalled in the following steps:

1. [Uninstall the OSM control plane](#uninstall-the-osm-control-plane)
1. [Remove User Provided Resources](#remove-user-provided-resources)
1. [Delete OSM Namespace](#delete-osm-namespace)

### Uninstall the OSM control plane

Use the `osm` CLI to uninstall the OSM control plane from a Kubernetes cluster. The following step will remove:

1. OSM controller resources (deployment, service, config map, and RBAC)
1. Prometheus, Grafana, Jaeger, and Fluentbit resources installed by OSM
1. Mutating webhook and validating webhook
1. CRDs installed/required by OSM: [CRDs for OSM](https://github.com/openservicemesh/osm/tree/main/charts/osm/crds). This will also remove all Custom Resources related to those CRDs. Refer to [Removal of OSM Cluster Wide resources](#remove-osm-cluster-wide-resources) for more details

Run `osm uninstall`:

```console
# Uninstall osm control plane components
$ osm uninstall --mesh-name=<mesh-name>
Uninstall OSM [mesh name: <mesh-name>] ? [y/n]: y
OSM [mesh name: <mesh-name>] uninstalled
```

Run `osm uninstall --help` for more options.

Alternatively, if you used Helm to install the control plane, run the following `helm uninstall` command:

```console
$ helm uninstall <mesh name> --namespace <osm namespace>
```

Run `helm uninstall --help` for more options.

### Remove User Provided Resources

If any resources were provided or created for OSM at install time, they can be deleted at this point.

For example, if [Hashicorp Vault](/docs/guides/certificates/#installing-hashi-vault) was deployed for the sole purpose of managing certificates for OSM, all related resources can be deleted.

### Delete OSM Namespace

When installing a mesh, the `osm` CLI creates the namespace the control plane is installed into if it does not already exist. However, when uninstalling the same mesh, the namespace it lives in does not automatically get deleted by the `osm` CLI. This behavior occurs because
there may be resources a user created in the namespace that they may not want automatically deleted.

If the namespace was only used for OSM and there is nothing that needs to be kept around, the namespace can be deleted at this time with `kubectl`.

```console
$ kubectl delete namespace <namespace>
namespace "<namespace>" deleted
```

Repeat the steps above for each mesh installed in the cluster. After there are no OSM control planes remaining, move to following step.

## Removal of OSM Cluster Wide resources

Uninstalling OSM will remove all the CRDs mentioned [here](https://github.com/openservicemesh/osm/tree/main/charts/osm/crds). This is the same behavior when using the Helm charts to uninstall OSM as well because OSM has a pre-delete helm hook that removes the CRDs. If there are other service meshes running in the same cluster, please ensure that you back the Custom Resources and re-apply the required CRDs and Custom Resources to the cluster using `kubectl` post a successful uninstallation of OSM.
