# blockchain-features
Sandbox for developing features contributed to Hyperledger

## Persistent Blockchain Events
Services need to provide fault tolerance. In the case of the eventing system in Hyperledger, both the event source (the network made up of the validating peers) and event listeners must be fault tolerant such that: 

* if one validating peer crashed, the event listener may still get events from the network
* if the event listener crashed, it will be able to pick up where it left off when it is restored

Some preliminary work has been done for the fault tolerance feature in the 2nd case above. The work is saved in a repository forked from the official Hyperledger project: [https://github.com/jimthematrix/fabric](https://github.com/jimthematrix/fabric)

![Hyperledger support for message queue](https://github.com/jimthematrix/blockchain-features/blob/master/events/hyperledger-ent-int.jpg "Hyperledger support for message queue")

### Extensible Messaging System Support Interface
The interface /github.com/hyperledger/fabric/events/producer.Connector defines the common behaviors of a message producer for a messaging system like Apache Kafka or WebSphere MQ. Extensions can be built on it to provide integration with an external messaging system. Support for the following systems have been implemented:

* Apache Kafka
* WebSphere MQ

##### Build
Branch *[events-producer-modular](https://github.com/jimthematrix/fabric/tree/events-producer-modular)* has both the interface declaration and the extensions. Follow these steps to build the code.

  * clone the repo and check out the branch
  * build the development environment by changing directory to the "fabric/devenv" folder and `vagrant up`
  * once the vagrant VM is successfully built, go into the VM host by `vagrant ssh`
  * follow the [instructions here](#mq-redist) to install the client libraries for WebSphere MQ
  * `cd $GOPATH/src/github.com/hyperledger/fabric`
  * `make peer`
    * If you get a build error saying "... Signal: killed", it usually means you don't have enough memory allocated for the vagrant VM. To fix the error, exit vagrant and modify vb.memory value to be at least "1024", and reload the new configuration by using command "vagrant reload"

##### Run
For Apache Kafka

* Set up a Kafka cluster. The easiest way is to use a docker image, follow the instructions here: [https://github.com/spotify/docker-kafka](https://github.com/spotify/docker-kafka)
* start a peer node and pass in the arguments to point the peer node at the messaging server and topic:

  `CORE_LOGGING_LEVEL=debug peer/peer node start --events-queue=kafka --kafka-brokers=192.168.99.100:9092 --kafka-topic=hlevents`

  Note: substitute "192.168.99.100" with IP of the zookeeper node(s) for the Kafka server/cluster, and substitute "hlevents" with any topic that exists in the Kafka server

* finally, start up a Kafka consumer to observe the messages produced by the peer. One easy way to get a Kafka consumer is installing the GO implementation:
  * in your vagrant VM host:

    `go get github.com/Shopify/sarama/tools/kafka-console-consumer`
    
    `kafka-console-consumer -topic=hlevents -brokers=192.168.99.100:9092`

* once the set up is complete, follow the [steps here](#test-tx) to submit test transactions and observe the events from the transaction processing displayed by the Kafka consumer

For WebSphere MQ

* Install a WebSphere MQ server and configure a remote queue manager by following [instructions here](#mq-install).
* start a peer node and pass in the arguments to point the peer node at the messaging server and topic:

  `CORE_LOGGING_LEVEL=debug MQSERVER='HLCHANNEL/TCP/192.168.99.100 1414' peer/peer node start --events-queue=wmq --queue-manager=HL --queue=HL.QUEUE`

  Note: 
  * substitute `192.168.99.100` with IP of the MQ server
  * in the value string for MQServer, it's a space b/w the IP and port, rather than a colon
  * HLCHANNEL, HL and HL.QUEUE are the channel, queue manager and queue names respectively configured on the MQ server. Refer to instructions below for details to define them.

* to check the resulted message that has been put on the queue, launch the following command from the MQ server machine or VM:

  `/opt/mqm/samp/bin/amqsget HL.QUEUE`

* once the set up is complete, follow the [steps here](#test-tx) to submit test transactions and observe the events from the transaction processing displayed by the MQ Get program

### Enhanced event listener approach
Another alternative to pumping the messages directly out of the peer node is to enhance the event listener client to connect with the messaging system instead. There are exemplary code below that demonstrates how that can be done for Apache Kafka and WebSphere MQ.

1. branch *[events-listener-kafka](https://github.com/jimthematrix/fabric/tree/events-listener-kafka)* has the local block event listener process do the message pumping into a Kafka topic

  To build and run this:

  * clone the repo and check out the branch
  * build the development environment by changing directory to the "fabric/devenv" folder and `vagrant up`
  * once the vagrant VM is successfully built, go into the VM host by `vagrant ssh`
  * `cd $GOPATH/src/github.com/hyperledger/fabric`
  * `make peer`
  * start a peer node:

    `peer/peer node start`

  * build and start the event listener in fabric/examples/events/block-listener and pass in the following arguments (substitute "192.168.99.100" with IP of the docker host, and "hlevents" with any topic that exists in the Kafka server):

    `./block-listener -events-address=10.0.2.15:31315 -kafka-brokers=192.168.99.100:9092 -kafka-topic=hlevents`

2. branch *[events-listener-mq](https://github.com/jimthematrix/fabric/tree/events-listener-mq) modifies the block event listener in fabric/examples/events/block-listener to pump event messages into a WebSphere MQ queue

  To run this:

  * clone the repo and check out the branch
  * build the development environment by changing directory to the "fabric/devenv" folder and `vagrant up`
  * once the vagrant VM is successfully built, go into the VM host by `vagrant ssh`
  * follow the [instructions here](#mq-redist) to install the client libraries for WebSphere MQ
  * `cd $GOPATH/src/github.com/hyperledger/fabric`
  * `make peer`
  * start a peer node:

    `peer/peer node start`

  * change directory to examples/events/block-listener and build the go program
    * If you get a build error saying "... Signal: killed", it usually means you don't have enough memory allocated for the vagrant VM. To fix the error, exit vagrant and modify vb.memory value to be at least "1024", and reload the new configuration by using command "vagrant reload"
  * start the event listener in fabric/examples/events/block-listener and pass in the following arguments:

    `MQSERVER='HLCHANNEL/TCP/192.168.99.100 1414' ./block-listener -events-address=10.0.2.15:31315 -queue-manager=HL -queue=HL.QUEUE`

    Note: 
    * substitute `192.168.99.100` with IP of the MQ server
    * in the value string for MQServer, it's a space b/w the IP and port, rather than a colon
    * HLCHANNEL, HL and HL.QUEUE are the channel, queue manager and queue names respectively configured on the MQ server. Refer to instructions below for details to define them.

### <a name="mq-redist"></a>Install Build and Runtime WebSphere MQ Pre-requisites
This support requires WebSphere MQ client libraries for C to build and execute.

* Download the MQ Client redistributable libraries on the same system where the Hyperledger code resides.
  * Go to [http://www-01.ibm.com/support/docview.wss?&uid=swg24037500](http://www-01.ibm.com/support/docview.wss?&uid=swg24037500)
  * Download a client redistributable version corresponding to the MQ server and the OS architecture of the peer node, such as _8.0.0.4-WS-MQC-Redist-LinuxX64_
  * Create a folder `mq-redist` in /home/vagrant directory
* Set the environment variable LD_LIBRARY_PATH to `/home/vagrant/mq-redist/lib64` (or `lib` for 32-bit systems). This is needed for both compile and execution of the WebSphere MQ support code.

### <a name="mq-install"></a>Set up a WebSphere MQ Queue Manager
If you don't already have an MQ server, download and install a trial from [http://www.ibm.com/developerworks/downloads/ws/wmq/](http://www.ibm.com/developerworks/downloads/ws/wmq/).

After install, first need to make an update to the VM configuration to expose the port needed for remote connections

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

### <a name="test-tx"></a>Test Transaction Submissions
Once the set up above is complete, you can test by using the "peer" command to submit transactions:

* `CORE_PEER_ADDRESS=10.0.2.15:30303 ./peer chaincode deploy -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02 -c '{"Function":"init", "Args": ["a","100", "b", "200"]}'`
* `CORE_PEER_ADDRESS=10.0.2.15:30303 ./peer chaincode invoke -n '<unique ID returned in the command result above>' -c '{"Function":"invoke", "Args": ["a","b","10"]}'`

