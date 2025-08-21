## Install the OpenShift GitOps operator

```shell
oc create ns openshift-gitops-operator 
```
```shell
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-gitops-operator
spec:
  channel: gitops-1.17
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: openshift-gitops-operator.v1.17.0
EOF
```

## Install the EPAM community Keycloak operator

```shell
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: edp-keycloak-operator
  namespace: openshift-operators
spec:
  channel: alpha
  installPlanApproval: Automatic
  name: edp-keycloak-operator
  source: community-operators
  sourceNamespace: openshift-marketplace
  startingCSV: edp-keycloak-operator.v1.19.0
EOF
```

## Create the ClusterRole and ClusterRoleBinding required for the Application controller to be able to create those resources

```shell
oc apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gitops-oauthclient-manager
rules:
- apiGroups:
  - oauth.openshift.io
  resources:
  - oauthclients
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete
EOF
```

```shell
oc apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gitops-controller-admin-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: gitops-oauthclient-manager
subjects:
- kind: ServiceAccount
  name: openshift-gitops-argocd-application-controller
  namespace: openshift-gitops
EOF
```

## Check if it is possible to create oauth client by Argo CD application controller
```shell
oc auth can-i create oauthclients --as=system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -A
```
If the RBAC resources are configured correctly, you should see a response saying `yes`

## Create the destination namespace and allow it to be managed by `openshif-gitops` ArgoCD instance.
```shell
oc create ns test-keycloak && oc label ns test-keycloak argocd.argoproj.io/managed-by=openshift-gitops
```

## Create the application

### Helm based

```shell
oc apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cci-keycloak
  namespace: openshift-gitops
spec:
  destination:
    namespace: openshift-gitops
    server: https://kubernetes.default.svc
  ignoreDifferences:
  - group: v1.edp.epam.com
    jsonPointers:
    - /spec/config/clientSecret
    kind: KeycloakRealmIdentityProvider
  - group: oauth.openshift.io
    jsonPointers:
    - /secret
    kind: OAuthClient
  - group: rbac.authorization.k8s.io
    kind: ClusterRole
    jsonPointers:
      - /rules/0/verbs
  project: default
  source:
    helm:
      valueFiles:
      - ./values.yaml
    path: helm-keycloak-oauth
    repoURL: https://github.com/anandf/argocd-example-apps
    targetRevision: master
  syncPolicy:
    syncOptions:
    - ApplyOutOfSyncOnly=true
EOF
```
### Kustomize based
```shell
oc apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cci-keycloak
  namespace: openshift-gitops
spec:
  destination:
    namespace: openshift-gitops
    server: https://kubernetes.default.svc
  ignoreDifferences:
  - group: v1.edp.epam.com
    jsonPointers:
    - /spec/config/clientSecret
    kind: KeycloakRealmIdentityProvider
  - group: oauth.openshift.io
    jsonPointers:
    - /secret
    kind: OAuthClient
  - group: rbac.authorization.k8s.io
    kind: ClusterRole
    jsonPointers:
      - /rules/0/verbs
  project: default
  source:
    path: kustomize-keycloak-oauth
    repoURL: https://github.com/anandf/argocd-example-apps
    targetRevision: master
  syncPolicy:
    syncOptions:
    - ApplyOutOfSyncOnly=true
EOF
```

## Sync the application from the UI or CLI

```shell
ADMIN_PASSWORD=$(oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d)
ARGOCD_SERVER_HOST=$(oc get route -n openshift-gitops openshift-gitops-server -o jsonpath='{.spec.host}')
argocd login "${ARGOCD_SERVER_HOST}:443" --username admin --password "${ADMIN_PASSWORD}"
```

```shell
argocd app list
argocd app sync cci-keycloak
```

## Make local edits to the resources

#### Patch the fields in `OAuthClient`, `KeyCloakRealmIdentityProvider` and `ClusteRole` using the following commands.
```shell
oc patch keycloakrealmidentityproviders -n test-keycloak my-realm-openshift-v4 --patch '{"spec": {"config": {"clientSecret": "abcde"}}}' --type merge
oc patch oauthclient my-keycloak-my-realm --patch '{"secret": "abcde"}'
oc patch clusterrole my-keycloak-my-app --type='json' -p='[{"op": "replace", "path": "/rules/0/verbs", "value": ["get"]}]'
```
