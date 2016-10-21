[//]: (Infrastructure discovery with consul is easy, mostly due to its known semantics of a server, a service and associated health checks. But using consul in production is a topic on its own - scale a running cluster is hard, ensuring all the zombie's are gone need to be implemented on your own and the solution for initial cluster startup ([atlas]) is commercial. With [etcd] you can achieve easily the same infrastructure discovery concept as consul has, with just a couple of lines of code. This post will guide you through a good alternative of infrastructure discovery, if you don't want to use consul in production.)
[//]: ("etcd": "3.0.12")
[//]: ("confd": "0.11")
[//]: ("haproxy": "1.6.7")
Infrastructure discovery with consul is easy, mostly due to its known semantics of a server, a service and associated health checks. But using consul in production is a topic on its own - scale a running cluster is hard, ensuring all the zombie's are gone need to be implemented on your own and the solution for initial cluster startup ([atlas]) is commercial. With [etcd] you can achieve easily the same infrastructure discovery concept as consul has, with just a couple of lines of code. This post will guide you through a good alternative of infrastructure discovery, if you don't want to use consul in production.

## Consul vs etcd
While consul actually has a page about [consul vs the others](https://www.consul.io/intro/vs/zookeeper.html), it's most likely the case, that you're already running etcd if you use kubernetes. Only maintaining 1 Service Registry technology in your environment might be a good idea. 

### Why not consul?
Well, this should not be an anti-consul post (I still love consul for PoC's), so I try to face some facts:
1. [Consul agent] takes part in Raft, so doing infrastructure discovery in your whole environment means every system takes part in [Raft]
2. Have seen many problems during leadership election of a running cluster
  * Already happened, that consul got problems during new leadership election and it was impossible to restore the data - cluster was empty afterwards (worst case)
3. Atlas is a more a less requirement, if you want to automate your environment (in a nice way)

Consul for local testing and development is great, the web ui helps you to find problems during development quiete fast. But there is a reason, why everyone uses the web ui - have you ever tried using the CLI with dozens of applications writing to it? Without a TTL? Probably over months? Good luck! You need the UI for consul, otherwise you will see more zombies than everything else after a bit.

### Why etcd?
Etcd is straight forward, it focuses on its main tasks: being a distributed key/value store. I personally don't have the feeling, that querying DNS for an endpoint is a must have for a service registry (although you can [enable it on etcd] as well). Same for the difference in handling nodes, services and the key/value pairs from consul - whats the point in treating them differently? So focusing on the basics was a good choice for etcd. But this also means, that you have to implement things like infrastructure discovery or health checks on your own. But you will see in the next sections, that its actually not hard to do so.

## Cluster setup
Setting up an etcd cluster is fairly easy - especially if you use CoreOS (like I do). Just ask the public discovery for a new token:

```
curl -w "\n" 'https://discovery.etcd.io/new?size=3'
```

And use the following user data to spin up 3 machines:

```
#cloud-config
  units:
    - name: etcd2.service
      command: start
  etcd2:
    discovery: https://discovery.etcd.io/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    initial-advertise-peer-urls: http://$private_ipv4:2380
    listen-peer-urls: http://$private_ipv4:2380,http://$private_ipv4:7001
    listen-client-urls: http://0.0.0.0:2379,http://0.0.0.0:4001
    advertise-client-urls: http://$private_ipv4:2379,http://$private_ipv4:4001
```

Remind yourself to always obtain a new token if you completly rebuild your environment.

## What is meant by Infrastructure discovery?
If cloud environments are build the right way, they're quiete short living. Not as short living as container environments (which are following kind of the same principles of discovery btw.), but autoscaling groups in AWS for example can fire up or shut down machines, same as spot fleet requests, based on various factors at any time. Therefor, it's quiete useful to have a place where you can query information such as: *give me all ip addresses of my kubernetes API servers*.
This is pretty much, what AWS sells as elastic load balancer as a ready to go solution. If you have infrastructure discovery in place, you can very easy rebuild the service provided by ELBs yourself.

### How infrastructure discovery looks like in consul
All machines in your environment are reporting information like ip address and health of corresponding services to a set of [consul servers]. This is what [consul agent] is for. The consul servers are most likely clustered using [atlas] and [consul-template] can be used to query and use this information.

![Consul discovery][Consul discovery]

I will not dig deeper into the details of how to setup such an environment. This is really straight forward, fairly easy and build for it - so you just need to follow the documentation of the linked tools from the toolchain.

## Setup infrastructure discovery with etcd
Since I mostly run [CoreOS] in my environment, I'll describe this setup via [Cloud-config] snippets using [systemd-timers]. But once you got the idea, it shouldn't be hard to transfer this implementation to CentOS and Cronjobs for example.

The whole idea of this setup is, that all machines in your environment periodically push information about themselves (such as IP address) to etcd. This information is e.g. provided with a TTL of 90 seconds - and the systemd-timers are scheduled to update this data every 60 seconds. If a server does not update the data in etcd, the data will be gone from the service registry.
This is basically a bit like the [TTL based consul health check].

### Defining how a node looks like in etcd
Just to get an idea what this means, let's have a quick recap how [consul] defines a node in its topology.

* Name
* IP
* Services and associated ports
* health checks
* datacenter

All this information can be queried plus you can get the state of the executed checks.
For now, we will just focus on the name and the IP for etcd - so we will define the structure like this:

```
/infrastructure/${ROLE}/${HOSTNAME}/ip
/infrastructure/k8s-master/production-k8s-master/ip
```

Consul is data center aware for it's node entries. If you want that, you can easily extend the structure to:

```
/infrastructure/${DC}/${ROLE}/${HOSTNAME}/ip
/infrastructure/eu-west-1a/k8s-master/production-k8s-master/ip
```

### Implementing etcd infrastructure discovery

Let's see how the [Cloud-config] looks like.

```
#cloud-config
coreos:
    - name: discovery.service
      content: |
        [Unit]
        Description=Register periodically in etcd

        [Service]
        Type=oneshot
        Environment='ETCDCTL_ENDPOINT=http://etcd.example.com:2379'
        Environment='ROLE=k8s-master'
        Environment='TTL=121'
        ExecStart=/bin/sh -c "/usr/bin/etcdctl set /infrastructure/${ROLE}/`hostname`/ip $private_ipv4 -ttl ${TTL}"
        ExecStartPost=/bin/sh -c "/usr/bin/etcdctl updatedir /infrastructure/${ROLE}/`hostname` -ttl ${TTL}"
        ExecStartPost=/bin/sh -c "/usr/bin/etcdctl updatedir /infrastructure/${ROLE} -ttl ${TTL}"

        [Install]
        WantedBy=discovery.timer
    - name: discovery.timer
      command: start
      content: |
        [Unit]
        Description=Run discovery.service every 1 minute

        [Timer]
        OnCalendar=*:0/1
```

You might wonder about the `updatedir` calls and the aditional workload? Etcd sets TTL for every layer of the "tree" - which means, only the last `/ip` entry will disappear after the TTL has expired. For my [spot fleet instances], I don't use the `updatedir` call to keep a history of spawned instances. 

If you got the idea, adding health checks is very simple. Just set `$?` for the previously run bash script, but this will result in aditional `set` for every server on a frequent basis. So don't do too much stuff to buy ahead.

### ETCDv3 to the rescue
As you might asume, the etcd cluster might be under preasure if you have a lot of instances reporting to it - especially in a large scale cloud environment. Doing 3 client operations (at least) for every machine in your environment once in a minute (or even less) is a bad thing. If you for example also add the datacenter to your etcd key, than you even may need an aditional `updatedir`. 

Let's see, how [etcdv3] can help here. A quote from [this blog post]:

> Parts of the original design proved successful: etcd evolved into a key value store with JSON endpoints, watchers on continuous key updates, and time-to-live (TTL) keys. 
> Unfortunately, some of these features tended to be chatty on the wire with clients, could put quorum pressure on the cluster when idling, and could unpredictably garbage-collect older key revisions.

Exactly the chattyness is what I complained about.

#### Leases vs TTL
The reason why they replaced TTLs with leases in etcdv3 is very much exactly our use case:

> Leases reduce keep-alive traffic and eliminate steady-state consensus updates. 
> Instead of a key having a TTL, a lease with a TTL is attached to a key. 
> When the lease’s TTL expires, it deletes all attached keys. 
> This model reduces keep-alive traffic when multiple keys are attached to the same lease. 

Due to its hierarchy structure in etcdv2, we're indeed setting multiple keys - that's why we also need to set the TTL later on (the directory structure has gone away in etcdv3 too, guess why?!).

#### How to enable etcdv3?
Since etcd follows semantic versioning, v3 can be enabled by just setting an environment variable on the client:

```
export ETCDCTL_API=3
etcdctl put foo "bar"
```
Since the v3 api is even part of etcd version 2 (since version 2.2 to be concrete), it's quiete possible, that you can use it without any change or upgrade of etcd.

I personally prefer the [CoreOS] stable channel and etcdv3 hasn't made it into this branch yet.
So I'm using these cloud-configs to install and enable etcdv3:

```
#cloud-config
coreos:
  units:
    - name: etcd3.service
      command: start
      enable: true
      content: |
        [Unit]
        Description=etcd3
        Conflicts=etcd2.service
        Requires=install_etcd3.service
        [Service]
        Type=notify
        ExecStartPre=/usr/bin/mkdir -p /var/lib/etcd3
        ExecStart=/bin/sh -c "/opt/bin/etcd3 \
          --name `hostname` \
          --advertise-client-urls=http://$private_ipv4:2379 \
          --initial-advertise-peer-urls=http://$private_ipv4:2380 \
          --listen-client-urls=http://0.0.0.0:2379 \
          --listen-peer-urls=http://0.0.0.0:2380 \
          --discovery=https://discovery.etcd.io/43b28fe8e3b10ccde9633959d1fd1c2b \
          --data-dir=/var/lib/etcd3"
        Restart=on-failure
        RestartSec=10
        LimitNOFILE=40000
        TimeoutSec=4m
        [Install]
        WantedBy=multi-user.target
    - name: install_etcd3.service
      command: start
      content: |
        [Unit]
        Description=Installs etcd3

        [Service]
        Type=oneshot
        RemainAfterExit=yes
        Environment='VERSION=v3.0.12'
        ExecStartPre=/usr/bin/mkdir -p /opt/bin/
        ExecStartPre=/usr/bin/sh -c "/usr/bin/printf 'PATH=$PATH:/opt/bin' > /etc/profile.d/binary.sh"
        ExecStart=/bin/sh -c "[ -f /tmp/etcd3.tar.gz ] || /usr/bin/curl -L -o /tmp/etcd3.tar.gz -z /tmp/etcd3.tar.gz https://github.com/coreos/etcd/releases/download/${VERSION}/etcd-${VERSION}-linux-amd64.tar.gz"
        ExecStartPost=/usr/bin/tar xvzf /tmp/etcd3.tar.gz -C /tmp
        ExecStartPost=/bin/sh -c "/usr/bin/mv /tmp/etcd-${VERSION}-linux-amd64/etcd /opt/bin/etcd3"
        ExecStartPost=/bin/sh -c "/usr/bin/mv /tmp/etcd-${VERSION}-linux-amd64/etcdctl /opt/bin/etcdctl-etcd3"
```

The biggest issue here is, if something fails you could end up with and endless loop fetching stuff from github. With my first implementation I got blocked multiple times, that's why I prevent the `install_etcd3` task to run multiple times - and even if, the files remains in `/tmp/etcd3.tar.gz` and will not get fetched again if its there.

Surly that the etcd cluster should be [protected via SSL](https://coreos.com/etcd/docs/latest/etcd-live-http-to-https-migration.html), but certificate handling is a big part on its own, so I skip it here.

I can just asume, that etcd3 will get a kind of the same support via cloud config in future CoreOS releases and you don't need to struggle with the manual installation anymore.

#### Rewrite the discovery scripts
Since they replaced TTLs with leases in etcdv3, we don't need to set the keys again to refresh the TTL. Instead, we can write as many keys as we want and we just need to keep the lease alive. 
And we also get a nice command from `etcdctl` to do that:

```
$ ETCDCTL_API=3 etcdctl lease keep-alive ${LEASE_ID}
```

To remember our lease-id, we're extracting it from the response

```
$ ETCDCTL_API=3 etcdctl lease grant ${SECONDS}
lease 694d57ce55ceaf91 granted with TTL(10s)

```

and pass them to a file. But we need to bake it still into a nice systemd unit.

```
#cloud-config
coreos:
  units:
    - name: discovery.service
      command: start
      content: |
        [Unit]
        Description=Register periodically in etcd

        [Service]
        Environment='ETCDCTL_ENDPOINT=http://etcd-production.example.com:2379'
        Environment='ETCDCTL_API=3'
        Environment='ROLE=registry'
        Environment='TTL=121'
        Environment='DISCOVER_FILE=/tmp/discover_lease'
        ExecStartPre=/bin/sh -c "/opt/bin/etcdctl-etcd3 lease grant ${TTL} | grep -Po '[0-9|a-z]{16}' > ${DISCOVER_FILE}"
        ExecStart=/bin/sh -c "/opt/bin/etcdctl-etcd3 lease keep-alive `cat ${DISCOVER_FILE}`"
        ExecStartPost=/bin/sh -c "/opt/bin/etcdctl-etcd3 put /infrastructure/${ROLE}/`hostname`/ip $private_ipv4 --lease=`cat ${DISCOVER_FILE}`"
```

Looking at this implementation, I can't wait to use etcdv3 all the way in my environment.
Sure, health checks need a bit of another logic, cause you might also want to update the values on a state change. But nevertheless, even without looking at the underlaying detailed implementation, I can asume that the etcdv3 version of this idea is a lot faster and more scalable than the etcdv2 version.
But for now, I'm stuck with etcdv2 in my environment for infrastructure discovery - more on the reason in the next chapter.

#### How to benefit from infrastructure discovery
There was a reason, why I've implemented this in my environment - mostly to safe the AWS ELB costs for my auto scaling groups and build my own (independent) solution with [haproxy]. 

I've tried the following approaches:
1. Using puppet for initial bootstrapping, than handed over responsibility to [consul-template]
2. Using puppet and [hiera-consul]
3. Using puppet and [hiera-etcd]
4. Using puppet for initial bootstrapping, than handed over responsibility to [confd]

Option 2 and 3 haven't made it that long in my environment - the reason was, that I was not bound to changes in [consul] or [etcd], but to the next puppet run after the changes occured. That was just too much delay for me - but reducing the time for the puppet runs wasn't an option either.
Option 1 was a bit of a hustle, since it has not worked to reconnect if the [consul] server was available from the beginning. If you spin up a whole environment and [consul] has to be there, bootstrapped and reachable at your defined endpoint, to even provision the first service this simply doesn't felt right.
Although there's a `:failure: graceful` in the puppet module, this hiera configuration has never worked in my stack and puppet timed out completly. 

confd is configured in my environment, using [confd-puppet]. But I'm gonna show some plain-confd snippets, to not bind the post too much on my implementation.

conf.d/haproxy.toml:
```
[template]
src    = "haproxy.tmpl"
dest   = "/etc/haproxy/haproxy.cfg"
owner  = "root"
group  = "root"
mode   = "420"
keys   = [
  "/k8s-master",
  "/registry",
  "/jenkins",
]
reload_cmd = "/etc/init.d/haproxy reload"
check_cmd  = "/usr/sbin/haproxy -c -q -f {{.src}}"
```

I'm not posting my whole config, but just an example how you can query the entries, setted up via the previous scripts:

templates/haproxy.tmpl
```
backend jenkins_backend
  balance roundrobin
  mode http
  option httpchk /index.html
  {{range $name := lsdir "/jenkins"}}
        {{$host := printf "/jenkins/%s/ip" $name}}
        server {{$name}} {{getv $host}}:8080 check maxconn 1000
  {{end}}
```

Neither [etcdv3] is done with their documentation (that's why I linked the github repository) nor [confd] supports [etcdv3] for now - although there's [already a pull request](https://github.com/kelseyhightower/confd/pull/468) for it.
I'm gonna update this blogpost about an end-to-end etcd3 version, once the etcd3 support got merged - otherwise I'll need to check other alternatives again.

#### Conclusion
Service registries become more and more the *central place of truth* in every orchestrated docker environment, so why don't use it for your infrastructure as well? Tools like [confd] are following exactly these principles. I'm sure it won't take long until a stable provisioning system with native service registry link will show up - and I'm waiting for this day. But as of now, it's still a bit of a hustle to support short living components only with the help of puppet or chef - but that's only the beginning. Just a few years back we would have never imagined what is now possible in 2016, so let's wait for what's possible in 2020!

[Consul discovery]: img/infrastructure_discovery_with_etcd/consul.png
[consul agent]: https://www.consul.io/docs/agent/basics.html
[consul servers]: https://www.consul.io/docs/agent/options.html#_server
[atlas]: https://www.hashicorp.com/atlas.html
[consul-template]: https://github.com/hashicorp/consul-template
[Cloud-config]: https://coreos.com/os/docs/latest/cloud-config.html
[systemd-timers]: https://coreos.com/os/docs/latest/scheduling-tasks-with-systemd-timers.html
[TTL based consul health check]: https://www.consul.io/docs/agent/checks.html#TTL
[enable it on etcd]: https://github.com/coreos/etcd/blob/master/Documentation/op-guide/clustering.md#dns-discovery
[this blog post]: https://coreos.com/blog/etcd3-a-new-etcd.html
[hiera-consul]: https://github.com/lynxman/hiera-consul
[hiera-etcd]: https://github.com/garethr/hiera-etcd
[consul]: https://www.consul.io/docs/index.html
[confd]: https://github.com/kelseyhightower/confd
[confd-puppet]: https://forge.puppet.com/ajcrowe/confd
[etcd]: https://coreos.com/etcd/docs/latest/
[etcdv3]: https://github.com/coreos/etcd/tree/v3.0.12/Documentation
[Raft]: http://thesecretlivesofdata.com/raft/
[CoreOS]: https://coreos.com/
[spot fleet instances]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-fleet.html
[haproxy]: http://www.haproxy.org/