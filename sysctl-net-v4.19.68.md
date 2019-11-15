# Registering a new `sysctl` variable
##### Linux Kernel v4.19.68

This document describes the process I used to add a new `sysctl` variable for Project 2 in _Networking in the Linux Kernel_ at RPI, Fall 2019. The only source I found online for custom `sysctl`'s is for an outdated, unspecified kernel version [here](https://sar.informatik.hu-berlin.de/teaching/2012-s/2012-s%20Operating%20Systems%20Principles/lab/lab-1/sysctl_.pdf). I figured it'd be good to have a (slightly) more up-to-date version out there.

The kernel is always evolving, so this document could become obsolete at any time. Hell, it could be outdated already. Chances are I won't have time to keep it up-to-date myself, so feel free to open a Pull Request if one is necessary.

This document is pretty specific to the network stack, as well as the requirements for Project 2. The network stack as a whole uses a lot of custom structs in place of the more generic kernel-wide `sysctl` structs, but the general concept should be pretty extensible.

#### 1. Add new `sysctl` variable to the network stack ([`include/net/netns/ipv4.h`](https://elixir.bootlin.com/linux/v4.19.68/source/include/net/netns/ipv4.h#L43))
All network-related `sysctl` variables are internally represented in the members of `struct net`. I added the `ssthresh_scale` variable to `ipv4`, in `struct netns_ipv4`.

```C
// add project 2 sysctl variable
int sysctl_tcp_ssthresh_scale;
```

#### 2. Set default value for the new variable ([`net/ipv4/tcp_ipv4.c`](https://elixir.bootlin.com/linux/v4.19.68/source/net/ipv4/tcp_ipv4.c#L2501))
network-related `sysctl` values are initialized on system boot in the `tcp_sk_init()` function. I copied the example set by the other `sysctl` variables here.

```C
// init variable declared in step 1
net->ipv4.sysctl_tcp_ssthresh_scale = 20;
```

#### 3. Add `sysctl` table entry to `proc/sys/` ([`net/ipv4/sysctl_net_ipv4`](https://elixir.bootlin.com/linux/v4.19.68/source/net/ipv4/sysctl_net_ipv4.c#L558))
This file details how the kernel represents the `sysctl` variable in the virtual filesystem at `/proc/sys`. I added this entry to the bottom of `ipv4_net_table[]`, as that was where the rest of the TCP `sysctl` variables were defined.
* `procname` is the name of the virtual file as it appears in `/proc/sys`.
* `data` points to the `struct net` created on boot.
* `extra1` and `extra2` are the minimum and maximum values this entry can hold, respectively. The values are pointers to `static int`'s defined at the top of the file. `hundred` did not exist already, I added that myself.
* Due to the way procfs works, the last entry in the list has to be empty. Be sure to keep it as-is.

```C
// (at top of file)
static int hundred = 100;
...
// add procfs file representation of this sysctl variable
{
  .procname     = "tcp_ssthresh_scale",
  .data         = &init_net.ipv4.sysctl_tcp_ssthresh_scale,
  .maxlen       = sizeof(int),
  .mode         = 0644,
  .proc_handler = proc_dointvec_minmax,
  .extra1       = &one,
  .extra2       = &hundred
},
// (this empty list entry has to stay at the end)
{}
```

#### Note
At this point, after a rebuild, we should see our new variable at `proc/sys/net/ipv4/tcp_ssthresh_scale`. It can be modified via `/etc/sysctl.conf`, and should be fully functional. The `sysctl` command-line program won't see it yet, though. The next two steps allow us to view and modify it via that tool.

#### 4. Add internal representation number ([`include/uapi/linux/sysctl.h`](https://elixir.bootlin.com/linux/v4.19.68/source/include/uapi/linux/sysctl.h#L331))
This file lists IDs for every `sysctl` variable on the system. I placed mine at the end of the enum commented as `/proc/sys/net/ipv4`. This file has a big warning about modifying existing values, so I just stuck it at the end, #126.

```C
// claim a unique ID for our new sysctl variable.
NET_TCP_SSTHRESH_SCALE=126,
```
#### 5. Register variable in the sysctl binary ([`kernel/sysctl_binary.c`](https://elixir.bootlin.com/linux/v4.19.68/source/kernel/sysctl_binary.c#L333))
This file maps representation numbers to procfs files and data types. I added this new `sysctl` variable to the `bin_net_ipv4_table[]` list. Order does not matter here, unlike in step 4.

```C
// Register the internal ID to the procfs filename. and respective data type.
{ CTL_INT,  NET_TCP_SSTHRESH_SCALE, "tcp_ssthresh_scale"},
```

#### Done!
After a rebuild, typing `sysctl -a | grep net.ipv4.tcp_ssthresh_scale` in a terminal should return `tcp_ssthresh_scale = 20`.

Updating the sysctl variable via `sysctl -w net.ipv4.tcp_ssthresh_scale=[1-100]` should also take effect properly.

#### Sources
* _Adding your own sysctls_ - Vitalik Nikolyenko - https://sar.informatik.hu-berlin.de/teaching/2012-s/2012-s%20Operating%20Systems%20Principles/lab/lab-1/sysctl_.pdf

* _Elixir Cross Referencer_ - Bootlin - https://elixir.bootlin.com/linux/v4.19.68/source
