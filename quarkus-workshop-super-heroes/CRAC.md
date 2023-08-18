# Running Super-heroes demo with CRaC

1. Make sure that JAVA_HOME points to CRaC JVM, and ensure that CRaC works (CRIU permissions etc).
2. Checkout and build Super-heroes:

```
git clone https://github.com/quarkusio/quarkus-workshops
cd quarkus-workshops/quarkus-workshop-super-heroes
./mvnw clean package -DskipTests -Pcomplete
```

3. Run the infrastructure:
```
docker-compose -f super-heroes/infrastructure/docker-compose.yaml up -d
```

4. Start the UI
```
$JAVA_HOME/bin/java -jar super-heroes/ui-super-heroes/target/quarkus-app/quarkus-run.jar
```

## Optional part:

5. In individual consoles start the apps
```
$JAVA_HOME/bin/java -XX:CRaCCheckpointTo=/tmp/heroes -jar super-heroes/rest-heroes/target/quarkus-app/quarkus-run.jar 
$JAVA_HOME/bin/java -XX:CRaCCheckpointTo=/tmp/villains -jar super-heroes/rest-villains/target/quarkus-app/quarkus-run.jar
$JAVA_HOME/bin/java -XX:CRaCCheckpointTo=/tmp/fights \
  -Dquarkus.http.cors.origins='*' \
  -Dcom.arjuna.ats.internal.arjuna.utils.processImplementation=com.arjuna.ats.internal.arjuna.utils.UuidProcessId \ 
  -jar super-heroes/rest-fights/target/quarkus-app/quarkus-run.jar
```

6. Open http://localhost:8080, click on 'New fighters' and 'Fight' a few times. Note that the 'fights' the first request usually fails due to a timeout; this is unrelated to CRaC. Subsequent requests usually succeed.

7. Checkpoint the three microservices:

```
for pid in $(jps -l | grep super-heroes/rest- | cut -f 1 -d ' '); do jcmd $pid JDK.checkpoint; done;
```

Without patched libraries you'll see several errors (exact form depends on the version of CRaC):

```
1586473:
An exception during a checkpoint operation:
jdk.crac.CheckpointException
	at java.base/jdk.crac.Core.checkpointRestore1(Core.java:182)
	at java.base/jdk.crac.Core.checkpointRestore(Core.java:287)
	at java.base/jdk.crac.Core.checkpointRestoreInternal(Core.java:303)
	Suppressed: jdk.crac.impl.CheckpointOpenSocketException: tcp6 localAddr ::ffff:127.0.0.1 localPort 48768 remoteAddr ::ffff:127.0.0.1 remotePort 5432
		at java.base/jdk.crac.Core.translateJVMExceptions(Core.java:120)
		at java.base/jdk.crac.Core.checkpointRestore1(Core.java:186)
		... 2 more
1586666:
CR: Checkpoint ...
1586548:
An exception during a checkpoint operation:
jdk.crac.CheckpointException
	at java.base/jdk.crac.Core.checkpointRestore1(Core.java:122)
	at java.base/jdk.crac.Core.checkpointRestore(Core.java:246)
	at java.base/jdk.crac.Core.checkpointRestoreInternal(Core.java:262)
	Suppressed: java.nio.channels.IllegalSelectorException
		at java.base/sun.nio.ch.EPollSelectorImpl.beforeCheckpoint(EPollSelectorImpl.java:384)
		at java.base/jdk.crac.impl.AbstractContextImpl.beforeCheckpoint(AbstractContextImpl.java:66)
		at java.base/jdk.crac.impl.AbstractContextImpl.beforeCheckpoint(AbstractContextImpl.java:66)
		at java.base/jdk.crac.Core.checkpointRestore1(Core.java:120)
		... 2 more
	Suppressed: jdk.crac.impl.CheckpointOpenSocketException: tcp6 localAddr ::ffff:127.0.0.1 localPort 51778 remoteAddr ::ffff:127.0.0.1 remotePort 9092
		at java.base/jdk.crac.Core.translateJVMExceptions(Core.java:91)
		at java.base/jdk.crac.Core.checkpointRestore1(Core.java:145)
		... 2 more
	Suppressed: jdk.crac.impl.CheckpointOpenSocketException: tcp6 localAddr ::ffff:127.0.0.1 localPort 52200 remoteAddr ::ffff:127.0.0.1 remotePort 5432
		at java.base/jdk.crac.Core.translateJVMExceptions(Core.java:91)
		at java.base/jdk.crac.Core.checkpointRestore1(Core.java:145)
		... 2 more
	Suppressed: jdk.crac.impl.CheckpointOpenResourceException: anon_inode:[eventpoll]
		at java.base/jdk.crac.Core.translateJVMExceptions(Core.java:97)
		at java.base/jdk.crac.Core.checkpointRestore1(Core.java:145)
		... 2 more
	Suppressed: jdk.crac.impl.CheckpointOpenResourceException: anon_inode:[eventfd]
		at java.base/jdk.crac.Core.translateJVMExceptions(Core.java:97)
		at java.base/jdk.crac.Core.checkpointRestore1(Core.java:145)
		... 2 more
	Suppressed: jdk.crac.impl.CheckpointOpenSocketException: tcp6 localAddr ::ffff:127.0.0.1 localPort 58142 remoteAddr ::ffff:127.0.0.1 remotePort 9092
		at java.base/jdk.crac.Core.translateJVMExceptions(Core.java:91)
		at java.base/jdk.crac.Core.checkpointRestore1(Core.java:145)
		... 2 more
```

## Replacing CRaC non-compatible artifacts

The vanilla version of Quarkus Superheroes relies on artifacts that do not handle checkpoint
and this would fail due to open files or sockets. Until CRaC becomes mainstream and these
libraries handle notifications we provide a CRaC-ed version with some of the problems fixed.

In order to identify these dependencies before running into errors we have created a Maven Enforcer
rule that can be included in the build and highlights the incompatible artifacts.
To use it conveniently we have added a parent module to the microservices and set up Maven Enforcer
in there:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-enforcer-plugin</artifactId>
            <version>3.3.0</version>
            <dependencies>
                <dependency>
                    <groupId>io.github.crac</groupId>
                    <artifactId>crac-enforcer-rule</artifactId>
                    <version>1.0.0</version>
                </dependency>
            </dependencies>
            <executions>
                <execution>
                    <goals>
                        <goal>enforce</goal>
                    </goals>
                </execution>
            </executions>
            <configuration>
                <rules>
                    <cracDependencies />
                </rules>
            </configuration>
        </plugin>
    </plugins>
</build>
```

Now when you try to build the project, the enforcer will spit out errors, including suggestions
for a replacement artifact. The offending dependencies can be removed using `<exclusions>`,
CRaC'ed dependencies should be added as any other dependency; usually these have `groupId` prefixed
with `org.github.crac.`, `artifactId` is identical and version is suffixed with `.CRAC.N` where `N`
stands for counter of CRaC-related changes; these should be added on top of the original (tagged) version.
It's likely that there won't be a CRaC'ed version exactly matching the original version; please test
for any compatibility issues thoroughly. 

```
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-enforcer-plugin:3.3.0:enforce (default) on project rest-villains: 
[ERROR] Rule 0: io.github.crac.CracDependencies failed with message:
[ERROR] io.quarkus.workshop.super-heroes:rest-villains:jar:1.0.0-SNAPSHOT
[ERROR]    io.quarkus:quarkus-resteasy-reactive:jar:2.15.3.Final
[ERROR]       io.quarkus.resteasy.reactive:resteasy-reactive-vertx:jar:2.15.3.Final
[ERROR]          io.vertx:vertx-web:jar:4.3.6
[ERROR]             io.vertx:vertx-web-common:jar:4.3.6
[ERROR]                io.vertx:vertx-core:jar:4.3.6 <--- replace with io.github.crac.io.vertx:vertx-core:4.3.8.CRAC.0
[ERROR]             io.vertx:vertx-auth-common:jar:4.3.6
[ERROR]                io.vertx:vertx-core:jar:4.3.6 <--- replace with io.github.crac.io.vertx:vertx-core:4.3.8.CRAC.0
[ERROR]             io.vertx:vertx-bridge-common:jar:4.3.6
[ERROR]                io.vertx:vertx-core:jar:4.3.6 <--- replace with io.github.crac.io.vertx:vertx-core:4.3.8.CRAC.0
[ERROR]             io.vertx:vertx-core:jar:4.3.6 <--- replace with io.github.crac.io.vertx:vertx-core:4.3.8.CRAC.0
[ERROR]          io.smallrye.reactive:smallrye-mutiny-vertx-core:jar:2.29.0
[ERROR]             io.smallrye.reactive:smallrye-mutiny-vertx-runtime:jar:2.29.0
[ERROR]                io.vertx:vertx-core:jar:4.3.6 <--- replace with io.github.crac.io.vertx:vertx-core:4.3.8.CRAC.0
[ERROR]             io.vertx:vertx-core:jar:4.3.6 <--- replace with io.github.crac.io.vertx:vertx-core:4.3.8.CRAC.0
[ERROR]       io.quarkus:quarkus-vertx-http:jar:2.15.3.Final
[ERROR]          io.smallrye.common:smallrye-common-vertx-context:jar:1.13.2
[ERROR]             io.vertx:vertx-core:jar:4.3.6 <--- replace with io.github.crac.io.vertx:vertx-core:4.3.8.CRAC.0
[ERROR]          io.smallrye.reactive:smallrye-mutiny-vertx-web:jar:2.29.0
[ERROR]             io.smallrye.reactive:smallrye-mutiny-vertx-uri-template:jar:2.29.0
[ERROR]                io.vertx:vertx-uri-template:jar:4.3.6
[ERROR]                   io.vertx:vertx-core:jar:4.3.6 <--- replace with io.github.crac.io.vertx:vertx-core:4.3.8.CRAC.0
[ERROR]    io.quarkus:quarkus-hibernate-orm-panache:jar:2.15.3.Final
[ERROR]       io.quarkus:quarkus-hibernate-orm:jar:2.15.3.Final
[ERROR]          io.quarkus:quarkus-agroal:jar:2.15.3.Final
[ERROR]             io.agroal:agroal-pool:jar:1.16 <--- replace with io.github.crac.io.agroal:agroal-pool:1.18.CRAC.0
```

## With patched libraries

8. Make sure that the non-patched version of the microservices is not running anymore, e.g. using Ctrl+C in console. 

9. Rebuild the patched microservices using `./mvnw clean package -DskipTests -Pcomplete`
 
10. Repeat steps 5 - 7, starting the apps, issuing a few requests through UI and performing the checkpoint.

The checkpoint might fail for some services: In this scenario Quarkus loads some classes during checkpoint
(e.g. from finalizers or when closing connections) opening some files. This may happen after the code that
was supposed to close all cached open files has executed, and leads to checkpoint failure.
If this is the case please trigger checkpoint again; this time everything should be loaded and the checkpoint should succeed. 

11. Restore the services in individual consoles:

```
$JAVA_HOME/bin/java -jar -XX:CRaCRestoreFrom=/tmp/heroes
$JAVA_HOME/bin/java -jar -XX:CRaCRestoreFrom=/tmp/villains
$JAVA_HOME/bin/java -jar -XX:CRaCRestoreFrom=/tmp/fights
```

12. Go to http://localhost:8080 and try to click through the UI few times. You can check consoles to see that everything works.

