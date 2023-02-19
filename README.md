# Build a RISC-V QEMU Docker

```
docker run --rm --privileged --net host multiarch/qemu-user-static --reset -p yes
docker buildx build --network host --cache-from=type=local,src=/root/dockercache/qemu --cache-to=type=local,dest=/root/dockercache/qemu,mode=max . -t docker-qemu-riscv64 --load
```

Note that the `multiarch/qemu-user-static` updates binfmt so that RISC-V binaries could be recognizable, or the `docker build` command may fail.
If the reset command fails by reporting `sh: write error: Invalid argument`, please remove the [`-p yes`](https://github.com/multiarch/qemu-user-static/issues/100)

# Run the RISC-V QEMU Docker

```
docker run -it --rm docker-qemu-riscv64 uname -a
> Linux ce7b6092122a 5.10.134-12.2.al8.x86_64 #1 SMP Thu Oct 27 10:07:15 CST 2022 riscv64 GNU/Linux
```

Please have fun.

# Things deserving mention as an OpenJDK developer

1. Better to use vfork instead of spawn if you use a high version JDK (like JDK13+) and meet the following error when starting a child process:

```
This command is not for general use and should only be run as the result of a call to
ProcessBuilder.start() or Runtime.exec() in a java application
Error getting java.specification.version for <jdk>: java.io.IOException: Cannot run program "<jdk>/bin/java": error=0, Failed to exec spawn helper: pid: 545, exit value: 1
```

Use `export JAVA_TOOL_OPTIONS="-Djdk.lang.Process.launchMechanism=vfork"` and
pass `-e JAVA_TOOL_OPTIONS="-Djdk.lang.Process.launchMechanism=vfork"` to jtreg
instead. For all commands are pulled up by qemu-user-static, jspawnhelper would retrieve the command line including qemu and Java.
But what we want is only the Java part, so jspawnhelper will fail.

Then you can run jtregs inside the container:

```
export JAVA_TOOL_OPTIONS="-Djdk.lang.Process.launchMechanism=vfork"

root@07e9e42c06db:/jdk/workspace/openjdk-jdk# ./jtreg/bin/jtreg -v:error,fail -jdk:/jdk/workspace/jdk -timeout:10 -e JAVA_TOOL_OPTIONS="-Djdk.lang.Process.launchMechanism=vfork" test/hotspot/jtreg/compiler/
Picked up JAVA_TOOL_OPTIONS: -Djdk.lang.Process.launchMechanism=vfork
Picked up JAVA_TOOL_OPTIONS: -Djdk.lang.Process.launchMechanism=vfork
Picked up JAVA_TOOL_OPTIONS: -Djdk.lang.Process.launchMechanism=vfork
--------------------------------------------------
TEST: compiler/allocation/TestAllocArrayAfterAllocNoUse.java
TEST RESULT: Passed. Execution successful
--------------------------------------------------
TEST: compiler/allocation/TestNewArrayOutsideLoopValidLengthTestInLoop.java
TEST RESULT: Passed. Execution successful
--------------------------------------------------
TEST: compiler/arguments/TestUseCompiler.java
TEST RESULT: Passed. Execution successful
--------------------------------------------------
TEST: compiler/arguments/TestTraceICs.java
TEST RESULT: Passed. Execution successful
--------------------------------------------------
TEST: compiler/allocation/TestCCPAllocateArray.java
TEST RESULT: Passed. Execution successful
--------------------------------------------------
TEST: compiler/allocation/TestNewArrayBadSize.java
TEST RESULT: Passed. Execution successful
--------------------------------------------------
TEST: compiler/allocation/TestAllocation.java
TEST RESULT: Passed. Execution successful
--------------------------------------------------
TEST: compiler/arraycopy/ACasLoadsStoresBadMem.java
TEST RESULT: Passed. Execution successful
--------------------------------------------------
TEST: compiler/arguments/TestStressOptions.java
TEST RESULT: Passed. Execution successful
--------------------------------------------------
TEST: compiler/arraycopy/TestACSameSrcDst.java
TEST RESULT: Passed. Execution successful
--------------------------------------------------
TEST: compiler/arraycopy/TestArrayCloneBadAssert.java
TEST RESULT: Passed. Execution successful
--------------------------------------------------
...
```

Though several tests would fail because `JAVA_TOOL_OPTIONS` would print messages to the error stream. See [JDK-8219323](https://bugs.openjdk.org/browse/JDK-8219323) for more details.
