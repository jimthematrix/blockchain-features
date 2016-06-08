# blockchain-features
Sandbox for developing features contributed to Hyperledger

## Persistent Blockchain Events
Services need to provide fault tolerance. In the case of the eventing system in Hyperledger, both the event source (the network made up of the validating peers) and event listeners must be fault tolerant such that: 
* if one validating peer crashed, the event listener may still get events from the network
* if the event listener crashed, it will be able to pick up where it left off when it is restored

Some preliminary work has been done for the fault tolerance feature in the 2nd case above. The work is saved in a repository forked from the official Hyperledger project: [https://github.com/jimthematrix/fabric](https://github.com/jimthematrix/fabric)

![Hyperledger support for message queue](https://github.com/jimthematrix/blockchain-features/blob/master/events/hyperledger-ent-int.jpg "Hyperledger support for message queue")

### Kafka Support
1. branch *[events-producer-kafka](https://github.com/jimthematrix/fabric/tree/events-producer-kafka)* has one of the approaches to provide integration with Apache Kafka, by modifying the fabric/events/producer package to pump messages into a Kafka topic

  To run this:

  * clone the repo and check out the branch
  * build the development environment by changing directory to the "fabric/devenv" folder and `vagrant up`
  * once the vagrant VM is successfully built, go into the VM host by `vagrant ssh`
  * `cd $GOPATH/src/github.com/hyperledger/fabric`
  * `make peer`
  * start a peer node and pass in the arguments to point the peer node at the Kafka server and topic (substitute "192.168.99.100" with IP of the docker host, and "hlevents" with any topic that exists in the Kafka server):

    `peer/peer node start --kafka-brokers=192.168.99.100:9092 --kafka-topic=hlevents`


2. branch *[events-listener-kafka](https://github.com/jimthematrix/fabric/tree/events-listener-kafka)* has another approach, by having the local block event listener process do the message pumping into a Kafka topic

  To run this:

  * clone the repo and check out the branch
  * build the development environment by changing directory to the "fabric/devenv" folder and `vagrant up`
  * once the vagrant VM is successfully built, go into the VM host by `vagrant ssh`
  * `cd $GOPATH/src/github.com/hyperledger/fabric`
  * `make peer`
  * start a peer node:

    `peer/peer node start`

  * build and start the event listener in fabric/examples/events/block-listener and pass in the following arguments (substitute "192.168.99.100" with IP of the docker host, and "hlevents" with any topic that exists in the Kafka server):

    `./block-listener -events-address=10.0.2.15:31315 -kafka-brokers=192.168.99.100:9092 -kafka-topic=hlevents`

If you don't have a Kafka server handy, the easiest is to use a docker image, follow the instructions here: [https://github.com/spotify/docker-kafka](https://github.com/spotify/docker-kafka)

Finally, start up a Kafka consumer to observe the messages produced by the peer (case #1 above) or the event listener (case #2). One easy way to get a Kafka consumer is installing the GO implementation:

In your vagrant VM host:

  * `go get github.com/Shopify/sarama/tools/kafka-console-consumer`
  * `kafka-console-consumer -topic=hlevents -brokers=192.168.99.100:9092`

Once the set up above is complete, you can test by using the "peer" command to submit transactions:

* `CORE_PEER_ADDRESS=10.0.2.15:30303 ./peer chaincode deploy -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02 -c '{"Function":"init", "Args": ["a","100", "b", "200"]}'`
* `CORE_PEER_ADDRESS=10.0.2.15:30303 ./peer chaincode invoke -n '<unique ID returned in the command result above>' -c '{"Function":"invoke", "Args": ["a","b","10"]}'`

### WebSphere MQ Support
This library requires WebSphere MQ client libraries for C to build and execute.

* Install the MQ Client on the same system where the Hyperledger code resides.  
  * Go to link [http://www-01.ibm.com/support/docview.wss?uid=swg24037500](http://www-01.ibm.com/support/docview.wss?uid=swg24037500)
  * Download a client version corresponding to the MQ server
  * Follow the [instructions here](https://www.ibm.com/support/knowledgecenter/SSFKSJ_8.0.0/com.ibm.mq.ins.doc/q009010_.htm) to install the client
* After install, double check the library folder needed by cgo linker
  * /opt/mqm/lib64 folder should contain file libmqm.so

1. branch *[events-listener-mq](https://github.com/jimthematrix/fabric/tree/events-listener-mq) modifies the block event listener in fabric/examples/events/block-listener to pump event messages into a WebSphere MQ queue

  To run this:

  * clone the repo and check out the branch
  * build the development environment by changing directory to the "fabric/devenv" folder and `vagrant up`
  * once the vagrant VM is successfully built, go into the VM host by `vagrant ssh`
  * `cd $GOPATH/src/github.com/hyperledger/fabric`
  * `make peer`
  * start a peer node:

    `peer/peer node start`

  * change directory to examples/events/block-listener and build the go program
    * If you get a build error saying "... Signal: killed", it usually means you don't have enough memory allocated for the vagrant VM. To fix the error, exit vagrant and modify vb.memory value to be at least "1024", and reload the new configuration by using command "vagrant reload"
  * start the event listener in fabric/examples/events/block-listener and pass in the following arguments:

    `LD_LIBRARY_PATH=/opt/mqm/lib64/ MQSERVER='HLCHANNEL/TCP/192.168.99.100 1414' ./block-listener -events-address=10.0.2.15:31315 -queue-manager=HL -queue=HL.QUEUE`

    Note: 
    * substitute `192.168.99.100` with IP of the MQ server
    * in the value string for MQServer, it's a space b/w the IP and port, rather than a colon
    * HLCHANNEL, HL and HL.QUEUE are the channel, queue manager and queue names respectively configured on the MQ server. Refer to instructions below for details to define them.

#### Set up a WebSphere MQ Queue Manager

* If you don't already have an MQ server, download and install a trial from [http://www.ibm.com/developerworks/downloads/ws/wmq/](http://www.ibm.com/developerworks/downloads/ws/wmq/)
* After install, first need to make an update to the VM configuration to expose the port needed for remote connections
  * if using vagrant, open Vagrantfile and add the following line in the configure section:

    `config.vm.network "forwarded_port", guest: 1414, host: 1414`

* switch to the user "mqm" that is required to run the following commands
* Create a Queue Manager: `crtmqm HL`
* Start the Queue Manager: `strmqm HL`
* Start the MQ command console to complete the remaining tasks:
  * `runmqsc`
  * `define qlocal(HL.QUEUE)` define a new queue dedicated to messages from the Hyperledger network
  * `set authrec objtype(QMGR) principal('vagrant') authadd(CONNECT)` allow the "vagrant" user to connect to the Queue Manager ("HL" defined above which is the target of the command console)
  * `set authrec profile(HL.QUEUE) objtype(QUEUE) principal('vagrant') authadd(PUT,GET)` allow the "vagrant" user to access the queue, this is the user account that the client program will be launch from
  * `define channel(HLCHANNEL) chltype(SVRCONN) trptype(TCP)` define a channel for the remote connection
  * `set chlauth(HLCHANNEL) type(ADDRESSMAP) address('192.168.99.1') mcauser('vagrant')` replace "192.168.99.1" with the IP for the host VM or machine of the MQ client install, in this case the vagrant host
  * `define listener(HLLISTENER) trptype(TCP) control(QMGR) port(1414)` define a listener to monitor port 1414 for incoming messages
  * `start listener(HLLISTENER)` start the listener
* The server is now ready to take remote connections for incoming messages
