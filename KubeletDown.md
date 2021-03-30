# PagerDuty Alert ID: Kubelet Down
# Description
This alert fires when the Kubelet service on the host is down.


# Investigation and Triage

This alert is cuased by one of several issues. In order to diagnose this, you will most likely have to be able to SSH into the host in question as restarting the Kubelet will destroy any `debug` pods that you may create for troubleshooting.

Start off by SSHing into the host in question as the `core` user. NOTE: You will need to have a copy of the SSH key that was used to create the cluster as all other methods of authentication are disabled by default.

```
ssh core@master-0.lab-cluster.ocp4.lab
```

Start by investigating the resource consumption on the host. Use `top` to determine if there are any run-away processes or high disk contention.
```
top - 16:41:33 up 5 days, 22:52,  1 user,  load average: 4.55, 3.03, 2.15
Tasks: 323 total,   1 running, 322 sleeping,   0 stopped,   0 zombie
%Cpu0  :  8.6 us,  6.6 sy,  0.0 ni, 84.5 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
%Cpu1  :  7.2 us,  5.5 sy,  0.0 ni, 86.6 id,  0.3 wa,  0.0 hi,  0.3 si,  0.0 st
%Cpu2  : 13.7 us,  6.0 sy,  0.0 ni, 76.8 id,  1.8 wa,  0.0 hi,  1.8 si,  0.0 st
%Cpu3  : 12.5 us, 11.4 sy,  0.0 ni, 75.8 id,  0.3 wa,  0.0 hi,  0.0 si,  0.0 st
```

In the above example, there is no real I/O wait as indicated by the `0.0 wa` portion of the output.

If there is no significant CPU or I/O load, check for free memory:

```
free -m
```

Next, check the logs for the kubelet service

```
sudo journalctl -u kubelet.service
```

Look for any errors or any entries that might indicate a problem.

If you have determined that the best course of action is to restart the kubelet service, make sure that you drain the node in question first. This is done from the host you normally issue `oc` commands from. 

```
oc adm drain node/node1 --ignore-daemonsets --delete-local-data
```

Back on the host in question, restart the kubelet service. Restarting this service can cause the pods on this host to die unexpectedly which is why we drained the node first.

```
sudo systemctl stop kubelet.service; sleep 5; sudo systemctl start kubelet.service
```

The above command uses the `sleep` command to allow for time for the `kubelet.service` to shutdown cleanly. There are occasions where using `sudo systemctl restart kubelet` will attempt to restart the service before it is completely shutdown and thus it is advisable to stop and then start the service after a slight delay.

# Verification

After restarting the `kubelet.service` verify the logs again ensuring that the startup process has not encountered problems:

```
sudo journalctl -u kubelet.service -S -1h
```

If there are no further problems, uncordon the node:

```
oc adm uncordon node/node1
```

Wait 15 minutes for the alert to clear

