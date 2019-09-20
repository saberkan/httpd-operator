# Main goal
Example : https://github.com/operator-framework/operator-sdk/blob/master/doc/ansible/user-guide.md
Create an operator to deploy one http pod : DONE
Create lifecycle manager to upgrade pod : TO DO

# Step 1 : Create template for httpd pod
<pre>
oc login -u system:admin
oc new-project operators
oc policy add-role-to-user admin -z developer -n operators
oc create user global-admin
oc adm policy add-cluster-role-to-user cluster-admin global-admin
oc import-image rhscl/httpd-24-rhel7 --from=registry.access.redhat.com/rhscl/httpd-24-rhel7 --confirm -n openshift
oc create -f httpd_resources.yaml
</pre>

# Step 2 : Create ansible script to deploy 
<pre>
mkdir roles && cd roles
ansible-galaxy init --force --offline httpd_openshift
# develop the role
</pre>

# Step 3 : Create the operator using sdk
<pre>
# install : https://github.com/operator-framework/operator-sdk/blob/master/doc/user/install-operator-sdk.md#install-from-homebrew-macos
brew install operator-sdk
operator-sdk new httpd-operator --api-version=httpd.saberkan.fr/v1 --kind=Httpd --type=ansible
cp roles/httpd_openshift/tasks/main.yml httpd-operator/roles/httpd/tasks/
cd httpd-operator/
cp ../playbook.yaml .
# Edit docker file, and watcher
operator-sdk build quay.io/saberkan/httpd-operator:v1
docker push quay.io/saberkan/httpd-operator:v1
# make image public in quay
oc login -u global-admin
oc project operators
vim deploy/operator.yaml
# Set the built image in the file twice, and imagePullPolicy to always
oc create -f deploy/service_account.yaml
oc create -f deploy/role_edited.yaml #be carreful this was adapted, not default
oc create -f deploy/clusterrole_edited.yaml #be carreful this was adapted, not default
oc create -f deploy/role_binding.yaml 
oc create -f deploy/crds/httpd_v1_httpd_crd.yaml
oc create -f deploy/operator.yaml
</pre>

# Step 4 : Deploy a first httpd resource
<pre>
echo 'apiVersion: httpd.saberkan.fr/v1
kind: Httpd
metadata:
  name: example-httpd
spec:
  my_var_value: "IT WORKS WITH OPERATOR"
  size: 1' > httpd.yaml
</pre>

At this point, operator works and deploys httpd server

#TODO :
consider to use jinja for template in roles:
<pre>
- name: Read definition file from the Ansible controller file system
  k8s:
    state: present
    definition: "{{ lookup('template', 'deployment-config.yaml') }}"
</pre>

# Step 5 : Generate operator manifest
<pre>
operator-sdk olm-catalog gen-csv --csv-version 1.0.0
</pre>

check the generated dir with csv and package under deploy/olm-catalog

# Step 6 : Deploy the operator
<pre>
oc create -f operator_group.yaml
oc create -f deploy/olm-catalog/httpd-operator/1.0.0/httpd-operator.v1.0.0.clusterserviceversion.yaml
</pre>
At this step all requirements are missing, CRD and roles should be created
You need to edit it and add customresourcedef requirements, and minKubeversion
If requirement still missing, edit csv on yaml, delete previous one and recreate otherwise, midification is not checked again. //TODO : this might be a big, check with team


# Step 7 : 
<pre>
# IF it doesn't exist
oc create -f deploy/crds/httpd_v1_httpd_crd.yaml
oc create -f deploy/service_account.yaml
oc create -f deploy/role_edited.yaml 
oc create -f deploy/role_binding.yaml 
</pre>

# Step 8 : Deploy httpd cluster
<pre>
oc create -f httpd.yaml
</pre>

ALL ok
<pre>
oc get pods
NAME                              READY   STATUS      RESTARTS   AGE
httpd-24-rhel7-1-deploy           0/1     Completed   0          22s
httpd-24-rhel7-1-lqkq6            1/1     Running     0          15s
httpd-operator-5fc9f48797-pxjfk   2/2     Running     0          3m25s
</pre>

-----------------------------------------------
How does OLM and Catalog work to download operators under the cluster ?

# Step 1 : First you need to have a sourcecatalog, by default openshift has 3 Source catalogs
<pre>
$ oc get catalogsource -n openshift-marketplace
NAME                  NAME                  TYPE   PUBLISHER   AGE
certified-operators   Certified Operators   grpc   Red Hat     3d3h
community-operators   Community Operators   grpc   Red Hat     3d3h
redhat-operators      Red Hat Operators     grpc   Red Hat     3d3h
</pre>

For example community one looks like this
<pre>
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  labels:
    csc-owner-name: community-operators
    csc-owner-namespace: openshift-marketplace
    olm-visibility: hidden
    openshift-marketplace: "true"
    opsrc-datastore: "true"
    opsrc-owner-name: community-operators
    opsrc-owner-namespace: openshift-marketplace
    opsrc-provider: community
  name: community-operators
  namespace: openshift-marketplace
spec:
  address: 172.30.176.102:50051
  displayName: Community Operators
  icon:
    base64data: ""
    mediatype: ""
  publisher: Red Hat
  sourceType: grpc
</pre>

opsrc-provider this field relates to csPublisher field in CatalogSourceConfig

# Step 2 : Configure CatalogSourceConfig to allow subscription
<pre>
apiVersion: operators.coreos.com/v1
kind: CatalogSourceConfig
metadata:
  name: installed-community-test
  namespace: openshift-marketplace
spec:
  csDisplayName: Community Operators
  csPublisher: Community  
  packages: prometheus
  targetNamespace: test
</pre>

name field relates to subscription field source

# Step 3: Create operatorgroup for namespace, to manage operators installed
<pre>
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: operatorgroup-for-test
  namespace: test
spec:
  targetNamespaces:
  - test
</pre>

# Step 4 : Create subscription
<pre>
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  generateName: prometheus-
  namespace: test
spec:
  source: installed-community-test
  sourceNamespace: test
  name: prometheus
  startingCSV: prometheusoperator.0.27.0
  channel: beta
</pre>

Then installplan will be generated, then operator deployed
<pre>
$ oc get pods
NAME                                   READY   STATUS    RESTARTS   AGE
prometheus-operator-67dfc557c7-s86bw   1/1     Running   0          59m
</pre>