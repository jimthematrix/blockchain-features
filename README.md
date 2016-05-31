# blockchain-features
Sandbox for developing features contributed to Hyperledger

## Persistent Blockchain Events
Services need to provide fault tolerance. In the case of the eventing system in Hyperledger, both the event source (the network made up of the validating peers) and event listeners must be fault tolerant such that: 
* if one validating peer crashed, the event listener may still get events from the network
* if the event listener crashed, it will be able to pick up where it left off when it is restored

Some preliminary work has been done for the fault tolerance feature in the 2nd case above. The work is saved in a repository forked from the official Hyperledger project: [https://github.com/jimthematrix/fabric](https://github.com/jimthematrix/fabric)

* branch "persistent-events" has one of the approaches to provide integration with a message queue (Apache Kafka), by having the fabric/events/producer package be modified to pump messages into a Kafka topic

  To run this:

  * clone the repo and check out the branch
  * build the development environment by changing directory to the "fabric/devenv" folder and `vagrant up`
  * once the vagrant VM is successfully built, go into the VM host by `vagrant ssh`
  * `cd $GOPATH/src/github.com/hyperledger/fabric`
  * `make peer`
  * start a peer node and pass in the arguments to point the peer node at the Kafka server and topic (substitute "hlevents" with any topic that exists in the Kafka server):

    `peer/peer node start --kafka-brokers=192.168.99.100:9092 --kafka-topic=hlevents`


* branch "persistent-events-1" has another approach, by having the local block eventlistener process do the message pumping into a Kafka topic

  To run this:

  * clone the repo and check out the branch
  * build the development environment by changing directory to the "fabric/devenv" folder and `vagrant up`
  * once the vagrant VM is successfully built, go into the VM host by `vagrant ssh`
  * `cd $GOPATH/src/github.com/hyperledger/fabric`
  * `make peer`
  * start a peer node:

    `peer/peer node start`

  * build and start the event listener in fabric/examples/events/block-listener and pass in the following arguments (substitute "hlevents" with any topic that exists in the Kafka server):

    `./block-listener -events-address=10.0.2.15:31315 -kafka-brokers=192.168.99.100:9092 -kafka-topic=hlevents`

If you don't have a Kafka server handy, the easiest is to use a docker image, follow the instructions here: [https://github.com/spotify/docker-kafka](https://github.com/spotify/docker-kafka)

Once the set up above is complete, you can test by using the "peer" command to submit transactions:

* `CORE_PEER_ADDRESS=10.0.2.15:30303 ./peer chaincode deploy -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02 -c '{"Function":"init", "Args": ["a","100", "b", "200"]}'`
* `CORE_PEER_ADDRESS=10.0.2.15:30303 ./peer chaincode invoke -n '<unique ID returned in the command result above>' -c '{"Function":"invoke", "Args": ["a","b","10"]}'`