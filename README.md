
Create an ansible.cfg file with the following

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
token=<token>

```

Replace <token> with a token retrieved from:

oc create secret generic rhel8-eap-ssh-key --from-file=key=/home/phayes/.ssh/id_rsa.pub --type=opaque -n eap

oc apply -f vm-sample.yml 

oc apply -f service.yml 

oc get virtualmachines

virtctl start rhel8-eap

ansible-galaxy collection install -r ./collections/requirements.yml

ansible-playbook -i ./inventory/hosts pre-reqs.yml -e username=$USERNAME -e password=$PASSWORD

ansible-playbook -i ./inventory/hosts jboss.yml -e rhn_username=$RHN_USERNAME -e rhn_password=$RHN_PASSWORD --ask-become-pass

/opt/jboss_eap/jboss-eap-7.4/bin/jboss-cli.sh --connect
/subsystem=datasources:installed-drivers-list

ansible-playbook -i ./inventory/hosts app.yml 


