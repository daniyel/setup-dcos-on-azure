# Setting up DCOS on Azure

## Prerequisites

Linux, macOS, or Windows
command-line shell terminal such as Bash or PowerShell
verified Azure Resource Manager account with the necessary permissions

### Install Terraform

Visit the [Terraform download page](https://www.terraform.io/downloads.html) for bundled installations and support for Linux, macOS and Windows.

If you’re on a Mac environment with [Homebrew](https://brew.sh/) installed, simply run the following command:

```sh
$ brew install terraform
```

### Install and configure the Azure CLI

1. Set up an [Azure Resource Manager account](https://azure.microsoft.com/en-us/free/) if you don’t already have on. Make sure to have at least one [user role set up](https://docs.microsoft.com/en-us/azure/security-center/security-center-permissions).

2. Install the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest). For macOS users, it is available using Homebrew:

```sh
$ brew install azure-cli
```

3. Once you have the Azure CLI, it needs to be connected to the account you would like to use. If you already had the CLI installed, you may already have your credentials set up. To set up your credentials, or to update them anytime as needed, run:

```sh
$ az login
```

Follow any directions, including signing in from your browser, to enable your CLI.

4. You can insure that you are logged in by listing your account permissions:

```sh
$ az account
```

It should return something like:

```sh
$ az account
[
  {
    "cloudName": "AzureCloud",
    "id": "12345678-abcd-efgh-9876-abc123456789",
    "isDefault": true,
    "name": "DC/OS Production Subscription",
    "state": "Enabled",
    "tenantId": "987654321-abcd-efgh-9876-abc123456789",
    "user": {
      "name": "myaccount@azuremesosphere.onmicrosoft.com",
      "type": "user"
    }
  }
]
```

Set the `ARM_SUBSCRIPTION_ID`. The current Terraform Provider for Azure requires that the default Azure subscription be set before terraform can start. provide the Azure subscription ID. You can set the default account with the following command:

An example:

```sh
$ export ARM_SUBSCRIPTION_ID="12345678-abcd-efgh-9876-abc123456789"
```

Ensure it is set:

```
$ echo $ARM_SUBSCRIPTION_ID
```

### Set up SSH credentials for your cluster

Terraform will need to send out SSH keys to connect securely to the nodes it creates. You should have ansible.pem file locally on your computer.

## Creating a DC/OS Cluster

1. Start by creating a local folder and cd’ing into it. This folder will be used as the staging ground for downloading all required Terraform modules and holding the configuration for the cluster you are about to create.

```sh
$ mkdir dcos-tf-azure-demo && cd dcos-tf-azure-demo
```

2. Create a file in that folder called main.tf, which is the configuration file the Mesosphere Universal Installer will call on each time when creating a plan. The name of this file should always be main.tf. Open the file in the code editor of your choice and paste in the following. Copy the following into the main.tf file. Adjust values accordingly.


```sh
variable "dcos_install_mode" {
  description = "specifies which type of command to execute. Options: install or upgrade"
  default = "install"
}
 
data "http" "whatismyip" {
  url = "http://whatismyip.akamai.com/"
}
 
provider "azurerm" {
  version = "~> 1.16.0"
}
 
module "dcos" {
  source  = "dcos-terraform/dcos/azurerm"
  version = "~> 0.1.0"
 
  dcos_instance_os    = "coreos_1855.5.0"
  cluster_name        = "my-dcos-cluster"
  ssh_public_key_file = "~/.ssh/my_public_key.pub"
  admin_ips           = ["${data.http.whatismyip.body}/32"]
  location            = "West Europe"
 
  num_masters        = "1"
  num_private_agents = "3"
  num_public_agents  = "2"
 
  dcos_version = "1.12.0"
 
  # dcos_variant              = "ee"
  # dcos_license_key_contents = "${file("./license.txt")}"
  dcos_variant = "open"
 
  providers = {
    azurerm = "azurerm"
  }
 
  dcos_install_mode = "${var.dcos_install_mode}"
}
 
output "masters-ips" {
  value       = "${module.dcos.masters-ips}"
}
 
output "cluster-address" {
  value       = "${module.dcos.masters-loadbalancer}"
}
 
output "public-agents-loadbalancer" {
  value = "${module.dcos.public-agents-loadbalancer}"
}
```

3. Now the action of actually creating your cluster and installing DC/OS begins. First, initialise the project’s local settings and data. Make sure you are still working in the dcos-tf-azure-demo folder where you created your main.tf file, and run the initialisation.

```sh
$ terraform init
```

Result:

```sh
Terraform has been successfully initialized!
 
You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.
 
If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your environment. If you forget, other
commands will detect it and remind you to do so if necessary.
```

4. After Terraform has been initialised, the next step is to run the execution planner and save the plan to a static file - in this case, plan.out.

```sh
$ terraform plan -out=plan.out
```

5. The next step is to get Terraform to build/deploy our plan. Run the command below.

```sh
$ terraform apply plan.out
```

:warning: If for whatever reason the apply command fails, you should repeat step 4. and 5. respectively, but with different **-out=plan.out names**. For example **plan1.out** and execute apply again.

After the Cluster is created you should use one of the authentication providers Google, GitHub or Microsoft and create account.

### Creating a JWT token

For us to be able to use Marathon API (deploying, creating etc. apps), we need to provide JWT token for our requests. JWT token can be generate with the secret from master node. SSH to the master node and execute:

```sh
$ cat /var/lib/dcos/dcos-oauth/auth-token-secret
```

Using Python script we can now generate our JWT token. Be sure you can execute Python scripts on your computer and you can instal modules via pip.

```sh
$ sudo pip install --upgrade pip
$ sudo pip install PyJWT
$ sudo pip install cryptography
```

Create a Python script with following content. Replace the `CLUSTER_SECRET` below with the hash value from previous steps. Use email account you provided when you first created authorization user and set the duration of the token according to your needs.

```python
import jwt
import time
from datetime import datetime
from datetime import timedelta
 
clusterSecret = '<CLUSTER_SECRET>' # token we retrieved from master node
uid = 'user@somewhere.com'         # email, that was initially used when creating authentication on DCOS
hours = 876000                     # how long we want our token to be valid
 
expTime = time.time() + (3600 * hours)
 
token = jwt.encode({'exp':expTime, 'uid': uid}, clusterSecret, algorithm='HS256')
 
print token
```

Testing the token:

```sh
$ curl -H "Authorization: token=<JWT_TOKEN>" "<DCOS_CLUSTER_URL>/marathon/v2/apps"
```

### Setting Up Logging

To setup logging on DCOS public and private agents we need to SSH to each one of them and download Filebeat tarball file:

```sh
$ curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.0.0-linux-x86_64.tar.gz
$ tar xzvf filebeat-7.0.0-linux-x86_64.tar.gz
$ mv filebeat-7.0.0-linux-x86_64 filebeat
```

Create necessary folders:

```sh
$ sudo mkdir /var/log/filebeat
$ sudo mkdir /var/lib/filebeat
$ sudo mkdir /etc/filebeat
```

Backup default filebeat.yml configuration file in `/home/dcos_admin/filebeat` folder:

```sh
$ cp filebeat.yml filebeat.yml.BAK
```

Empty the configuration file with:

```sh
$ echo '' > filebeat.yml
```

and copy and paste this and replace `ELASTICSEARCH_USERNAME` and `ELASTICSEARCH_PASSWORD`, if applicable:

```sh
filebeat.inputs:
  - type: log
    paths:
      - '/var/lib/mesos/slave/slaves/*/frameworks/*/executors/*/runs/latest/stdout*'
      - '/var/lib/mesos/slave/slaves/*/frameworks/*/executors/*/runs/latest/stderr*'
      - '/var/log/mesos/*.log'
      - '/var/log/dcos/dcos.log'
    exclude_files:
      - stdout.logrotate.state
      - stdout.logrotate.conf
      - stderr.logrotate.state
      - stderr.logrotate.conf
    tail_files: true
    fields:
      environment: testing
      vendor: nyo
output.elasticsearch:
  hosts: 'logpush.convercus.io:9443'
  protocol: https
  username: <ELASTICSEARCH_USERNAME>
  password: <ELASTICSEARCH_PASSWORD>
```
  
Copy configuration file to:

```sh
$ sudo cp filebeat.yml /etc/filebeat/filebeat.yml
```

Create systemd service configuration file `/etc/systemd/system/filebeat.service`, with content below:

```sh
[Unit]
Description=Filebeat sends log files to Logstash or directly to Elasticsearch.
Documentation=https://www.elastic.co/products/beats/filebeat
Wants=network-online.target
After=network-online.target
 
[Service]
Environment="BEAT_LOG_OPTS=-e"
Environment="BEAT_CONFIG_OPTS=-c /etc/filebeat/filebeat.yml"
Environment="BEAT_PATH_OPTS=-path.home /home/dcos_admin/filebeat -path.config /etc/filebeat -path.data /var/lib/filebeat -path.logs /var/log/filebeat"
ExecStart=/home/dcos_admin/filebeat/filebeat $BEAT_LOG_OPTS $BEAT_CONFIG_OPTS $BEAT_PATH_OPTS
Restart=always
 
[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```sh
$ sudo systemctl enable filebeat.service
$ sudo systemctl start filebeat.service
```

If everything went well, service should be in active mode. You can check with:

```sh
$ sudo systemctl status filebeat.service
```

### Set up service for parsing the journal

Create a script `/etc/systemd/system/dcos-journalctl-filebeat.service` that parses the output of the DC/OS master journalctl logs and funnels them to `/var/log/dcos/dcos.log`.

```sh
[Unit]
Description=DCOS journalctl parser to filebeat
Wants=filebeat.service
After=filebeat.service
 
[Service]
Restart=always
RestartSec=5
ExecStart=/bin/sh -c '/bin/journalctl  --since="5 minutes ago" --no-tail --follow --unit="dcos*.service" >> /var/log/dcos/dcos.log 2>&1'
 
[Install]
WantedBy=multi-user.target
```

Create necessary folder:

```sh
$ sudo mkdir /var/log/dcos
```

For all nodes, start and enable the Filebeat log parsing services created above:

```sh
$ sudo chmod 0755 /etc/systemd/system/dcos-journalctl-filebeat.service
$ sudo systemctl start dcos-journalctl-filebeat.service
$ sudo systemctl enable dcos-journalctl-filebeat.service
```

### Setup Logrotate


You should configure logrotate on all of your nodes to prevent the file `/var/log/dcos/dcos.log` from growing without limit and filling up your disk. Your logrotate config should contain copytruncate because otherwise the journalctl pipe remains open and pointing to the same file even after it has been rotated. When using copytruncate, there is a very small window between copying the file and truncating it, so some logging data might be lost - you should balance pros and cons between filling up the disk and losing some lines of logs.

```sh

$ sudo vi /etc/logrotate.d/dcos
```

For example, your logrotate configuration should look like this:

```sh
/var/log/dcos/dcos.log {   
  size 100M
  copytruncate
  rotate 5
  compress
  compresscmd /bin/xz
}
```
