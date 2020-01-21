---
title: Generic Data Populators
authors:
  - "@bswartz"
owning-sig: sig-storage
participating-sigs:
  - sig-api-machinery
reviewers:
  - "@thockin"
  - "@saad-ali"
  - "@smarterclayton"
  - "@j-griffith"
approvers:
  - "@thockin"
  - "@saad-ali"
creation-date: 2019-12-03
last-updated: 2020-01-21
status: provisional
see-also:
replaces:
superseded-by:
---

# Generic Data Populators

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1](#vm-images)
    - [Story 2](#backuprestore)
  - [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Examples](#examples)
      - [Alpha -&gt; Beta Graduation](#alpha---beta-graduation)
      - [Beta -&gt; GA Graduation](#beta---ga-graduation)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Implementation History](#implementation-history)
<!-- /toc -->

## Release Signoff Checklist

**ACTION REQUIRED:** In order to merge code into a release, there must be an issue in [kubernetes/enhancements] referencing this KEP and targeting a release milestone **before [Enhancement Freeze](https://github.com/kubernetes/sig-release/tree/master/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core Kubernetes i.e., [kubernetes/kubernetes], we require the following Release Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These checklist items _must_ be updated for the enhancement to be released.

- [ ] kubernetes/enhancements issue in release milestone, which links to KEP (this should be a link to the KEP location in kubernetes/enhancements, not the initial KEP PR)
- [ ] KEP approvers have set the KEP status to `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] Graduation criteria is in place
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

**Note:** Any PRs to move a KEP to `implementable` or significant changes once it is marked `implementable` should be approved by each of the KEP approvers. If any of those approvers is no longer appropriate than changes to that list should be approved by the remaining approvers and/or the owning SIG (or SIG-arch for cross cutting KEPs).

**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://github.com/kubernetes/enhancements/issues
[kubernetes/kubernetes]: https://github.com/kubernetes/kubernetes
[kubernetes/website]: https://github.com/kubernetes/website

## Summary

In Kubernetes 1.12, we added the `DataSource` field to the PVC spec. The field was implemented
as a `TypedLocalObjectReference` to give flexibility in the future about what objects could be
data sources for new volumes.

Since then, we have allowed only two things to be the source of a
new volume -- existing PVCs (indicating a user's intent to clone the volume) and snapshots
(indicating the user's intent to restore a snapshot). Implementation of these two data sources
relies on a CSI plugin to do the actual work, and no generic implementation exists for cloning
volumes or restoring snapshots.

Since the original design of the `DataSource` API field, we have been aware of the desire to
populate volumes with data from other sources, both CSI-specific and generic (compatible with
any CSI plugin). For new CSI-specific data sources, the path forward is clear, but for other
sources of data, which I call "Generic Data Populators" we don't have a mechanism. The main
problem is that current API validation logic uses a white list of object types, driven by
feature gates for each object type. This approach won't scale to generic populators, which
will by their nature be too numerous and varied.

This proposal recommends that we relax validation on the `DataSource` field to allow
arbitrary object types to be data sources, and leave the implementation to controllers. It
includes example implementations to show why this is a safe thing to do.

## Motivation

### Goals

- Enable users to create pre-populated volumes in a manner consistent with current practice
- Enable developers to innovate with new an interesting data sources
- Avoid changing existing workflows and breaking existing tools with new behavior

### Non-Goals

- In the past it was suggested that we might implement volume taints/tolerations as
  a mechanism to facilitate the populator pod attaching to the volume before an
  end-user pod could do so. This proposal doesn't require that.
- In the past it was suggested that Kubernetes might implement an end-user
  accessible interface for rebinding PVC and PVs -- in particular for swapping
  the bindings between a pair of bound PVCs. This proposal instead relies on
  direct modification of the PV to do rebinding. We do not think end-users
  need a way to rebind PVCs, because it's too hard to do safely in the general
  case.
- We do not propose any new object types. Specifically, no special "populator"
  object type. Populators should function based on their own object types.

## Proposal

The validation for `DataSource` field should be removed entirely, in the fullness of time.

Short term, we should add a new alpha feature gate, which allows arbitrary objects to be
specified for the `DataSource` field, with the intention to eventually make this behavior
the standard. 

Populators will work by responding to PVC objects with a data source they understand,
and producing a PV with the expected data, such that ordinary Kubernetes workflows are
not disrupted. In particular the PVC should be attachable to the end user's pod the
moment it is bound, similar to a PVC created from currently supported data sources.

### User Stories

There are a long list of possible use cases around generic data populators, and I won't
try to detail them all here. I will detail a few that illustrate the challenges faced by
users and developers, but it's important to see these as a few examples among many.

#### VM Images

One use case was relevant to KubeVirt, where the Kubernetes `Pod` object is actually a
VM running in a hypervisor, and the `PVC` is actually a virtual disk attached to a VM.
It's common for virtualization systems to allows VMs to boot from disks, and for disks
to be pre-populated with various OS images. OS images tend to be stored in external
repositories dedicated to that purpose, often with various mechanisms for retrieving
them efficiently that are external to Kubernetes.

One way to achieve this is to represent disk images as custom resources that point to
the image repository, and to allow creation of PVCs from these custom resources such
that the volumes come pre-populated with the correct data. Efficient population of the
data could be left up to a purpose-built controller that knows how to get the bits
where they need to be with minimal I/O.

#### Backup/Restore

Without getting into the details of how backup/restore should be implemented, it's
clear that whatever design one chooses, a necessary step is to have the user
(or higher level controller) create a PVC that points to the backup they want to
restore, and have the data appear in that volume somehow.

One can imagine backups simply being a special case of snapshots, in which case the
existing design is sufficient, but if you want anything more sophisticated, there
will inevitably be a new object type that represents a backup. While it's arguable
that backup should be something CSI plugins should be exclusively responsible for,
one can also argue that generic backup tools should also exist which can backup
and restore all kind of volumes. Those tools will be apart from CSI plugins and
yet need a way to populate volumes with restored volumes.

It's also likely that multiple backup/restore implementations will be developed,
and it's not a good idea to pick a winner at the Kubernetes API layer. It makes
more sense to enable developers to try different approaches by making the API allow
restoring from various kinds of things. 

### Implementation Details/Notes/Constraints

As noted above, the proposal is extremely simple -- just remove the validation on
the `DataSource` field. This raises the question of WHAT will happen when users
put new things in that field, and HOW populators will actually work with so small
a change.

It's first important to note that only consumers of the `DataSource` field today
are the various dynamic provisioners, most notably the external-provisioner CSI
sidecar. If the external-provisioner sidecar sees a data source it doesn't
understand, it simply ignores the request, which is both important for forward
compatibility, and also perfect for the purposes of a data populator. This allows
developers to add new types of data sources that the dynamic provisioners will
simply ignore, enabling a different controller to see these objects and respond
to them.

The way I expect most data populators to work in practice is that they will see
a request for a PVC `foo` with a data source of `bar`. The populator will create
a shadow PVC `foo'` in another namespace with all of the same spec except the
data source will be empty. The dynamic provisioner will ignore `foo` but will
respond to `foo'` by creating an empty PV. The populator can create a pod
attached to `foo'` and fill it with data pointed to by `bar` using whatever
mechanism the developer chooses. After the volume is populated, the pod can be
deleted, and the PV can be rebound to `foo`.

A working example of the above approach is illustrated with a "hello world"
prototype populator here:

https://github.com/NetApp/hello-populator

Other implementations are also possible, where the populator either works more
closely with the dynamic provisioner, or goes around the CSI interface
entirely to create the PV in a proprietary way. For example, there may be
some kinds of data sources that we want to support in the CSI standard itself,
but also allow for non-CSI things to populator as a fall-back. In this case
the CRD that acts as the data source would need to include enough detail that
the external-provisioner sidecar could decide whether to handle the request
using CSI (perhaps through a capability query), or whether to ignore the
request, so that some other fall-back controller could handle the request in
a generic manner.

An example of a more proprietary implementation would be
an additional sidecar that implements features not yet part of CSI by talking
directly to the storage controller. It's likely that brand new features will
be delivered first as proprietary extensions before they become standardized
and supported by the CSI spec and the community-supported sidecars.

### Risks and Mitigations

Clearly there is concern that bad things might happen if we don't restrict
the contents of the `DataSource` field, otherwise the validation wouldn't
have been added. The main risk that I'm aware of is that badly-coded dynamic
provisioners might crash if they see something they don't understand.
Fortunately, the external-provisioner sidecar correctly handles this case,
and so would any other dynamic provisioner designed with forward compatibility
in mind.

Removing validation of the field relinquishes control over what kind of
data sources are okay, and gives developers the freedom to decide. It's hard
to see how giving developers freedom goes badly, but I'm willing to consider
that it might.

Security issues are hard to measure, because any security issues would be the
result of badly designed data populators that failed to put appropriate
limits on user's actions. Such security issues are going to be present with
any new controller, though, so they don't seem relevant to this change. The
main thing to realize is that the `DataSource` field is a "local" typed
object reference, so no matter what, the object in that field has to either
be in the same namespace as the PVC that references it, or it must be a
non-namespaced object. This seems like an appropriate and desirable
limitation for security reasons.

If we think about who can install populators, the RBAC required for a
populator to operate requires at minimum, the ability to either create or
modify PVs. Also the CRD for the data source type needs to be installed.
This means that populators will generally be installed by cluster admins
or similarly-powerful users, and those users can be expected to understand
the uses and implications of any populators they chose to install. 

## Design Details

TBD

### Test Plan

TBD

### Graduation Criteria

TBD

#### Examples

TBD

##### Alpha -> Beta Graduation

- Need to see several implementations of working data populators that solve real world problems
- Automated tests for at least one flavor of data population

##### Beta -> GA Graduation

- Distributions including data populators as part of their distros (possibly
  a backup/restore implementation layered on top)
- Allowing time for feedback

### Upgrade / Downgrade Strategy

Data sources are only considered at provisioning time -- once the PVC becomes
bound, the `DataSource` field becomes merely a historical note.

On upgrade, there are no potential problems because this change merely
relaxes an existing limitation.

On downgrade, there is a potential for unbound (not yet provisioned) PVCs to
have data sources that never would have been allowed on the lower version. In
this case we might want to revalidate the field and possibly wipe it out on
downgrade. 

### Version Skew Strategy

No issues

## Implementation History

- The idea of data populators has been discussed abstractly in SIG-storage
  since 2018 at least.
- John Griffith did the original work to propose something like this, but
  that work got scaled down to just PVC clones.
- Ben Swartzlander picked up the populator proposal developed 2 prototypes
  in December 2019.
- New KEP proposed January 2020
