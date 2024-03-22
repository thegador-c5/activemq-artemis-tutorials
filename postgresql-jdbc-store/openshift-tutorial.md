# postgresql-jdbc-store

## Init
```
mkdir -p workspace
wget -O workspace/apache-artemis-2.32.0-bin.zip https://dlcdn.apache.org/activemq/activemq-artemis/2.32.0/apache-artemis-2.32.0-bin.zip
unzip workspace/apache-artemis-2.32.0-bin.zip -d workspace/apache-artemis-2.32.0-bin
mv workspace/apache-artemis-2.32.0-bin/apache-artemis-2.32.0 workspace/apache-artemis-2.32.0
rm -rf workspace/apache-artemis-2.32.0-bin*
```

## Openshift
Log in to your server by using the command `oc login` and create new project y using the command `oc new-project`, i.e.
```
oc login --token=sha256~n1eCPRPn2jSgVXJU8ObmfmvRqlIfFxz-MjJ2f6WnYqM --server=https://a603c8cd206bf4fe98f4d3817cda2dda-011da12348812db5.elb.us-east-1.amazonaws.com:6443
oc new-project postgresql-jdbc-store
```

## PostgreSQL
```
oc apply -f postgresql-jdbc-store/postgresql-install.yaml
```

## Certificates
```
wget -O workspace/server-keystore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/server-keystore.jks
wget -O workspace/server-ca-truststore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/server-ca-truststore.jks
wget -O workspace/client-ca-truststore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/client-ca-truststore.jks
oc create secret generic ext-acceptor-ssl-secret \
--from-file=broker.ks=workspace/server-keystore.jks \
--from-file=client.ts=workspace/client-ca-truststore.jks \
--from-literal=keyStorePassword=securepass \
--from-literal=trustStorePassword=securepass
```

## ArtemisCloud Operator
```
wget -O workspace/activemq-artemis-operator-1.1.0.zip https://github.com/artemiscloud/activemq-artemis-operator/releases/download/1.1.0/activemq-artemis-operator-1.1.0.zip
unzip workspace/activemq-artemis-operator-1.1.0.zip -d workspace/activemq-artemis-operator-1.1.0
rm -rf workspace/activemq-artemis-operator-1.1.0.zip
workspace/activemq-artemis-operator-1.1.0/install_opr.sh
```

## ActiveMQ Artemis
```
oc apply -f postgresql-jdbc-store/activemq-artemis-install.yaml
oc apply -f postgresql-jdbc-store/ext-acceptor-openshift-install.yaml
```

## Hosts
```
export BROKER_EXT_ACCEPTOR_HOST=$(oc get route broker-ext-acceptor-0-svc-rte -o json | jq -r '.spec.host')
export BROKER_CONSOLE_HOST=$(oc get route broker-wconsj-0-svc-rte -o json | jq -r '.spec.host')
```

## Producer
```
workspace/apache-artemis-2.32.0/bin/artemis producer --verbose --destination queue://TEST --user admin --password admin --protocol core --sleep 1000 --url "tcp://${BROKER_EXT_ACCEPTOR_HOST}:443?sslEnabled=true&verifyHost=false&trustStorePath=workspace/server-ca-truststore.jks&trustStorePassword=securepass&useTopologyForLoadBalancing=false"
```

## Verify the persistent messages
Checking the database - Find the pod running PostgresQL in the default namespace/project. From the Admin perspective Workloads -> Pods -> Terminal tab.

### List the tables
```
PGPASSWORD=postgres psql -h localhost -p 5432 --username postgres -c '\dt'
```

### Query the persistent messages
```
PGPASSWORD=postgres psql -h localhost -p 5432 --username postgres -c 'select count(*) from messages'
```

### Query the message data
```
PGPASSWORD=postgres psql -h localhost -p 5432 --username postgres -c 'select * from messages'
```

## Consumer
```
workspace/apache-artemis-2.32.0/bin/artemis consumer --verbose --destination queue://TEST --user admin --password admin --protocol core --sleep 1000 --url "tcp://${BROKER_EXT_ACCEPTOR_HOST}:443?sslEnabled=true&verifyHost=false&trustStorePath=workspace/server-ca-truststore.jks&trustStorePassword=securepass&useTopologyForLoadBalancing=false"
```

## Test
```
workspace/apache-artemis-2.32.0/bin/artemis check queue --name TEST --produce 10 --browse 10 --consume 10 --url "tcp://${BROKER_EXT_ACCEPTOR_HOST}:443?sslEnabled=true&verifyHost=false&trustStorePath=workspace/server-ca-truststore.jks&trustStorePassword=securepass&useTopologyForLoadBalancing=false"
```

## Resources
https://activemq.apache.org/components/artemis/documentation/latest/persistence.html#jdbc-persistence 
https://access.redhat.com/documentation/en-us/red_hat_amq_broker/7.11/html/configuring_amq_broker/assembly-br-persisting-message-data_configuring 
https://www.postgresql.org/docs/current/app-psql.html 
https://github.com/artemiscloud/activemq-artemis-operator/tree/1.1.0 

## Notes
dbruscin
  9 days ago
postgresql should be in the default namespace and the broker pod in the postgresql-jdbc-store.


dbruscin
  9 days ago
your ActiveMQArtemis CRD is not updated, indeed it doesn't include the new required field spec.resourceTemplates.patch (edited) 


dbruscin
  9 days ago
the tutorial requires the ActivemQArtemis CRD included in the upstream version 1.1.0
https://github.com/artemiscloud/activemq-artemis-operator/blob/1.1.0/deploy/crds/broker_activemqartemis_crd.yaml (edited) 


dbruscin
  9 days ago
I updated the ActiveMQArtemis CRD in your OpenShift cluster, now it is working as expected (edited) 


thegador
  9 days ago
Thank you!


thegador
  9 days ago
Is there any mapping between supported AMQ Broker versions and Operator versions that I can refer to?


dbruscin
  9 days ago
AMQ Broker Operator 7.12 will be based on upstream ArtemisCloud Operator 1.1.0


thegador
  9 days ago
Ok. Thank you!


thegador
  8 days ago
Hello Domenico, is it okay to share this tutorial with a customer who is starting a proof of concept?


dbruscin
  8 days ago
Hi Thomas, we can share it with a customer but we need to clarify that it is a POC and the official doc for AMQ Broker Operator 7.12 could suggest a different supported way to deploy a broker pod with JDBC storage because other improvements could be included in 7.12
