## Rohit's adventure with Kyverno


#### Overview

On one of my client sites, I was asked to work on an OpenShift cluster compliance and security. Openshift by default provides various security and compliance features by default, it has role based access control to allow role separation between cluster-admins vs devs etc, it also has admission controllers to make sure incoming resources adhere to the syntax and the CRD rules as well as restrictions to run privileged loads on the platform. However, there are more advance use cases that OpenShift does not take into account. This is where kyverno comes in - it allows for putting up cluster guardrails so developer teams can stay within bounds of what they're supposed to do and allow enforcing best practices.

#### Architecture

Kyverno is basically made up of different controllers that sit between the api server and the etcd database. It can allow for rules that can stop resources with certain labels to come in or allow for resources to be admitted only if they carry a certain annotations, this is achieved using the admission controller. Kyverno consists of Background controller, cleanup controller, and reports controller. If you want to make changes to already existing resources, the background controller comes in handy to mutate and delete resource as well. The reports policy is used to generate reports of it's findings and violation of certain rules.

<Insert Kyverno architecture diagram here>  

#### Use cases

Before I talk about the specific rules I want to introduce the concept of Policies and ClusterPolicies in Kyverno. You can either allow for the rules to apply to all the namespace via ClusterPolicy and any rules that apply only to certain namespaces can be applied through Policy Resource. There are two types of action these policies can do, either audit all the violations and observances of these rules or Block certain resources based on the rules.

Some of the policies I created to implement guardrails and best practices were:

1. Audit if deployments are setting secrets as environment variables.

```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: secrets-not-from-env-vars
  annotations:
    policies.kyverno.io/title: Disallow Secrets from Env Vars
    policies.kyverno.io/category: Sample, EKS Best Practices
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Pod, Secret
    kyverno.io/kyverno-version: 1.6.0
    policies.kyverno.io/description: >-
      Secrets used as environment variables containing sensitive information may, if not carefully controlled, 
      be printed in log output which could be visible to unauthorized people and captured in forwarding
      applications. This policy disallows using Secrets as environment variables.
spec:
  validationFailureAction: Audit
  background: true
  rules:
  - name: secrets-not-from-env-vars
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Secrets must be mounted as volumes, not as environment variables."
      pattern:
        spec:
          containers:
          - name: "*"
            =(env):
            - =(valueFrom):
                X(secretKeyRef): "null"
  - name: secrets-not-from-envfrom
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Secrets must not come from envFrom statements."
      pattern:
        spec:
          containers:
          - name: "*"
            =(envFrom):
            - X(secretRef): "null"
```

This cluster policy works on all the Pods and make sure they do not have `secretKeyRef` in the spec. The syntax `=()` in the above policy checks if the key env is present and only moves forward to the key `valueFrom`. This basically ensures the policy won't error out if the previous field is not present. Also a policy can have multiple rules of usually of similar requirements, in our case above we have two rules that check for secret keys inside of pods. 

2. Signature validation for container images.


```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image
  annotations:
    policies.kyverno.io/title: Verify Image
    policies.kyverno.io/category: Sample, EKS Best Practices
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Pod
    policies.kyverno.io/minversion: 1.7.0
    policies.kyverno.io/description: >-
      Using the Cosign project, OCI images may be signed to ensure supply chain
      security is maintained. Those signatures can be verified before pulling into
      a cluster. This policy checks the signature of an image repo called
      ghcr.io/kyverno/test-verify-image to ensure it has been signed by verifying
      its signature against the provided public key. This policy serves as an illustration for
      how to configure a similar rule and will require replacing with your image(s) and keys.      
spec:
  validationFailureAction: enforce
  background: false
  rules:
    - name: verify-image
      match:
        any:
        - resources:
            kinds:
              - Pod
      verifyImages:
      - imageReferences:
        - "ghcr.io/kyverno/test-verify-image:*"
        mutateDigest: true
        attestors:
        - entries:
          - keys:
              publicKeys: |
                -----BEGIN PUBLIC KEY-----
                MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE8nXRh950IZbRj8Ra/N9sbqOPZrfM
                5/KAQN0/KjHcorm/J5yctVd7iEcnessRQjU917hmKO6JWVGHpDguIyakZA==
                -----END PUBLIC KEY-----                

``` 

You can also use kyverno to verify the image present in your platform are signed or not. The above policy checks on all the pods with imageReferences from `ghcr.io/kyverno/test-verify-image:*` which means all the test-verify-image with any tag needs to be verified against this public key. You can change the imageReference to check for all the images from ghcr.io, quay.io etc with the following syntax:

```
- imageReferences:
  - "ghcr.io/*"
  - "quay.io/*"
```

You can also check for predicates, annotations in the image metadata, attestations, and signatures for more granular checks. Along with key-pair checks, there are other ways to verify signature with cosign including keyless and certificate verification.

PS: these policies are from the offical kyverno documentation, Check it out for more uses cases: [Kyverno Policies](https://release-1-8-0.kyverno.io/policies/other/verify_image/)


#### Good to know

1. You can avoid certain namespaces to be managed by kyverno using a configMap called kyverno. You can specify the namespaces you want kyverno to ignore using labels and wilcards for the names. You can use `resourceFilter` and `webhook` keys to achieve it.

```
  resourceFilters: '[*/*,test,*]', [*/*, openshift-*,*]
  webhook:   webhooks: '[{\"namespaceSelector\":{\"matchExpressions\":[{\"key\":\"kubernetes.io/metadata.name\",\"operator\":\"NotIn\",\"values\":[\"kube-system\"]},{\"key\":\"kubernetes.io/metadata.name\",\"operator\":\"NotIn\",\"values\":[\"kyverno\"]}],\"matchLabels\":null}}]'
```

The resource filter will not apply any ClusterPolicy and Policy in test as well as any namespace starting with openshift- - like openshift-apiserver, openshift-etcd etc.
Whereas webhook uses namespace selector with the help of labels to avoid the namespaces.

FYI - Webhooks are an instruction to the API server for what NOT to send to Kyverno. The resource filter (in the ConfigMap) is an instruction to Kyverno on what NOT to process (i.e., ignore).

Any Background requests are not affected by resourceFilters and webhooks since they are applied only to admission review, so if you wanna mutate existing resources you can use match/exclude keys on the policies.

