# HTTP DDoS detector 
This repository contains an ONOS application that is focused to detect and mitigate HTTP DDoS attacks. Through the usage of a random forest classifier.

## Requirements:
- Intermediate Java knowledge
- SDN basics
- ONOS basics
- Random Forest classifier (Optional)

## Project structure
- [mx.itesm.api.flow](./src/main/java/mx/itesm/api/flow) contains the code that interacts with the REST flow api.
- [mx.itesm.httpddosdetector.classifier](./src/main/java/mx/itesm/httpddosdetector/classifier) contains a generic Classifier class that every classifier used should inherit from.
- [mx.itesm.httpddosdetector.classifier.randomforest](./src/main/java/mx/itesm/httpddosdetector/classifier/randomforest) contains the implementation of the random forest classifier that loads a JSON file containing a trained model
- [mx.itesm.httpddosdetector.flow.parser](./src/main/java/mx/itesm/httpddosdetector/flow/parser) contains the code implementation in java of [flowtbag](https://github.com/DanielArndt/flowtbag) to convert packets into flows
- [mx.itesm.httpddosdetector.keys](./src/main/java/mx/itesm/httpddosdetector/keys) contain keys used for identifying flows, attacks and distributed attacks.

## Processing packets 
In order to process and analyze the packets from the network traffic, we will use a [packet processor](http://api.onosproject.org/1.7.0/org/onosproject/net/packet/PacketProcessor.html). We will be based on an ONOS sample application from the onos [repository](https://wiki.onosproject.org/display/ONOS/Building+the+ONOS+Sample+Apps), to clone it run `git clone https://gerrit.onosproject.org/onos-app-samples`. In that repository we will use the **oneping** sample app, which process a packet and just allow one ping per minute.

### Converting packets into flows
Afte we have the packet processor ready, we need to convert the packets into flows so we can pass them through our classifier. To convert them we use the [FlowData](./flow/parser/FlowData.java) class to append each packet to its corresponding flow. 

## Detecting malicious flows
When a flow is closed, we can pass it through our classifiers, in this implementation we will use a random forest classifier. 

We have previously trained the model and you can find it in the resources folder [here](./src/main/resources/models/random_forest_bin.json). The classifier has to previously load the model with the _RandomForestClassifier.Load_ method, and after that we can use the _RandomForestClassifier.Classify_ to obtain the predicted class of the provided flow.

This is done on the HttpDdosDetector [here](./src/main/java/mx/itesm/httpddosdetector/HttpDdosDetector.java#L131-L41)

## Mitigating attacks
You can access the onos api docs through the following url http://192.168.99.124:8181/onos/v1/docs/ in there check the flow api to access the endpoints that will add or delete flow rules. For more info on flow rules check the wiki https://wiki.onosproject.org/display/ONOS/Flow+Rules. 

In order to block the attacker we will use the following criteria on the flow rule:

- IP_PROTO	Ip protocol	uint16 (0x06 TCP)
- TCP_DST	match port	int64 (the attacked host port)
- IPV4_SRC	source ip	string (the attacker host ip)
- IPV4_DST	destination ip	string (the attacked host ip)

To drop the matched packets we will leave empty the treatment property of the flow, so the JSON object required to add the flow rule will be:

```
{
  "flows": [
    {
      "priority": 40000,
      "timeout": 0,
      "isPermanent": true,
      "deviceId": "of:0000000000000001",
    // Leave treatment empty to drop the package
    // "treatment": {},
      "selector": {
        "criteria": [
          {
            "type": "IP_PROTO",
            "protocol": "0x06"
          },
          {
            "type": "TCP_DST",
            "tcpPort": 80
          },
          {
            "type": "IPV4_SRC",
            "ip": "192.168.1.2/24"
          },
          {
            "type": "IPV4_DST",
            "ip": "192.168.1.1/24"
          }
        ]
      }
    }
  ]
}
```

- run simple server on host `python -m SimpleHTTPServer 80`
- on karaf check the logs to see that the detector is running