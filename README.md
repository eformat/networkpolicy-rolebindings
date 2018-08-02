### Customizing Network Policy

The default `networkpolicy` configuration in OpenShift 3.10 allows project `admin` and `edit` roles
the ability to create/update/delete networkpolicy objects.

You can restrict `networkpolicy` access (e.g. to view/get/list only normal project admins, editors)
by creating custom roles `custom-admin`, `custom-edit` and set the default project template to use
one of these clusterroles when projects are created.

As a `custer-admin`, Create custom roles:

```
oc login -u system:admin
# export default roles
oc export clusterrole admin  > clusterrole-admin.yaml
oc export clusterrole edit  > clusterrole-edit.yaml

# edit them and create new roles
oc create -f clusterrole-admin-custom.yaml
oc create -f clusterrole-edit-custom.yaml
```

Set default project template that has admin role and networkpolicy adjusted

```
oc login -u system:admin
oc apply -f project-request-temaplate.yaml -n default

## sets up the admin to use our custom role
- apiVersion: rbac.authorization.k8s.io/v1beta1
  kind: RoleBinding
  metadata:
    creationTimestamp: null
    name: admin
    namespace: ${PROJECT_NAME}
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: custom-admin
  subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: ${PROJECT_ADMIN_USER}

## adds default networkpolicy

- apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-from-same-namespace
  spec:
    podSelector:
    ingress:
    - from:
      - podSelector: {}
- apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-from-default-namespace
  spec:
    podSelector:
    ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            name: default


# set this template as default project
ansible masters -m shell -a "sed -i -e 's|projectRequestTemplate:.*$|projectRequestTemplate: "default/project-request"|g' /etc/origin/master/master-config.yaml"
#ansible masters -m shell -a "systemctl restart atomic-openshift-master-api"
#ansible masters -m shell -a "systemctl restart atomic-openshift-master-controllers"
/usr/local/bin/master-restart api
/usr/local/bin/master-restart controllers

# label default ns for networkpolicy
oc label namespace default name=default
```

As a normal user create project

```
oc login -u developer
oc new-project welcome

$ oc get networkpolicy
NAME                           POD-SELECTOR   AGE
allow-from-default-namespace   <none>         10m
allow-from-same-namespace      <none>         10m

$ oc delete networkpolicy allow-from-default-namespace
Error from server (Forbidden): networkpolicies.extensions "allow-from-default-namespace" is forbidden: User "developer" cannot delete networkpolicies.extensions in the namespace "welcome": User "developer" cannot delete networkpolicies.extensions in project "welcome"

$ oc get rolebinding
NAME                    ROLE                    USERS       GROUPS                           SERVICE ACCOUNTS   SUBJECTS
admin                   /custom-admin           developer
system:deployers        /system:deployer                                                     deployer
system:image-builders   /system:image-builder                                                builder
system:image-pullers    /system:image-puller                system:serviceaccounts:welcome
```

Show rolebindings

```
oc policy who-can delete networkpolicy -n welcome
oc policy who-can get networkpolicy -n welcome
```

Manually set the custom-admin role on a project and remove default admin as a `cluster-admin`

```
oc policy add-role-to-user custom-admin developer -n welcome
oc policy remove-role-from-user admin developer -n welcome
```
