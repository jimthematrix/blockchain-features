# blockchain-features
Sandbox for developing features contributed to Hyperledger

* [Persistent Blockchain Events](#feature-events) (incubating)
* [Asynchronous Transaction Submission](#feature-async) (incubating)

## <a name="feature-events"></a>Persistent Blockchain Events
Services need to provide fault tolerance. In the case of the eventing system in Hyperledger, both the event source (the network made up of the validating peers) and event listeners must be fault tolerant such that: 

* if one validating peer crashed, the event listener may still get events from the network
* if the event listener crashed, it will be able to pick up where it left off when it is restored

Some preliminary work has been done for the fault tolerance feature in the 2nd case above. The work is saved in a repository forked from the official Hyperledger project: [https://github.com/jimthematrix/fabric](https://github.com/jimthematrix/fabric)

![Hyperledger support for message queue](https://github.com/jimthematrix/blockchain-features/blob/master/events/hyperledger-ent-int.jpg "Hyperledger support for message queue")

### Extensible Messaging System Support Interface
The interface /github.com/hyperledger/fabric/events/producer.Connector defines the common behaviors of a message producer for a messaging system like Apache Kafka or WebSphere MQ. Extensions can be built on it to provide integration with an external messaging system. 

The interface is designed as follows:

```go
type Connector interface {
  SystemName() string
  RuntimeFlags() [][]string
  Initialize() error
  Publish(msg *pb.Event) error
  Close() error
}
```

* `SystemName()` returns the flag value for identifying the external system, such as "kafka", "wmq" (for WebSphere MQ), which informs the framework to load the appropriate implementation
* `RuntimeFlags()` returns the two-dimensional string array that describes the command line flags needed by the Connector implementation, for instance an Apache Kafka connector requires the Kafka broker address string and the topic
* `Initialize()` is called when the events sub-system is initialized, usually connections to the external messaging system is established here
* `Publish()` is called when the event has been triggered to allow the Connectors to pass the event along to the messaging system
* `Close()` is called when the event sub-system is torn down, persistent connections should be closed at this point

Support for the following systems have been prototyped in this fork:

* Apache Kafka
* WebSphere MQ

##### Build
Branch *[events-producer-modular](https://github.com/jimthematrix/fabric/tree/events-producer-modular)* has both the interface declaration and the extensions. Follow these steps to build the code.

If you have an existing vagrant-based development environment:

  * clone the repo and check out the branch
  * change directory to the "fabric/devenv" folder and `vagrant ssh`
  * follow the [instructions here](#mq-redist) to install the client libraries for WebSphere MQ
  * `cd $GOPATH/src/github.com/hyperledger/fabric`
  * `make peer`
    * If you get a build error saying "... Signal: killed", it usually means you don't have enough memory allocated for the vagrant VM. To fix the error, exit vagrant and modify vb.memory value to be at least "1024", and reload the new configuration by using command "vagrant reload"

If you don't have an existing vagrant-based development environment, you need to follow some special steps first to set up the vagrant VM otherwise the vagrant build process (`vagrant up`) will fail.

  * clone the repo and check out the branch
  * follow these steps to re-build the base image for the vagrant environment
    * Go to [http://www-01.ibm.com/support/docview.wss?&uid=swg24037500](http://www-01.ibm.com/support/docview.wss?&uid=swg24037500)
  * Download a client version corresponding to the MQ server and the OS architecture of the peer node, such as _8.0.0.4-WS-MQC-LinuxX64_
    * Make the downloaded archive available via HTTP download. this can be done by firing up a local HTTP file server such as `python -m SimpleHTTPServer <port>` from the directory containing the archive
    * Open the file `fabric/images/base/scripts/common/setup.sh, replace the IP address and the file name of the following lines according to your set up:

      `wget http://192.168.99.1:8000/mqc8_8.0.0.4_linuxx86-64.tar.gz`

      `tar -xvf mqc8_8.0.0.4_linuxx86-64.tar.gz --directory mqc`

    * Change directory to `fabric/images/base` and launch command `make vagrant`. The the make script uses the following tools that must be installed first:
      * json parser: https://stedolan.github.io/jq/
      * packer tool: https://www.packer.io/
    * Upon successful completion, the vagrant base image `hyperledger/fabric-baseimage (virtualbox, 0)` will be visible from the output of command "vagrant box list". This will be the basis of the vagrant environment for event producer code.
  * Change directory to `fabric/devenv` and launch command `USE_LOCAL_BASEIMAGE=true vagrant up` to build the vagrant development environment for the fabric
  * Upon successful completion, `USE_LOCAL_BASEIMAGE=true vagrant ssh` to log in to the vagrant environment and your fabric peer code will have been built already

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
Another alternative, instead of pumping the messages directly out of the peer node, is to enhance the event listener client to connect with the external messaging system instead. There are exemplary code below that demonstrates how that can be done for Apache Kafka and WebSphere MQ.

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

## <a name="feature-async"></a>Asynchronous Transactions API
The asynchronous transactions API allows transactions to be submitted asynchronously via a message queue, to compensate for the speed gap b/w client submitting transactions and the Blockchain network's processing and committing the transactions.

Support for different message queues is extensible via the following interface:

```go
type Connector interface {
  SystemName() string
  RuntimeFlags() [][]string
  Start() error
  Close() error
}
```

The following message queues support have been prototyped:

* Apache Kafka
* WebSphere MQ

## Appendix
### <a name="mq-redist"></a>Install Build and Runtime WebSphere MQ Pre-requisites
This support requires WebSphere MQ client libraries for C to build and execute.

* Install the MQ Client on the same system where the Hyperledger code resides.
  * Go to [http://www-01.ibm.com/support/docview.wss?&uid=swg24037500](http://www-01.ibm.com/support/docview.wss?&uid=swg24037500)
  * Download a client version corresponding to the MQ server and the OS architecture of the peer node, such as _8.0.0.4-WS-MQC-LinuxX64_
  * Unzip to a folder and install from the list of rpm's: `sudo rpm -ivh *.rpm`
  * You may need to install rpm first with `sudo apt-get install rpm`
* Copy the needed libraries to the folder used by the linker to find shared libraries
  * `sudo cp /opt/mqm/lib64/* /usr/local/lib`

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

