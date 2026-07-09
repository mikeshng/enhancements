# Replacing Validating Webhooks with CEL Validation and ValidatingAdmissionPolicy

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [X] Design details are appropriately documented from clear requirements
- [X] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

Historically, OCM has relied heavily on Go-based Validating Admission Webhooks to enforce complex validation rules on Custom Resources. While effective, webhooks introduce operational overhead, including managing TLS certificates, ensuring high availability of webhook pods, and adding network latency to every API request. 

With the maturity of Common Expression Language (CEL) validation embedded directly in CRDs and the introduction of `ValidatingAdmissionPolicy` in modern Kubernetes clusters, we can move validation in-tree. This proposal details a phased migration strategy to replace the existing OCM validating webhooks with native CEL validation and `ValidatingAdmissionPolicy`, controlled via a FeatureGate.

## Motivation

Moving validation logic into the API server drastically improves performance and reduces the blast radius of webhook failures.

### Goals
- Migrate simple field validation logic from Go webhooks to `+kubebuilder:validation:XValidation` CEL markers in the `api` repository.
- Use `ValidatingAdmissionPolicy` for cross-field or complex authorization validations that cannot be handled by structural CRD CEL markers.
- Implement a `CELValidation` FeatureGate to allow users to opt-in during the transition.
- Provide a smooth migration path where the webhook remains enabled by default but can eventually be phased out.

### Non-Goals
- Replacing **Mutating** Webhooks. Mutation cannot be achieved via CEL validation, so mutating webhooks will remain in the project.
- Forcing legacy Kubernetes clusters (pre-v1.28) to use features they do not support.

## Proposal

### User Stories

#### Story 1: Improved Cluster Latency and Reliability
An operator deploying OCM components across thousands of clusters notices that API server latency spikes when applying large batches of Custom Resources due to the network round-trips to the webhook server. By enabling the `CELValidation` feature gate, validation shifts into the Kubernetes API server natively. The operator observes lower API latency, and API requests no longer fail if the OCM webhook pod temporarily restarts.

#### Story 2: Reduced Operational Complexity
A cluster administrator struggles with rotating the self-signed TLS certificates for the OCM webhook server. As the project matures and graduates the CEL validation feature, the validation webhook is completely disabled. The administrator no longer needs to monitor or rotate certificates for the validating webhook.

### Risks and Mitigation
**Risk:** Migrating complex Go logic to CEL might miss edge cases, resulting in invalid resources being accepted.
**Mitigation:** The webhook will remain enabled by default during the initial rollout. We will implement extensive integration tests in the `api` repository comparing the output of the legacy webhook against the new CEL validations to ensure 1:1 parity before deprecating the webhook.

## Design Details

### Phase 1: CRD CEL Markers (`+kubebuilder:validation:XValidation`)
We will analyze the existing Go webhook validation code and translate all structural and field validation into Kubebuilder CEL markers. Based on the current OCM codebase, the following validations will be migrated:
- **`ManagedCluster`:** Validating that `Spec.ManagedClusterClientConfigs` contains valid HTTPS URLs, and `metadata.name` matches the required regex.
- **`ManagedClusterSetBinding`:** Ensuring `metadata.name` exactly matches `spec.clusterSet` using cross-field transition rules (`self.metadata.name == self.spec.clusterSet`).
- **`ManifestWork` & `ManifestWorkReplicaSet`:** Ensuring the `manifests` array is not empty and contains structurally valid JSON/YAML.

*Example (Old Webhook Code for ManagedClusterSetBinding):*
```go
if binding.Name != binding.Spec.ClusterSet {
    return nil, apierrors.NewBadRequest("The ManagedClusterSetBinding must have the same name as the target ManagedClusterSet")
}
```

*Example (New CEL Marker in `api` repo):*
```go
// +kubebuilder:validation:XValidation:rule="self.metadata.name == self.spec.clusterSet",message="The ManagedClusterSetBinding must have the same name as the target ManagedClusterSet"
type ManagedClusterSetBindingSpec struct {
    ClusterSet string `json:"clusterSet"`
}
```

### Phase 2: ValidatingAdmissionPolicy (Authorization Checks)
For complex rules that require checking authorization (`authorizer()`) via SubjectAccessReview (SAR), we will deploy `ValidatingAdmissionPolicy` and `ValidatingAdmissionPolicyBinding` resources. Based on the OCM codebase, these include:
- **`ManagedCluster`:** Setting `HubAcceptsClient=true` requires a SAR check for `subresource: accept`. Updating the `clusterset` label requires a SAR check for `subresource: join`.
- **`ManagedClusterSetBinding`:** Creating the binding requires a SAR check for `subresource: bind` on the target cluster set.
- **`ManifestWork`:** Modifying `Spec.Executor` requires a SAR check for `verb: execute-as` on the target ServiceAccount.

### Phase 3: Validation Points Retained in Go
Some validations cannot be migrated to CEL or ValidatingAdmissionPolicy and must remain in the Go webhooks.
- **Mutating Webhooks:** CEL is validation-only, so mutation logic stays in Go.
- **FeatureGate Runtime Checks:** Validating deletions against runtime FeatureGates (e.g., `ManifestWorkReplicaSetWebhook.ValidateDelete` checks if the `ManifestWorkReplicaSet` feature gate is enabled dynamically) cannot be done via CEL.

### Phase 4: The Feature Gate
We will introduce a new FeatureGate: `CELValidation` (Default: `false`).

When `CELValidation` is:
- **false (default):** The legacy Validating Webhook is registered and active. The `ValidatingAdmissionPolicy` resources are NOT deployed.
- **true:** The Validating Webhook registration is skipped for validation (mutating webhook remains active). The `ValidatingAdmissionPolicy` resources ARE deployed. (Note: CRD-level CEL markers will always be present, as they are compiled into the CRD).

### Test Plan
- **API Integration Tests:** A new test suite in the `api` repo will programmatically apply invalid resources to an `envtest` environment. We will assert that the API server natively rejects them with the exact CEL validation `message` defined in the markers.
- **E2E Testing:** E2E tests will run two variants: one with `CELValidation=false` (webhook) and one with `CELValidation=true` (native validation) to ensure identical behavior.
- **Rollback / Version Skew:** Add a regression case for disabling `CELValidation` after it was enabled, and for clusters below v1.28 falling back to the legacy webhook without error.

### Upgrade / Downgrade Strategy
This is a purely opt-in feature initially. The operator manages the rollout sequence as follows:

**Upgrading (Enabling `CELValidation`):**
1. The operator detects the `CELValidation` gate is enabled.
2. The operator deploys the necessary `ValidatingAdmissionPolicy` and `ValidatingAdmissionPolicyBinding` objects.
3. The operator updates the `ValidatingWebhookConfiguration` to remove the rules that are now covered by VAP/CEL (or skips deploying the validating webhook entirely if everything is covered).

**Downgrading (Disabling `CELValidation`):**
1. If a user experiences issues, they disable the `CELValidation` feature gate.
2. The operator restores the full `ValidatingWebhookConfiguration` pointing back to the legacy Go webhook pods.
3. The operator deletes both the `ValidatingAdmissionPolicy` and the `ValidatingAdmissionPolicyBinding` resources.

### Version Skew Strategy
`ValidatingAdmissionPolicy` requires Kubernetes v1.28+. 
The controller will verify the Kubernetes API server version before deploying `ValidatingAdmissionPolicy` resources. If the cluster is older than v1.28, the feature gate will log a warning and fall back to registering the legacy webhook, ensuring backwards compatibility for older OCM deployments.

## Alternatives
**Alternative 1: Maintain Webhooks Indefinitely**
We could avoid CEL validation entirely and stick to Go webhooks. This was rejected because it ignores modern Kubernetes best practices, forces operators to endure unnecessary API latency, and retains the operational burden of managing webhook TLS configurations.
