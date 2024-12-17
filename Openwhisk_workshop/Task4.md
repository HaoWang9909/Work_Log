## Task 4:

Firstly, try two machines, deploy openwhisk on the server, and then use your own computer as the client terminal.

Open the server where openwhisk was previously deployed, and after restarting, some settings need to be reset.

Re-deploy.

```bash

export ENVIRONMENT=local
sudo ansible-playbook -i environments/local setup.yml
sudo ansible-playbook -i environments/$ENVIRONMENT prereq.yml
sudo systemctl stop couchdb
cd ..
./gradlew distDocker
cd ansible
sudo ansible-playbook -i environments/$ENVIRONMENT couchdb.yml
sudo ansible-playbook -i environments/$ENVIRONMENT initdb.yml
sudo ansible-playbook -i environments/$ENVIRONMENT wipe.yml
sudo ansible-playbook -i environments/$ENVIRONMENT openwhisk.yml
```

Install wsk on local computer

My computer is a Mac

```bash
brew update 
brew install wsk
```

On the local computer

```bash
wsk property set --apihost <server's address>
wsk property set --auth <key>
```

Test

Create a function locally`client_hello.js`

```bash
 /**
 * Hello world as an OpenWhisk action.
 */
function main(params) {
  var name = params.name || "from client";
  return { payload: "Hello World " + name + "!" };
}

```

```bash
wsk -i action create client_hello client_hello.js 
```

```bash
wsk -i action invoke client_hello --result
```

Output

```bash
{
    "payload": "Hello World from client!"
}
```

Success!

Check actionlist

```bash
wsk -i action list                           
actions
/guest/client_hello                                                    private nodejs:20
/guest/hello                                                           private nodejs:20
```

Below, try to deploy invoker on another machine

Establish an SSH trust relationship

Enter on the master node

```bash
sudo -i
ssh-keygen -t rsa -b 4096
```

Just keep pressing Enter.

Then enter

```bash
ssh-copy-id <username>@<worker_node_address>
```

Just follow the prompts and enter 'yes' and the server password
Can be tested by:

```bash
ssh <username>@<worker_node_address>
```

On the **secondary node (the disk space must be large! Otherwise, an error will occur in the final step)**:

```bash
sudo apt-get update
sudo apt-get install -y npm
```

Copy the 'openwhich/tools' on the main control node, and then we need to install the required packages

```bash
cd tools
find . -type f -name "*.sh" -exec chmod +x {} \;
cd ubuntu-setup
./all.sh
```

```bash
sudo groupadd docker
sudo usermod -aG docker $USER	
newgrp docker
sudo systemctl start docker
```

Return to the master control node

Modify `openwhisk/ansible/environments/local/hosts.j2.ini` 

```
; the first parameter in a host is the inventory_hostname

; used for local actions only
ansible ansible_connection=local

[edge]
<master_node_address>          ansible_host=<master_node_address> ansible_connection=local

[controllers]
controller0         ansible_host=<master_node_address> ansible_connection=local
;{% if mode is defined and 'HA' in mode %}
;controller1         ansible_host=<master_node_address> ansible_connection=local
;{% endif %}

[kafkas]
kafka0              ansible_host=<master_node_address> ansible_connection=local
{% if mode is defined and 'HA' in mode %}
kafka1              ansible_host=<master_node_address> ansible_connection=local
{% endif %}

[zookeepers:children]
kafkas

[invokers]
invoker0  ansible_host=<worker_node_address> ansible_user=exouser ansible_connection=ssh

[schedulers]
scheduler0       ansible_host=<master_node_address> ansible_connection=local

; db group is only used if db.provider is CouchDB
[db]
<master_node_address>         ansible_host=<master_node_address> ansible_connection=local

[redis]
<master_node_address>          ansible_host=<master_node_address> ansible_connection=local

[apigateway]
<master_node_address>          ansible_host=<master_node_address> ansible_connection=local

[etcd]
etcd0            ansible_host=<master_node_address> ansible_connection=local
```

Modify dblocal

```bash
[db_creds]
db_provider=CouchDB
db_username=admin
db_password=admin
db_protocol=http
db_host=<master_node_address>
db_port=5984
```

Modify`openwhisk/ansible/roles/invoker/tasks/deploy.yml` 

Change Task start invoker (line440) to:

```bash
  image: "{{ docker_registry }}openwhisk/invoker:nightly"
```

Run again

```bash
export ENVIRONMENT=local
sudo ansible-playbook -i environments/local setup.yml
sudo ansible-playbook -i environments/$ENVIRONMENT prereq.yml
sudo systemctl stop couchdb
sudo systemctl stop etcd
cd ..
./gradlew distDocker
cd ansible
sudo ansible-playbook -i environments/$ENVIRONMENT couchdb.yml
sudo ansible-playbook -i environments/$ENVIRONMENT initdb.yml
sudo ansible-playbook -i environments/$ENVIRONMENT wipe.yml
sudo ansible-playbook -i environments/$ENVIRONMENT openwhisk.yml
```

Test

Use `docker ps` in both machines to see if it is successful.

Create a function locally`distribute_hello.js` 

```bash
 /**
 * Hello world as an OpenWhisk action.
 */
function main(params) {
  var name = params.name || "from invoker";
  return { payload: "Hello World " + name + "!" };
}

```

```bash
wsk -i action create distribute_hello distribute_hello.js
wsk -i action invoke distribute_hello --result
```

success!

