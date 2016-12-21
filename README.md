# Hyperledger fabric and chaincode deployment using docker (natively)

This guide assumes you have docker installed.

##Pull the docker images present on docker hub

~~~~~
	docker pull hyperledger/fabric-peer:latest
	docker pull hyperledger/fabric-membersrvc:latest

~~~~~



Create a docker-compose.yml file and paste the below content

~~~~~
membersrvc:
  image: hyperledger/fabric-membersrvc
  ports:
    - "7054:7054"
  command: membersrvc
vp0:
  image: hyperledger/fabric-peer
  ports:
    - "7050:7050"
    - "7051:7051"
    - "7053:7053"
  environment:
    - CORE_PEER_ADDRESSAUTODETECT=true
    - CORE_VM_ENDPOINT=unix:///var/run/docker.sock
    - CORE_LOGGING_LEVEL=DEBUG
    - CORE_PEER_ID=vp0
    - CORE_PEER_PKI_ECA_PADDR=membersrvc:7054
    - CORE_PEER_PKI_TCA_PADDR=membersrvc:7054
    - CORE_PEER_PKI_TLSCA_PADDR=membersrvc:7054
    - CORE_SECURITY_ENABLED=true
    - CORE_SECURITY_ENROLLID=test_vp0
    - CORE_SECURITY_ENROLLSECRET=MwYpmSRjupbT
  links:
    - membersrvc
  command: sh -c "sleep 5; peer node start --peer-chaincodedev"

~~~~~~


Run the docker compose using below command and it should start 2 docker conatiners

~~~~~
	docker-compose up

~~~~~


# Deploying the chaincode.

The chaincode example 02 is an simple chain code in which you can have "n" number of parties each having some account balance
You can then invoke the chaincode to transfer amount from one account to another and at any point query the ledger to check the balance.

After deploying we will have 2 parties "a" and "b" with account balance of "100" and "200" each.
We will then transform "20" from a to b and query the ledger to find the account balance of both a and b



# Building the chaincode

We will have to loginto the container created from the image "hyperledger/fabric-peer" So execute the below command and note down the name/id of the conatiner

~~~~~~
	docker ps 

~~~~~~


Sample output (It may differ based on the number of docker containers you are running)

~~~~~~

CONTAINER ID        IMAGE                           COMMAND                  CREATED             STATUS              PORTS                                                      NAMES
a09537ccf5ac        hyperledger/fabric-peer         "sh -c 'sleep 5; peer"   19 hours ago        Up 19 hours         0.0.0.0:7050-7051->7050-7051/tcp, 0.0.0.0:7053->7053/tcp   githubcom_vp0_1
00910248abae        hyperledger/fabric-membersrvc   "membersrvc"             19 hours ago        Up 19 hours         0.0.0.0:7054->7054/tcp                                     githubcom_membersrvc_1

~~~~~~


The container id/name in sample output is a09537ccf5ac / githubcom_vp0_1



Login to the container by below command

~~~~~~~
	docker exec -it <containerid> /bin/bash

~~~~~~~


Go to the directory where chaincode example 2 is present (in the container)

~~~~~~
	cd /opt/gopath/src/github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02

~~~~~~


Now build the chaincode using the below command

~~~~~~
	go build

~~~~~~


After execution when you do "ls" you should be able to see "chaincode_example02" created.

Run the following chaincode command to start and register the chaincode with the validating peer

~~~~~~~
	CORE_CHAINCODE_ID_NAME=mycc CORE_PEER_ADDRESS=0.0.0.0:7051 ./chaincode_example02

~~~~~~~


# Running the chaincode using API

All the enpoints are exposed on '<ip of host>:7050'


Registering user through API

The username and password are present in the file /opt/gopath/src/github.com/hyperledger/fabric/membersrvc/membersrvc.yaml 
Search for enrollmentID and enrollmentPW listed in the eca.users section to know more credentials if needed

This request contains user jim along with the password you can directly use it.


Rest request
~~~~~
	POST localhost:7050/registrar

	{
	  "enrollId": "jim",
	  "enrollSecret": "6avZQLwcUe9b"
	}
~~~~~

Output

~~~~~
	{
	  "OK": "User jim is already logged in."
	}
~~~~~




# Chaincode deploy

REST Request
~~~~~
	POST host:port>/chaincode
	{
	  "jsonrpc": "2.0",
	  "method": "deploy",
	  "params": {
	    "type": 1,
	    "chaincodeID":{
	        "name": "mycc"
	    },
	    "ctorMsg": {
	        "args":["init", "a", "100", "b", "200"]
	    },
	    "secureContext": "jim"
	  },
	  "id": 1
	}

~~~~~

Output
~~~~~
	{
	  "jsonrpc": "2.0",
	  "result": {
	    "status": "OK",
	    "message": "new"
	  },
	  "id": 1
	}
~~~~~




# Chaincode invoke

This is where we are performing a transaction of transferring 20 from a to b


~~~~~
	POST <host:port>/chaincode
	{
	  "jsonrpc": "2.0",
	  "method": "invoke",
	  "params": {
	      "type": 1,
	      "chaincodeID":{
	          "name":"mycc"
	      },
	      "ctorMsg": {
	         "args":["invoke", "a", "b", "20"]
	      },
	      "secureContext": "jim"
	  },
	  "id": 3
	}
~~~~~~

Output
~~~~~~~~~
	{
	  "jsonrpc": "2.0",
	  "result": {
	    "status": "OK",
	    "message": "b4188084-07b4-4bef-9387-93f3fec26749"
	  },
	  "id": 3
	}

~~~~~~~~~




# Chaincode Query

Query to find the value of "a"

~~~~~
	POST <host:port>/chaincode
	{
	  "jsonrpc": "2.0",
	  "method": "query",
	  "params": {
	      "type": 1,
	      "chaincodeID":{
	          "name":"mycc"
	      },
	      "ctorMsg": {
	         "args":["query", "a"]
	      },
	      "secureContext": "mycc"
	  },
	  "id": 5
	}
~~~~~~

Output
~~~~~~~~~
	{
	  "jsonrpc": "2.0",
	  "result": {
	    "status": "OK",
	    "message": "80"
	  },
	  "id": 5
	}

~~~~~~~~~

Query to find the value of "b"

~~~~~
	POST <host:port>/chaincode
	{
	  "jsonrpc": "2.0",
	  "method": "query",
	  "params": {
	      "type": 1,
	      "chaincodeID":{
	          "name":"mycc"
	      },
	      "ctorMsg": {
	         "args":["query", "b"]
	      },
	      "secureContext": "mycc"
	  },
	  "id": 5
	}
~~~~~~

Output
~~~~~~~~~
	{
	  "jsonrpc": "2.0",
	  "result": {
	    "status": "OK",
	    "message": "120"
	  },
	  "id": 5
	}

~~~~~~~~~