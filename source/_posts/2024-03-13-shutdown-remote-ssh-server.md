---
title: Automatic Shutdown Server under VS Code Remote SSH
date: 2024-03-13 10:09:26
tags: [vscode, tools, docker]
---

I have migrated all my development environments to GCP's Spot Instance and connect using the VS Code Remote SSH plugin. The advantage is that GCP charges by the second, making it cost-effective as long as you shut down promptly, even with high-spec machines. The downside is that forgetting to shut down during holidays can lead to high costs. After losing more than 10 dollars, I decided to find a way to automatically shut down the machine when VS Code is idle. This involved some tricky maneuvers to execute the shutdown command within a Docker container.

# How to Determine Idle State

This feature is similar to the idle timeout in CodeSpace, but the Remote SSH plugin doesn't expose this functionality, so I had to implement it myself. The main challenge was determining idle state on the server side. I couldn't find an exposed interface in VS Code, so I wondered if there was a simple way to determine idle state outside of VS Code.

The most straightforward idea was to monitor SSH connections, as Remote SSH connects via SSH. If there are no active SSH connections on the machine, it can be considered idle and shut down. However, the connection timeout for Remote SSH is quite long, reportedly four hours, and even closing the VS Code client didn't terminate the server-side SSH connection. Setting a timeout on the server side would lead to quick client reconnection, so the number of connections wouldn't decrease.

Since directly monitoring the number of SSH connections wasn't feasible, I looked into whether it was possible to determine if no traffic under existing SSH connections. Using `tcpdump`, I found that there were TCP heartbeat packets every second, even without client interaction, so no traffic wasn't a valid indicator. However, the size of these heartbeat packets was consistently 44 bytes, which could be used to identify idle state—if there were no packets larger than 44 bytes on the SSH port for a certain period, it was considered idle.

Here's the first version of the script:

```bash
#!/bin/bash

while true; do
  if [ -f /root/dump ]; then
    rm /root/dump
  fi

  timeout 1m tcpdump -nn -i ens4 tcp and port 22 and greater 50 -w /root/dump

  line_count=$(wc -l < /root/dump)

  if [ $line_count -gt 0 ]; then
    rm /root/dump
  else
    rm /root/dump
    shutdown -h now
  fi
done
```

This script achieves the functionality of shutting down the machine if there are no non-heartbeat packets on the SSH port within one minute. The next step was to automate this script's execution.

# Docker Dark Magic

When packaging the script into a Docker image, I encountered an interesting issue: none of the base images contained the `shutdown` command, nor could it be easily installed. To execute the `shutdown` command on the host, it was necessary to switch to the host's namespace from within the Docker container:

```bash
nsenter -a -t 1 shutdown -h now
```

The `nsenter` command targets the process with Pid 1 and enters all its namespaces (pid, mount, network), making it appear as if operating directly on the host when Docker runs in the shared host Pid mode. To run the container with this capability, use the following command:

```bash
docker run --name=close --pid=host --network=host --privileged --restart=always -d close:v0.0.1
```

# Conclusion

By monitoring SSH traffic, we can infer if VS Code is idle and then use some Docker "dark magic" to achieve automatic shutdown. However, this method is quite complex; is there a simpler way?