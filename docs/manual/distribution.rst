..
   Copyright(c) 2026 Sven Trittler

   This program and the accompanying materials are made available under the
   terms of the Eclipse Public License v. 2.0 which is available at
   http://www.eclipse.org/legal/epl-2.0, or the Eclipse Distribution License
   v. 1.0 which is available at
   http://www.eclipse.org/org/documents/edl-v10.php.

   SPDX-License-Identifier: EPL-2.0 OR BSD-3-Clause


Development distributions
=========================

Build and validate the binding before producing archives:

.. code-block:: sh

   cmake --build build --parallel
   ctest --test-dir build --output-on-failure
   cmake --build build --target package

CPack creates platform-specific ``cyclonedds-java-0.1.0-dev-*.tar.gz`` and ``.zip``
files and accompanying SHA-256 checksum files in the build directory. These are
experimental development artifacts, not official Eclipse releases. Nothing is
published by this command. Archive bit-for-bit reproducibility is not established.

A distribution includes:

* ``share/java/cyclonedds-java-0.1.0-<platform>.jar``: standalone runtime, APIs and bundled natives.
* ``share/java/omg-dds-api.jar``\ : unmodified normative DDS-Java API and Apache notices.
* ``share/java/idl4-java-api.jar``\ : supported sequence interfaces and project license.
* ``share/java/cyclonedds-java.jar``\ : provider, codec runtime and project notices.
* ``lib/libcycloneddsjava.*``\ : generic JNI runtime.
* ``lib/libcycloneddsidljava.*``\ : IDL compiler backend.
* ``lib/cmake/CycloneDDSJava``\ : downstream generator/package integration.
* ``share/doc/cyclonedds-java``\ : license, README, coverage and usage documentation.
* ``share/cyclonedds-java/consumer-example``\ : a standalone Java-only consumer project.

Running the standalone platform JAR requires JDK 17+ and the matching OS/CPU,
without a separate Cyclone DDS installation. It includes the core and JNI
libraries, extracts them to a private temporary directory, verifies SHA-256 digests,
and loads dependencies in order. The temporary filesystem must allow native code
to load. The platform JAR is not a universal cross-platform artifact.

Generating types or building the binding still requires the matching external
Cyclone DDS installation, compiler and development headers. Thin development JARs
remain available for explicit system-native loading. Windows is unverified.

Verify an extracted archive
---------------------------

Extract it into a fresh directory and configure the included consumer:

.. code-block:: sh

   cmake -S /path/to/distribution/share/cyclonedds-java/consumer-example \
     -B consumer-build \
     -DCMAKE_PREFIX_PATH="/path/to/distribution;/path/to/cyclonedds-install"
   cmake --build consumer-build --parallel
   ctest --test-dir consumer-build --output-on-failure

The consumer project declares only ``LANGUAGES Java``. It generates an appendable
keyed type, compiles against the normative API and generated types without the
provider JAR on its compilation classpath, then loads the bundled provider/core/JNI
and exchanges data using a native waitset. The root ``installed-consumer`` test
performs this install/configure/build/run workflow automatically.

The provider is selected by:

.. code-block:: text

   -Dorg.omg.dds.serviceClassName=org.eclipse.cyclonedds.omg.CycloneServiceEnvironment

At runtime, include only the platform runtime JAR and generated application JAR
on the classpath. Application code does not load libraries or set native paths.
The installed CMake package exposes ``CycloneDDSJava_BUNDLED_RUNTIME_JAR`` for
this purpose, alongside the thin API/runtime paths used for compilation.

Maven and CI
------------

``mvn package`` builds the combined Java artifact, with the normative API and license
notices. It does not build JNI, run ``idlc``\ , package native classifiers or execute
native integration tests. Use the CMake distribution for the complete toolchain.
The README shows how to install the platform JAR into a local Maven repository
as a classified artifact. Publishing platform artifacts and a generator plugin
remain future work; no Maven Central publication is claimed.

CI is configured to build the pinned core and Java binding, run integration tests,
then archive the platform distributions as workflow artifacts on Linux and macOS.
Local validation has been performed on macOS arm64. Remote CI and Windows results
must be verified separately.

Unmodified core and reproducibility
-----------------------------------

The binding builds against unmodified Cyclone DDS revision
``3a5526315e64ac1d2a52e6ac1ae7f3e861c593d3``. CI checks out and builds that revision
directly. No binding-specific core patches are applied, installed or bundled.
The standalone runtime JAR embeds the matching unmodified core and its original
license notices. Runtime behavior is verified against independent C peers built
with the same unmodified installation.

Core limitations remain explicit: unsupported enabled-reader time-based filter
updates are rejected, wchar/negative signed 64-bit union labels are diagnosed,
and generated representation restrictions keep unsupported schemas off XCDR1
endpoints. Java codec-only vectors do not establish native DDS interoperability.
