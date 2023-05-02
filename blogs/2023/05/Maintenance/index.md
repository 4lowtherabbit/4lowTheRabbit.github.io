# How does the cloud upgrade customer instances - Concept and Best Practice

The worker instances of customer sites are virtual machines running server OS. The VMs and their host servers are upgraded regularly to keep them secure and healthy.

A well-designed upgrade process can keep customer sites high available through maintenance. IaaS and PaaS take different processes.

## Infrastructure as a Service (IaaS)

IaaS provides virtual machines to cloud customers. Cloud customers manage the software environment of their VMs.

IaaS uses the following upgrade process to upgrade the host environment of a virtual machine.

|Steps|Cloud Platform|Site Admin|Website Requests Routing|Web Application|
|-|-|-|-|-|
|1|Notify Admin| |**Instance 0 (Active) - To be upgraded**<br/>Instance 1 (Inactive)|Running in both instances|
|2| |Switch|Instance 0 (Inactive) - To be upgraded<br/>**Instance 1 (Active)**|Running in both instances|
|3|Apply Maintenance| |Instance 0 (Inactive) - Upgrading<br/>**Instance 1 (Active)**|Running in both instances|
|4| |Switch back |**Instance 0 (Active) - Upgraded**<br/>Instance 1 (Inactive)|Running in both instances|

### Pros
1. Can achieve zero down time through maintenance.
1. The web application does not need to restart in any instance.

   Friendly to slow or fragile to start up web applications.

1.	Maintenance time widow of the instance can be planned out of business hours.
1. A simple and easy-to-understand process for site admins with an on-prem background.
1. Less cloud maintenance activities because the process is mostly for host environment upgrades.

### Cons
1. Additional cost by the standby instances.
1. Higher manual efforts required to site admins, especially when sites have lost of worker instances.
1. Site admins need to schedule and maintain software environment inside the virtual machine by themselves.

## Platform as a Service (PaaS)

|Steps|Cloud Platform|Website Requests Routing|Web Application|
|-|-|-|-|
|1| |**Instance 0 (Active) - To be upgraded**|Running in #0|
|2|Allocate new|**Instance 0 (Active) - To be upgraded**<br/>Instance 1 (Inactive)|Running in #0<br/>**Starting up in #1**|
|2|Overlapped|**Instance 0 (Active) - To be upgraded**<br/>**Instance 1 (Active)**|Running in both|
|3|Switch|Instance 0 (Inactive) - To be upgraded<br/>**Instance 1 (Active)**|Running in both
|4|Shut down|Instance 0 (Inactive) - To be upgraded<br/>**Instance 1 (Active)**|Shutting down from #0<br/>Running in #1
|5|Deallocate|Instance 1 (Active)|Running in #1|


### Pros
1. Can achieve zero down time through maintenance too.
1. No additional cost by any standby instance.
1. Transparent to site admins, delegated to the cloud platform.

### Cons

1. The web application needs to start up in a new instance quickly and successfully.

   In other words, it requires a cloud-friendly web application.

1. Difficult to predict accurately for when the maintenance will be applied to a specific instance.

   See [Scale Unit](#Scale-Unit)

1. More upgrade activities, because the process update both the host environment and the stack inside the virtual machine.

1. A new process far site admins who used to the IaaS process.

### Scale Unit

As illustrated in the above process, PaaS allocates a new instance to the site and deallocates the original instance after the maintenance. That implies PaaS has a pool of available instances.

Taking Azure App Service for example: App Service has hundreds of instances in each scale unit. The instances in a scale unit can be allocated to various roles and different customer sites of that scale unit dynamically.

An App Service scale unit uses a global maintenance job to upgrade all its instances, instead of one job per site. The instances are divided into a few [update domains](https://www.rupeshtiwari.com/azure-update-domain-vs-fault-domain/#why-group-servers-in-update-domains). The job shuts down instances in turn to upgrade them. It could take hours or days to walk through all the instances in each update domain. It is difficult to predict when a specific instance of a site in a scale unit will be upgraded during the long maintenance batch.

### Common Questions to the PaaS Upgrade Process
1. What if my web application fails to start up quickly and successfully in the new instance.

   A cloud-ready web application should be design be resilient to restarts, that can start up quickly and successfully.

   Some safeguarding features can used to handle unexpected start up failures. Taking Azure App Service for example:

   * To reserve more time for the web application to start up before a new instance starts to take requests.
  
     Use [Warm Up](https://azure.github.io/AppService/2020/05/15/Robust-Apps-for-the-cloud.html#use-application-initialization).

   * To automatically heal the application if it does not start up successfully.

     Use [Auto Heal](https://azure.github.io/AppService/2020/05/15/Robust-Apps-for-the-cloud.html#auto-heal) to restart the application in-place of the worker instance.
     Use [Health Check](https://azure.github.io/AppService/2020/05/15/Robust-Apps-for-the-cloud.html#set-your-health-check-path) to replace an unhealthy instance.

     Most start-up errors are caused by web applications and can be healed by restarting the worker process of the web application only.
     
     Auto Heal restarts are light-weight actions. It is not limited by any replacing quota limit. 

1. I received maintenance notification emails from the cloud platform. Will my site be down through the maintenance?

   You receive email notifications either because you previously subscribed to future maintenance activities of a scale unit or is enlisted by the platform.
   
   As illustrated above, the PaaS maintenance process is designed be transparent to cloud customers.

   Please review and follow the best practices in [The Ultimate Guide to Running Healthy Apps in the Cloud](https://azure.github.io/AppService/2020/05/15/Robust-Apps-for-the-cloud.html#set-your-health-check-path) for a resilient web application in the cloud.

1. I see keyword "Unplanned" in the notification email. Is there any 0-day security vulnerability that you are patching in a hurry?

   Having an unplanned maintenance does not imply there is a 0-day vulnerability found in the cloud.
   
   For example: The PaaS environment updates are rolled out through deployment rings. If a regression is found in an outer ring, the deployment would be paused, and an unplanned fix would need to be applied to inner ring instances who deployed the same build previously.
   
   PaaS maintains not only the host servers but also the app stack insider the worker instances. Potentially there would be more planned and unplanned maintenance activities in PaaS, while PaaS guarantees the same high availability as IaaS.

1. Why was my site restarted at business hours.

   An App Service scale unit can take hours or days to upgrade. Although the maintenance job is started out of the local business hours, the job's execution would extend to business hours and we would see restarts of a specific instance at business hours.

   On the other hand, seeing restarts at business hours does not mean site availability is impacted. Please review and follow the best practices in [The Ultimate Guide to Running Healthy Apps in the Cloud](https://azure.github.io/AppService/2020/05/15/Robust-Apps-for-the-cloud.html#set-your-health-check-path) for a resilient web application in the cloud.

1. My worker instances depend on a file storage instance. What is the maintenance process of upgrading a file storage instance?

   * File storage instances follow the same overlapped restart process of PaaS for maintenance.

   * Because the web applications in worker instances depend on the file storage, they need to restart too to apply the change.
     
     The web applications are restarted in the worker instances overlapped and in-place to achieve high availability through the restarts.
      
     However, because two processes of the web application run side-by-side in a worker instance, memory, CPU, and IO resource could be tensed during the overlapped restarts. App Service could alter to non-overlapped restarts if [density](https://azure.github.io/AppService/2020/05/15/Robust-Apps-for-the-cloud.html#minimize-app-service-plan-density) is too high.
     
    Please review and follow the best practices in [The Ultimate Guide to Running Healthy Apps in the Cloud](https://azure.github.io/AppService/2020/05/15/Robust-Apps-for-the-cloud.html#set-your-health-check-path) for a resilent web application in the cloud.
