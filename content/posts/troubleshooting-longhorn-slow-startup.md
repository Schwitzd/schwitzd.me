+++
title = 'Troubleshooting Longhorn Slow Startup'
date = 2025-07-13T11:29:42Z
draft = false
+++

In my K3s home cluster, I use Longhorn as the storage engine for my stateful workloads. Since I'm just starting out and shutting down the cluster every day (to safe my power bill), I've noticed that Longhorn takes a long time to be ready, with a messy startup involving a lot of errors and pods going into the `CrashLoopBackOff` state.

Spoiler: It's always DNS :)

## Troubleshooting

I decided to take a look, so I began my troubleshooting journey by analyzing one of the affected pods.

```
$ kubectl -n database describe pod redis-master-0
...
Events:
  Type     Reason                  Age                From                     Message
  ----     ------                  ----               ----                     -------
  Normal   Scheduled               23m                default-scheduler        Successfully assigned database/redis-master-0 to sheep
  Normal   SuccessfulAttachVolume  23m                attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-c228e06b-4fbf-4e1f-8f07-1ff0dcc0f8ab"
  Warning  FailedMount             22m (x6 over 23m)  kubelet                  MountVolume.MountDevice failed for volume "pvc-c228e06b-4fbf-4e1f-8f07-1ff0dcc0f8ab" : kubernetes.io/csi: attacher.MountDevice failed to create newCsiDriverClient: driver name driver.longhorn.io not found in the list of registered CSI drivers
```

Spotted the error: `driver name driver.longhorn.io not found in the list of registered CSI drivers`
The Longhorn CSI drivers are managed by workloads bundled with Longhorn’s deployment in the cluster. Examining the `longhorn-system` namespace using the command `kubectl get pods -n longhorn-system` revealed a significant number of pods in error states, including those in `CrashLoopBackOff`, which was a nightmare.

I started to investigate why pods longhorn-csi-plugin they was in `CrashLoopBackOff` on both my worker and control-plane node:

```
$ kubectl -n longhorn-system logs longhorn-csi-plugin-22q68 -c node-driver-registrar --previous
I0622 06:08:12.136386       1 main.go:150] "Version" version="v2.13.0"
I0622 06:08:12.136480       1 main.go:151] "Running node-driver-registrar" mode=""
I0622 06:08:12.136488       1 main.go:172] "Attempting to open a gRPC connection" csiAddress="/csi/csi.sock"
I0622 06:08:22.137195       1 connection.go:253] "Still connecting" address="unix:///csi/csi.sock"
I0622 06:08:32.136932       1 connection.go:253] "Still connecting" address="unix:///csi/csi.sock"
I0622 06:08:42.137271       1 connection.go:253] "Still connecting" address="unix:///csi/csi.sock"
E0622 06:08:42.137425       1 main.go:176] "Error connecting to CSI driver" err="context deadline exceeded"
```

By examining the logs, it looked like the `node-driver-registrar` container was unable to connect to the CSI socket on the node. This socket is created and managed by the main `longhorn-csi-plugin` container within the same pod—so if that container isn’t running properly, the registrar won't be able to connect.

To dig deeper, my next troubleshooting step was to check the logs for the main `longhorn-csi-plugin` container itself, to see why it might be failing to create the CSI socket. If the main plugin isn’t starting up correctly, it would explain why the registrar can’t communicate with it.

Here’s what I found:

```
k3s@cow:~ $ kubectl -n longhorn-system logs longhorn-csi-plugin-22q68 -c longhorn-csi-plugin --previous
time="2025-06-22T06:06:09Z" level=info msg="CSI Driver: driver.longhorn.io version: v1.9.0, manager URL http://longhorn-backend:9500/v1" func="csi.(*Manager).Run" file="manager.go:23"
time="2025-06-22T06:06:19Z" level=fatal msg="Error starting CSI manager: Failed to initialize Longhorn API client: Get \"http://longhorn-backend:9500/v1\": context deadline exceeded (Client.Timeout exceeded while awaiting headers)" func=app.CSICommand.func1 file="csi.go:37"
```

The main `longhorn-csi-plugin` container is crashing because it can’t connect to the Longhorn backend API—it just times out trying to reach `http://longhorn-backend:9500/v1`. At this point, my suspicion turned to DNS, specifically the CoreDNS pod.
CoreDNS usually resides in the `kube-system` namespace. I checked to see if it was running with the following command `kubectl -n kube-system get pods`, Ta da! The DNS pod is not yet started!

## Lesson learned

What I learned is that *Longhorn* really needs DNS to work properly. The pods, especially the ones managed by DaemonSets, would try to start as soon as K3s was up, but CoreDNS wasn’t ready yet. This led to lots of timeouts and those annoying `CrashLoopBackOff` errors. Kubernetes isn’t really designed to be shut down and started up every day, especially with the ever-complicated setup I’ve created. This can lead to all kinds of race conditions like the one I ran into.

In the end, I solved this issue by adding a `dns-unready` taint and corresponding toleration during my [cluster shutdown & startup workflow](https://github.com/Schwitzd/IaC-HomeK3s?tab=readme-ov-file#cluster-shutdown--startup-workflow). This kept Longhorn pods from starting up until CoreDNS was actually ready, and finally got rid of those frustrating startup errors.

## Credits

I want to give a huge shoutout to the author of [Troubleshooting Longhorn and DNS Networking](https://www.ekervhen.xyz/posts/troubleshooting-longhorn-and-dns-networking/). That article pointed me in the right direction after days of troubleshooting! The issue was DNS in both of our cases, but the motivation behind it was different. The post also inspired me to write this blog and share my own experience.
