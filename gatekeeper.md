## Enable Gatekeeper(Open Policy Agent) in Kubernetes

> Gatekeeper is an admission controller that validates requests to create and update Pods on Kubernetes clusters, using the Open Policy Agent (OPA). Using Gatekeeper allows administrators to define policies with a constraint, which is a set of conditions that permit or deny deployment behaviors in Kubernetes


- RBAC Permission
```
#kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user <YOUR USER NAME>
```
- Installation Of Gatekeeper Using Helm
```
#helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
#helm install gatekeeper/gatekeeper --name-template=gatekeeper --namespace gatekeeper-system --create-namespace
```

- Delete the Gatekeeper
```
#helm delete gatekeeper --namespace gatekeeper-system
#kubectl delete crd -l gatekeeper.sh/system=yes (Helm v3 will not cleanup Gatekeeper installed CRDs. Run command uninstall Gatekeeper CRDs)
```

- Create Constraint Templates
```
#cat ConstraintTemplate.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("you must provide labels: %v", [missing])
        }
```

- Create Constraints
```
#cat Constraints.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: ns-must-have-gk
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
  parameters:
    labels: ["gatekeeper"]
```

- **The match field**
  **The match field defines the scope of objects to which a given constraint will be applied. It supports the following matchers**
  - `kinds` accepts a list of objects with apiGroups and kinds fields that list the groups/kinds of objects to which the constraint will apply. If multiple groups/kinds objects are specified, only one match is needed for the resource to be in scope.
  - `scope` determines if cluster-scoped and/or namespaced-scoped resources are matched. Accepts *, Cluster, or Namespaced. (defaults to *)
  - `namespaces` is a list of namespace names. If defined, a constraint only applies to resources in a listed namespace. Namespaces also supports a prefix-based glob. For example, namespaces: `[kube-*]` matches both `kube-system` and `kube-public`.
  - `excludedNamespaces` is a list of namespace names. If defined, a constraint only applies to resources not in a listed namespace. ExcludedNamespaces also supports a prefix-based glob. For example, excludedNamespaces: [kube-*] matches both kube-system and kube-public.
  - `labelSelector` is the combination of two optional fields: matchLabels and matchExpressions. These two fields provide different methods of selecting or excluding k8s objects based on the label keys and values included in object metadata. All selection expressions are ANDed to determine if an object meets the cumulative requirements of the selector.
  - `namespaceSelector` is a label selector against an object's containing namespace or the object itself, if the object is a namespace.
  - `name` is the name of an object. If defined, it matches against objects with the specified name. Name also supports a prefix-based glob. For example, name: `pod-*` matches both `pod-a` and `pod-b`.

- The parameters field
**The parameters field describes the intent of a constraint. It can be referenced as input.parameters by the ConstraintTemplate's Rego source code. Gatekeeper populates input.parameters with values passed into the parameters field in the Constraint.**
```
Example

      rego: |
        package k8srequiredlabels

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("you must provide labels: %v", [missing])
        }
```
**The schema for the input Constraint parameters is defined in the ConstraintTemplate. The API server will reject a Constraint with an incorrect parameters field if the data types do not match**
```
Example

# Apply the Constraint with incorrect parameters schema
$ cat << EOF | kubectl apply -f -
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: ns-must-have-gk
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
  parameters:
    # Note that "labels" is now an array item, rather than an object
    - labels: ["gatekeeper"]
EOF
The K8sRequiredLabels "ns-must-have-gk" is invalid: spec.parameters: Invalid value: "array": spec.parameters in body must be of type object: "array"
```
- The enforcementAction field
**The enforcementAction field defines the action for handling Constraint violations. By default, enforcementAction is set to deny as the default behavior is to deny admission requests with any violation. Other supported enforcementActions include dryrun and warn. Refer to Handling Constraint Violations for more details.**

- You can list all constraints in a cluster with the following command
```
#kubectl get constraints
```

- The `input.review` object stores the admission request under evaluation. It has the following fields

  - `dryRun`: Describes if the request was invoked by `kubectl --dry-run`. This cannot be populated by Kubernetes for audit.
  - `kind`: The resource kind, group, version of the request object under evaluation.
  - `name`: The name of the request object under evaluation. It may be empty if the deployment expects the API server to generate a name for the `requested resource`.
  - `namespace`: The namespace of the request object under evaluation. Empty for cluster scoped objects.
  - `object`: The request object under evaluation to be created or modified.
  - `oldObject`: The original state of the request object under evaluation. This is only available for `UPDATE` operations.
  - `operation`: The operation for the request `(e.g. CREATE, UPDATE)`. This cannot be populated by Kubernetes for audit.
  - `uid`: The request's unique identifier. This cannot be populated by Kubernetes for audit.
  - `userInfo`: The request's user's information such as username, uid, groups, extra. This cannot be populated by Kubernetes for audit.

- Validation
```
#cat namespace-withoutlabel.yaml
kind: Namespace
apiVersion: v1
metadata:
  name: test

#kubectl apply -f namespace-withoutlabel.yaml

#cat namespace-withlabel.yaml
kind: Namespace
apiVersion: v1
metadata:
  name: test
  labels:
    name: gatekeeper

#kubectl apply -f namespace-withlabel.yaml
```
