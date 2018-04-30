# Play with our list of Developer's features

## Roles - Features

The table hereafter summarizes the roles available that you can call using the `post_installation` playbook.

| Role Name | Description  |
| --------- | ------------ | 
| [add_extra_users](#command-add_extra_users) | Create list of users/passwords and their corresponding project |
| [enable_cluster_admin](#command-enable_cluster_admin) | Grant Cluster admin role to an OpenShift user |
| [identity_provider](#command-identity_provider) | Set the Master-configuration of OpenShift to use `htpasswd` as its identity provider |
| [persistence](#command-persistence) | Enable Persistence using `hotPath` as `persistenceVolume` |
| [install_nexus](#command-install_nexus) | Install Nexus Repository Server |
| [install_jenkins](#command-install_jenkins) | Install Jenkins and configure it to handle `s2i` builds started within an OpenShift project |
| [install_jaeger](#command-install_jaeger) | Install Distributed Tracing - Jaeger |
| [install_istio](#command-install_istio) | Install ServiceMesh - Istio |
| [service_catalog](#command-service-catalog) | Deploy the [Ansible Service Broker](http://automationbroker.io/) |
| [install_launcher](#command-install_launcher) | Install and enable the Fabric8 [Launcher](http://fabric8-launcher) |
| [install_oc](#command-install_oc) | Install oc client within the vm

## Prerequisite

  - Linux VM (CentOS7, ...) running, that you can ssh on port 22 and where your public key has been imported
  - Ansible [2.4](http://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
  - OpenShift installed using `cluster` role or `openshift playbook`

## General command

The post_installation playbook, that yu can execute as presented hereafter performs various tasks.
When you will run it, make sure that the `openshift_admin_pwd` is specified when invoking the command as it contains the 'openshift' admin user
to be used to executed `oc` commands on the cluster.

```bash
ansible-playbook -i inventory/simple_host playbook/post_installation.yml -e openshift_admin_pwd=admin
```

To install one of the roles, you will specify it using the `--tags` parameter as showed hereafter.

```bash
ansible-playbook -i inventory/cloud_host playbook/post_installation.yml -e openshift_admin_pwd=admin --tags "enable_cluster_admin"
```

**Remarks** : 

- Refer to the `ROLE/defaults/main.yml` to learn what are the parameters and their default value
- To only install specific roles, you will pass a comma separated values list using the `--tags install_nexus,install_jaeger` parameter
- If you would like to execute all roles except some, you can use Ansible's `--skip-tags` in the same fashion. 

## Role's command

### Command identity_provider

  ```bash
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml -e openshift_admin_pwd=admin --tags "identity_provider"
  ```
  
### Command add_extra_users

  **WARNING**: Role `identity_provider` must be executed before !
  
  For the first machine the following will create an admin user (who is granted cluster-admin priviledges) and an extra 5 users (user1 - user5)

  ```bash
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml --tags add_extra_users \
     -e number_of_extra_users=5 \
     -e first_extra_user_offset=1 \
     -e openshift_admin_pwd=admin
  ```
  
  This step will create 5 users with credentials like `user1/pwd1` while also creating a project for like `user1` for each user
  
  By default these users will have admin roles (although not cluster-admin) and will each have a project that corresponds to the user name.
  These defaults can be changed using the `make_users_admin` and `create_user_project` flags. See [here](playbook/roles/add_extra_users/defaults/main.yml) 

### Command enable_cluster_admin

  ```bash
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml \
     -e openshift_admin_pwd=admin \
     --tags "enable_cluster_admin"
  ```
  
### Command Persistence

  ```bash
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml \
    --tags persistence \
    -e openshift_admin_pwd=admin
  ```
  
  The number of PVs to be created can be controlled by the `number_of_volumes` variable. See [here](playbook/roles/persistence/defaults/main.yml).
  By default, 10 volumes of 5Gb each will be created.

### Command install_nexus

  The nexus server will be installed under the project `infra` and will contain the Red Hat proxy servers
  
  ```bash
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml \
     --tags nexus
  ```

### Command install_jenkins

  The Jenkins server will be installed under the project `infra` 
  
  ```bash
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml \
     --tags jenkins
  ```

### Command install_jaeger

  ```bash
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml \
     --tags jaeger \
     -e infra_project=infra
  ```
  **WARNING**: the `infra_project` parameter is mandatory

### Command install_istio

  ```bash
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml \
     --tags istio
  ```
 
    
  The `istio` role also accepts the following parameters that can be added using Ansible's `extra-vars` [syntax](http://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#passing-variables-on-the-command-line):
  
  
| Name | Description  | Default
| --------- | ------------ | ------------ |
| istio_git_repo | The git repository where the ansible playbook for installing Istio exists | https://github.com/istio/istio.git |
| istio_git_branch | The git branch where the ansible playbook for installing Istio exists | master |
| istio_repo_dest | Directory where the project will be cloned on the target machine | ~/.istio/playbooks |

### Command install_launcher

  ```bash
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml \
     --tags {install-launcher|uninstall-launcher}
  ```
  
    
  The `launcher` role also accepts the following parameters that can be added using Ansible's `extra-vars` [syntax](http://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#passing-variables-on-the-command-line):
  
  
| Name | Description  | Default
| --------- | ------------ | ------------ |
| launcher_project_name | The project where the launcher will be installed | devex |
| launcher_github_username | The github username to use |  |
| launcher_github_token | The github token to use |  |
| launcher_keycloak_url | The keycloak URL to use |  |
| launcher_keycloak_realm | The keycloak realm to use | |
| launcher_openshift_user | The launcher openshift user | admin |
| launcher_openshift_pwd | The launcher openshift user | admin |
| launcher_template_version | The launcher template version | master |
| launcher_template_name | The launcher template name | fabric8-launcher |  
| launcher_template_url | The launcher template URL | https://raw.githubusercontent.com/fabric8-launcher/launcher-openshift-templates/master/openshift/launcher-template.yaml |  
| launcher_catalog_git_repo | Git Repo where the catalog is defined | https://github.com/fabric8-launcher/launcher-booster-catalog.git |  
| launcher_catalog_git_branch | Git branch where the catalog is defined | master |  
| launcher_catalog_filter | The filter used to limit which catalog entries will appear |  |  
| launcher_openshift_console_url | The URL where the Openshift console is accessible | https://192.168.99.50:8443/ |  
| launcher_keycloak_template_name | The project where the launcher will be installed | devex |  

### Command install_oc

  ```bash
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml \
     --tags install_oc
  ```

### Command Service catalog

  To install the service catalog, execute this command
  ```bash
  ansible-playbook -i inventory/cloud_host openshift-ansible/playbooks/openshift-service-catalog/config.yml
  ```
