# Deployment

## Task 1:

```bash
sudo apt-get update
sudo apt-get install -y npm
```

```bash
git clone [https://github.com/apache/openwhisk.git](https://github.com/apache/openwhisk.git)
cd openwhisk
./gradlew core:standalone:bootRun
```

It can be found that a website address is generated on the server.

http://172.17.0.1:3232/playground.

Then use `ssh -L 3232:172.17.0.1:3232 <remote server username>@<remote server IP>`.

After that, this website address can be opened locally.

***Download and install the wsk CLI from (Linux, Mac or Windows):***

```bash
wget [https://github.com/apache/openwhisk-cli/releases/download/1.2.0/OpenWhisk_CLI-1.2.0-linux-amd64.tgz](https://github.com/apache/openwhisk-cli/releases/download/1.2.0/OpenWhisk_CLI-1.2.0-linux-amd64.tgz)
```

```bash
tar -xvzf OpenWhisk_CLI-1.2.0-linux-amd64.tgz
```

```bash
sudo mv wsk /usr/local/bin
```

test:

```bash
wsk --help
```

use

```bash
wsk property set --apihost 'http://172.17.0.1:3233' --auth '23bc46b1-71f6-4ed5-8c54-816aa4f8c502:123zO3xZCLrMN6v2BKK1dXYFpXlPkccOFqm12CdAsMgRU4VrNZ9lyGVCGuMDGIwP'
```

(The specific website address and key can be seen by just entering./gradlew core:standalone:bootRun.)

Run again:

```bash
./gradlew core:standalone:bootRun
```

success

## Task 2:

Save the following JavaScript code to the hello.js file:

```bash
/**
 * Hello world as an OpenWhisk action.
 */
function main(params) {
    var name = params.name || 'World';
    return {payload: 'Hello, ' + name + '!'};
}
```

`wsk action create hello hello.js`

See the output

`ok: created action hello`

`wsk action invoke hello --result` 

See the output

```bash
{
"payload": "Hello, World!"
}
```

Use

`wsk action invoke hello --result --param name NSDI` 

See the output

```bash
{
"payload": "Hello, NSDI!"
}
```

use`wsk action list` to view existing Actions：

```bash
exouser@lucaswang-test:~/openwhisk$ wsk action list
actions
/guest/hello                                                           private nodejs:20
```

Update Action:`wsk action update hello hello.js`

Delete Action:`wsk action delete hello` 

## Task 3:

```bash
# Install git if it is not installed
sudo apt-get install git -y

# Clone openwhisk
git clone https://github.com/apache/openwhisk.git openwhisk

# Change current directory to openwhisk
cd openwhisk
```

Here, the entire "tool" folder needs to be replaced with a new one from https://github.com/IntelliSys-Lab/InstaInfer/tree/main (replace it with the one inside this).

`cd tools`

`find . -type f -name "*.sh" -exec chmod +x {} \;`

add `sudo pip install setuptools==57.5.0` in `tools/ubuntu-setyp/ansible.sh`

and delete `sudo pip install --upgrade setuptools pip`

```bash
# Install all required software
(cd tools/ubuntu-setup && ./all.sh)
```

Set environment variables

Set default environment variables and specify the local environment.

```bash
export ENVIRONMENT=local
```

Install CouchDB

```bash
sudo apt update && sudo apt install -y curl apt-transport-https gnupg
```

```bash
curl https://couchdb.apache.org/repo/keys.asc | gpg --dearmor | sudo tee /usr/share/keyrings/couchdb-archive-keyring.gpg >/dev/null 2>&1
```

```bash
source /etc/os-release
```

```bash
echo "deb [signed-by=/usr/share/keyrings/couchdb-archive-keyring.gpg] https://apache.jfrog.io/artifactory/couchdb-deb/ ${VERSION_CODENAME} main" \
    | sudo tee /etc/apt/sources.list.d/couchdb.list >/dev/null

```

```bash
sudo apt update
```

```bash
sudo apt install -y couchdb
```

When configuring, change the port selection inside to 0.0.0.0.

```bash
sudo mkdir /home/logconf
sudo nano wsk_env.sh
```

add / append / insert

```bash
export OW_DB=CouchDB 
export OW_DB_USERNAME=admin
export OW_DB_PASSWORD=admin
export OW_DB_PROTOCOL=http
export OW_DB_HOST=172.17.0.1
export OW_DB_PORT=5984
export OPENWHISK_TMP_DIR=/home/logconf

```

```bash
source wsk_env.sh
```

```bash
sudo ansible-playbook -i environments/local setup.yml
```

```bash
#sudo ansible-playbook prereq.yml
```

It can be seen that a db_local.ini file is generated.`db_local.ini`file
It is found that the username and password inside are still incorrect.

Change it.

Then go to`/openwhisk/common/scala/src/main/resources/reference.conf` and change to this:

```bash
  DurationCheckerProvider = org.apache.openwhisk.core.scheduler.queue.NoopDurationCheckerProvider
```

go to `/home/exouser/openwhisk/ansible/group_vars/all` and change to this

```bash
  artifact_store:
    backend: "CouchDB"
  activation_store:
    backend: "CouchDB"
```

Then

```bash
sudo systemctl start docker
cd <openwhisk_home>
./gradlew distDocker
```

Stop couchdb

```bash
sudo systemctl stop couchdb
```

Run

```bash
cd ansible
ansible-playbook -i environments/$ENVIRONMENT couchdb.yml
```

Success

move on

```bash
sudo ansible-playbook -i environments/$ENVIRONMENT initdb.yml
```

```bash
sudo ansible-playbook -i environments/$ENVIRONMENT wipe.yml
```

```bash
sudo ansible-playbook -i environments/$ENVIRONMENT openwhisk.yml
```

```bash
sudo ansible-playbook -i environments/$ENVIRONMENT postdeploy.yml
```

Test

```bash
wsk property set --apihost '172.17.0.1'
wsk property set --auth `cat ansible/files/auth.guest`
```

Create a file 

```bash
/**
 * Hello world as an OpenWhisk action.
 */
function main(params) {
    var name = params.name || 'World';
    return {payload:  'Hello, ' + name + '!'};
}

```

`wsk -i action create hello hello.js`

`wsk -i action invoke hello --result`

see the out put

```bash
exouser@lucaswang-test:~/openwhisk$ wsk -i action invoke hello --result
{
    "payload": "Hello, World!"
}
```

Success!
