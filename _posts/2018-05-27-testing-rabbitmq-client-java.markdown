---
layout: post
title:  "Testing RabbitMQ Client in Java"
date:   2018-05-27 15:35:00 +0200
categories: java rabbitmq queue testing
---
Writing RabbitMQ client in Java is very well described in the [tutorial][rabbit-java-tutorial] and client [docs][rabbit-java-client], but writing test-coverage for such client can be tricky.

It makes sense to test message queue client against real message queue, and it makes sense to pick the message queue broker that can be embedded into tests code. The answer I found is Apache [QPID][apache-qpid].

Next in this post I will show how to set up embedded QPID broker and run send/receive test workload against it. Here I will provide the full solution without much comments (as it is extremely straightforward), but for more details like broker configuration etc the QPID [docs][qpid-java-broker] is a good starting point.

This is how to add required Java dependencies using Gradle.

```groovy
compile('com.rabbitmq:amqp-client:5.0.0')
testCompile('org.apache.qpid:qpid-broker:6.1.4')
```

This is QPID broker configuration file that will be used for this example.

```json
{
    "name" : "broker",
    "modelVersion" : "6.1",
    "accesscontrolproviders" : [ {
        "name" : "AllowAll",
        "type" : "AllowAll",
        "priority" : 9999
    } ],
    "authenticationproviders" : [ {
        "name" : "plain",
        "type" : "Plain",
        "users" : [ {
            "id" : "5c009963-eb49-4249-a150-833db543fe9e",
            "type" : "managed",
            "name" : "guest",
            "password" : "guest"
        } ]
    } ],
    "keystores" : [ {
        "name" : "default",
        "password" : "broker",
        "storeUrl" : "qpid/brokerkeystore"
    } ],
    "ports" : [ {
        "name" : "AMQP",
        "port" : "${qpid.amqp_port}",
        "authenticationProvider" : "plain",
        "keyStore" : "default",
        "transports" : [ "SSL" ],
        "virtualhostaliases" : [ {
            "name" : "defaultAlias",
            "type" : "defaultAlias"
        }, {
            "name" : "hostnameAlias",
            "type" : "hostnameAlias"
        }, {
            "name" : "nameAlias",
            "type" : "nameAlias"
        } ]
    } ],
    "virtualhostnodes" : [ {
        "name" : "default",
        "type" : "JSON",
        "defaultVirtualHostNode" : "true",
        "virtualHostInitialConfiguration" : "${qpid.initial_config_virtualhost_config}"
    } ]
}
```

For the sake of SSL support, configuration requires keystore. Useful links on keystores from [digitalocean][digitalocean-java-keystore] and [oracle][oracle-java-keystore]. The keystore described in this post looks like this:

```bash
$ keytool -list -keystore brokerkeystore
Enter keystore password:

Keystore type: JKS
Keystore provider: SUN

Your keystore contains 1 entry

broker, May 27, 2017, PrivateKeyEntry,
Certificate fingerprint (SHA1): <removed>
```

Now the RabbitMQ client code:

```java
public class RabbitClient implements Supplier<String>, Consumer<String> {
    private static final Logger logger = LoggerFactory.getLogger(RabbitClient.class);
    private static final String EXCHANGE_TYPE = "direct";

    private static final boolean QUEUE_DURABLE = true;
    private static final boolean QUEUE_EXCLUSIVE = false;
    private static final boolean QUEUE_AUTODELETE = false;

    public static class Config {
        final String host;
        final int port;
        final String virtualHost;
        final String username;
        final String password;
        final String queueName;
        final String exchangeName;
        final String routingKey;
        final boolean useSsl;

        public Config(
            String host,
            int port,
            String virtualHost,
            String username,
            String password,
            String queueName,
            String exchangeName,
            String routingKey,
            boolean useSsl
        ) {
            this.host = host;
            this.port = port;
            this.virtualHost = virtualHost;
            this.username = username;
            this.password = password;
            this.queueName = queueName;
            this.exchangeName = exchangeName;
            this.routingKey = routingKey;
            this.useSsl = useSsl;
        }
    }

    private final RabbitClient.Config config;
    private final ConnectionFactory connectionFactory;

    public RabbitClient(final RabbitClient.Config config) {
        this.config = config;
        this.connectionFactory = new ConnectionFactory();
        this.connectionFactory.setUsername(config.username);
        this.connectionFactory.setPassword(config.password);
        this.connectionFactory.setVirtualHost(config.virtualHost);
        this.connectionFactory.setHost(config.host);
        this.connectionFactory.setPort(config.port);
        if (config.useSsl) {
            try {
                this.connectionFactory.useSslProtocol();
            } catch (Exception e) {
                logger.error("Error when enabling SSL", e);
            }
        }
    }

    private String withConnection(Function<Connection, String> block) {
        try (Connection connection = connectionFactory.newConnection()) {
            return block.apply(connection);
        } catch (Exception e) {
            logger.error("Error during using rabbit connection", e);
            return null;
        }
    }

    @Override
    public void accept(String payload) {
        withConnection(c -> {
            try(Channel channel = c.createChannel()) {
                channel.exchangeDeclare(config.exchangeName, EXCHANGE_TYPE, true);
                channel.queueDeclare(config.queueName, QUEUE_DURABLE, QUEUE_EXCLUSIVE, QUEUE_AUTODELETE, null);
                channel.queueBind(config.queueName, config.exchangeName, config.routingKey);
                channel.basicPublish(
                    config.exchangeName,
                    config.routingKey,
                    true,
                    MessageProperties.PERSISTENT_TEXT_PLAIN,
                    payload.getBytes());
            } catch (Exception e) {
                logger.error("Creating rabbit channel failed", e);
            }
            return null;
        });
    }

    @Override
    public String get() {
        return withConnection(c -> {
            try(Channel channel = c.createChannel()) {
                channel.exchangeDeclare(config.exchangeName, EXCHANGE_TYPE, true);
                channel.queueDeclare(config.queueName, QUEUE_DURABLE, QUEUE_EXCLUSIVE, QUEUE_AUTODELETE, null);
                channel.queueBind(config.queueName, config.exchangeName, config.routingKey);
                GetResponse response = channel.basicGet(config.queueName, false);
                if (response != null) {
                    byte[] body = response.getBody();
                    long tag = response.getEnvelope().getDeliveryTag();
                    channel.basicAck(tag, false);
                    return new String(body);
                }
                return null;
            } catch (Exception e) {
                logger.error("Creating rabbit channel failed", e);
                return null;
            }
        });
    }
}
```

And finally the piece of code that checks that send() and receive() work as expected.

```java
public class RabbitClientTests {
    private static final Logger logger = LoggerFactory.getLogger(RabbitClientTests.class);

    private static final RabbitClient.Config config = new RabbitClient.Config(
        "localhost", 5672, "/", "guest", "guest", "q", "X", "key", true);

    private static final ConnectionFactory cf = new ConnectionFactory();
    static {
        cf.setUsername(config.username);
        cf.setPassword(config.password);
        cf.setHost(config.host);
        cf.setPort(config.port);
        cf.setVirtualHost(config.virtualHost);
        if (config.useSsl) {
            try {
                cf.useSslProtocol();
            } catch (NoSuchAlgorithmException | KeyManagementException e) {
                logger.error("Error when enabling SSL", e);
            }
        }
    }

    private static final Broker broker = new Broker();

    @BeforeClass
    public static void beforeClass() throws Exception {
        final BrokerOptions brokerOptions = new BrokerOptions();
        brokerOptions.setConfigurationStoreLocation("qpid/broker-config.json");
        broker.startup(brokerOptions);
    }

    @AfterClass
    public static void afterClass() {
        broker.shutdown();
    }

    private static Channel getChannel() throws Exception {
      Connection connection = cf.newConnection();
      Channel channel = connection.createChannel();
      channel.exchangeDeclare(config.exchangeName, "direct", true);
      channel.queueDeclare(config.queueName, false, false, false, null);
      channel.queueBind(config.queueName, config.exchangeName, config.routingKey);
      return channel;
    }

    private static void send(String message) throws Exception {
        Channel channel = getChannel();
        channel.basicPublish(config.exchangeName, config.routingKey, null, message.getBytes());
    }

    private static String recv() throws Exception {
        Channel channel = getChannel();
        GetResponse response = channel.basicGet(config.queueName, true);
        return new String(response.getBody());
    }

    @Test
    public void receive() throws Exception {
        String key = "{}";
        Supplier<String> supplier = new RabbitClient(config);
        send(key);
        assertEquals(key, supplier.get());
    }

    @Test
    public void send() throws Exception {
        String key = "{}";
        Consumer<String> consumer = new RabbitClient(config);
        consumer.accept(key);
        assertEquals(recv(), key);
    }
}
```

This is how one can test RabbitMQ client. Clear and without much comments, as promised.

[rabbit-java-tutorial]: https://www.rabbitmq.com/tutorials/tutorial-one-java.html
[rabbit-java-client]: https://www.rabbitmq.com/java-client.html
[apache-qpid]: https://qpid.apache.org/
[qpid-java-broker]: https://qpid.apache.org/releases/qpid-java-6.0.7/java-broker/book/Java-Broker-Introduction.html
[digitalocean-java-keystore]: https://www.digitalocean.com/community/tutorials/java-keytool-essentials-working-with-java-keystores
[oracle-java-keystore]: https://docs.oracle.com/cd/E19509-01/820-3503/ggfen/index.html
