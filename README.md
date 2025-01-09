# Red Hat EX280 - Study Guide

## Details

Name: `Red Hat Certified OpenShift Administrator`

Version: `V414K`

**Disclaimer**: My notes may not cover all the commands required for the exam and, they do not guarantee success on exam. 

## Table of Contents

1. [Common commands](#common-commands)

    1.1. [With oc-cli](#with-oc-cli)

    1.2. [With git](#with-git)

2. [Deploy an application](#deploy-an-application)

    2.1. [Method 1: Using CLI](#method-1-using-cli)

    2.2. [Method 2: Using YAML file](#method-2-using-yaml-file)

    2.3. [Method 3: Using Kustomize](#method-3-using-kustomize)

    2.4. [Method 4: Using OpenShift Templates](#method-4-using-openshift-templates)

    2.5. [Method 5: Using Helm](#method-5-using-helm)

3. [Manage authentication and authorization](#manage-authentication-and-authorization)

    3.1. [Configuring HTPasswd Identity Provider](#configuring-htpasswd-identity-provider)

    3.2. [Administering users and groups permissions](#administering-users-and-groups-permissions)

## Notes

### Common commands

#### With `oc-cli`

Create a new project

```shell
$ oc new-project xp
```

Get the deployment status

```shell
$ watch oc get deployments,pods
```

#### With `git`

Clone a Git repository

```shell
$ git clone https://github.com/kubernetes-sigs/kustomize.git

# It clones a particular branch of the Git repository
# add: -b release-kustomize-v5.4.3
$ git clone https://github.com/kubernetes-sigs/kustomize.git -b release-kustomize-v5.4.3
```

Show all the branches (local and remote) of a Git repository

```shell
$ git branch -a
```

Switch from `master` to `release-kustomize-v5.4.3` branch

```shell
$ git checkout release-kustomize-v5.4.3
```

### Deploy an application

#### Method 1: Using CLI

Deploy a MySQL instance

```shell
$ oc create deployment db \
--image quay.io/redhattraining/mysql-80 \
--port 3306
```

**Note**: The deployment `db` failed due to missing parameters.

Set needed parameters to start the container

```shell
$ oc set env deployment/db \
MYSQL_USER=user1 \
MYSQL_PASSWORD=mypa55w0rd \
MYSQL_DATABASE=items
```

#### Method 2: Using YAML file

If no deployment exists, use the command `oc create` first.

```shell
$ oc create -f deployment.yaml
```

If a deployment exists, it's recommended to use the `oc apply` command to update the existing deployment.

```shell
$ oc apply -f deployment.yaml
```

#### Method 3: Using Kustomize

I use the template available in the [kubernetes-sig/kustomize](https://github.com/kubernetes-sigs/kustomize/tree/master/examples) repository to learn.

I moved the content into `base/` folder to enable working with overlays.

```shell
$ tree helloWorld/

helloWorld/
├── base
│   ├── configMap.yaml
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
└── README.md
```

Replace the deprecated `commonLabels` with `labels` and modify the value of the `app` key.

```shell
$ sed -i 's/commonLabels/labels/g' base/kustomization.yaml
$ sed -i 's/hello/my-hello-world/g' base/kustomization.yaml
```

Deploy the package

```shell
$ cd helloWorld/

$ oc create -k base/
```

#### Method 4: Using OpenShift Templates

List available templates

```shell
$ oc get templates -n openshift
```

Show the available parameters for a template

```shell
$ oc process mysql-ephemeral --parameters -n openshift
```

Deploy a MySQL instance using the `mysql-ephemeral` template.

```shell
$ oc new-app db \
--template=mysql-ephemeral \
-p MYSQL_USER=user1 \
-p MYSQL_PASSWORD=mypa55w0rd \
-p MYSQL_DATABASE=items
```

**Note**: The command `oc new-app` enables the use of a template from `openshift` namespace

#### Method 5: Using Helm

Add a Helm chart repository

```shell
$ helm repo add bitnami https://charts.bitnami.com/bitnami
```

List all added Helm repositories

```shell
$ helm repo list
```

Search for Helm charts

```shell
# It shows only the most recent version of the chart
$ helm search repo bitnami

# OR

# It shows all the versions available of the chart
# add: --versions
$ helm search repo bitnami --versions
```

Deploy a MySQL instance using `bitnami/mysql` chart

```yaml
# values.yaml
auth:
  username: user1
  password: pa55w0rd
```

```shell
# If the namespace is absent, replace --namespace with --create-namespace parameter
$ helm install db \
--namespace xp \
bitnami/mysql \
--version 12.2.0 \
--values values.yaml

# OR

$ helm install db \
--namespace xp \
bitnami/mysql \
--version 12.2.0 \
--set auth.username="user1" \
--set auth.password="pa55w0rd"
```

List all installed Helm charts

```shell
# In the current namespace context
$ helm list

# OR

# Across all namespaces
$ helm list -A
```

### Manage authentication and authorization

#### Configuring HTPasswd Identity Provider

Create an HTPasswd file

```shell
# httpd-tools package needed
$ htpasswd -c -B -b ./htpasswd developer pa55w0rd1
```

Add a new user to the existing HTPasswd file

```shell
$ htpasswd -b ./htpasswd tester pa55w0rd2
```

Create the `secret` using the HTPassword file

```shell
$ oc create secret generic localusers \
--from-file=htpasswd=./htpasswd
-n openshift-config
```

Modify the `oauth/cluster` configuration

```shell
$ oc get oauth/cluster -o yaml > oauth.yaml
```

Add the `htpasswd` parameters

```yaml
# .spec.identityProviders[]
- htpasswd:
    fileData:
      name: localusers
  mappingClaim: claim
  name: myusers
  type: HTPasswd
```

Apply the modified `oauth/cluster` configuration

```shell
$ oc replace -f ./oauth.yaml
```

Wait for the new configuration to take effect

```shell
$ watch oc get pods -n openshift-authentication
```

#### Administering Users and Groups permissions

Create a new group

```shell
$ oc adm groups new dev-group
```

Manage users in a group

- Add a user to a group

```shell
$ oc adm groups add-users dev-group developer
```

- Remove a user to a group
```shell
$ oc adm groups remove-users dev-group developer
```

List groups and their users

```shell
$ oc get groups
```

Disallow users creating new projects

```shell
$ oc get clusterrolebindings | grep self-provisioner

$ oc adm policy remove-role-from-group self-provisioner system:authenticated:oauth
```

Grant privileges to a group:

- At the namespace level:

```shell
$ oc policy add-role-to-group edit dev-group
```

- At the cluster level:

```shell
$ oc adm policy add-role-to-group cluster-admin dev-group
```

Work in progress..

## Author

**Hoang Thanh VO**

- Email: [thanh@thanh-vo.com](mailto:thanh@thanh-vo.com) 

## License

This project is [MIT](https://github.com/ht-vo/rh-ex280-study-guide/blob/main/LICENSE) licensed.
