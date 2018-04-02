# Instructions to install OpenShift using oc cluster command

- To create an all in one cluster using oc cluster command, then it is required to have a CentOS vm that you can access using ssh
- Next you can execute one of the role created to `up`, `down` or `clean` the cluster created

```bash
~/MyProjects/cloud-native/infra/ansible
ansible-playbook -i inventory/cloud_host playbook/cluster.yml -e openshift_node=masters --tags "up" 
```

- To override the default parameters of the `oc cluster up` command, then pass extra vars within the command 
```bash
ansible-playbook -i inventory/cloud_host playbook/cluster.yml -e openshift_node=masters -e openshift_release_tag_name=v3.9.0-alpha.3 --tags "up" 
-e "@extra_vars.yml"
```

# Post installation

The post_installation playbook performs various tasks, like enabling the cluster admin user, installing Istio etc.
Make sure that the `openshift_admin_pwd` is specified when invoking the command. 

```bash
ansible-playbook -i inventory/cloud_host playbook/post_installation.yml -e openshift_admin_pwd=admin
```
