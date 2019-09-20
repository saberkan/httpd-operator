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
