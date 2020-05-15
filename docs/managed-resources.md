---
title: Managed Resources
toc: true
weight: 400
indent: true
---

# Managed Resources

## Overview

Managed resources are Crossplane representation of the external provider resources
and they are considered primitive low level custom resources that are used as
building blocks in other abstractions such as your application or a layer of composition.

For example, `RDSInstance` in AWS Provider corresponds to an actual RDS Instance
in AWS. There is a one-to-one relationship and the changes on managed resources are
reflected directly on the corresponding resource in the provider.

## Syntax

Crossplane has some API conventions that are built on top of Kubernetes API
conventions for how the schema of the managed resources should look like. Following
is an example of `RDSInstance`:

```yaml
apiVersion: database.aws.crossplane.io/v1beta1
kind: RDSInstance
metadata:
  name: foodb
spec:
  forProvider:
    dbInstanceClass: db.t2.small
    masterUsername: root
    allocatedStorage: 20
    engine: mysql
  writeConnectionSecretToRef:
      name: mysql-secret
      namespace: crossplane-system
  providerRef:
    name: mycreds
  reclaimPolicy: Delete
```

In Kubernetes, `spec` top field represents the desired state of the user.
Crossplane adheres to that and has its own conventions about how the fields under
`spec` should look like.

* `writeConnectionSecretToRef`: A reference to the secret that you want this
  managed resource to write its connection secret that you'd be able to mount
  to your pods in the same namespace. For `RDSInstance`, this secret would contain
  `endpoint`, `username` and `password`.
  
* `providerRef`: Reference to the `Provider` resource that will provide information
  regarding authentication of Crossplane to the provider. `Provider` resources
  refer to `Secret` and potentially contain other information regarding authentication.

* `reclaimPolicy`: Enum to specify whether the actual cloud resource should be
  deleted when this managed resource is deleted in Kubernetes API server. Possible
  values are `Delete` and `Retain`.
  
* `forProvider`: While the rest of the fields relate to how Crossplane should
  behave, the fields under `forProvider` are solely used to configure the actual
  external resource. In most of the cases, the field names correspond to the what
  exists in provider's API Reference.
  
  The objects under `forProvider` field can get huge depending on the provider
  API. For example, GCP `ServiceAccount` has only a few fields while GCP `CloudSQLInstance`
  has over 100 fields that you can configure.

## Behavior

As a general rule, managed resource controllers try not to make any decision
that is not specified by the user in the desired state since managed resources
are the lowest level components of your infrastructure that you can build other
abstractions for doing magic.

### Continuous Reconciliation

Crossplane providers continuously reconcile the managed resource to achieve the
desired state. The parameters under `spec` is considered one and only source of
truth for the external resource. This means that if someone changed a configuration
in the UI of the provider, Crossplane will change it back to what's given under
`spec`.

#### Immutable Properties

There are configuration parameters in external resources that provider does not
allow to be changed. If the corresponding field in the managed resource is changed
by the user, Crossplane submits the new desired state to the provider and returns
the error, if any. For example, in AWS, you cannot change the region of
an `RDSInstance`.

Some infrastructure tools such as Terraform deletes and recreates the resource
to accommodate those changes but Crossplane does not take that route. Unless
the managed resource is deleted and its `reclaimPolicy` is `Delete`, its controller
never deletes the external resource in the provider.

> Immutable fields are marked as `immutable` in Crossplane codebase
but Kubernetes does not yet have immutable field notation in CRDs.

### Late Initialization

For some of the optional fields, users rely on the default that provider chooses
for them. Since Crossplane treats the managed resource as the source of the truth,
values of those fields need to exist in `spec` of the managed resource. So, in each
reconciliation, Crossplane will fill the value of a field that is left empty by
the user but is assigned a value by the provider. For example, there could be
two fields like `region` and `availabilityZone` and you might only want to give
`region`. In that case, if provider assigns an availability zone, Crossplane gets
that value and fills `availabilityZone` if it's not given already. Note that if
the field is already filled, the controller won't override its value.

### Deletion

When a deletion request is made for a managed resource, its controller starts the
deletion process immediately. However, it blocks the managed resource disappearence
via a finalizer so that the managed resource disappears only when the controller
confirms that the external resource is gone. This way, controller ensures that
if any error happens during deletion and resource is still there, you can still
see and manage it via its managed resource.

## Dependencies



## Importing Existing Resources

## Backup and Restore