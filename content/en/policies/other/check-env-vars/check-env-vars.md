---
title: "Check Environment Variables"
category: Other
version: 
subject: Pod
policyType: "validate"
description: >
    Environment variables control many aspects of a container's execution and are often the source of many different configuration settings. Being able to ensure that the value of a specific environment variable either is or is not set to a specific string is useful to maintain such controls. This policy checks every container to ensure that if the `DISABLE_OPA` environment variable is defined, it must not be set to a value of `"true"`.
---

## Policy Definition
<a href="https://github.com/kyverno/policies/raw/main//other/check-env-vars/check-env-vars.yaml" target="-blank">/other/check-env-vars/check-env-vars.yaml</a>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-env-vars
  annotations:
    policies.kyverno.io/severity: medium
    policies.kyverno.io/title: Check Environment Variables
    policies.kyverno.io/category: Other
    policies.kyverno.io/subject: Pod
    kyverno.io/kubernetes-version: "1.24"
    kyverno.io/kyverno-version: 1.8.2
    policies.kyverno.io/description: >-
      Environment variables control many aspects of a container's execution and are
      often the source of many different configuration settings. Being able to ensure that
      the value of a specific environment variable either is or is not set to a specific string
      is useful to maintain such controls. This policy checks every container to ensure that if the
      `DISABLE_OPA` environment variable is defined, it must not be set to a value of `"true"`.
spec:
  background: true
  validationFailureAction: Audit
  rules:
    - name: check-disable-opa
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "DISABLE_OPA must not be set to true."
        pattern:
          spec:
            containers:
              - name: "*"
                =(env):
                  - (name): DISABLE_OPA
                    value: "!true"
```
