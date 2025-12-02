# Unstract workflow

[Open source Edition](https://docs.unstract.com/unstract/editions/open_source_edition/)

## Purpose of the project

We want to demonstrate it is feasible with some AI to extract quote data about any types of work and material such as prices or VAT from PDF files
and inject this data into excel. 


## Prerequisites

    * Linux or MacOS (Intel or M-series)
    * Docker
    * Docker Compose (sometimes you need to install this separately)
    * Git (if you plan to clone the repo rather than download a release)


## Get up and running

You can find the Unstract Open Source Edition source code repository on GitHub. To run Unstract you can either download one of the releases (recommended) or clone the Unstract repository. The package should contain the run-platform.sh script, which you need to then run to get all the required containers pulled and started. The steps look something like this:

    * Step 1: Run the script: $ sudo ./run-platform.sh
    * Step 2: Now visit http://frontend.unstract.localhost in your browser.
    * Step 3: Use user name and password unstract to login

Once the services are up, visit http://frontend.unstract.localhost in your browser.
user/pwd unstract/unstract

See logs with:
    docker compose -f docker/docker-compose.yaml logs -f
    
Configure services by updating corresponding <service>/.env files.

Make sure to restart the services with:
```
    docker compose -f docker/docker-compose.yaml up -d
```

## Note:

###################### BACKUP ENCRYPTION KEY ######################
Copy the value of ENCRYPTION_KEY in any of the following env files
to a secure location:

- backend/.env: ENCRYPTION_KEY="d4WNw-9............="
- platform-service/.env: ENCRYPTION_KEY="d4WNw-9...........="

Adapter credentials are encrypted by the platform using this key.
Its loss or change will make all existing adapters inaccessible!
###################################################################

## Setting up Unstract

After login in, we need to connect LLM, DB, Model, Extractor


### Connect Embedding Model
* install Ollama: https://docs.ollama.com/linux
*  curl -o ollama-install.sh -fsSL https://ollama.com/install.sh
*  chmod 755 ollama-install.sh
*  /ollama-install.sh

#### Testing Ollama:

First we need to pull a model:
```
ollama pull mxbai-embed-large
```

Test model:
```
curl http://localhost:11434/api/embeddings -d '{
  "model": "mxbai-embed-large",
  "prompt": "Represent this sentence for searching relevant passages: The sky is blue because of Rayleigh scattering"
}'
```

For Ollama to be accessible we need to tell it to accept connections from any of the IP addresses i.e. 0.0.0.

To change the listen IP:

```
sudo vi /etc/systemd/system/ollama.service
Under [Service]
add:
Environment="OLLAMA_HOST=0.0.0.0:11434"

sudo systemctl stop ollama.service
sudo systemctl status ollama.service
```

We set in Unstract the Embedding Model:

* BASE URL http://192.168.1.73:11434

Meaning the host ip not localhost

### Connect Vector DB Qdrant

Qdrant runs in the cloud.

We need to create an account to get API key

```
curl \
    -X GET 'https://eb616ac7-bea6-4992-8150-3a8d9d1b1d9d.europe-west3-0.gcp.cloud.qdrant.io:6333' \
    --header 'api-key: eyJ..............yA_L3Ks'
```

### Connect An LLM

We opted for the free tier MISTRAL AI.

We created  an account and got an API key
wjMF1...............cW

Due to free tier limitation we have to use the mistral-tiny model and now everything seems
to work well.



### Connect Text Extractor

We are using LLMWhisperer https://unstract.com/blog/llmwhisperer-ocr-api/

From us-central.unstract.com we created under **LLM Whisperer** an API key

URL: https://us-central.unstract.com/llm-whisperer/fa5686..........635/playground

## Using Unstract 

Steps to use Unstract

* create a prompt studio
* in setting (gear) set the default LLM


[Troubleshooting FAQ](https://docs.unstract.com/unstract/unstract_faq/troubleshooting/unstract_faq_troubleshooting/)

### Setting up a Connector

#### Google drive
We wanted to use google drive as a connector but it appears we need a client_id to get authencated but set one up
we need access to the Google Console API which requires a admin account in Google Workspace which is of course
not free. I did create an account pjmd @ptech-app.com but did not follow through as I didn't want to pay.

#### Local MySql

See [installation guide](https://documentation.ubuntu.com/server/how-to/databases/install-mysql/) to set it up.


#### SqlDeveloper

Trying SqlDeveloper to connect to the DB.

For SqlDevolper install first Java sudo apt install openjdk-25-jdk

Download [SqlDeveloper](https://www.oracle.com/database/sqldeveloper/technologies/download/) OtherPlatforms.

[info](https://www.youtube.com/watch?v=trVCq2SMWKM)

```
cd ~/Downloads/
unzip  sqldeveloper-24.3.1.347.1826-no-jre.zip
sudo mv sqldeveloper /opt
sh /opt/sqldeveloper/sqldeveloper.sh
```
** SqlDeveloper did not work because JAVA security issue.**

Instead we decided to use a VS Code extension to connect and interact with the DB

#### SQLTools VS Code extension

First we created a user pjmd 
```
sudo mysql -u root
mysql> CREATE USER 'pjmd'@'localhost' IDENTIFIED WITH caching_sha2_password BY '****';
mysql> GRANT SELECT on *.* TO 'pjmd'@'localhost';
mysql> exit
```

From extension in VS Code we installed (SQLTools)[https://www.youtube.com/watch?v=KSxqIDOP1G0] from Matheues Teixeira, then we let it find a list of driver and select from the list MySql.

We created a connection. MySql port is 3606 user pjmd, password to is set to AskOnConnect (VS code will prompt us at the top 
for the password)since we haven't created a DB yet, we set database to sys and the connection timeout to 30.

Next we create a DB.
```
sudo mysql -u root
mysql> create database ETL_DB;
mysql> show databases;
mysql> exit
```

Finally we update the extension connection settings with pjmd and ETL_DB.

#### Setting the Unstract Connector to MySql

First we change the MySql bind address so it accepts connection fron any interface.

```
$ sudo vi /etc/mysql/mysql.cnf
[mysql]
bind-address            = 0.0.0.0
$ sudo systemctl restart mysql.service
```

I set up the **pjmd user** so it could connect from all possible addresses

My host IP is 192.168.1.73

```
sudo mysql -u root

mysql> CREATE USER 'pjmd'@'192.168.1.73' IDENTIFIED WITH caching_sha2_password BY 'philippe';
Query OK, 0 rows affected (0.02 sec)

mysql> GRANT SELECT on *.* TO 'pjmd'@'172.18.0.13';
Query OK, 0 rows affected (0.01 sec)
```

Then I created the Connect with 192.168.1.73 but when I tested it I realized it did not accept a connection
from the docker service IP 172.18.0.13.

So I added pjmd user at 172.18.0.13 and it fixed the issue.

```
mysql> CREATE USER 'pjmd'@'172.18.0.13' IDENTIFIED WITH caching_sha2_password BY 'philippe';
Query OK, 0 rows affected (0.02 sec)

mysql> GRANT SELECT on *.* TO 'pjmd'@'172.18.0.13';
Query OK, 0 rows affected (0.01 sec)
```
#### More SQL

General commands from **mysql -u root**
```
mysql> show databases;

mysql> connect information_schema;

mysql> show tables;

select * from USER_privileges;

```


Specifi user commands

```
mysql> GRANT insert,delete,update,create  on *.* TO 'pjmd'@'localhost';
Query OK, 0 rows affected (0.01 sec)

mysql> GRANT insert,delete,update,create  on *.* TO 'pjmd'@'192.168.1.73';
Query OK, 0 rows affected (0.00 sec)

mysql> GRANT insert,delete,update,create  on *.* TO 'pjmd'@'172.18.0.13';
Query OK, 0 rows affected (0.01 sec)

mysql> select * from USER_privileges where grantee like '%pjmd%';

Or to mak sure

mysql> GRANT ALL PRIVILEGES  on *.* TO 'pjmd'@'172.18.0.13';
Query OK, 0 rows affected (0.01 sec)

mysql> GRANT ALL PRIVILEGES  on *.* TO 'pjmd'@'192.168.1.73';
Query OK, 0 rows affected (0.01 sec)

mysql> GRANT ALL PRIVILEGES  on *.* TO 'pjmd'@'localhost';
Query OK, 0 rows affected (0.01 sec)

```

Remember to create the user first the grant the right:
```
mysql> CREATE USER 'pjmd'@'172.18.0.%' IDENTIFIED BY 'philippe';
Query OK, 0 rows affected (0.01 sec)

mysql> GRANT ALL PRIVILEGES  on *.* TO 'pjmd'@'172.18.0.%';
Query OK, 0 rows affected (0.01 sec)

```

## Create an ETL pipline

Follow these [instructions](https://docs.unstract.com/unstract/unstract_platform/etl_pipeline/set_up_etl_pipeline/)

As input connector we used SFPT.

See these [instructiopns](https://sftpcloud.io/learn/sftp/ubuntu-sftp-server-how-to-install-and-configure) to install it.


