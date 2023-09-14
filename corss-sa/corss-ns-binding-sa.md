
## Background

Have two namespace:
- sn-system: In this namespace, we have custom resource `connector-catalogs`(Namespaced scoped).
- sn-users: In this namespace, we have a service account `users-sa`.
 
We want to let `users-sa` have permissions to read `connector-catalogs` of `sn-system`. 

PS: Don't allow create `ClusterRole` and `ClusterRoleBinding`.

## How do it?

Refer to ChartGPT:

If you wish to bind a RoleBinding in the `sn-system` namespace to a ServiceAccount in the `sn-users` namespace, you can do so as follows. This would allow the ServiceAccount in the `sn-users` namespace to perform operations in the `sn-system` namespace that are allowed by the Role.

Here is an example:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: connector-catalog-reader-binding
  namespace: sn-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role  # or ClusterRole if you have the permission to create it
  name: connector-catalog-reader
subjects:
- kind: ServiceAccount
  name: your-service-account
  namespace: sn-users
```

## Test step

1. Create Namespace `sn-system` and `sn-users`:

```bash
kubectl create namespace sn-system
kubectl create namespace sn-users
```

2. Create CRD define

```bash
kubectl apply -f test-crd-define.yaml
```

3. Create CRD data on `sn-system`
```bash
kubectl apply -f test-crd.yaml
kubectl get connectorcatalogs.example.com my-connector-catalog -n sn-system -o yaml
```

4. Create ServiceAccount
```
kubectl apply -f sn-system-sa.yaml
kubectl apply -f sn-users-sa.yaml
```

5. Verify sn-system account not permissions to reader connectorcatalogs on sn-system
```bash
kubectl apply -f sn-system-k8s-pod.yaml
kubectl exec -ti alpine-k8s-pod -n sn-system -- kubectl get connectorcatalogs.example.com my-connector-catalog -n sn-system -o yaml

Error from server (Forbidden): connectorcatalogs.example.com "my-connector-catalog" is forbidden: User "system:serviceaccount:sn-system:sn-system-sa" cannot get resource "connectorcatalogs" in API group "example.com" in the namespace "sn-system"
```

6. Verify sn-users account not permissions to reader connectorcatalogs on sn-system
```bash
kubectl apply -f sn-users-k8s-pod.yaml
kubectl exec -ti alpine-k8s-pod -n sn-users -- kubectl get connectorcatalogs.example.com my-connector-catalog -n sn-system -o yaml

Error from server (Forbidden): connectorcatalogs.example.com "my-connector-catalog" is forbidden: User "system:serviceaccount:sn-users:sn-users-sa" cannot get resource "connectorcatalogs" in API group "example.com" in the namespace "sn-users"
```

7. Create Role and RoleBinding on `sn-system`
```bash
kubectl apply -f sn-system-role-binding.yaml

// verify sn-system have reader permissions
kubectl exec -ti alpine-k8s-pod -n sn-system -- kubectl get connectorcatalogs.example.com my-connector-catalog -n sn-system -o yaml
```

7. **Create Role and RoleBinding on `sn-system` to binding `sn-users-sa` of `sn-users`  with `connectorcatalogs-reader` of `sn-system` **
```bash
kubectl apply -f sn-system-role-binding-sn-users.yaml

// verify sn-users have reader connectorcatalogs on sn-system permissions
kubectl exec -ti alpine-k8s-pod -n sn-users -- kubectl get connectorcatalogs.example.com my-connector-catalog -n sn-system -o yaml
```


8. We still can binding `sn-users-sa` to other role
```
kubectl apply -f sn-users-sa-binding-pod-test.yaml 
kubectl exec -ti alpine-k8s-pod -n sn-users -- kubectl get pods -n sn-users 
```