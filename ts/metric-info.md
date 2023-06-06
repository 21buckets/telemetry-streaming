# System Poller

This is my ramblings abut the System Poller and returned metrics as I keep forgetting.


## CPU and Memory

This KB article explains which metrics to use for CPU and Memory: https://my.f5.com/manage/s/article/K72233013

| Metrics	| Details |
| ------------- | ---------------- |
| f5_system_cpu |	Average CPU usage of the system in 1 minute. It equals the Last 1 Min System from the tmsh show sys host-info. |
| f5_system_memory |	Calculated by formula ( memoryUsed / memoryTotal ) * 100. </n> The memoryUsed and memoryTotal are outputs of the Total block of Sys::Host Memory from tmsh show sys memory. |
| f5_system_swap |	The System Swap usage. It equals the outputs of  Current Swap Used ratios of the tmsh show sys memory. |
| f5_system_tmmCpu |	Average CPU usage of the tmm in 1 minute. It equals the  Last 1 Minute CPU Usage Ratio (%) from the tmsh show sys tmm-info. |
| f5_system_tmmMemory |	Calculated by formula ( tmmMemoryUsed / tmmMemoryTotal ) * 100. </n> The tmmMemoryUsed and tmmMemoryTotal are outputs of the Tmm block of Sys::Host Memory from tmsh show sys memory. |


## Other useful information

### FAQ

Link: https://clouddocs.f5.com/products/extensions/f5-telemetry-streaming/1.20/faq.html

* Telemetry streaming uses iControl REST to return metrics about big-ip

### Troubleshooting

Link: https://clouddocs.f5.com/products/extensions/f5-telemetry-streaming/1.20/troubleshooting.html

* Check `/var/log/restnoded/restnoded.log` for Telemetry Streaming logs
* 
