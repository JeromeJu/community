---
status: implementable
title: Decouple api and feature versioning
creation-date: '2023-07-07'
last-updated: '2023-08-16'
authors:
- '@JeromeJu'
- '@chitrangpatel'
- '@lbernick'
---

# TEP-0138: Decouple API and Feature Versioning

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [Use Cases](#use-cases)
- [Requirements](#requirements)
- [Proposal](#proposal)
- [Design Details](#design-details)
  - [New value for `enable-api-fields`](#introduce-new-stable-value-for-enable-api-fields)
  - [Immediate decoupling solution](#provide-an-immediate-solution-to-decouple-api-and-feature-versioning-now)
  - [Migrate default to `stable`](#migrate-enable-api-fields-to-stable-in-9-months)
  - [Pros and cons](#pros-and-cons)
- [Alternatives](#alternatives)
  - [Per feature flag with new value for enable-api-fields `none`](#per-feature-flag-with-new-value-for-enable-api-fields-none) 
  - [New `legacy-enable-beta-features-by-default` flag](#new-legacy-enable-beta-features-by-default-flag)
  - [Make `beta` feature validation changes now](#make-beta-feature-validation-changes-now-migrate-enable-api-fields-to-stable-in-9-months)
  - [New`legacy-stable` value for `enable-api-fields`](#new-legacy-stable-value-for-enable-api-fields-migrate-enable-api-fields-to-stable-in-9-months)
  - [Make validation changes only for new beta features](#make-validation-changes-only-for-new-beta-features)
  - [Give 9-month warning before making v1beta1 validation changes and default to stable](#give-9-month-warning-before-making-v1beta1-validation-changes-and-default-to-stable)
  - [Wait until v1beta1 is removed to swap `enable-api-fields` back to `stable`](#wait-until-v1beta1-is-removed-to-swap-enable-api-fields-back-to-stable)
- [Implementation Plan](#implementation-plan)
  - [Test Plan](#test-plan)
- [Future Work](#future-work)
- [References](#references)
<!-- /toc -->

## Summary

This document proposes updating Tekton Pipelines' feature flags design, as originally proposed in [TEP-0033](https://github.com/tektoncd/community/blob/main/teps/0033-tekton-feature-gates.md), to decouple API versioning from feature versioning.

## Motivation

Per [TEP-0033](https://github.com/tektoncd/community/blob/main/teps/0033-tekton-feature-gates.md), the behavior of `enable-api-fields` depends on the CRD API version being used. In v1beta1 CRDs, `beta` features can be enabled by setting `enable-api-fields` to `beta` or to "`stable`", but in v1 CRDs, `beta` features can only be enabled by setting `enable-api-fields` to `beta`. This couples API versioning to feature stability, and has led to the following pain points:

- [Feedback indicates](https://github.com/tektoncd/pipeline/issues/6592#issuecomment-1533268522) that users upgrading their CRDs from v1beta1 to v1 were confused to find `beta` features that worked by default in v1beta1 did not work by default in v1 when `enable-api-fields` was set to "`stable`" (its default value). This is especially confusing for users who are not cluster operators and cannot control the value of `enable-api-fields`, especially if they are not aware they are using `beta` features.

- For maintainers, the maintenance operation of swapping the storage version from v1beta1 to v1 should not have affected our users. However, we had to [change the user-facing default value of enable-api-fields from `stable` to `beta` ](https://github.com/tektoncd/pipeline/pull/6732) before changing the storage version of the API to [avoid breaking PipelineRuns using `beta` features](https://github.com/tektoncd/pipeline/pull/6444#issuecomment-1580926707).

- When promoting features, it could cause confusions for contributors to be dependent on the fact whether an apiVersion is available. For example, during [the promotion to beta for projected workspaces](https://github.com/tektoncd/pipeline/pull/5530), the `v1` api's existence led to confusions of what to do with `beta` features in `v1beta1` and its difference with in `v1`.

## Goals

- Feature validations and implementation should be independent from any API version.
- Come up with a plan that makes the migration easier for setting feature flags to enable stable features only by default in the long term.
- Changes and updates made to the existent feature validaitons regarding decoupling api and feature versioning should keep much backwards compatiblity as possible.

## Non goals

- Better guidance on feature promotion and when features can be promoted
  - This is a nice-to-have but not necessarily a blocker, since the feature graduating process should not affect the implementation of how features are enabled.
- Ensure pending resources don't break with changing feature flags on downgrades or upgrades
  - As [handling backwards incompatible changes for pending resources](https://github.com/tektoncd/pipeline/issues/6479) pointed out, we have run into the cases where [feature flag info are changed or lost](https://github.com/tektoncd/pipeline/issues/5999) when handling deprecated fields which led the pending resources to break. However, this issue was introduced by the implementation of feature flags rather than its design, and can be addressed separately.
  - Users can downgrade their pipeline versions without invalidating stored resources, even if stored resources cannot be run with the downgraded server. Keeping the stored resources valid relates with the storage migration instead of our feature flags implementations, which has been covered in [Storage version migrator v1beta1 -> v1](https://github.com/tektoncd/pipeline/issues/6667) and is out of scope.

### Use Cases

**End Users**
- As a users who has newly adopted Tekton:
  - I want to have a consistent and easily understandable feature flag UX.

- As an end user currently on `v1beta1`:
  - I want to migrate to `v1` and have as seamless of an experience as possible.

**Cluster Operators**
- As a cluster operator whose most users are on `v1beta1`:
  - I want to control the features that my users use and have enough notice of any backwards incompatible changes going to be made in the Tekton pipeline releases.
  - When migrating to `v1`, I may want my users to keep using `stable` opt-in features that have already been turned on by default in `v1beta1`.

- As a cluster operator with users who have migrated to `v1`:
  - I would like to get notice of the plan for the breaking change if `enable-api-fields` is going to be changed to `stable` in the future.

- As a cluster operator who accept default values of all pipeline releases:
  - I want minimal changes to the configs to keep the same set of features for my users.
  - When there is a breaking change, I would like to have workarounds to keep the existing set of features.

**Tekton Maintainers**
- I would like to be able to migrate the apiVersion without having to make backwards incompatible changes.

## Requirements
- It should not require all existing features to get promoted to `beta` or `stable` to decouple the existing coupling of feature and api versioning.
- It should not block the promotion of `beta` features from the existing `alpha` features.
- It should have a testing stratey that will give us confidence in our impelmentations.

## Proposal
The solutions and alternatives aim to decouple api and feature versioning, and will provide a migration plan for setting the feature flags to enable stable features only by default in the long term. 

Note that for the existent api driven feature flag `enable-api-fields`, we propose to sunset it after all exisiting beta and alpha features stabilize to avoid the breaking change defaulting it back to `stable` while there are some alternatives suggesting that we will change back to `stable` with the breaking change.

## Design Details

### Change existent validation to decouple feature and api versioning
We will change the current validation for `enable-api-fields=stable` to only allow using stable features regardless of api version. This will resolve the current issue of the coupling of api and feature versioning in v1beta1. More specifically, beta features resolvers, object/ array params and results will require `enable-api-fields` set to `beta` to be used.

Note that although this is a behavior change, it is more of a bug fix for the coupling of feature and api versioning. Currently, with `enable-api-fields` set to `stable`,  PipelineRuns like this one fail because the controller cannot create child TaskRuns. This change will result in a validation failure instead.

- **Impacts on users:**
    - Cluster operators: 
    This makes it possible for cluster operators to have full control over only stable feature usages, rather than user overrides. For example, currently if cluster operators want their users to only opt-in “stable” features, they cannot do so for v1beta1 apiVersion for the exception of resolver, object params and results.
    - Pipeline and Task authors: 
    This affects pipeline users in the situation where cluster operators have chosen to set `enable-api-fields` to `stable` (i.e. those who have changed the current default `beta` value) and who are accidentally using beta features, like resolvers, in v1beta1.

### Per feature flag for _new_ api-driven features
Introduce per-feature flags for each **new** api driven feature. Each feature will have its own flag, instead of using the group api driven flag `enable-api-fields` to enable or disable all features of a stability level. Note that behavioural flags will remain the existent beahviour.

The flag will also include the maintainer information and stability level for the feature as the source of truth. New behavioural features will also have a new per-feature flag to either enable or disable the feature. This will allow behavioural features that has values leading to behaviors in different stability levels to be turned on or off instead of depending on the stability level of features that are enabled.

For example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
data:
  # <Source of truth>
  # <Stability level>: since <release version>
  #
  # <Description of feature>
  enable-new-template-feature-0: "true"
  # owner: tektonCD@contributor0
  # alpha: v0.41
  # beta: v0.50
  # 
  # Enables new feature-1 that has foo functionalities. 
  enable-new-api-feature-1: "true"
```

See [implementation plan](#implementation-plan) for more details on the PerFeatureFlag struct.

All **new** features can only be enabled via per-feature flags. When they first get introduced as alpha, they will be disabled by default. When new features get promoted to stable, they will be enabled by default according to the following table:

| Feature stability level | Default  |
| ----------------------- | -------- |
| Stable                  | Enabled  |
| Beta                    | Disabled |
| Alpha                   | Disabled |

The behaviour of existing `enable-api-fields` flag with per feature flag:
- Any current beta features can be enabled with `enable-api-fields` set to “beta” or “alpha”.
- Any current alpha features can be enabled with `enable-api-fields` set to “alpha”. When current alpha features are promoted to beta, they can be enabled with `enable-api-fields` set to “beta”.
- It will not be possible to enable existing features using per-feature flags.
  - We cannot enable existing features using per-feature flags because this would not be backwards compatible. If we allowed this, the individual flag would have precedence over the grouped flag, which means that to preserve backwards compatibility, the individual flag would need to be on by default for beta features and off by default for alpha features. However, this would not be backwards compatible for cluster operators who set `enable-api-fields` to `stable`, since they would also need to override the new `beta` level per feature flags. 

- **Cluster operators perspective:** For new features, cluster operators will explicitly turn on or off each features in the ConfigMap. They will be able to choose to turn on a single feature.

- **End users perspective:** End users will get to know the list of features that are turned on from their service providers.

### Sunset `enable-api-fields` after existent features have stabilized
When all existing alpha and beta features have either been stabilized or removed, we will be able to remove the `enable-api-fields` flag.

Existent beta and alpha features as of today:
| Feature                                                                                               | Stability level | Individual flag                                 |
| ----------------------------------------------------------------------------------------------------- | --------------- | ----------------------------------------------- |
| [Array Results and Array Indexing](pipelineruns.md#specifying-parameters)                             | beta            |                                                 |
| [Object Parameters and Results](pipelineruns.md#specifying-parameters)                                | beta            |                                                 |
| [Remote Tasks](./taskruns.md#remote-tasks) and [Remote Pipelines](./pipelineruns.md#remote-pipelines) | beta            |                                                 |
| [`Provenance` field in Status](pipeline-api.md#provenance)                                            | beta            | `enable-provenance-in-status`                   |
| [Isolated `Step` & `Sidecar` `Workspaces`](./workspaces.md#isolated-workspaces)                       | beta            |                                                 |
| [Bundles ](./pipelineruns.md#tekton-bundles)                                                          | alpha           | `enable-tekton-oci-bundles`                     |
| [Hermetic Execution Mode](./hermetic.md)                                                              | alpha           |                                                 |
| [Windows Scripts](./tasks.md#windows-scripts)                                                         | alpha           |                                                 |
| [Debug](./debug.md)                                                                                   | alpha           |                                                 |
| [Step and Sidecar Overrides](./taskruns.md#overriding-task-steps-and-sidecars)                        | alpha           |                                                 |
| [Matrix](./matrix.md)                                                                                 | alpha           |                                                 |
| [Task-level Resource Requirements](compute-resources.md#task-level-compute-resources-configuration)   | alpha           |                                                 |
| [Trusted Resources](./trusted-resources.md)                                                           | alpha           | `trusted-resources-verification-no-match-policy`|
| [Larger Results via Sidecar Logs](#enabling-larger-results-using-sidecar-logs)                        | alpha           | `results-from`                                  |
| [Configure Default Resolver](./resolution.md#configuring-built-in-resolvers)                          | alpha           |                                                 |
| [Coschedule](./affinityassistants.md)                                                                 | alpha           | `coschedule`                                    |

#### Example use cases:
- A single feature "foo" being introduced in `v1`:
  We will add a new feature flag "enable-foo" to the configMap, it will have the `alpha` stability level as a new feature and will be disabled by default.
  - Cluster operators will now be able to 
  - Tekton Pipeline authors will get to know the new feature being turned on 
- Two more features are  being introduced while the feature introduced in (1) is alpha
  Two more features will have their own flags

## Design Evalutaion

#### Pros
- This is backwards compatible.
- Cluster operators can more granular control over features to be turned on. Previously they can only have features of a certain stability level all on or off, but now they can enable individual alpha or beta features controlled by API fields.
- Unblocks the internal version work where the validations in the internal version does not depend on apiVersion, which requires the decoupling of feature and api versioning.
- Improves consistency among existing features enabled with `enable-api-fields` set to `beta`, since the existent beta feature isolated step and sidecar workspaces(since v0.50) is validated differently.
- The validations for per feature flag will have clear source of truth of feature levels and traceability and there will not be coupling in the future.

#### Cons
- This is adding complexity to both the implementations and the testing matrix for the newly introduced flag.

## Alternatives
### Per feature flag with new value for enable-api-fields `none`
Introduce a new value `none` for `enable-api-fields` and it must be used for per feature flags for existing features. All new features enabled via per feature flags will be off by default, regardless of the value of “enable-api-fields”.
- Any current beta features can be enabled with “enable-api-fields” set to “beta” or “alpha”.
- If “enable-api-fields” is set to “none”, you can also enable them with per-feature flags. These are off by default.
- Any current alpha features can be enabled with “enable-api-fields” set to “alpha”.
- If “enable-api-fields” is set to “none”, you can also enable them with per-feature flags. These are off by default.
When promoting an alpha feature to beta, it can be enabled with enable-api-fields set to “alpha”, “beta”, or “none”.
Disallowing it when enable-api-fields is set to “beta” wouldn’t actually help us phase out the flag more quickly, as we’d still need to wait until the feature is stabilized or removed.

### Introduce “new-stable” value for enable-api-fields; migrate `enable-api-fields` to `stable` in 9 months

- A new option, `enable-api-fields` = `new-stable`, will be added to the API. This option will use the preferred validation where `stable` and `beta` features are validated the same across apiVersions. In 9 months, `new-stable` will be renamed as `stable`.
- The current behavior for “stable” `enable-api-fields` remains the same for now where it allows users to use `beta` features in `v1beta1` that have coupled feature and api versioning in `v1beta1` i.e. remote resolution. In 9 months “stable” will be renamed as “legacy-stable” and be marked as deprecated.
- The existing `beta` option will remain the same for now and in 9 months.

- Taking the remote resolution feature, which currently couples feature and api versioning, as an example, after the change:
  - With `new-stable` in `v1beta1`, we disable resolver.
  - With `legacy-stable` in `v1beta1`, we do not validate resolvers as a field needs `enable-api-fields` as beta, so they are still turned on by default.
  - When `enable-api-fields` is set to `beta` in both `v1` and `v1beta1`, we are turning on resolver.

This solution proposes that we provide an immediate solution to address the current coupling issue. We are adding a new value to turn on stable features only with the `new-stable` `enable-api-fields` and require `enable-api-fields` to be set to `beta` when using beta features, regardless of CRD API version. This will solve the unintended behavior that taskruns cannot be created for version coupled features with `enable-api-fields` set to “stable” during the [v1 storage swap](https://github.com/tektoncd/pipeline/pull/6444#issuecomment-1580926707) right away.

The default value for `enable-api-fields` will be switched back to `stable` in the long run, as discussed in [issue #6948](https://github.com/tektoncd/pipeline/issues/6948). The `new-stable` flag introduced will be renamed to `stable` in 9 months, at which point it will become the default value. And the current `stable` behavior will be renamed to `legacy-stable` for backwards compatibility. The `legacy-stable` flag will be deprecated and then removed when `v1beta1` is removed or once all current `beta` features that are coupled(object params and results, array results and indexing and remote resolution) become `stable`, whichever happens first.

### Pros
- The proposed solution provides immediate a new option for cluster operators who want to restrict usage for only `stable` features, regardless of what apiversion their users have defined in their crds. This helps to prevent adding more complexity to the existing validation if more `beta` features are being added to `v1beta1`.
- The migration of `enable-api-fields` back to `stable` is backwards incompatible. However, the option for a easier migration of the defaulting `enable-api-fields` back to `stable`. For example, users that accidentally have `beta` features opt-in with `v1beta1` CRDs could migrate to the usage with `legacy-stable` validation.

### Cons
- The introduced new flag will add complexity to the testing matrix. We will also need to introduce tests for `new-stable`, and to keep testing `beta` and `legacy-stable` in 9 months.

### New `enable-api-fields-new` flag
This alternative proposes introducing a new flag `enable-api-fields-new` that validates new alpha and beta features and leave the existing flag `enable-api-fields` as is for existing alpha/beta features. Once all existing beta/alpha features become stable or v1beta1 apiVersion is removed, we could remove the existing `enable-api-fields`.

#### Pros
- This has more consistency of usage for existing v1beta1 api users.

#### Cons
- This is adding complexity to both the implementations and the testing matrix for the newly introduced flag.

### New `legacy-enable-beta-features-by-default` flag
This alternative proposes introducing a new flag `legacy-enable-beta-features-by-default ` that takes boolean value and it would be phased out in 9 months. The existing `enable-api-fields` will continue applying to existing beta features when the flag `legacy-enable-beta-features-by-default` is `true`. 

This chart would apply to the existing beta features (array results, array indexing, object params and results, and remote resolution):

| enable-api-fields |	legacy-enable-beta-features-by-default | enabled in v1beta1? | enabled in v1? |
| ------ | ------ | --- | --- |
| beta   |	true  |	yes |	yes |
| beta   |	false |	yes	| yes |
| stable |	true  |	yes	| no  |
| stable |	false |	no	| no  |

For new beta features(e.g. matrix in the future):

| enable-api-fields |	legacy-enable-beta-features-by-default | enabled in v1beta1? | enabled in v1? |
| ------ | ------ | --- | --- |
| beta   |	true  |	yes |	yes |
| beta   |	false |	yes |	yes |
| stable |	true  |	no  |	no  |
| stable |	false |	no  |	no  |

Once all existing beta features become stable, `legacy-enable-beta-features-by-default` can be removed and we will deprecate and then remove `legacy-enable-beta-features-by-default` and to use `stable` `enable-api-fields`. We would default `true` for the new flag and after 9 months, we default `enable-api-fields` to `stable` to preserve the existing behavior.

#### Pros
- This will provide a smoother transition for switching default `enable-api-fields` value back to `stable`.

#### Cons
- It is not clear what to do with alpha features for the feature promotion process when there are two `enable-api-fields` related flags, which might lead to confusions.
- This is adding complexity to both the implementations and the testing matrix for the newly introduced flag.

### Make `beta` feature validation changes now; migrate `enable-api-fields` to `stable` in 9 months
Require `enable-api-fields` to be set to `beta` when using beta features, regardless of CRD API version. Immediately make this change for existing beta features. This will solve the unintended behavior that taskruns cannot be created for version coupled features with `enable-api-fields` set to `stable` during the `v1` storage swap right away.

For the default value of `enable-api-fields` in the long run, this alternative will give a nine month notice for users for the breaking change setting the default to `stable`.

#### Pros
- This solution could provide best clarity of feature stability levels across different apiVersions by keeping validation for beta features the same in `v1` and `v1beta1`.

#### Cons
- Updating existing beta features valiations in `v1beta1` is a backwards incompatible change. This will break users who are accidentally using the coupled beta features while `enable-api-fields` is set to stable.

### New `legacy-stable` value for `enable-api-fields`; migrate `enable-api-fields` to `stable` in 9 months
Change the validation for beta features when `enable-api-fields` is set to `stable` to validate only stable features across apiVersions. Add a `legacy-stable` value to the `enable-api-fields` to keep the behavior of the current `stable` feature validations in `v1beta1` that includes the coupled features. 

For the default value of `enable-api-fields` in the long run, in 9 months, it will be switched back to `stable`, and the `beta` option will remain the same, with `legacy-stable` being removed.

#### Pros
- Users who want to opt-in beta features in `v1beta1` could migrate to `legacy-stable` after 9 months.
- No need to update the testing matrix for new values in 9 months.

#### Cons
- Adding complexity to the testing matrix, we will need to introduce tests for `legacy-stable` now.

### Make validation changes only for new beta features
This alternative proposes that we will keep the current implementations and validations for beta features in `v1` and `v1beta1` apiVersion and only make new features have the synchronized validations across different apiVersions.

#### Pros
- This is backwards compatible.
- Changes are minimal for the current codebase with probably only documentation required.
#### Cons
- We would still need to use `beta` as the default stability of features that are turned on. Some beta features are still coupled in feature and api versioning. This will not allow cluster operators to opt-in only stable features in v1beta1.

### Give 9-month warning before making v1beta1 validation changes and default to stable
This alternative proposes the validation change for beta features in v1beta1 to be done with giving a notice for the breaking change for 9 months.

#### Cons:
- Since v1beta1 only has 12 months of support period left, making a breaking change to it might have more impacts on v1beta1 users than its worth after 9 months.

### Wait until v1beta1 is removed to swap `enable-api-fields` back to `stable`
#### Pros
- Straightforward and no code changes need to be done.

#### Cons
- This is still a breaking change for v1 even when v1beta1 is removed.
- This does not provide cluster operators with options to only turn on stable features. Since our desired state of default features being turned on is stable, we should make the change as soon as possible for v1 otherwise there could be more features being promoted to beta and become enabled by default during the migration process.

## Implementation Plan

```go
type PerFeatureFlag struct {
    // Name of the feature flag
    Name string
    
    // Stability level of the feature, one of "stable", "beta" or "alpha"
    Stability string
    
    // Enabled is whether the feature is enabled
    Enabled bool
    
    // Deprecated is whether the feature is deprecated
    // +optional
    Deprecated bool
}
```

## Test Plan
- With the addition of per feature flags, we should have sets of end-to-end test for the expected behaviour for per feature flags along with the existent `enable-api-fields`.
    - Per feature flag with all beta feature enabled and others disabled with `enable-api-fields` set to `beta`. Similar for `alpha`.
    - Some per feature flags for beta features are enabled while others are disabled.
- For per feature flags, we will test out each single feature flag in end to end test and the combinations of features that could have overlapps. The integration tests for each flag will be in place for the CI while the combinations will be in the nightly tests.
    - Note that it is not feasible to exhuast the combinations of per feature flags which is going to be 2^n with n as the number of per feature flags we are going to introduce. The 
- For fields at `alpha`, `beta` and `stable` stability level, we will continue to use the existing testing matrix for end to end tests and examples. 
    - Add beta features enabled to `pull-tekton-pipeline-beta-integration-tests` and similar for `alpha` and `stable`.

## Future Work
- For seeking better way of communicating enabled features from cluster operators and downstream users including service providers and pipeline authors. CLI feature request to query all enabled features at each stability level via tkn.
    - Example output:
      ```
      $: 'tkn list enabled-features'
      stable: <feature-0>
      beta: <feature-1>, <feature-2>
      alpha: <feature-3>
      ```
- For preserving the usage of the group flag, we could provide some script that turns on all alpha/beta features by modifying the group of per feature flags.

## References

- [TEP-0033](https://github.com/tektoncd/community/blo9b/main/teps/0033-tekton-feature-gates.md)
- [Decoupling API versioning and Feature versioning for features turned on by default](https://github.com/tektoncd/pipeline/issues/6592)
- [Versioned validation of referenced Pipelines/Tasks](https://github.com/tektoncd/pipeline/issues/6616)
- [Default enable-api-fields value for opt-in features once feature and API versioning are decoupled](https://github.com/tektoncd/pipeline/issues/6948)
