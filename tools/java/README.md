# Tips and tricks around using Java

## Debugging
Attaching a debugger to a running JVM can be done using these commands
    jsadebugd <pid>
    jdb -connect sun.jvm.hotspot.jdi.SADebugServerAttachingConnector:debugServerName=localhost

Finding the right connector:
    jdb -listconnectors

See also the [jsadebug documentation](https://docs.oracle.com/javase/7/docs/technotes/tools/share/jsadebugd.html)
and the [JDK development tools](https://docs.oracle.com/javase/7/docs/technotes/tools/) as well as this
[discussion](https://stackoverflow.com/questions/376201/debug-a-java-application-without-starting-the-jvm-with-debug-arguments).

