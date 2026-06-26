# Hosted Mode Support for AddOnTemplate

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal adds Hosted Mode support to `AddOnTemplate`-based addons. Today, addons built with the
addon-framework Go library can deploy their agents on a hosting cluster (outside the managed cluster) using the
Hosted Mode mechanism introduced in [enhancement #63](../63-hosted-addon/README.md). Template-based addons cannot
do this because the template agent controller does not enable Hosted Mode and does not expose the install mode to
template rendering. This proposal makes template-based addons work with Hosted Mode the same way code-based addons
already do.

## Motivation

The [AddOnTemplate enhancement (#82)](../82-addon-template/README.md) explicitly listed Hosted Mode as a non-goal
at the time of its initial design. Since then, Hosted Mode has become widely used in OCM. It allows addon agents
to run on a hosting cluster rather than the managed cluster, which is useful when:

- The managed cluster should not run addon workloads (e.g., resource-constrained edge clusters).
- The managed cluster is imported in Hosted Mode (klusterlet runs externally), and addon agents should follow the
  same pattern to avoid requiring direct hub-to-managed-cluster connectivity.
- Security requirements mean the managed cluster should not have credentials to reach the hub cluster.

Without Hosted Mode support, addon developers who need this pattern are forced to fall back to the code-based
addon-framework approach, which defeats the purpose of having a template-based model.

### Goals

- Enable template-based addons to deploy agent manifests on a hosting cluster using the existing Hosted Mode
  annotations (`addon.open-cluster-management.io/hosting-cluster-name` and
  `addon.open-cluster-management.io/hosted-manifest-location`).
- Expose a new built-in template variable `INSTALL_MODE` so that template manifests can conditionally include
  configurations specific to Hosted Mode (e.g., managed-kubeconfig volume mounts).
- Maintain backward compatibility. Existing `AddOnTemplate` addons continue to work without changes.

### Non-Goals

- Modifying the `AddOnTemplate` CRD schema. Hosted Mode uses annotations on the `ManagedClusterAddOn` and on
  individual manifests, which is how the addon-framework already handles it for code-based addons.
- Automatically determining the hosting cluster. The addon deployer explicitly sets the hosting cluster annotation,
  as defined in enhancement #63.
- Supporting all addons in Hosted Mode. Some addons need direct access to managed cluster nodes or local resources
  and cannot run externally. This is a Hosted Mode limitation in general, not specific to templates.

## Proposal

### User Stories

#### Story 1

As an addon developer using `AddOnTemplate`, I want to annotate my addon manifests with
`addon.open-cluster-management.io/hosted-manifest-location: hosting` so that the addon-framework deploys those
manifests on the hosting cluster instead of the managed cluster when the addon is in Hosted Mode.

#### Story 2

As an addon deployer, I want to set the `addon.open-cluster-management.io/hosting-cluster-name` annotation on a
`ManagedClusterAddOn` that references an `AddOnTemplate`, and have the addon agent deployed on the specified hosting
cluster, following the same pattern as code-based addons.

#### Story 3

As an addon developer, I want to use the `{{INSTALL_MODE}}` variable in my `AddOnTemplate` manifests so that my
agent container receives the deploy mode as an environment variable and can adjust its behavior for Hosted vs
Default mode at runtime.

## Design Details

### Overview

The implementation requires two changes to the template agent controller in the `ocm` repository
(`pkg/addon/templateagent/`):

1. **Enable Hosted Mode** in the template agent's `AgentAddonOptions` so that the addon-framework's hosted syncer
   processes template-based addons.
2. **Expose `INSTALL_MODE`** as a built-in template variable so that template manifests can reference the current
   deploy mode.

No API (CRD) changes are required. The existing annotation-based mechanism for Hosted Mode
([enhancement #63](../63-hosted-addon/README.md)) applies directly to `AddOnTemplate` manifests.

### Enabling Hosted Mode in the Template Agent

Currently, the `CRDTemplateAgentAddon.GetAgentAddonOptions()` method returns an `AgentAddonOptions` struct with
`HostedModeEnabled` defaulting to `false`. The addon-framework's `hostedSyncer` checks this flag and skips all
hosted-mode processing when it is false.

We propose to set `HostedModeEnabled: true` in the returned options. The template agent already sets
`HostedModeInfoFunc: addonconstants.GetHostedModeInfo`, which reads the hosting cluster name from the
`addon.open-cluster-management.io/hosting-cluster-name` annotation on the `ManagedClusterAddOn`. With
`HostedModeEnabled` set to true, the addon-framework's existing syncer chain handles all the hosted-mode logic:

- The **defaultSyncer** (managed manifest processor) filters out manifests annotated with
  `hosted-manifest-location: hosting` when `installMode == "Hosted"`, deploying only managed-cluster resources
  via ManifestWork in the managed cluster namespace.
- The **hostedSyncer** (hosting manifest processor) creates a separate ManifestWork in the hosting cluster namespace
  containing only the manifests annotated with `hosted-manifest-location: hosting`.
- Health checks, finalizers, and pre-delete hooks work identically to code-based addons.

### New Built-in Template Variable: `INSTALL_MODE`

A new built-in variable `INSTALL_MODE` is added to the template rendering context. Its value is `"Hosted"` when
the addon is deployed in Hosted Mode, or `"Default"` otherwise. This value is determined by the existing
`GetHostedModeInfo` function, which reads the hosting cluster annotation.

The variable is available in two ways:

1. **As a template variable**: addon developers can use `{{INSTALL_MODE}}` in their `AddOnTemplate` manifests.
   `AddOnTemplate` uses simple string substitution (not Go template conditionals), so in practice this is used
   to inject the mode into container arguments or environment variables.

2. **As an injected environment variable**: like all built-in variables (`CLUSTER_NAME`, `HUB_KUBECONFIG`),
   `INSTALL_MODE` is automatically injected as an environment variable into all Deployment and DaemonSet containers
   defined in the template. The addon agent can read this at runtime to adjust its behavior.

The built-in values struct becomes:

```go
type templateCRDBuiltinValues struct {
    ClusterName      string `json:"CLUSTER_NAME,omitempty"`
    InstallNamespace string `json:"INSTALL_NAMESPACE,omitempty"`
    InstallMode      string `json:"INSTALL_MODE,omitempty"`
}
```

This mirrors the existing `templateBuiltinValues` struct in the addon-framework's `addonfactory` package, which
already includes `InstallMode` for code-based template addons.

### Manifest Location Annotations

Template-based addons use the same annotation as code-based addons to control manifest placement in Hosted Mode:

```
addon.open-cluster-management.io/hosted-manifest-location
```

Values:
- `hosting`: deploy on the hosting cluster
- `managed` (or not set): deploy on the managed cluster
- `none`: do not deploy in Hosted Mode

These annotations are placed directly on the manifest resources inside
`AddOnTemplate.spec.agentSpec.workload.manifests`. The addon-framework reads the annotations from the rendered
manifests and routes each one to the correct cluster.

When manifests are deployed on the hosting cluster, the addon developer must ensure the agent namespace exists
on the hosting cluster. This is typically done by including a Namespace manifest with the
`hosted-manifest-location: hosting` annotation. The namespace should also carry the label
`addon.open-cluster-management.io/namespace: "true"` so that the addon-framework can manage image pull secrets
in that namespace.

### How Hosted Deployment Works with Templates

The deployment flow for a template-based addon in Hosted Mode:

1. The addon deployer creates a `ManagedClusterAddOn` with the annotation
   `addon.open-cluster-management.io/hosting-cluster-name: <hosting-cluster>`.
2. The addon template controller renders the template manifests, injecting `INSTALL_MODE=Hosted` and other
   built-in values.
3. The addon-framework's deploy controller splits the rendered manifests by their
   `hosted-manifest-location` annotation:
   - Manifests with `hosting` go into a ManifestWork in the hosting cluster namespace
   - Manifests with `managed` or no annotation go into a ManifestWork in the managed cluster namespace
   - Manifests with `none` are not deployed
4. The work agents on each cluster apply their respective ManifestWorks.

### Status Conditions and Finalizers

When a template-based addon is deployed in Hosted Mode, the addon-framework sets the following additional status
conditions on the `ManagedClusterAddOn`:

- **`HostingClusterValidity`**: Indicates whether the hosting cluster (specified in the annotation) is a valid
  managed cluster of the hub. Set to `True` with reason `HostingClusterValid` when the hosting cluster is found,
  or `False` with reason `HostingClusterInvalid` when it is not.
- **`HostingManifestApplied`**: Indicates whether the hosting ManifestWork has been applied successfully. Set to
  `True` with reason `AddonManifestApplied` when the ManifestWork is created and applied on the hosting cluster.

The addon-framework adds the finalizer `addon.open-cluster-management.io/hosting-manifests-cleanup` so that
ManifestWorks on the hosting cluster are cleaned up when the addon is deleted.

### Registration in Hosted Mode

The `AddOnTemplate` registration mechanism (CSR-based client certificate issuance) works without changes in
Hosted Mode. The registration agent on the hosting cluster handles CSR creation and certificate rotation for
the addon agent, just as it does for code-based hosted addons.

When the addon is deployed in Hosted Mode, the hub-kubeconfig secret is created in the agent install namespace
on the hosting cluster. If the addon agent needs to access the managed cluster, the addon deployer must provide a
`{addon-name}-managed-kubeconfig` secret in the install namespace on the hosting cluster, following the same
pattern described in [enhancement #63](../63-hosted-addon/README.md).

### Example

#### AddOnTemplate with Hosted Mode Annotations

```yaml
apiVersion: addon.open-cluster-management.io/v1beta1
kind: AddOnTemplate
metadata:
  name: my-addon-template-v1
spec:
  addonName: my-addon
  agentSpec:
    workload:
      manifests:
      # Namespace for the addon agent, deployed on the hosting cluster
      - apiVersion: v1
        kind: Namespace
        metadata:
          name: my-addon-agent
          annotations:
            addon.open-cluster-management.io/hosted-manifest-location: "hosting"
          labels:
            addon.open-cluster-management.io/namespace: "true"

      # ServiceAccount, deployed on the hosting cluster
      - apiVersion: v1
        kind: ServiceAccount
        metadata:
          name: my-addon-agent-sa
          namespace: my-addon-agent
          annotations:
            addon.open-cluster-management.io/hosted-manifest-location: "hosting"

      # Deployment, deployed on the hosting cluster
      - apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: my-addon-agent
          namespace: my-addon-agent
          annotations:
            addon.open-cluster-management.io/hosted-manifest-location: "hosting"
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: my-addon-agent
          template:
            metadata:
              labels:
                app: my-addon-agent
            spec:
              serviceAccountName: my-addon-agent-sa
              containers:
              - name: agent
                image: quay.io/my-org/my-addon-agent:latest
                args:
                  - "--cluster-name={{CLUSTER_NAME}}"
                  - "--hub-kubeconfig={{HUB_KUBECONFIG}}"

      # CRD owned by the addon, deployed on the managed cluster
      - apiVersion: apiextensions.k8s.io/v1
        kind: CustomResourceDefinition
        metadata:
          name: myresources.my-addon.io
          annotations:
            addon.open-cluster-management.io/hosted-manifest-location: "managed"
        spec:
          group: my-addon.io
          # ... CRD spec
  registration:
  - type: KubeClient
    kubeClient:
      hubPermissions:
      - type: CurrentCluster
        currentCluster:
          clusterRoleName: my-addon-hub-role
```

#### ManagedClusterAddOn in Hosted Mode

```yaml
apiVersion: addon.open-cluster-management.io/v1beta1
kind: ManagedClusterAddOn
metadata:
  name: my-addon
  namespace: managed-cluster-1
  annotations:
    addon.open-cluster-management.io/hosting-cluster-name: hosting-cluster-1
    addon.open-cluster-management.io/install-namespace: my-addon-agent
spec:
  configs: []
```

In this example:
- The Namespace, ServiceAccount, and Deployment are created on `hosting-cluster-1` via a ManifestWork in the
  `hosting-cluster-1` namespace on the hub.
- The CRD is created on `managed-cluster-1` via a ManifestWork in the `managed-cluster-1` namespace on the hub.
- The addon agent receives `INSTALL_MODE=Hosted` as an environment variable.
- The addon-framework sets `HostingClusterValidity` and `HostingManifestApplied` status conditions on the
  `ManagedClusterAddOn`.

### Test Plan

#### Unit Tests

- Verify that `INSTALL_MODE` is included in the built-in template values with the correct value (`"Default"` or
  `"Hosted"`) based on the hosting cluster annotation.
- Verify that `HostedModeEnabled` is `true` in the options returned by `GetAgentAddonOptions()`.

#### Integration Tests

- Deploy a template-based addon in Hosted Mode and verify that ManifestWorks are created in the correct namespaces
  (hosting cluster namespace for hosting-annotated manifests, managed cluster namespace for managed-annotated
  manifests).
- Verify the `HostingManifestApplied` and `HostingClusterValidity` conditions are set correctly.
- Verify that the addon agent receives `INSTALL_MODE=Hosted` as an environment variable.

#### E2E Tests

- Set up a hub cluster, a hosting cluster, and a managed cluster (imported in Hosted Mode).
- Deploy a template-based addon with hosted manifest annotations.
- Verify the addon agent runs on the hosting cluster and can access both the hub and managed clusters.
- Verify addon deletion cleans up resources on both the hosting and managed clusters.

### Graduation Criteria

Since the `AddOnTemplate` API is already beta and this change does not introduce new API fields, there is no
separate feature gate or phased rollout. The criteria for considering this complete:

- `HostedModeEnabled` is set to `true` in the template agent options.
- `INSTALL_MODE` built-in variable is available in template rendering.
- Unit and integration tests cover the new behavior.
- E2E test demonstrates a template-based addon deployed in Hosted Mode on a KinD cluster.
- Documentation is updated with guidance for template addon developers on using Hosted Mode.

### Upgrade / Downgrade Strategy

This change is backward compatible:

- **Existing addons**: Template-based addons without the `hosting-cluster-name` annotation continue to deploy
  in Default Mode. The `INSTALL_MODE` variable is injected with value `"Default"`, which does not affect existing
  templates that do not reference it.
- **Downgrade**: If the addon-manager is downgraded to a version without this change, template-based addons with
  Hosted Mode annotations will fall back to Default Mode behavior (all manifests deployed on the managed cluster).
  The `{{INSTALL_MODE}}` variable will be rendered as an empty string.

### Version Skew Strategy

- The `INSTALL_MODE` variable is a template-side concern. If the addon-manager supports it but the template does
  not reference it, there is no impact.
- If the template references `{{INSTALL_MODE}}` but the addon-manager does not support it, the variable renders
  as an empty string, which is safe for most use cases (the agent can treat an empty install mode as Default).

## Alternatives

1. **Add a `hostedMode` field to the `AddOnTemplate` spec**: This would make Hosted Mode explicit in the CRD
   schema. But Hosted Mode is a deployment-time decision made by the addon deployer (via `ManagedClusterAddOn`
   annotations), not something the template author should hard-code. Code-based addons already use annotations
   for this, and templates should work the same way.

2. **Require addon developers to use the code-based addon-framework for Hosted Mode**: This is the status quo.
   It forces developers back to writing Go code just to get Hosted Mode, even when the rest of their addon is
   simple enough for a template.
