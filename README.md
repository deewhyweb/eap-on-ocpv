
## Deploying and configuring EAP on OpenShift Virtualization using Ansible Middleware

This repository demonstrates how to create OpenShift Virtual machines using the OpenShift CLI and installs and configures an EAP installation using the Ansible-middleware collection.

### Pre-requisites

* An OpenShift cluster (tested with version 4.13) with bare metal nodes supporting OpenShift Virtualization e.g. https://demo.redhat.com/catalog?search=virt&item=babylon-catalog-prod%2Fsandboxes-gpte.ocp4-virtualization-lab.prod
* The OpenShift CLI logged into the OpenShift cluster as a user with permissions to create Virtual Machines
* Ansible (tested with version 2.14.8)
* virtctl CLI, (download instructions https://docs.openshift.com/container-platform/4.13/virt/virt-using-the-cli-tools.html#installing-virtctl_virt-using-the-cli-tools)

Create a file called ansible.cfg file with the following

```
#ansible.cfg:
[defaults]
host_key_checking = False
retry_files_enabled = False
nocows = 1

[inventory]
# fail more helpfully when the inventory file does not parse (Ansible 2.4+)
unparsed_is_failed=true

[galaxy]
server_list = automation_hub, galaxy
[galaxy_server.galaxy]
url=https://galaxy.ansible.com/
[galaxy_server.automation_hub]
url=https://cloud.redhat.com/api/automation-hub/
auth_url=https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
token=paste-your-token-here

```

Replace "paste-your-token-here" with a token retrieved from console.redhat.com by following these instructions:

* Navigate to https://cloud.redhat.com/ansible/automation-hub/token/.
* Click Load Token.
* Click copy icon to copy the API token to the clipboard.

Install the required ansible collections:

`ansible-galaxy collection install -r ./collections/requirements.yml`

Create an namespace called eap

`oc new-project eap`

Create a secret containing your local public key

`oc create secret generic rhel8-eap-ssh-key --from-file=key=~/.ssh/id_rsa.pub --type=opaque -n eap`

Create a PostgreSQL database

`oc apply -f ./postgres`

Create a virtual machine called rhel8-eap

`oc apply -f vm-sample.yml`

Create a loadbalancer service accepting connections on ports 22000->22 and 80->8080

`oc apply -f service.yml`

Start the virtual machine

`virtctl start rhel8-eap`

Give the virtual machine a minute or two to start and for the loadbalancer service hostname to resolve.

`oc get virtualmachines`

## installing and configuring EAP

Get the hostname of the service

`oc get svc`

Copy the "EXTERNAL-URL" and paste this into inventory/hosts

Set local environment variables with your red hat subscription credentials

`export USERNAME=xxx`

`export PASSWORD=xxx`

Run the pre-reqs, i.e. register the RHEL instance and mount the Postgres secret

`ansible-playbook -i ./inventory/hosts pre-reqs.yml -e username=$USERNAME -e password=$PASSWORD`

Set local environment variables with credentials of a service account to download JBoss EAP.
You can manage service accounts using the hybrid cloud console. Within this portal, on the [service accounts tab](https://console.redhat.com/application-services/service-accounts), you can create a new service account if one does not already exist.

`export RHN_USERNAME=xxx`

`export RHN_PASSWORD=xxx`

Deploy JBoss EAP using Ansible-middleware

`ansible-playbook -i ./inventory/hosts jboss.yml -e rhn_username=$RHN_USERNAME -e rhn_password=$RHN_PASSWORD --ask-become-pass`

When this script runs you will need to enter the password to your local account to allow local file access.

Test the postgres datasource using the JBoss CLI

`sss cloud-user@<external hostname> -p 22000`

On the ssh session, run the following

`sudo su`

`/opt/jboss_eap/jboss-eap-7.4/bin/jboss-cli.sh --connect`

`/subsystem=datasources:installed-drivers-list`

You should see the postgresql datasource listed.

Exit the ssh session and run the following to deploy the address book application

`ansible-playbook -i ./inventory/hosts app.yml`

You should now be albe to connect to the address book using the url http://\<external hostname\>/addressbook


