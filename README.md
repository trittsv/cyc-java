<!--
Copyright(c) 2026 Sven Trittler

This program and the accompanying materials are made available under the
terms of the Eclipse Public License v. 2.0 which is available at
http://www.eclipse.org/legal/epl-2.0, or the Eclipse Distribution License
v. 1.0 which is available at
http://www.eclipse.org/org/documents/edl-v10.php.

SPDX-License-Identifier: EPL-2.0 OR BSD-3-Clause
-->

# Cyclone DDS Java

Java binding for Cyclone DDS using the **official OMG DDS-Java API** and generated
IDL4-Java types. Generate your types once, then run with a platform runtime JAR.
The JAR contains the API, Java provider, JNI and matching Cyclone DDS native core;
it extracts and loads native libraries automatically.

**Status:** experimental, with a tested API subset. Full OMG profile conformance
and production maturity are not established. See [coverage](docs/manual/standards.rst)
and the [RTI/OpenSplice comparison](docs/manual/comparison.rst).

## 1. Generate a type

Save this as `Message.idl`:

```idl
module Example {
  @final struct Message { @key long id; string text; };
};
```

With an installed binding and Cyclone DDS compiler, use this `CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.21)
project(MyTypes LANGUAGES Java)
find_package(CycloneDDSJava CONFIG REQUIRED)
idljava_generate(TARGET message-types FILES Message.idl)
```

```sh
cmake -S . -B type-build \
  -DCMAKE_PREFIX_PATH="/path/to/java-binding;/path/to/cyclonedds-install"
cmake --build type-build --parallel
```

This produces `type-build/message-types.jar`. Generated classes have a default
constructor, an all-values constructor, and `get_id()` / `set_id(...)` accessors.
No application-specific C library is generated or needed.
Optional/external fields, inheritance, recursive types, mutable XCDR1/XCDR2, wide text
and arrays of sequences are supported; see [richer IDL schemas](docs/manual/howtos/idl.rst)
for examples, annotations and restrictions.

Already have generated types? Start with the next step. The compiler and CMake
are build-time tools, not application runtime dependencies.

## 2. Use the type

Save as `Demo.java`. It publishes and receives one sample through the standard API:

```java
import Example.Message;
import java.util.Set;
import java.util.concurrent.TimeUnit;
import org.omg.dds.core.ServiceEnvironment;
import org.omg.dds.core.WaitSet;
import org.omg.dds.core.status.PublicationMatchedStatus;
import org.omg.dds.domain.DomainParticipantFactory;

public final class Demo {
    public static void main(String[] args) throws Exception {
        System.setProperty("org.omg.dds.serviceClassName",
            "org.eclipse.cyclonedds.omg.CycloneServiceEnvironment");
        var env = ServiceEnvironment.createInstance(Demo.class.getClassLoader());
        var waitset = WaitSet.newWaitSet(env);
        try (var participant = DomainParticipantFactory.getInstance(env).createParticipant(0);
             var topic = participant.createTopic("ExampleMessage", Message.class);
             var publisher = participant.createPublisher();
             var subscriber = participant.createSubscriber();
             var writer = publisher.createDataWriter(topic);
             var reader = subscriber.createDataReader(topic);
             var ready = reader.select().getCondition()) {
            var matched = writer.getStatusCondition();
            matched.setEnabledStatuses(Set.of(PublicationMatchedStatus.class));
            waitset.attachCondition(matched);
            if (writer.getPublicationMatchedStatus().getCurrentCount() == 0)
                waitset.waitForConditions(5, TimeUnit.SECONDS);
            waitset.detachCondition(matched);
            waitset.attachCondition(ready);
            writer.write(new Message(42, "Hello from Java!"));
            waitset.waitForConditions(5, TimeUnit.SECONDS);
            for (var sample : reader.select().take(null)) {
                if (sample.getData() != null)
                    System.out.println(sample.getData().get_text());
            }
        } finally {
            if (waitset instanceof AutoCloseable resource) resource.close();
        }
    }
}
```

Choose the JAR matching your OS/CPU. For example, on macOS arm64:

```sh
DDS_JAR=/path/to/java-binding/share/java/cyclonedds-java-0.1.0-macos-aarch64.jar
javac -cp "$DDS_JAR:type-build/message-types.jar" Demo.java
java -cp "$DDS_JAR:type-build/message-types.jar:." Demo
```

Output: `Hello from Java!`. No `System.loadLibrary`, `java.library.path`, or
separate Cyclone DDS installation is required **at runtime** for the bundled
platform JAR. JDK 17+ is required. Use `linux-x86_64` or `linux-aarch64` artifacts
on Linux; Windows remains unverified. Windows classpaths use `;` instead of `:`.

For separate publisher/subscriber processes, see
[HelloWorld](examples/helloworld/java/org/eclipse/cyclonedds/examples/HelloWorld.java).
The [manual](docs/manual/index.rst) covers QoS, instance selectors, conditions,
type mappings and installation.

Readers and writers also support listener callbacks, standard status getters,
query conditions and content filters, next-instance traversal, key lookup and
timestamped registration overloads. See the short [operation examples and
restrictions](docs/manual/howtos/operations.rst), including callback threading and
the supported SQL subset.

## Build the binding

Requirements: JDK 17+, CMake 3.21+, a C11 compiler and a Cyclone DDS installation
with `idlc`, IDL/DDSI development headers and type-library support. The tested core
revision is the unmodified `3a5526315e64ac1d2a52e6ac1ae7f3e861c593d3` (11.0.1).
No Cyclone DDS core patches are required. Build that core first:

```sh
git clone https://github.com/eclipse-cyclonedds/cyclonedds.git core
git -C core checkout 3a5526315e64ac1d2a52e6ac1ae7f3e861c593d3
cmake -S core -B core-build -DCMAKE_INSTALL_PREFIX="$PWD/core-install" \
  -DBUILD_TESTING=OFF -DBUILD_EXAMPLES=OFF -DENABLE_SECURITY=OFF -DENABLE_SSL=OFF
cmake --build core-build --parallel
cmake --install core-build
```

Then build and install the binding:

```sh
cmake -S . -B build -DCMAKE_PREFIX_PATH="$PWD/core-install"
cmake --build build --parallel
ctest --test-dir build --output-on-failure
cmake --install build --prefix /path/to/java-binding
cmake --build build --target package
```

The build creates the standalone platform JAR, thin development JARs, JNI runtime
and IDL backend. CPack produces development `.tar.gz` / `.zip` archives and SHA-256
checksums. Nothing is published automatically. The tests cover wire formats,
standard API behavior, three independent C interoperability paths, a Java-only
installed consumer, and bundled-native loading with native search paths disabled.
Local validation is macOS arm64; Linux/macOS CI is configured.

The [upstream test ledger](tests/upstream/README.md) compares pinned fresh Python
and C++ downloads with runnable Java scenarios. It records remaining gaps;
complete upstream test parity is not yet established.

For Maven, the platform JAR can be installed locally as a classified artifact:

```sh
mvn install:install-file -Dfile="$DDS_JAR" \
  -DgroupId=org.eclipse.cyclonedds -DartifactId=cyclonedds-java \
  -Dversion=0.1.0-SNAPSHOT -Dclassifier=macos-aarch64 -Dpackaging=jar
```

```xml
<dependency>
  <groupId>org.eclipse.cyclonedds</groupId>
  <artifactId>cyclonedds-java</artifactId>
  <version>0.1.0-SNAPSHOT</version>
  <classifier>macos-aarch64</classifier>
</dependency>
```

These coordinates describe a local install; this project is not yet published to
Maven Central. `mvn package` currently builds Java only. See
[distribution details](docs/manual/distribution.rst) and
[documentation build instructions](docs/how-to-build.md).

## License

EPL 2.0 or EDL 1.0, using the exact [LICENSE](LICENSE) from the Cyclone DDS Python
binding. Bundled third-party components retain their notices; see
[THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).

XCDR2 is the default. To use XCDR1, set the writer's representation policy to
`policyFactory.Representation().withValue((short) 0)`; readers can accept both
with `withValue((short) 0, (short) 2)`. Optional and mutable fields and top-level appendable evolution are supported
within the unmodified core's representation limits. Generated type support records
those limits and rejects unsupported XCDR1 endpoint configurations. Regenerate
existing types when upgrading.
See [IDL and wire formats](docs/manual/howtos/idl.rst) for nested appendable
layout restrictions.

Discovery metadata is available through `participant.getBuiltinSubscriber()`,
`reader.getMatchedPublicationData(handle)` and
`writer.getMatchedSubscriptionData(handle)`. The
[operations guide](docs/manual/howtos/operations.rst) includes a built-in reader
example and documents the available metadata.

## Runnable examples

See [the example catalog](examples/README.md) for HelloWorld, roundtrip, throughput,
vehicles, asynchronous reception, all four built-in topics, DDS listeners,
waitsets and instance lifecycle. Build with `-DBUILD_EXAMPLES=ON` and start with:

```sh
java @build/examples/java.args org.eclipse.cyclonedds.examples.SimpleHelloWorld
java @build/examples/java.args org.eclipse.cyclonedds.examples.Listeners
```
