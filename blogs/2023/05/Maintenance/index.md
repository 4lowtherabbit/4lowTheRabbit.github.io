# How does the cloud upgrade customer instances - Concept and Best Practice

The worker instances of customer sites are virtual machines running server OS. They and their host servers are upgraded regulary to keep them security and healthy.

A well designed upgrade process can keep customer sites high available through maintenances. IaaS and PaaS takes difference processes.
## Infrastracture as a Service (IaaS)

IaaS provides virtual machines whoes software environment is managed by cloud customers.

The following upgrade process is used when the host environment of a virutal machine needs to take a cloud side maintanance.

|Steps|Cloud Platform|Site Admin|Website Requests Routing|Web Application|
|-|-|-|-|-|
|1|Notify Admin| |**Instance 0 (Active) - To be upgraded**<br/>Instance 1 (Inactive)|Running in both instances|
|2| |Switch|Instance 0 (Inactive) - To be upgraded<br/>**Instance 1 (Active)**|Running in both instances|
|3|Apply Maintenance| |Instance 0 (Inactive) - Upgrading<br/>**Instance 1 (Active)**|Running in both instances|
|4| |Switch back |**Instance 0 (Active) - Upgraded**<br/>Instance 1 (Inactive)|Running in both instances|

### Pros
1. Can achieve zero down time through maintenance.
1. The web application doesn't need to restart in any instance.

   Friendly to slow or fragile to start up web applications.

1. Maintenance time widow can be strictly out of business hours.
1. A simple and easy process to understand.
1. Less upgrade activities, because the process is mostly for host environemnt upgrades.

### Cons
1. Additional cost by the standby instances.
1. High manual efforts required to site admins, espeically when sites have many worker instances.
1. Site admins need to schedule and maintain software environemnt inside the virtual machine by themselves.

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

1. The web application need to start up in a new instance quickly and successfully.

   In other words, it requires a cloud-friendly web application.

1. Difficult to predict accurately for when the maintenance will be applied to a specific instance.

   See [Scale Unit](#Scale-Unit)

1. More upgrade activities, because the process update both the host environment and the stack inside the virtual machine.

1. A new process to admins who used to the IaaS process.

### Scale Unit

As illustrated in the above process, PaaS allocates a new instance to the site and deallocates the original instance after the maintenance. That implies PaaS has a pool of available instances.

Taking Azure App Service for example: App Service has hundreds of instances in each scale unit. The instances in a scale unit can be allocated to different roles and different customer sites of that scale unit dynamically.

An App Service scale unit uses a global maintenance job to upgrade all its instances, instead of one job per site. The instances are divided into a few [update domains](https://www.rupeshtiwari.com/azure-update-domain-vs-fault-domain/#why-group-servers-in-update-domains). The job shuts down instances in turn to upgrade them. It could take hours or days to walk through all the instances in each update domains. It is difficult to predict when a specific instance of a site in a scale unit will be upgraded during the long maintenance batch.

### Common Questions to the PaaS Process
1. What if my web application fails to start up quickly and successfully in the new instance.

   A cloud-ready web application should be design be resilent to restarts, that can start up quickly and successfully.

   Some safe guarding features can used to handle unexpected start up failures. Taking Azure App Service for example:

   * To reserver more time for the web application to start up, because the new instance handles requests.
  
     Use [Warm Up](https://azure.github.io/AppService/2020/05/15/Robust-Apps-for-the-cloud.html#use-application-initialization).

   * To automatically heal the application if it doesn't start up successfully.

     Use [Auto Heal](https://azure.github.io/AppService/2020/05/15/Robust-Apps-for-the-cloud.html#auto-heal) to restart the application in-place of the worker instance.
     Use [Health Check](https://azure.github.io/AppService/2020/05/15/Robust-Apps-for-the-cloud.html#set-your-health-check-path) to replace an unhealthy instance.

     Most start-up errors are caused from the web application side and can be healed by restaring the worker process of the web application only.
     
     Auto Heal restarts only lighter-weighted and is not limited by any replacing quota limit. 

1. I received maintencne notification emails from the cloud platform. Will my site be down through the maintenance?

   As illustrated above,The PaaS maintenance process is designed be transparent to cloud customers.

   Please review and follow the best practices in [The Ultimate Guide to Running Healthy Apps in the Cloud](https://azure.github.io/AppService/2020/05/15/Robust-Apps-for-the-cloud.html#set-your-health-check-path) for a resilent web application in the cloud.

1. I am notified with the keyword "Unplanned", is there any 0-day security vulnerability that you are patching in a hurry?

   Having an unplanned mainteance doesn't mean there is a 0-day vulnerability found in the cloud.
   
   For example: The PaaS environment updates are rolled up through deployment rings. If a regression is found after deploying a release to an outter ring, the deployment would be paused and an unplanned fix would need to be applied to inner ring instances.
   
   PaaS maintains not only the host servers but the app stack insider the worker instances' virutal machines. Potentially there are more planned and unplanned maintenance activities in PaaS, while PaaS guaranteens high availability too.

1. Why my site was restarted at business hours.

   An App Service scale unit can take hours or days to upgrade. Although the mainteance job are triggered out of the business hours of the local region, the job's execution would extend to business hours and we would see restarts of a specific instance at business hours.

   On the other hand, seeing restarts at business hours doesn't mean site availability is impact. Please review and follow the best practices in [The Ultimate Guide to Running Healthy Apps in the Cloud](https://azure.github.io/AppService/2020/05/15/Robust-Apps-for-the-cloud.html#set-your-health-check-path) for a resilent web application in the cloud.

1. My worker instances are connected to a file storage instance. What is the maintenance process of upgrading the file storage instance?

   * File storage instances follow the same overlapped restart process for maintenance.

   * Because the web applications in worker instances depend on the file storage, they are restarted too to apply the change.
     
     They are restarted overlapped and in-place in the worker instance to achieve high availability through the restarts.
      
     However, because 2 instances of the web application run side-by-side in a worker instance, memory, CPU and IO resource could be tensed during the overlapped restarts. App Service could alter to non-overlapped restarts if [density](https://azure.github.io/AppService/2020/05/15/Robust-Apps-for-the-cloud.html#minimize-app-service-plan-density) is too high.
     
   Please review and follow the best practices in [The Ultimate Guide to Running Healthy Apps in the Cloud](https://azure.github.io/AppService/2020/05/15/Robust-Apps-for-the-cloud.html#set-your-health-check-path) for a resilent web application in the cloud.
