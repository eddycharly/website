---
title: JMESPath 
description: The JSON query language behind Kyverno.
weight: 12
---

{{% pageinfo color="warning" %}}
This page is currently under construction.
{{% /pageinfo %}}

[JMESPath](https://jmespath.org/) (pronounced "James path") is a JSON query language created by James Saryerwinnie and is the language that Kyverno supports to perform more complex selections of fields and values and also manipulation thereof by using one or more [filters](https://jmespath.org/specification.html#filter-expressions). If you're familiar with `kubectl` and Kubernetes already, this might ring a bell in that it's similar to [JSONPath](https://github.com/json-path/JsonPath). JMESPath can be used almost anywhere in Kyverno although is an optional component depending on the type and complexity of a Kyverno policy or rule that is being written. While many policies can be written with simple overlay patterns, others require more detailed selection and transformation. The latter is where JMESPath is useful.

While the complete specifications of JMESPath can be read on the official site's [specifications page](https://jmespath.org/specification.html), much of the specifics may not apply to Kubernetes use cases and further can be rather thick reading. This page serves as an easier guide and tutorial on how to learn and harness JMESPath for Kubernetes resources for use in crafting Kyverno policies. It should not be a replacement for the official JMESPath documentation but simply a use case specific guide to augment the already comprehensive literature.

## Getting Set Up

In order to position yourself for success with JMESPath expressions inside Kyverno policies, a few tools are recommended.

1. `jp`, the JMESPath CLI tool [here](https://github.com/jmespath/jp). This program allows you to test out JMESPath expressions live in a command line interface by passing in a JSON document and seeing the results without having to repeatedly test Kyverno policies.

2. `kubectl`, the Kubernetes CLI [here](https://kubernetes.io/docs/tasks/tools/). While having `kubectl` is a given, it comes in handy especially when building a JMESPath expression around performing API lookups.

3. `kyverno`, the Kyverno CLI [here](https://github.com/kyverno/kyverno/releases) or via [krew](https://krew.sigs.k8s.io/). Kyverno acts as a webhook (when run in-cluster) but also as a standalone CLI when run outside giving you the ability to test policies and, more recently, to test custom JMESPath filters which are endemic to only Kyverno.

4. `yq`, the YAML processor [here](https://github.com/mikefarah/yq). `yq` allows reading from a Kubernetes manifest and converting to JSON, which is helpful in order to be piped to `jp` in order to test expressions.

5. `jq`, the JSON processor [here](https://stedolan.github.io/jq/download/). `jq` is an extremely popular tool for working with JSON documents and has its own filter ability, but it's also useful in order to format JSON on the terminal for better visuals.

## Basics

JMESPath is used when you need fine-grained selection of a document and need to perform some type of query logic against the result. For example, if in a given field you need to refer to the value of another field either in the same resource or in a different one, you'll need to use JMESPath. This sample policy performs a simple mutation on a Pod to add a new label named `appns` and set the value of it based on the value of the Namespace in which that Pod is created.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-labels
spec:
  rules:
  - name: add-labels
    match:
      any:
      - resources:
          kinds:
          - Pod
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            appns: "{{request.namespace}}"
```

JMESPath expressions in most places in Kyverno must be enclosed in double curly braces like `{{request.namespace}}`. If an expression is used as the value of a field and contains nothing else, the expression needs to be wrapped in quotes: `appns: "{{request.namespace}}"`. If the value field contains other text outside of the expression, then it can be unquoted and treated as a string but this isn't strictly required: `message: The namespace name is {{request.namespace}}`.

When building a JMESPath expression, a dot (`.`) character is called a "sub-expression" and used to descend into nested structures. In the `{{request.namespace}}` example, this expression is looking for the top-most object the key of which is called `request` and then looking for a child object the key of which is called `namespace`. Whatever the value of the `namespace` key is will be inserted where the expression is written. Given the below AdmissionReview snippet, which will be explained in a moment, the value that would result from the `{{request.namespace}}` expression is `foo`.

```json
{
    "apiVersion": "admission.k8s.io/v1",
    "kind": "AdmissionReview",
    "request": {
        "namespace": "foo"
    }
}
```

When submitting a Pod, which matches the policy above, the result which gets created after Kyverno has mutated it would then look something like this.

**Incoming Pod**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: busybox
    image: busybox
```

**Outgoing Pod**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  labels:
    appns: foo
spec:
  containers:
  - name: busybox
    image: busybox
```

Notice that the new label `appns` has been added to the Pod and the value set equal to the expression `{{request.namespace}}` which, in this instance, happened to be `foo` because it was created in the `foo` Namespace. Should this Pod be created in another Namespace called `bar`, as you might guess, the label would be `appns: bar`. And this is a nice segue into AdmissionReview resources.

{{% alert title="Remember" color="primary" %}}
JMESPath, like JSONPath, is a query language _for JSON_ and, as such, it only works when fed with a JSON-encoded document. Although most work with Kubernetes resources using YAML, the API server will convert this to JSON internally and use that format when storing and sending objects to webhooks like Kyverno. This is why the program `yq` will be invaluable when building the correct expression based upon Kubernetes manifests written in YAML.
{{% /alert %}}

### AdmissionReview

Kyverno is an example, although there are many others, of an [admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-are-they). As the name implies, these are pieces of software which have some stake in whether a given resource is _admitted_ into the cluster or not. They may be either validating, mutating, or both. The latter applies to Kyverno as it has both capabilities. For a graphical representation of the order in which these requests make their way into Kyverno, see the [introduction page](/docs/introduction/#how-kyverno-works).

{{% alert title="Note" color="info" %}}
As the name "admission" implies, this process only takes place when an object _does not_ already exist. Pre-existing objects have already been admitted successfully in the past and therefore do not apply here. Certain other _operations_ on pre-existing objects, however, are subject to the admissions process including examples like executing (`exec`) commands inside Pods and deleting objects but, importantly, not when reading back objects.
{{% /alert %}}

When a resource that matches the criteria of a selection statement gets sent to the Kubernetes API server, after the API server performs some basic modifications to it, it then gets sent to webhooks which have told the API server via a MutatingWebhookConfiguration or ValidatingWebhookConfiguration resource--which Kyverno creates for you based upon the policies you write--that it wishes to be informed. The API server will "wrap" the matching resource in another resource called an [AdmissionReview](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#request) which contains a bunch of other descriptive data about that request, for example what _type_ or request this is (like a creation or deletion), the user who submitted the request, and, most importantly, the contents of the resource itself. Given the simple Pod example above that a user wishes to be created in the `foo` Namespace, the AdmissionReview request that hits Kyverno might look like following.

```json
{
    "kind": "AdmissionReview",
    "apiVersion": "admission.k8s.io/v1",
    "request": {
        "uid": "3d4fc6c1-7906-47d9-b7da-fc2b22353643",
        "kind": {
            "group": "",
            "version": "v1",
            "kind": "Pod"
        },
        "resource": {
            "group": "",
            "version": "v1",
            "resource": "pods"
        },
        "requestKind": {
            "group": "",
            "version": "v1",
            "kind": "Pod"
        },
        "requestResource": {
            "group": "",
            "version": "v1",
            "resource": "pods"
        },
        "name": "mypod",
        "namespace": "foo",
        "operation": "CREATE",
        "userInfo": {
            "username": "thomas",
            "uid": "404d34c4-47ff-4d40-b25b-4ec4197cdf63"
        },
        "object": {
            "kind": "Pod",
            "apiVersion": "v1",
            "metadata": {
                "name": "mypod",
                "creationTimestamp": null
            },
            "spec": {
                "containers": [
                    {
                        "name": "busybox",
                        "image": "busybox",
                        "resources": {}
                    }
                ]
            },
            "status": {}
        },
        "oldObject": null,
        "dryRun": false,
        "options": {
            "kind": "CreateOptions",
            "apiVersion": "meta.k8s.io/v1"
        }
    }
}
```

As can be seen, the full Pod is represented along with other metadata surrounding its creation.

These AdmissionReview resources serve as the most common source of data when building JMESPath expressions, specifically `request.object`. For the other data properties which can be consumed via an AdmissionReview resource, refer back to the [variables page](/docs/writing-policies/variables/#variables-from-admission-review-requests).

## Formatting

Because there are various types of values in differing fields, there are differing ways values must be supplied to JMESPath expressions as inputs in order to generate not only a valid expression but produce the output desired. Specifying values in the correct format is key to this success. Values which are supported but need to be differentiated in formatting are numbers (i.e., an integer like `6` or a floating point like `6.7`), a quantity (i.e., a number with a unit of measure like `6Mi`), a duration (i.e., a number with a unit of time like `6h`), a [semver](https://semver.org/) (i.e., a version number like `1.2.3`), and others. Because Kyverno (and therefore most custom JMESPath filters built for Kyverno) is designed for Kubernetes, it is Kubernetes aware. Therefore, specifying \`6\` as an input to a filter is not the same as specifying \'6\' where the former is interpreted as "the number six" and latter as "six bytes". The _types_ which map to the possible values are either JSON or string. In JMESPath, these are [literal expression](https://jmespath.org/specification.html#literal-expressions) and [raw string literals](https://jmespath.org/specification.html#raw-string-literals). Use the table below to find how to format the type of value which should be supplied.

| Value Type   | Input Type | JMESPath Type | Formatting |
|--------------|------------|---------------|------------|
| Number       | Integer    | Literal       | backticks  |
| Quantity     | String     | Raw           | quotes     |
| Duration     | String     | Raw           | quotes     |
| Labels (map) | Object     | Literal       | backticks  |

Paths in a JMESPath expression may also need escaping or literal quoting depending on the contents. For example, in a ResourceQuota the following schema elements may be present:

```yaml
spec:
  hard:
    limits.memory: 3750Mi
    requests.cpu: "5"
```

To represent the `limits.memory` field in a JMESPath expression requires literal quoting of the key in order to avoid being interpreted as child nodes `limits` and `memory`. The expression would then be `{{ spec.hard.\"limits.memory\" }}`. A similar approach is needed when individual keys contain special characters, for example a dash (`-`). Quoting and then escaping is similarly needed there, ex., `{{ images.containers.\"my-container\".tag }}`.

## Custom Filters

In addition to the filters available in the upstream JMESPath library which Kyverno uses, there are also many new and custom filters developed for Kyverno's use found nowhere else. These filters augment the already robust capabilities of JMESPath to bring new functionality and capabilities which help solve common use cases in running Kubernetes. The filters endemic to Kyverno can be used in addition to any of those found in the upstream JMESPath library used by Kyverno and do not represent replaced or removed functionality.

For instructions on how to test these filters in a standalone method (i.e., outside of Kyverno policy), see the [documentation](/docs/kyverno-cli/#jp) on the `kyverno jp` subcommand.

Information on each subcommand, its inputs and output, and specific usage instructions can be found below along with helpful and common use cases that have been identified.

### Add

<details><summary>Expand</summary>
<p>

The `add()` filter very simply adds two values and produces a sum. The official JMESPath library does not include most basic arithmetic operators such as add, subtract, multiply, and divide, the exception being `sum()` as documented [here](https://jmespath.org/specification.html#sum). While `sum()` is useful in that it accepts an array of integers as an input, `add()` is useful as a simplified filter when only two individual values need to be summed. Note that `add()` here is different from the `length()` [filter](https://jmespath.org/specification.html#length) which is used to obtain a _count_ of a certain number of items. Use `add()` instead when you have values of two fields you wish to add together.

`add()` is also value-aware (based on the formatting used for the inputs) and is capable of adding numbers, quantities, and durations without any form of unit conversion.

| Input 1            | Input 2            | Output   |
|--------------------|--------------------|----------|
| Number             | Number             | Number   |
| Quantity or Number | Quantity or Number | Quantity |
| Duration or Number | Duration or Number | Duration |
<br>

**Example:** This policy denies a Pod if any of its containers which specify memory requests and limits exceed 200Mi.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-demo
spec:
  validationFailureAction: enforce
  background: false
  rules:
  - name: add-demo
    match:
      any:
      - resources:
          kinds:
          - Pod
    preconditions:
      any:
      - key: "{{ request.operation }}"
        operator: In
        value: ["CREATE","UPDATE"]
    validate:
      message: "The total memory defined in requests and limits must not exceed 200Mi."
      foreach:
      - list: "request.object.spec.containers"
        deny:
          conditions:
            any:
            - key: "{{ add('{{ element.resources.requests.memory || `0` }}', '{{ element.resources.limits.memory || `0` }}') }}"
              operator: GreaterThan
              value: 200Mi
```

</p>
</details>

### Base64_decode

### Base64_encode

### Compare

### Divide

### Equal_fold

### Label_match

### Modulo

### Multiply

### Parse_json

### Parse_yaml

### Path_canonicalize

### Pattern_match

### Regex_match

### Regex_replace_all

### Regex_replace_all_literal

### Replace

### Replace_all

### Semver_compare

### Split

### Subtract

### Time_since

### To_lower

### To_upper

### Trim

### Truncate