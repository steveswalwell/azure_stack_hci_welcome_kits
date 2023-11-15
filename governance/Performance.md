[cpubottleneck]:https://learn.microsoft.com/en-us/windows-server/administration/performance-tuning/role/hyper-v-server/detecting-virtualized-environment-bottlenecks#processor-bottlenecks
[memorybottleneck]:https://learn.microsoft.com/en-us/windows-server/administration/performance-tuning/role/hyper-v-server/detecting-virtualized-environment-bottlenecks#memory-bottlenecks
[networkbottleneck]:https://learn.microsoft.com/en-us/windows-server/administration/performance-tuning/role/hyper-v-server/detecting-virtualized-environment-bottlenecks#network-bottlenecks
[storagebottleneck]:https://learn.microsoft.com/en-us/windows-server/administration/performance-tuning/role/hyper-v-server/detecting-virtualized-environment-bottlenecks#storage-bottlenecks
[storageqos]:https://learn.microsoft.com/en-us/windows-server/storage/storage-qos/storage-qos-overview
[createstorageqospilicy]:https://learn.microsoft.com/en-us/windows-server/storage/storage-qos/storage-qos-overview#BKMK_CreateQoSPolicies
[testrdma]:./../tools/testrdma.md
# Welcome Kit - Azure Stack HCI - Performance - Draft

Welcome to the "Azure Stack HCI - Performance Welcome Kit." This comprehensive guide is designed to provide you with essential insights into the areas of focus and consideration when managing your Azure Stack HCI Monitoring. As part of your preparation before engaging with Microsoft engineers, this resource aims to demystify these technologies, offering clear explanations and answers to frequently asked questions. Whether you're new to these concepts or seeking to deepen your understanding, this Welcome Kit is your stepping stone to navigating the world of modern cloud-native solutions. 

## Performance to Benchmark

When the Azure Stack HCI cluster is deployed then it can be advisable to perform some checks to ensure that it is understood the performance which can be expected from the cluster.  The types of areas to look to can be:

- CPU - this needs to be an understanding of what CPU usage looks like with no workloads, to understand with the features turned on what is the CPU usage of the system.  One of the major advantages of virtualization is that you can provision more resources than you have available and the workloads will only use what it needs up to the maximum you have configured.  It is then key to look at what "over provisioning" you are willing to tolerate.  
-  Memory - as with CPU, this needs to be an understanding of what memory usage looks like with no workloads, to understand with the features turned on what is the memory usage of the system.  One of the major advantages of virtualization is that you can provision more resources than you have available and the workloads will only use what it needs up to the maximum you have configured.  With memory you need to again consider over provisioning and how the memory will be allocated.
-  Storage - an understanding of the number of IOPS which can be expected and also ensure the configuration of the network is optimal for the storage networks

## CPU

With most clusters the nodes will have hyper threading enabled so on a system with 16 cores, 2 cores per node and 4 nodes in the cluster there will be 128 cores which can be allocated and there will still be the 1:1 physical to virtual core ratio.  To get the most out of the investment it is typical to see the physical to virtual core ratio to go as high as 1:6.   The key think to remember is that the high the ratio it is more likely that a VM will need to wait longer for  CPU cycle.  This can be detrimental for realtime workloads such as SQL.  

An additional consideration is also the size of the VM's which are created.  When a VM wishes to perform an action then there needs to be a physical core available for each virtual core that it has available.  This means that a VM which only has a single virtual CPU which be more likely to get CPU time then a VM with 8 vCPU's are the chances of a single physical CPU becoming available with be much higher than 8 physical cores being available.  This does not mean deploy all single virtual core VM's which there needs to be a balance and try not to have too many VM's with large virtual core counts running on the same node.

One of the key performance counters which you can use to monitor this is **Hyper-V Hypervisor Virtual processor\CPU Wait time per dispatch**.  This metric is measured in milliseconds and is showing the amount of time the virtual core is waiting to run on a physical core.  There is no magical number of what is seen as good and bad as it is totally dependant on the workloads which are running, but it may be worth trying to gather such metrics overtime incase this does increase and you begin to see performance issue.

Using monitoring and alerting can help identify those [CPU bottlenecks][cpubottleneck] and using a combination of Azure Monitor, for alerting and viewing historical data, and Windows Admin Centre, for real-time metric, and help with this.

## Memory

The memory allocation of the VM's is possible with either 2 memory types

- Static - this will allocate the configured amount of memory to that VM at startup and it will always have that amount of memory available to it while the VM running
- Dynamic - this this has a number of parameters associated with it:
    - Start-up - this is the amount of memory the VM will have when it first boots and until the Operating system is booted.  There must be this amount of memory available to allow the VM to be powered on
    - Minimum -  once the VM is running then memory available to the VM will drop down to the amount that it using ( plus the Memory Buffer) or the minimum amount configured, which ever is higher.
    - Maximum - this is the amount of memory which the VM can consume, this can be more than the start-up amount
    - Memory Buffer - this is a percentage of memory which the hyper-visor will "reserve" so that sudden spikes in memory required are better handled
    - Memory Weight - this helps set the priority the VM will receive when there is a memory constraint on the host running the VM.  The higher the weighting the higher the priority.  Be aware that trying to start a VM with a low weighting when there is a constraint may cause the VM not to start.

To monitor is at a host level then the metric **MemoryAvailable Mbytes** and enable the relevant warnings for this metric to notify when a memory constraint may be occurring, this may need to be fine tuned and taken over maybe a 5 minute period to help prevent any false alerting for any spikes in usage.  Within the VM's also monitor and track the **MemoryCommitted Bytes** metric as this will allow the tracking of the amount of memory a VM is actually using which could assist with "true sizing" of the VM's.

Using monitoring and alerting can help identify those [memory bottlenecks][memorybottleneck] and using a combination of Azure Monitor, for alerting and viewing historical data, and Windows Admin Centre, for real-time metric, and help with this.

## Storage

The amount of IOPS which the storage can deliver is key to be able to deliver a consistent experience.  Unlike with CPU and Memory, where you have a defined amount of how much of these, the amount of IOPS for storage is determined by the factors of:

- Number of Disks
- Size of Disks
- Speed of Disks
- Is caching available 

To help provide a guide line for the number of IOPS available then there is a tool named VMFleet.  This is best ran on a cluster which has no workloads running and it will build out an environment and then "stress" the environment until it is no longer providing the level of IOPS expected for those tests.  There are a series of built in tests it will run such as different read:write ratio's, differing levels of storage redirection.  This will then allow you to see if you had for example a 70:30 read:write ratio then you would expect to see X amount of IOPS for the entire cluster.

Additionally with the use of [storage quality of service][storageqos] VM's can be limited to the amount of IOPS they can consume to prevent single VM's from using up vast amounts of the IOPS.  It is possible to create [Storage Qos Policies][createstorageqospilicy] and then assign these to VM's, even having a pool or IOPS which can then be shared between a group of VM's.

Using monitoring and alerting can help identify those [storage bottlenecks][storagebottleneck] and using a combination of Azure Monitor, for alerting and viewing historical data, and Windows Admin Centre, for real-time metric, and help with this.

### RDMA

RDMA is a key feature which allows for a better performance of the cluster.  Having a storage interface and network which supports RDMA can help move the processing requirement from the node CPU to the network interface card itself.  In some environment it can be seen that there is a 30% drop in node CPU usage when RDMA is used.  A tool name [test rdma][testrdma] is available which can perform tests on the interfaces to ensure that RDMA is successfully enabled end-to-end on your storage connections.