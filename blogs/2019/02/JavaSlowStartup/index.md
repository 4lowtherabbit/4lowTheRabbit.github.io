# Workshop - Java Web App slow boot up



## Short Story


### Issue:
A few number of java apps are not starting up successfully. This seems to be happening with the ANT 79 upgrade. This does not appear to be related to any particular version of java. Java apps felt the impact of this the most if a startup time wasn’t long enough. It would result in a startup loop.

### Customer impact:
The java process tries to start up but ends up restarting.  Requests to the web app return 500 errors. 
 

### How to mitigation:
* Option 1:

  Increase startupTimeLimit to value at least 20% higher than current. If none was specified, the default is 120 seconds, so a new default of 180/240 will be appropriate. 

  <httpPlatform startupTimeLimit="180"  ..../>

* Option 2:

  Scale up app:
  Depending on current SKU/VM size, customers have a variety of options to scale up to.

* Option 3:

  Enable Local Cache Most incident were fixed by enabling local cache.



## Long Story

1) One day....

   Google’s Project Zero team discovered serious security flaws caused by “speculative execution,” a technique used by most modern processors (CPUs) to optimize performance.

   The Project Zero researcher, Jann Horn, demonstrated that malicious actors could take advantage of speculative execution to read system memory that should have been inaccessible. For example, an unauthorized party may read sensitive information in the system’s memory such as passwords, encryption keys, or sensitive information open in applications. Testing also showed that an attack running on one virtual machine was able to access the physical memory of the host machine, and through that, gain read-access to the memory of a different virtual machine on the same host.
   
   These vulnerabilities affect many CPUs, including those from AMD, ARM, and Intel, as well as the devices and operating systems running on them.

2) Azure deployed the security patch:
   
   Links about How Azure handle this vulnerablity:
   
   https://docs.microsoft.com/en-us/azure/virtual-machines/windows/mitigate-se
   https://support.microsoft.com/en-us/help/4073119/protect-against-speculative-execution-side-channel-vulnerabilities-in

   Other than what is in that article, we won’t go into detail on exactly what we do. 
   It impacted the Java apps that had low start up timeouts.  We won’t go into detail on what exact change caused that  - just that if they had lower start up times, to increase them and they will be fine.

   "Note Enabling mitigations that are off by-default may affect performance. The actual performance effect depends on multiple factors, such as the specific chipset in the device and the workloads that are running."

3) Someone else run some tests, got some results:
   
   How meltdown and spectre patches drag down older hardware
   https://www.pcworld.com/article/3250645/laptop-computers/how-meltdown-and-spectre-patches-drag-down-older-hardware.html

   "Generally, performance in common tasks will be hard for the average person to notice most of the time. So yeah, breathe a sigh of relief."

   And then sometimes, it’ll just hit you in the face, with wait times taking 25 percent more on I/O-intensive tasks, such as decompressing a file. "


## Troubleshooting and resolution

### How to handle the issue:
- Confirm the Tomcat boot up speed
  Get D:\home\LogFiles\catalina.xxxx-xx-xx.log
  ```
  01-Jun-2018 08:10:12.505 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server version:        Apache Tomcat/9.0.0.M26
  01-Jun-2018 08:10:12.974 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server built:          Aug 2 2017 20:29:05 UTC
  01-Jun-2018 08:10:12.974 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server number:         9.0.0.0
  01-Jun-2018 08:10:12.974 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log OS Name:               Windows NT (unknown)
  01-Jun-2018 08:10:12.974 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log OS Version:            10.0
  01-Jun-2018 08:10:12.974 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Architecture:          amd64
  01-Jun-2018 08:10:12.974 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Java Home:             D:\Program Files\Java\jdk1.8.0_111\jre
  01-Jun-2018 08:10:12.974 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log JVM Version:           1.8.0_111-b14
  01-Jun-2018 08:10:12.974 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log JVM Vendor:            Oracle Corporation
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log CATALINA_BASE:           D:\Program Files (x86)\apache-tomcat-9.0.0
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log CATALINA_HOME:           D:\Program Files (x86)\apache-tomcat-9.0.0
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log Command line argument:   -Djava.util.logging.config.file=D:\Program Files (x86)  \apache-tomcat-9.0.0\conf\logging.properties
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log Command line argument:   -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log Command line argument:   -Djavaagent=D:\home\site\wwwroot\webapps\ROOT\WEB-INF\applicationinsights-agen  t-2.0.2.jar
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log Command line argument:   -Dapplicationinsights.  configurationDirectory=D:\home\site\wwwroot\webapps\ROOT\WEB-INF\classes
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log Command line argument:   -Djdk.tls.ephemeralDHKeySize=2048
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log Command line argument:   -Djava.protocol.handler.pkgs=org.apache.catalina.webresources
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log Command line argument:   -Djava.util.logging.config.file=D:\Program Files (x86)  \apache-tomcat-9.0.0\conf\logging.properties
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log Command line argument:   -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log Command line argument:   -Dsite.logdir=d:\home\LogFiles\
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log Command line argument:   -Dsite.tempdir=D:\local\Temp
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log Command line argument:   -Dport.http=33732
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log Command line argument:   -Djava.net.preferIPv4Stack=true
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log Command line argument:   -Dcatalina.base=D:\Program Files (x86)\apache-tomcat-9.0.0
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log Command line argument:   -Dcatalina.home=D:\Program Files (x86)\apache-tomcat-9.0.0
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.startup.VersionLoggerListener.log Command line argument:   -Djava.io.tmpdir=D:\local\Temp
  01-Jun-2018 08:10:12.974 INFO [main]   org.apache.catalina.core.AprLifecycleListener.lifecycleEvent The APR based   Apache Tomcat Native library which allows optimal performance in production   environments was not found on the java.library.path: [D:\Program   Files\Java\jdk1.8.0_111\bin;D:\Windows\Sun\Java\bin;D:\Windows\system32;  D:\Windows;D:\Program Files (x86)\nodejs\6.9.1;D:\Windows\system32;D:\Windows;  D:\Windows\System32\Wbem;D:\Windows\System32\WindowsPowerShell\v1.0\;  D:\Program Files (x86)\Git\cmd;  D:\Users\Administrator\AppData\Local\Microsoft\WindowsApps;;D:\Program Files   (x86)\dotnet;  D:\Windows\system32\config\systemprofile\AppData\Local\Microsoft\WindowsApps;  D:\Program Files (x86)\PHP\v5.6;D:\Python27;;.]
  01-Jun-2018 08:10:18.847 INFO [main] org.apache.coyote.AbstractProtocol.init   Initializing ProtocolHandler ["http-nio-127.0.0.1-33732"]
  01-Jun-2018 08:10:19.832 INFO [main]   org.apache.tomcat.util.net.NioSelectorPool.getSharedSelector Using a shared   selector for servlet write/read
  01-Jun-2018 08:10:19.863 INFO [main]   org.apache.catalina.startup.Catalina.load Initialization processed in 16787 ms
  01-Jun-2018 08:10:20.195 INFO [main]   org.apache.catalina.core.StandardService.startInternal Starting service   [Catalina]
  01-Jun-2018 08:10:20.195 INFO [main]   org.apache.catalina.core.StandardEngine.startInternal Starting Servlet   Engine: Apache Tomcat/9.0.0.M26
  01-Jun-2018 08:10:22.190 INFO [main]   org.apache.catalina.startup.HostConfig.deployWAR Deploying web application   archive [D:\home\site\wwwroot\webapps\ROOT.war]
  01-Jun-2018 08:11:23.206 INFO [main]   org.apache.jasper.servlet.TldScanner.scanJars At least one JAR was scanned   for TLDs yet contained no TLDs. Enable debug logging for this logger for a   complete list of JARs that were scanned but no TLDs were found in them.   Skipping unneeded JARs during scanning can improve startup time and JSP   compilation time.
  01-Jun-2018 08:11:30.313 INFO [main]   org.apache.catalina.startup.HostConfig.deployWAR Deployment of web   application archive [D:\home\site\wwwroot\webapps\ROOT.war] has finished in   [68,064] ms
  01-Jun-2018 08:11:30.313 INFO [main] org.apache.coyote.AbstractProtocol.start   Starting ProtocolHandler ["http-nio-127.0.0.1-33732"]
  01-Jun-2018 08:11:30.343 INFO [main]   org.apache.catalina.startup.Catalina.start Server startup in 70482 ms
  ```

- Check the system CPU usage
  
- Try those 3 options

- If still not work
  
  * About TldScanner:
  
    org.apache.jasper.servlet.TldScanner.scanJars At least one JAR was scanned for TLDs yet contained no TLDs. Enable debug logging for this logger for a complete list of JARs that were scanned but no TLDs were found in them. Skipping unneeded JARs during scanning can improve startup time and JSP compilation time.
    
    https://wiki.apache.org/tomcat/HowTo/FasterStartUp

  * Try not to use .war file. Means you need to manully unpack the war file. So, Tomcat do not need to unpack it during the boot up.
  
    In D:\Program Files (x86)\apache-tomcat-8.5.6\conf\server.xml, you can find the following setting:
    ```
    <Host name="localhost" appBase="d:\home\site\wwwroot\webapps" xmlBase="d:\home\site\wwwroot\"
            unpackWARs="true" autoDeploy="true" workDir="${site.tempdir}">
    ```
    You will need to use your own tomcat to make change in server.xml. 