---
title: "Ensure Read Only hostPath in CEL expressions"
category: Other in CEL
version: 1.11.0
subject: Pod
policyType: "validate"
description: >
    Pods which are allowed to mount hostPath volumes in read/write mode pose a security risk even if confined to a "safe" file system on the host and may escape those confines (see https://blog.aquasec.com/kubernetes-security-pod-escape-log-mounts). The only true way to ensure safety is to enforce that all Pods mounting hostPath volumes do so in read only mode. This policy checks all containers for any hostPath volumes and ensures they are explicitly mounted in readOnly mode.
---

## Policy Definition
<a href="https://github.com/kyverno/policies/raw/main//other-cel/ensure-readonly-hostpath/ensure-readonly-hostpath.yaml" target="-blank">/other-cel/ensure-readonly-hostpath/ensure-readonly-hostpath.yaml</a>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: ensure-readonly-hostpath
  annotations:
    policies.kyverno.io/title: Ensure Read Only hostPath in CEL expressions
    policies.kyverno.io/category: Other in CEL 
    policies.kyverno.io/severity: medium
    policies.kyverno.io/minversion: 1.11.0
    kyverno.io/kyverno-version: 1.11.0
    kyverno.io/kubernetes-version: "1.26-1.27"
    policies.kyverno.io/subject: Pod
    policies.kyverno.io/description: >-
      Pods which are allowed to mount hostPath volumes in read/write mode pose a security risk
      even if confined to a "safe" file system on the host and may escape those confines (see
      https://blog.aquasec.com/kubernetes-security-pod-escape-log-mounts). The only true way
      to ensure safety is to enforce that all Pods mounting hostPath volumes do so in read only
      mode. This policy checks all containers for any hostPath volumes and ensures they are
      explicitly mounted in readOnly mode.
spec:
  background: false
  validationFailureAction: Audit
  rules:
  - name: ensure-hostpaths-readonly
    match:
      any:
      - resources:
          kinds:
            - Pod
          operations:
          - CREATE
          - UPDATE
    validate:
      cel:
        variables:
          - name: allContainers
            expression: "object.spec.containers + object.spec.?initContainers.orValue([]) + object.spec.?ephemeralContainers.orValue([])"
          - name: hostPathVolumes
            expression: "object.spec.?volumes.orValue([]).filter(volume, has(volume.hostPath))"
        expressions:
          - expression: >-
              variables.hostPathVolumes.all(hostPath, variables.allContainers.all(container, 
              container.volumeMounts.orValue([]).all(volume, (hostPath.name != volume.name) || volume.?readOnly.orValue(false) == true)))
            message: All hostPath volumes must be mounted as readOnly.
      

```
