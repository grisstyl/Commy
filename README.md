# Commy

Introducing **Commy**, a simple yet powerful framework which simplifies the use of [Plugin Messaging Channels](https://www.spigotmc.org/wiki/bukkit-bungee-plugin-messaging-channel/). _The name is not communist-inspired._ 

## Getting Started

These instructions will help you setup a plugin using _Commy_.

### Prerequisites

Java 8

You will need to download Maven, as the modules _commy-spigot_ and _commy-bungee_ needs to be shaded with their respective plugins.

### Installing

Your IDE most likely comes with maven pre-installed (like IntelliJ), but if it  doesn't you can [manually download Maven from here](https://maven.apache.org/download.cgi).

Add commy to your pom.xml file. **Note:** If you are making a Spigot plugin, use _commy-spigot_. If you are making a BungeeCord plugin, use _commy-bungee_.
````xml
<project>
    <repositories>
        <repository>
            <id>ossrh</id>
            <name>Commy Repository</name>
            <url>https://oss.sonatype.org/content/groups/public/</url>
        </repository>
    </repositories>
    
    <dependencies>
        <!-- Use this for any Spigot plugin -->
        <dependency>
            <groupId>com.github.expdev07</groupId>
            <artifactId>commy-spigot</artifactId>
            <version>1.4</version>
        </dependency>
        <!-- Use this for any BungeeCord plugin -->
        <dependency>
            <groupId>com.github.expdev07</groupId>
            <artifactId>commy-bungee</artifactId>
            <version>1.4</version>
        </dependency>
    </dependencies>
</project>
````

Maven repo can be found at [The Central Repository](https://search.maven.org/beta/search?q=g:com.github.expdev07). 

#### Shading
These dependencies are not packaged in Spigot or BungeeCord, so you have to manually shade them. It can be achieved by adding this to your pom.xml. Note that by default, gson will also be shaded. This is because it is not apparent in some Spigot versions, however, you can exclude gson from being shaded in by adding the configurations. [Here's how you use the Maven Shade Plugin](https://maven.apache.org/plugins/maven-shade-plugin/usage.html).

It is important that you have ``<scope>provided</scope>`` on the BungeeCord and Spigot dependency, otherwise they will also be shaded (which we do not want).

#### Compile
Now, to compile, just run ``mvn clean install``.

## Examples

You will find some examples below on how to use **Commy** with Spigot and BungeeCord. 

_Please note that both commy-spigot and commy-bungee uses the same base-interface, which means that setting it up will be the same, just different implementations (e.g BungeeCommy instead of SpigotCommy)._

### Bukkit/Spigot

Setting up _Commy_ and assigning a default handler, a handler for intercepting a string message, and a handler for receiving a custom object. The default handler will be used when a message does not find its pipe. You can set it as null (not set it).
```java
/**
 * A simple Spigot plugin demonstrating the use of Commy
 */
public class SpigotPlugin extends JavaPlugin {

    // Universal logger
    static Logger logger;

    // Pre-"defining" a commy at class-level
    private SpigotCommy commy;

    @Override
    public void onEnable() {
        logger = Bukkit.getLogger();

        // Initialize commy, calling setup will start the engines
        this.commy = new SpigotCommy(this);

        // Adding handlers, you can add as many as you want
        // The first parameter here is the "pipe" the handler will handle messages for
        commy.addHandler("test", new TestHandler());
        commy.addHandler("test_msg", new AbstractTestHandler());
    }

    /**
     * A method demonstrating some usage
     */
    private void sendMessage() {
        // Get a connection with a player
        Connection<Player> connection = commy.getConnection(Bukkit.getPlayer("ExpDev"));

        // Now, there are many ways you can send a message
        //   * You can just send a simple string
        connection.sendMessage("test", "This is a message");
        //   * You can send a custom object!
        connection.sendMessage("test", new Object());
        //   * You can send bytes like you normally would
        //     Use our helper class "BytesOutput" to quickly write to an array
        byte[] bytes = new BytesOutput()
                .write("a string", "another string")
                .getBytes();
        connection.sendMessage("test_proxy", bytes);

        // You can also "quick send" a message
        commy.sendMessage("test_proxy", "Message to send");
    }

    /**
     * Handles a test message. The parameter of MessageHandler is the type
     * of source we will communicate with, which for Spigot's case is
     * always Player
     */
    private static class TestHandler extends StringMessageHandler<Player> {

        @Override
        public void handle(Connection<Player> conn, String tag, String message) {
            // We know tag == test, otherwise it would have been intercepted through the default handler
            logger.log(Level.INFO, "Received a message through test from " + conn.getSender().getName() + ": " + message);

            // Respond! Here, the source we're communicating with will need to have a handler for the "test"
            // pipe, otherwise it will be rerouted to their default handler
            conn.sendMessage("test", "I heard your test message and is sending this back through the \"test\" pipe");
        }
    }

    /**
     * A simple handler to test out how to use the abstract handler to
     * send objects over the pipes
     */
    private static class AbstractTestHandler implements AbstractMessageHandler<Player, TestObject> {

        @Override
        public void handle(Connection<Player> conn, String tag, TestObject message) {
            // We received a "TestObject" object, manipulate it as you want
            logger.log(Level.INFO, String.format(
                    "Received a %s through %s from %s", message.getClass().getSimpleName(), tag, conn.getSender().getName())
            );
        }

        @Override
        public Class<TestObject> getMessageType() {
            // This is important as generics is not available in run-time,
            // which means this will have to manually be specified
            return TestObject.class;
        }

    }

}
```

### BungeeCord

Setting up _Commy_ in a BungeeCord plugin, then sending a custom object/message once a test message is received.
````java
/**
 * A simple Bungee plugin demonstrating the use of Commy
 */
public class BungeePlugin extends Plugin {

    // Universal logger
    private static Logger logger;

    // Pre-"defining" a commy at class-level
    private BungeeCommy commy;

    @Override
    public void onEnable() {
        logger = ProxyServer.getInstance().getLogger();

        // Initialize commy, calling setup will
        // start the engines
        this.commy = new BungeeCommy(this);

        // Adding handlers, you can add as many as you want
        commy.addHandler("test", new TestHandler());
    }

    /**
     * Handles a test message
     */
    private static class TestHandler implements MessageHandler<ServerInfo> {

        @Override
        public void handle(Connection<ServerInfo> conn, String tag, byte[] message) {
            // We know tag == test, otherwise it would have been intercepted through the default handler
            logger.log(Level.INFO, ("Received a message through test from " + conn.getSender().getName() + ": " + new String(message)));

            // Or... if you sent bytes, you can manipulate it like you normally would
            ByteArrayDataInput in = ByteStreams.newDataInput(message);
            logger.log(Level.INFO, in.readUTF());
            logger.log(Level.INFO, in.readUTF());

            // Let's respond by sending them a TestObject
            conn.sendMessage("test_msg", new TestObject("ExpDev", 2));
        }
    }

}
````

Find more examples [for SpigotMC](spigot-plugin) and [for BungeeCord](bungee-plugin).

## Deployment

Once you have shaded commy, all you have to do is put your plugin inside your _/plugins_ folder as regular.

## Built With

* [Gson](https://github.com/google/gson) - For deserialization/serialization of objects for sending over network.
* [Maven](https://maven.apache.org/) - Dependency Management framework.

## Contributing

Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on our code of conduct, and the process for submitting pull requests to us.

## Authors

* **Marius Richardsen** - *Initial work* - [ExpDev](https://github.com/ExpDev07)
* **Tyler Grissom** - *Contributor* - [grisstyl](https://github.com/grisstyl)

See also the list of [contributors](https://github.com/ExpDev07/Commy/contributors) who participated in this project.

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details

## Acknowledgments

* Dinnerbone for designing the Plugin Messaging Channels.
