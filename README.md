# AMQ 7 Interconnect Router RBAC Enforcement with LDAP


This project demonstrates how to setup an AMQ 7 Master / Slave cluster (using shared-nothing replication) together with an Interconnect Router acting as a front-end RBAC proxy.

![IC Router RBAC Enforcement with LDAP Diagram](images/logical-arch.png)

## Prerequisites

The following product / OS prerequisities exist:

* RHEL 7 (with an activated subscription)
* OpenJDK 1.8+
* AMQ 7.0.3+ Broker
* Interconnect Router 1.0+
* An LDAP server.  If you don't have an LDAP server, follow this [procedure](https://hub.docker.com/r/larrycai/openldap/) to install a Docker image with a pre-built LDAP server.  Additionally, follow this [tutorial](https://access.redhat.com/documentation/en-US/Fuse_ESB_Enterprise/7.1/html/ActiveMQ_Security_Guide/files/LDAP.html) to configure sample users / groups for your test.
* qtools (install from [here](https://github.com/ssorj/qtools)

## Procedure

### Test AMQ 7 Master / Slave Cluster

To get things started, we need to setup a master / slave broker cluster and ensure it works correctly with shared-nothing replication.

1. Follow the [instructions](https://github.com/jbossdemocentral/amq-ha-replicated-demo) for setting-up an AMQ 7 shared-nothing replication master / slave cluster.  You will need to increment the product version to `7.0.3` in the `init.sh` script contained in the project.
2. Update the Acceptor section of `/amq-broker-7.0.3/instances/replicatedMaster/etc/broker.xml` so we're listening on the correct network adapter:

```

      <acceptors>

         <!-- Acceptor for every supported protocol -->
         <acceptor name="artemis">tcp://0.0.0.0:61616?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=CORE,AMQP,STOMP,HORNETQ,MQTT,OPENWIRE;useEpoll=true;amqpCredits=1000;amqpLowCredits=300</acceptor>

         <!-- AMQP Acceptor.  Listens on default AMQP port for AMQP traffic.-->
         <acceptor name="amqp">tcp://0.0.0.0:5673?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=AMQP;useEpoll=true;amqpCredits=1000;amqpMinCredits=300</acceptor>

         <!-- STOMP Acceptor. -->
         <acceptor name="stomp">tcp://0.0.0.0:61613?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=STOMP;useEpoll=true</acceptor>

         <!-- HornetQ Compatibility Acceptor.  Enables HornetQ Core and STOMP for legacy HornetQ clients. -->
         <acceptor name="hornetq">tcp://0.0.0.0:5445?protocols=HORNETQ,STOMP;useEpoll=true</acceptor>

         <!-- MQTT Acceptor -->
         <acceptor name="mqtt">tcp://0.0.0.0:1883?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=MQTT;useEpoll=true</acceptor>

      </acceptors>
```
3. Do the same for `/amq-broker-7.0.3/instances/replicatedSlave/etc/broker.xml`:

```

      <acceptors>

         <!-- useEpoll means: it will use Netty epoll if you are on a system (Linux) that supports it -->
         <!-- amqpCredits: The number of credits sent to AMQP producers -->
         <!-- amqpLowCredits: The server will send the # credits specified at amqpCredits at this low mark -->

         <!-- Acceptor for every supported protocol -->
         <acceptor name="artemis">tcp://0.0.0.0:61716?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=CORE,AMQP,STOMP,HORNETQ,MQTT,OPENWIRE;useEpoll=true;amqpCredits=1000;amqpLowCredits=300</acceptor>

         <!-- AMQP Acceptor.  Listens on default AMQP port for AMQP traffic.-->
         <acceptor name="amqp">tcp://0.0.0.0:5772?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=AMQP;useEpoll=true;amqpCredits=1000;amqpMinCredits=300</acceptor>

         <!-- STOMP Acceptor. -->
         <acceptor name="stomp">tcp://0.0.0.0:61713?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=STOMP;useEpoll=true</acceptor>

         <!-- HornetQ Compatibility Acceptor.  Enables HornetQ Core and STOMP for legacy HornetQ clients. -->
         <acceptor name="hornetq">tcp://0.0.0.0:5545?protocols=HORNETQ,STOMP;useEpoll=true</acceptor>

         <!-- MQTT Acceptor -->
         <acceptor name="mqtt">tcp://0.0.0.0:1983?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=MQTT;useEpoll=true</acceptor>

      </acceptors>
```
4. Update `/amq-broker-7.0.3/instances/replicatedMaster/etc/bootstrap.xml` and `/amq-broker-7.0.3/instances/replicatedMaster/etc/bootstrap.xml` so that the web bind address is listening on the correct host e.g. `<web bind="http://0.0.0.0:XXXX" path="web">`
5. Test your setup as described in the instructions, ensuring that master / slave failover occurs as expected.  Ensure you can access both HawtIO consoles (on port 8161 + 8261).
6. For an additional test, try sending an AMQP message to either port 5673 (master) or 5772 (slave) depending on which broker is active.  You can use the following command for this:
```
qsend amqp://127.0.0.1:5673/haQueue -m Abc
```
If you browse for the message via HawtIO, you'll notice that Durable=false.  This means the message will disappear during failover and won't be replicated.  The test in step 5. was Durable=true, therefore the message should have been replicated


### Test LDAP connectivity with your broker cluster

Although we want to enforce RBAC at the IC router level, it's important to test out LDAP connectivity using our users / groups.  For this exercise, we'll use a single standalone broker instead of the cluster.

1. Create a new broker instance using the following command:

```
./bin/artemis create instances/adbroker
```

Use admin/admin for credentials and type 'Y' for anonymous access

2. Replace the `etc/broker.xml`, `etc/bootstrap.xml` and `etc/login.config` files with those contained in this project (found in `/conf/broker/ldap`)

3. Update the HawtIO role in artemis.profile with `-Dhawtio.role=admins`

4. Startup the broker using `bin/artemis run`

5. Test login to the HawtIO console using both your admin and user credentials.  Only the admin user should be authorized to enter the console.

6. Test sending messages using a known username e.g. jdoe:

```
./bin/artemis producer --message-count 10 --url "tcp://127.0.0.1:61616" --destination queue://defQueue --user jdoe --password sunflower
```

7. Test consuming messages using a known username e.g. jdoe:

```
./artemis consumer --message-count 10 --url "tcp://127.0.0.1:61616" --destination queue://defQueue --user jdoe --password sunflower
```

### Setup Interconnect Router for SSL Client connections

This procedure sets up PLAIN SSL for clients connecting via the router.  The router contains autolinks to a couple of static queues (queue.foo and queue.grs).  The router will loadbalance between the active/passive master/slave pair, depending on which is active.

1. Install the Interconnect Router on RHEL 7 as per the [instructions](https://access.redhat.com/documentation/en-us/red_hat_jboss_amq/7.0/html/using_amq_interconnect/installation)

2. Follow the instructions to setup an SSL profile from the [documentation](https://access.redhat.com/documentation/en-us/red_hat_jboss_amq/7.0/html/using_amq_interconnect/security#setting_up_ssl_for_encryption_and_authentication).  I used openssl to generate my cacerts and keystores: `openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem`.

3. Copy the `/conf/ic/qdrouterd.conf` to your RHEL server.

4. Update the qdrouterd.conf sslProfile section to include the correct path to your keys and keystores created in step 2.

5. Startup your master/slave brokers.  You should see the following in the router log indicating the connection to the master broker is active:

```
Fri Nov 10 13:17:52 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/0' on connection master
Fri Nov 10 13:17:52 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/1' on connection master
Fri Nov 10 13:17:52 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/2' on connection master
Fri Nov 10 13:17:52 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/3' on connection master
```

6. Send a test message to the master node by issuing the following command:

```
qsend amqps://127.0.0.1:5672/queue.foo -m Master
```

7. Login to the Master node HawtIO app.  You should see the message in the queue.foo by browsing.

8.  Failover to the Slave broker by killing the Master.  The following should appear in the router logs:

```
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Deactivated 'autoLink/0' on connection master
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Deactivated 'autoLink/1' on connection master
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Deactivated 'autoLink/2' on connection master
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Deactivated 'autoLink/3' on connection master
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/4' on connection slave
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/5' on connection slave
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/6' on connection slave
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/7' on connection slave
```

9. Send a test message to the slave node by issuing the following command:

```
qsend amqps://127.0.0.1:5672/queue.foo -m Slave
```

### Setup LDAP connection to IC Router

1. Ensure that the `cyrus-sasl-plain` package has been installed on your RHEL instance
2. Copy `/ic/sasl.qdrouterd.conf` file to your RHEL instance.  Rename the file to `qdrouterd.conf` and replace the same file in `/etc/sasl2` directory.
3. Restart your router.
4. Test your LDAP connectivity using `qdstat`:

```
qdstat -a --sasl-mechanisms=DIGEST-MD5 --sasl-username=admin --sasl-password=sunflower
PN_TRACE_FRM=1 qdstat -a --sasl-mechanisms=DIGEST-MD5 --sasl-username=admin --sasl-password=sunflower
```