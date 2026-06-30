# Crossplane v2 Migration The Hard Way

This guide stands up a Crossplane v1 control plane managing real AWS resources, then migrates it to
Crossplane v2 "[The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)", with
limited automation and tooling. The goal is to understand the actual steps and mechanics of a full
v2 migration, so you can run one yourself and know exactly what each step does and why.

A Crossplane control plane can be upgraded to v2 with *no manifest changes at all*: legacy claims,
XRs, and managed resources keep working untouched. This guide deliberately goes further. It migrates
the user-facing API from claims to namespaced XRs, and the composed managed resources (MRs) from
cluster-scoped to namespaced, doing it as an in-place adoption that never recreates the live AWS
resources. That cutover is where all the effort goes, and it's most of what you'll do here.

The guide has two parts: first you stand up a working v1 control plane, then you migrate it to v2.

## Set up the v1 control plane

You'll build a small but realistic stack: an XRD and Composition that provision a VPC, three subnets
(one per availability zone), and a security group, driven by a single claim.

Create a v1 control plane:

```bash
kind create cluster
```

Install Crossplane `v1.20`:

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm upgrade --install crossplane crossplane-stable/crossplane \
    --namespace crossplane-system --create-namespace \
    --version 1.20.10 \
    --wait
```

Install functions and the AWS provider:

```bash
kubectl apply -f functions.yaml
kubectl apply -f v1/providers.yaml
```

Make sure all packages become healthy:

```bash
kubectl get pkg
```

Configure AWS credentials and the ProviderConfig:

```bash
AWS_PROFILE=default && echo -e "[default]\naws_access_key_id = $(aws configure get aws_access_key_id --profile $AWS_PROFILE)\naws_secret_access_key = $(aws configure get aws_secret_access_key --profile $AWS_PROFILE)" > aws-credentials.txt

kubectl create secret generic aws-secret -n crossplane-system --from-file=creds=./aws-credentials.txt

kubectl apply -f v1/providerconfig.yaml
```

Apply the XRD and Composition:

```bash
kubectl apply -f v1/xrd.yaml
kubectl apply -f v1/composition.yaml
```

Apply the claim, then trace the stack it composes:

```bash
kubectl apply -f v1/claim.yaml
```

```bash
crossplane resource trace networkingstack.example.crossplane.io/cool-network
```

Wait for every resource in the trace to reach `READY` and `SYNCED` `True`. AWS provisioning takes a
minute or two. Once it's all green, you have a working v1 control plane to migrate.

## Migrate to v2

The migration happens in three phases:

- **Prepare and validate locally.** Update your compositions and XRs to v2 namespaced resources,
  render-test them, and run the upgrade check. Nothing touches your running control plane yet, so
  this is all safe to explore.
- **Upgrade the platform.** Upgrade the Crossplane core, activate the namespaced MR kinds, then
  upgrade the provider. The v1 stack keeps reconciling untouched the whole time.
- **Adopt and cut over.** Point the v2 stack at the existing AWS resources, retire v1, and promote
  v2 to full management, with no recreation of the live infrastructure.

### Update your compositions and XRs for v2

Now update your compositions and XRs to use v2-style namespaced resources, along with the XRD,
provider, and ProviderConfig that back them. The tables below are the changes you'll make to your own
code; the finished versions live in `v2/` to apply and compare against. Each is a v1 -> v2 diff, with
the reasoning that drives it.

#### XRD (`v1/xrd.yaml` -> `v2/xrd.yaml`)

| Field | v1 | v2 |
|---|---|---|
| `apiVersion` | `apiextensions.crossplane.io/v1` | `apiextensions.crossplane.io/v2` |
| `metadata.name` | `xnetworkingstacks.example.crossplane.io` | `networkingstacks.example.m.crossplane.io` |
| `spec.scope` | (not set) | `Namespaced` |
| `spec.group` | `example.crossplane.io` | `example.m.crossplane.io` |
| `spec.names.kind` | `XNetworkingStack` | `NetworkingStack` |
| `spec.names.plural` | `xnetworkingstacks` | `networkingstacks` |
| `spec.claimNames` | present | removed |

The `spec.group` rename dodges a Claim CRD name collision: the v1 Claim already owns
`networkingstacks.example.crossplane.io`, so the v2 XRD has to live under a different group.

#### Composition (`v1/composition.yaml` -> `v2/composition.yaml`)

| Field | v1 | v2 |
|---|---|---|
| `metadata.name` | `networkingstack` | `networkingstack-v2` |
| `spec.compositeTypeRef.apiVersion` | `example.crossplane.io/v1` | `example.m.crossplane.io/v1` |
| `spec.compositeTypeRef.kind` | `XNetworkingStack` | `NetworkingStack` |
| MR `apiVersion` (inline template) | `ec2.aws.upbound.io/v1beta1` | `ec2.aws.m.upbound.io/v1beta1` |
| MR `providerConfigRef` | (not set; relied on default) | `kind: ClusterProviderConfig`, `name: default` |
| SecurityGroup `forProvider.name` | `my-sg-{{ $xr.metadata.name }}` | (unset) |

A security group's name is immutable, and the v1 template derived it from the XR name. The v2 XR's
name differs from the claim-generated v1 one, so reusing that template would force a rename on
adoption, which the provider rejects. Leaving `name` unset lets the provider keep the existing name.

#### Claim becomes a namespaced XR (`v1/claim.yaml` -> `v2/xr.yaml`)

With a namespaced XR there is no Claim; the namespaced XR is the user-facing API now. We keep the
same kind name (`NetworkingStack`) and changed the group instead.

| Field | v1 (`claim.yaml`) | v2 (`xr.yaml`) |
|---|---|---|
| `apiVersion` | `example.crossplane.io/v1` | `example.m.crossplane.io/v1` |
| `metadata.namespace` | (not set) | `default` |

#### Provider package (`v1/providers.yaml` -> `v2/providers.yaml`)

Same provider, bumped to a v2 release that serves both cluster-scoped and namespaced MR CRDs out of
one package.

| Field | v1 | v2 |
|---|---|---|
| `spec.package` | `provider-aws-ec2:v1.23.0` | `provider-aws-ec2:v2.5.0` |

#### ProviderConfig (`v1/providerconfig.yaml` -> `v2/providerconfig.yaml`)

v2 namespaced MRs use a `ClusterProviderConfig` (cluster-scoped, under the `.m.` group) instead of
the legacy `ProviderConfig`.

| File | `apiVersion` | `kind` | When applied |
|---|---|---|---|
| `v1/providerconfig.yaml` | `aws.upbound.io/v1beta1` | `ProviderConfig` | during v1 setup |
| `v2/providerconfig.yaml` | `aws.m.upbound.io/v1beta1` | `ClusterProviderConfig` | along with the v2 provider upgrade |

### Test locally with `crossplane render`

`crossplane render` runs the composition pipeline locally without touching a cluster, so you can
build confidence in your updated compositions and XRs before applying anything. Two ways to use it.

Confirm v2 renders against the XRD schema and emits the expected MRs:

```bash
crossplane render v2/xr.yaml v2/composition.yaml functions.yaml --xrd v2/xrd.yaml --include-full-xr
```

Diff a v2 render against a v1 render to spot any unintended semantic changes:

```bash
crossplane render v1/xr.yaml v1/composition.yaml functions.yaml --include-full-xr > /tmp/v1.yaml
crossplane render v2/xr.yaml v2/composition.yaml functions.yaml --include-full-xr > /tmp/v2.yaml
diff /tmp/v1.yaml /tmp/v2.yaml
```

Everything in the diff should fall into one of these categories. Anything outside them is suspicious:

- XR
  - `apiVersion`, `kind`, and (on v2 only) `metadata.namespace: default`
- composed MRs
  - `apiVersion`: `ec2.aws.upbound.io` -> `ec2.aws.m.upbound.io`
  - `metadata.namespace: default` on v2 (render exercises v2's namespace propagation from XR to children)
  - `spec.providerConfigRef` block on v2 only (`kind: ClusterProviderConfig`, `name: default`; v1 composition omits it and relies on the legacy default)
  - SecurityGroup `spec.forProvider.name` (`my-sg-cool-network`) on v1 only; v2 leaves it unset so adoption keeps the existing name (see the Composition table above)
  - `ownerReferences[0]` apiVersion + kind (follows the XR)
  - generated name hash suffixes and ownerReference UIDs

### Run the pre-upgrade check

Download the `v1.20` `crossplane` CLI and run its `upgrade check` to verify the v1 control plane is
ready to safely upgrade to v2:

```bash
curl -sL "https://raw.githubusercontent.com/crossplane/crossplane/main/install.sh" | XP_VERSION=v1.20.10 sh
```

```bash
./crossplane beta upgrade check
```

Any flagged issues need to be addressed before moving on to the upgrade.

### Upgrade the Crossplane core to v2

Upgrade the core with `--set provider.defaultActivations={}` to skip the default MRAP that would
activate every MRD for every provider. We'll apply our own targeted MRAP in the next step instead.

```bash
helm upgrade crossplane crossplane-stable/crossplane \
  --namespace crossplane-system --version 2.0.8 \
  --set provider.defaultActivations={} \
  --wait
```

Sanity-check that the v1 workloads are still healthy. The v1 XRD becomes `LegacyCluster` scope
internally and keeps working through backwards compatibility.

```bash
kubectl get pods -n crossplane-system
```

```bash
crossplane resource trace networkingstack.example.crossplane.io/cool-network
```

Nothing should have changed: the pods reach `Running` and the trace is still all `Ready`/`Synced`.

### Activate the namespaced managed resources

In v2, a managed resource kind must be activated by a `ManagedResourceActivationPolicy` (MRAP) before
the provider installs its CRD. Apply our targeted MRAP, which activates just the three namespaced EC2
kinds this composition uses:

```bash
kubectl apply -f v2/mrap.yaml
kubectl get mrap
```

`v2/mrap.yaml` activates:

- `vpcs.ec2.aws.m.upbound.io`
- `subnets.ec2.aws.m.upbound.io`
- `securitygroups.ec2.aws.m.upbound.io`

We don't need to activate the v1 cluster-scoped kinds here. MRD activation is one-way, and they were
already considered "active" at v2 upgrade time.

### Upgrade the provider

Upgrade the provider to a v2 version so the namespaced MRs get installed:

```bash
kubectl apply -f v2/providers.yaml
```

Wait for the new provider version to become healthy:

```bash
kubectl get pkg
```

Once the provider reports healthy, apply the `ClusterProviderConfig` the namespaced MRs will use:

```bash
kubectl apply -f v2/providerconfig.yaml
```

Sanity-check that the v1 stack is still reconciling under the new v2 provider:

```bash
crossplane resource trace networkingstack.example.crossplane.io/cool-network
```

Still all `Ready`/`Synced`: the v1 stack rides through the provider swap untouched.

Confirm the v2 namespaced MRs are now available:

```bash
kubectl get crd | grep ec2.aws.m.upbound.io
```

### Capture the original external-names

From here we adopt the existing AWS resources, then cut over to v2. Applying the v2
XRD/composition/XR as-is would make the provider create a *second* set of resources, since it doesn't
know the v1 ones exist; instead we point each v2 MR at its existing cloud resource via its
`crossplane.io/external-name` before the provider can create anything. We do it by hand, matching v1
to v2 MRs by the `crossplane.io/composition-resource-name` annotation, which is consistent across
both compositions.

Start by capturing each v1 MR's `crossplane.io/external-name` (the id of the AWS resource it manages)
into a variable, so the later steps can apply them to the v2 resources. Keep this shell open through
the adoption steps that follow; they reuse these variables:

```bash
VPC=$(kubectl get managed -l crossplane.io/claim-name=cool-network \
  -o jsonpath="{.items[?(@.metadata.annotations.crossplane\.io/composition-resource-name=='vpc')].metadata.annotations.crossplane\.io/external-name}")

SUBNET_AZA=$(kubectl get managed -l crossplane.io/claim-name=cool-network \
  -o jsonpath="{.items[?(@.metadata.annotations.crossplane\.io/composition-resource-name=='subnetAZA')].metadata.annotations.crossplane\.io/external-name}")

SUBNET_AZB=$(kubectl get managed -l crossplane.io/claim-name=cool-network \
  -o jsonpath="{.items[?(@.metadata.annotations.crossplane\.io/composition-resource-name=='subnetAZB')].metadata.annotations.crossplane\.io/external-name}")

SUBNET_AZC=$(kubectl get managed -l crossplane.io/claim-name=cool-network \
  -o jsonpath="{.items[?(@.metadata.annotations.crossplane\.io/composition-resource-name=='subnetAZC')].metadata.annotations.crossplane\.io/external-name}")

SG=$(kubectl get managed -l crossplane.io/claim-name=cool-network \
  -o jsonpath="{.items[?(@.metadata.annotations.crossplane\.io/composition-resource-name=='securityGroup')].metadata.annotations.crossplane\.io/external-name}")
```

Sanity-check they're populated, and save the mapping for later in one go. Each line is `<role> <id>`;
a capture that missed shows up as a role with no id after it. We also save this list to a temp file for verifying the final migration state later.

```bash
printf '%s\n' "vpc $VPC" "subnetAZA $SUBNET_AZA" "subnetAZB $SUBNET_AZB" "subnetAZC $SUBNET_AZC" "securityGroup $SG" \
  | sort | tee /tmp/original-external-names.txt
```

### Set `deletionPolicy: Orphan` on the v1 MRs

Set `deletionPolicy: Orphan` on every v1 MR. This is what lets us delete the v1 claim later without
the provider deleting the underlying resources. We patch the live MRs directly; the v1 composition never sets
`deletionPolicy`, so it won't fight the patch.

```bash
for mr in $(kubectl get managed -l crossplane.io/claim-name=cool-network -o name); do
  kubectl patch "$mr" --type merge -p '{"spec":{"deletionPolicy":"Orphan"}}'
done

kubectl get managed -l crossplane.io/claim-name=cool-network \
  -o custom-columns='NAME:.metadata.name,DELETION:.spec.deletionPolicy'
```

Confirm every row shows `Orphan` before continuing.

### Bring up the v2 stack in Observe-only mode

`v2/composition.yaml` gates Observe mode on an annotation: when the XR carries
`example.crossplane.io/adopt: "true"`, every composed MR is rendered with
`managementPolicies: ["Observe"]`. Observe mode is read-only, so the MRs can never create a cloud
resource.

```bash
kubectl apply -f v2/xrd.yaml
kubectl apply -f v2/composition.yaml
kubectl apply -f v2/xr-adopt.yaml
```

```bash
kubectl get managed -n default -l crossplane.io/composite=cool-network
```

You'll see five MRs, each with an empty `EXTERNAL-NAME` and not Ready yet (nothing to observe until
we point them at the live resources). The adopt annotation made the composition render them `Observe`-only;
confirm on any one with `kubectl get <mr> -n default -o jsonpath='{.spec.managementPolicies}'`.
Because Observe is read-only, the provider can't create anything, so there's no duplicate
VPC/subnets/SG while we wire up the external-names.

### Adopt the existing resources (attach the external-names)

Now give each v2 MR the external-name of its existing AWS resource, and the provider adopts that
resource instead of creating a new one. We use `kubectl annotate` (out of band) on purpose: the
annotation is then owned by the `kubectl` field manager, not the composition, so promoting the stack
later won't strip it.

We do it one resource at a time, matching each MR by its `crossplane.io/composition-resource-name`
annotation (the same role key we captured by) and handing it the matching `$VPC` / `$SUBNET_AZ*` /
`$SG` id:

```bash
kubectl annotate vpc.ec2.aws.m.upbound.io -n default \
  $(kubectl get vpc.ec2.aws.m.upbound.io -n default \
    -o jsonpath="{.items[?(@.metadata.annotations.crossplane\.io/composition-resource-name=='vpc')].metadata.name}") \
  crossplane.io/external-name=$VPC --overwrite

kubectl annotate subnet.ec2.aws.m.upbound.io -n default \
  $(kubectl get subnet.ec2.aws.m.upbound.io -n default \
    -o jsonpath="{.items[?(@.metadata.annotations.crossplane\.io/composition-resource-name=='subnetAZA')].metadata.name}") \
  crossplane.io/external-name=$SUBNET_AZA --overwrite

kubectl annotate subnet.ec2.aws.m.upbound.io -n default \
  $(kubectl get subnet.ec2.aws.m.upbound.io -n default \
    -o jsonpath="{.items[?(@.metadata.annotations.crossplane\.io/composition-resource-name=='subnetAZB')].metadata.name}") \
  crossplane.io/external-name=$SUBNET_AZB --overwrite

kubectl annotate subnet.ec2.aws.m.upbound.io -n default \
  $(kubectl get subnet.ec2.aws.m.upbound.io -n default \
    -o jsonpath="{.items[?(@.metadata.annotations.crossplane\.io/composition-resource-name=='subnetAZC')].metadata.name}") \
  crossplane.io/external-name=$SUBNET_AZC --overwrite

kubectl annotate securitygroup.ec2.aws.m.upbound.io -n default \
  $(kubectl get securitygroup.ec2.aws.m.upbound.io -n default \
    -o jsonpath="{.items[?(@.metadata.annotations.crossplane\.io/composition-resource-name=='securityGroup')].metadata.name}") \
  crossplane.io/external-name=$SG --overwrite
```

Then confirm the adoption, look at all the MRs belonging to our v2 XR:

```bash
kubectl get managed -n default -l crossplane.io/composite=cool-network
```

Each MR's `EXTERNAL-NAME` should now be the ID you set, with `SYNCED` and `READY` both `True`.
`SYNCED=True` means the provider found and bound to the existing resource.

### Retire v1 and promote v2

Delete the v1 claim. It cascades to the v1 XR and its five MRs, but because they're orphaned the AWS
resources survive and only the Kubernetes objects go away.

```bash
kubectl delete -f v1/claim.yaml

kubectl get managed -l crossplane.io/claim-name=cool-network
```

The legacy MRs should now be gone, while the v2 MRs keep observing the (still present) cloud
resources. Promote the v2 stack to full management by applying the plain XR (`v2/xr.yaml`, which is
`v2/xr-adopt.yaml` minus the adopt annotation):

```bash
kubectl apply -f v2/xr.yaml
```

Without the annotation the composition re-renders the MRs without `managementPolicies`, reverting
them to the default `["*"]` (full management), and the external-names we set out of band persist.
The composition's `forProvider` should already match the live resources, so nothing should be
recreated or modified. The v2 MRs simply take over management of the existing infrastructure.

```bash
crossplane resource trace networkingstack.example.m.crossplane.io/cool-network -n default
```

The trace should show the v2 XR and all five MRs `Synced`/`Ready`, with the same ids you saw on the
v1 MRs. The live AWS resources are now fully managed by the namespaced v2 composition, with no
recreation along the way.

### Confirm the end state

Only the namespaced v2 MRs should be left. The `NAME` column shows each MR's full kind and group, so
every row should read `...ec2.aws.m.upbound.io/...` (no legacy `ec2.aws.upbound.io` rows), all
`Ready`, with `EXTERNAL-NAME` matching the ids you captured at the start:

```bash
kubectl get managed -A
```

Same cloud resources, nothing recreated. To prove that exactly rather than by eye, diff the live
mapping against the one you saved during capture:

```bash
kubectl get managed -n default -l crossplane.io/composite=cool-network \
  -o jsonpath='{range .items[*]}{.metadata.annotations.crossplane\.io/composition-resource-name}{" "}{.metadata.annotations.crossplane\.io/external-name}{"\n"}{end}' \
  | sort | diff /tmp/original-external-names.txt - && echo "exact match"
```

No output (just `exact match`) means every role still maps to its original id.

### Clean up the v1 definitions (optional)

Once you're confident in the v2 stack, remove the now-unused v1 XRD, composition, and ProviderConfig.
The provider package stays (the v2 release serves both the legacy and namespaced MR kinds).

```bash
kubectl delete -f v1/composition.yaml
kubectl delete -f v1/xrd.yaml
kubectl delete -f v1/providerconfig.yaml
```

## Tear down

When you're done, remove everything so you don't leave AWS resources running. Order matters: delete
the Crossplane resources first so the provider deletes the cloud resources, *then* delete the
cluster. Deleting the cluster first would orphan the live VPC, subnets, and security group in AWS.

Delete the v2 XR. It cascades to the five v2 MRs, which are fully managed (deletionPolicy defaults to
`Delete`), so the provider deletes the real AWS resources too. The command blocks on the XR's
finalizer until that finishes, so give it a few minutes; the VPC goes last, once its subnets and SG
are gone.

```bash
kubectl delete -f v2/xr.yaml
```

Confirm nothing is left, then delete the cluster:

```bash
kubectl get managed -A
kind delete cluster
```
