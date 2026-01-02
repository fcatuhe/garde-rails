# De-cloud and de-k8s — bringing our apps back home

**Author:** Farah Schüller, Senior Site Reliability Engineer, Ops
**Published:** March 21, 2023
**Source:** <https://dev.37signals.com/bringing-our-apps-back-home/>

*For the Operations team at 37signals, the biggest effort in 2023 is removing our dependencies on the cloud and migrating our application stacks back into the data center onto our own hardware. We’ve already made amazing progress in a fairly short time — let’s get into some details!*

---

## Why are we doing this?

A number of our applications have been on a long journey through various cloud providers and services over the years.

Originally, we started by moving them from our data centers to AWS ECS, with the promise of lovely contained Docker builds and eventual cost savings.

We liked Docker a lot, less so the lack of flexibility in ECS. Thus, we pivoted to GKE on Google Cloud Platform to give Kubernetes a try, only to be thwarted by network control plane outages that led us to a quick retreat. Undeterred by the cloud, we moved our legacy applications to AWS Kubernetes (EKS) where some of them still reside today.

You inevitably accrue some dimension of technical debt and complexity on this path. Your deployment strategy isn’t the only thing that has to change: you’ll have to invent new tooling to manage those stacks and create useful CI/CD to cater to the needs of both operations and programming. Most likely you will have to rethink your monitoring strategy as well.

Not even scratching the necessity of entirely different paradigms regarding informational and operational security as well. Oh, and at some point, you’d also have to train people about all of this! Gimme a second, we just got an email about the maintenance EOL of this resource from [public cloud provider]… wait, is us-east-1 down?

Bottom line: you need a lot of processes to do this right. In a lot of places, it became apparent we were spending more than we got out of it in return — [not just economically](https://dev.37signals.com/our-cloud-spend-in-2022/), but also operationally. This was our smallest application, [Tadalist](https://signalvnoise.com/archives/001021.php), when it ran on EKS, taken from our internal documentation.

```
+--------------------------------------------------------+
                              |    eksctl VPC                                          |
                              |    +----------------------------------------------+    |
                              |    |                      EKS                     |    |
                              |    | +------------------------------+ +---------+ |    |
                              |    | |app namespace                 | |default  | |    |
                              |    | |                              | |namespace| |    |
+-------------------+         |    | +--------+ +--------+ +--------+ +---------+ |    |
|   tadalist VPC    |         |    | |pod     | |pod     | |pod     | |pod      | |    |
|                   |         |    | +--------+ +--------+ +--------+ +---------+ |    |
| +---------------+ |         |    | |Unicorn | |Unicorn | |Unicorn +-+Logstash | |    |
| |   Services    | |         |    | |        | |        | |        | |         | |    |
| |               | |   VPC   |    | +--------+ +--------+ +--------+ +---+-----+ |    |
| | +-----------+ | | Peering |    | |Nginx   | |Nginx   | |Nginx   |     |       |    |
| | |    RDS    | | <--------->    | |        | |        | |        +-----+       |    |
| | +-----------+ | |         |    | +-^------+-+-^------+-+-^------+             |    |
| +---------------+ |         |    |   |          |          |                    |    |
+-------------------+         |    |   |          |          |                    |    |
                              |    +----------------------------------------------+    |
                              |        |          |          |                         |
                              |      +-+----------+----------+---+                     |
                              |      | Application Load Balancer |                     |
                              |      +------------^--------------+                     |
                              |                   |                                    |
                              +--------------------------------------------------------+
                                                  |
                                                  |
                                                  +
                                           Internet Traffic
```

Looks… easy? Well, note that this is just the rough infrastructural outline — it doesn’t include all the auxiliary tooling that runs on it, such as
- cluster-autoscaler
- ingress controllers
- storage drivers
- external-dns
- node termination handlers
- complex networking concepts around VPN, peering, route tables, NAT, …
- where DNS is handled
- …

It also misses the entire sphere around identity and access management for those resources that also needs to be maintained. Not even mentioning the infrastructure as code that has grown around this. It’s fair to say that we never realized the promise that the cloud would simplify our life.

And Tadalist is just a basic, pretty isolated Rails app at its core. It doesn’t get easier from here for our other, much more complex apps with a lot more dependencies on other backend services.

Our first impulse for de-clouding was to run (vendor-supported) Kubernetes on our own hardware, so we could keep the largest chunk of the work that we’ve invested over years and “just” repoint our tooling to a new place. An additional challenge was the fact that most of our apps have been containerized a few years ago to satisfy legacy requirements more easily — and we wanted to keep it that way.

It all sounded like a win-win situation, but it turned out to be a very expensive and operationally complicated idea, so we had to get back to the drawing board pretty soon.

Enter [mrsk](https://github.com/mrsked/mrsk).

---

## Containers, but nice: migrating Tadalist

`mrsk` became the center paradigm around our new de-clouding and de-k8sing strategy: it created a path to simplifying deployments of our existing containerized applications while speeding up the process of doing so tremendously. We were able to keep a lot of prior art around the containerization while still running apps in a somewhat new, shiny but also familiar way — we’ve been deploying with [`capistrano`](https://www.capify.org/index.html) for Basecamp 4 and other apps in the data center for years, and `mrsk` aimed at the same imperative model. No more control plane, fewer moving parts. Nice!

[Tadalist](https://signalvnoise.com/archives/001021.php) was the perfect candidate: it has the lowest criticality of all our applications and no paying customers. It has served as a canary in all our operational endeavors and it stayed within that role this time as well.

Of course, this wasn’t just a “throw it on there and run with it” kind of process. `mrsk` was in active development while the infrastructural side was incepted, in tandem. You don’t hit the ground running right from the start! We also had to take care of a couple of other things.

### Virtual machine provisioning

We focused on the cloud and surrounding technologies *a lot* in recent years, thus our processes for on-premise infrastructure provisioning fell a bit behind. Next to the de-clouding effort, we started to completely modernize and simplify our configuration management, upgrading to the current version of [Chef](https://www.chef.io/).

Since `mrsk` was going to target servers on-premise, we needed a new process for provisioning virtual machines quickly and painlessly as well. We leveraged our existing knowledge running pure KVM-based VM guests, simplified bootstrapping with cloud-init, and weaved all of this into a single cookbook. These improvements allowed us to cut down our process for a fully bootstrapped VM from around 20 minutes per guest to *less than a minute* — and we can now define and start several simultaneously with a single chef converge of the KVM host.

```
# Example definition from our cookbook
node.default['guests'] = case node['hostname']
                         when "kvm-host-123"
                           {
                            "test-guest-101": { ip_address: "XX.XX.XX.XX/XX", memory: "8192", cpu: "4", disk: "30G", os: "ubuntu22.04", os_name: "jammy", domain: "guest.internal-domain.com" },
                           }
                         when "kvm-host-456"
                           {
                            "test-guest-08": { ip_address: "XX.XX.XX.XX/XX", memory: "4096", cpu: "2", disk: "20G", os: "ubuntu18.04", os_name: "bionic", domain: "guest.internal-domain.com" },
                           }
                         end

include_recipe "::_create_guests"
```

```
# Excerpt from our ::_create_guests Chef recipe
node['guests'].each do |guest_name, guest_config|
  execute "Create and start a new guest with virt-install" do
    command "virt-install --name #{guest_name} --memory #{guest_config[:memory]} \
             --vcpus #{guest_config[:cpu]} --network bridge=br0,model=virtio --graphics none \
             --events on_reboot=restart --os-variant #{guest_config[:os]} \
             --import --disk /u/kvm/guests-vol/#{guest_name}-OS.qcow2 \
             --noautoconsole --cloud-init user-data=/u/kvm/guests-config/user-data-#{guest_name},network-config=/u/kvm/guests-config/network-config-#{guest_name} \
             --autostart"
  end
end
```

A huge speed boost, since we’d be creating VMs left and right soon! Most of these were simple boxes with not much more than basic user management, a Filebeat config and a Docker installation.

### Logging

Our logging pipeline is [a fully consolidated ELK stack](https://www.elastic.co/what-is/elk-stack), which is interoperable with both our cloud and on-premise stacks. `mrsk`’s logging is built on top of pure Docker logs, so all we had to do was reroute Filebeat to pick up those logs in `/var/lib/docker/containers/` and ship them to our Logstash installations.

### CDN

For years, we’ve leveraged CloudFront and Route53 for our CDN and DNS requirements, so naturally, we had to find a replacement and switched to [Cloudflare](https://www.cloudflare.com/), which is now fronting our on-premise F5 load balancers in the data centers.

### CI/CD

Previously running on Buildkite, we now moved to [GitHub Actions](https://github.com/features/actions) for CI/CD.

### Database replication and backups

RDS was doing a lot for us, but it’s not like we didn’t have prior experience in running MySQL databases on-prem before — we’ve been doing this for Basecamp for decades. For the `mrsk`-backed app deployments, we leveraged on that and upgraded our stack to run [Percona MySQL 8](https://docs.percona.com/percona-server/8.0/#) and implemented a cron-based backup process to a dedicated backup location within our data center — all encapsulated in a cookbook and applied to the relevant database servers.

In *less than a six-week cycle*, we built those operational foundations, shaped `mrsk` to its functional form and had Tadalist running in production on our own hardware.

The cutover process was a fairly straightforward call of transferring the small database and shifting DNS, but we did accept some downtime, as Tadalist doesn’t run under an SLO of zero downtime.

So after this process, this is where Tadalist landed:

```
+----------------------------------------------------------------+
|  On-Prem                                                       |
|            +--------------------------------------------+      |
|            |                                            |      |
|            |                                   +--------v--+   |
|  +---------+-------+ +-----------------+------>| db-01     |   |
|  | app-01          | | app-02          |       |           |   |
|  | +-------------+ | | +-------------+ |       |    mysql  |   |
|  | | tadalist-app| | | | tadalist-app| |   +---+-----------+   |
|  | |             | | | |             | |   |                   |
|  | +-------------+ | | +-------------+ |   |replicates to      |
|  | |   traefik   | | | |  traefik    | |   |   +------------+  |
|  | +-------------+ | | +-------------+ |   |   | db-02      |  |
|  +-+-----------+-+-+ +-+--+----------+-+   +-->|            |  |
|                |          |                    |   mysql    |  |
|                |          |                    +------------+  |
|             +--v----------v--+                                 |
|             |       F5       |                                 |
|             |  load balancer |                                 |
|             +--------+-------+                                 |
|                      |                                         |
+----------------------+-----------------------------------------+
                       |
              +--------+-------+
              |   Cloudflare   |
              +--------^-------+
                       |
                       |
               Internet traffic
```

A pretty standard Rails app deployment, right? Turns out that’s all Tadalist (and most of our apps) really needed.

The application hosts are configured as pools on the F5 load balancer, which takes care of routing the traffic where it should go and also provides the public endpoint that Cloudflare should talk to. Anything behind that is just basic Ruby, Rails and Docker, the latter wrapped into `mrsk`.

Did I mention we cut down our deployment times from several minutes to just roughly a minute, sometimes even less?

---

## Two is a party — migrating Writeboard

In terms of complexity, [Writeboard](https://signalvnoise.com/archives2/writeboard_is_live.php) was our next candidate. We could build on all those foundations we figured out for Tadalist already, so the entire process of migrating took less than two weeks.

We were also joined by our Security/Infrastructure/Performance team for extended verification and eventual changes from the application side, as Writeboard has a concept of scheduling jobs. Other than that, it pretty much matched Tadalist in terms of application and infrastructure requirements.

Since Writeboard has paying customers and a much larger database, the SLO changed. Thus we opted for a simple three-fold replication process to prepare the databases pre-migration.

```
+-----------------+            +-----------------+            +-----------------+
|       RDS       |            |     wb-db-01    |            |     wb-db-02    |
|                 | replicates |                 | replicates |                 |
|      (r/w)      +----------->|      (r/o)      +----------->|      (r/o)      |
+-----------------+            +-----------------+            +-----------------+
```

Once that was in place, the actual migration consisted of:
- enabling maintenance windows
- setting RDS to read-only
- make the on-prem databases writable
- stopping the replication from RDS
- switching DNS

The full cutover for Writeboard took 16 minutes with no downtime or complications.

We’re on a roll here, so let’s move on to Backpack!

---

## Three is a festival — migrating Backpack

[Backpack](https://signalvnoise.com/archives2/backpack_launches_a_new_breed_of_personal_and_business_information_manager.php) again could build on the previous migrations, with a small caveat: it runs a stateful mail pipeline implemented with [postfix](https://www.postfix.org/), thus it needs to process those mails on disk.

On EKS, this got implemented by running postfix on a separate EC2 node and mounting via a shared EFS-backed PVC into the jobs pods. For the `mrsk`-backed deployment, we decided on a similar scheme with a shared NFS between the mail-in hosts and the jobs containers.

Here, we came across the next big feature in `mrsk`, which is mounting the filesystem into containers. It’s actually not a huge deal in Docker — it just had to be implemented into `mrsk` and thoroughly tested.

Backpack also runs many more scheduled jobs than Writeboard — and we had to find a way to run those so they don’t step on each other’s toes within our given application requirements. Previously, this was solved by just running them all on the same pod/host and sharing a temporary file-based lock. With multiple nodes as we had planned, this wouldn’t work anymore (and it’s also not very flexible). Thus, this logic was rewritten to run with [Redis](https://redis.io/) as their backend, so we could isolate this concern from the jobs hosts.

```
^
                                                              |
--------------------------------------------------------------+----------
  On-Prem                                                     |
                                  +----------+            +---v---+
                 passes mail      |          |            |       |
                  content to +--->|  app-01  |<---------->|  F5   |
                     app     |    |          |            |       |
                             |    +----------+            +---+---+
                             v                                |
                      +---------------+                       |
    processes jobs    |    jobs-01    |     NFS share         |  Incoming
           +--------->|               |<---------+            |   mail
           |          +---------------+          |            |
           v                                     v            |
    +---------------+                   +---------------+     |
    |   redis-01    |                   |  mail-in-01   |<----+
    +---------------+                   +---------------+
```

The app and jobs VMs run containerized workloads deployed with `mrsk`. We opted to run critical stateful workloads such as MySQL and postfix non-containerized on VMs.

Backpack’s migration started next to Writeboard’s, but took us around three weeks in total because of the new unknowns and extended acceptance testing we had to figure out. Nevertheless — it was a fairly smooth, zero-downtime cutover following the same steps as for the previous migrations — we’ve been exercising that muscle for a while now!

---

## Summary

Following a standard pattern for all apps, we achieved multiple wins:
- Huge reduction of infrastructure complexity — fewer moving parts
- Alignment of deployment strategies — no more special snowflakes
- Cut down deployment times to a fraction
- Massive code cleanup — check this diff for Tadalist:

![Tadalist Code Cleanup](https://dev.37signals.com/assets/images/bringing-our-apps-home/tadalist-pr.png)

On top of that, this move also pushed us to:
- Rewrite our entire configuration management with the current version of Chef
- Implement a super fast VM provisioning for our needs
- Upgrade from MySQL 5.7 to 8

**And most importantly, it challenged us to completely reconsider how our applications could be run.** Did they really need all that the cloud and Kubernetes had to offer? Wasn’t there a better, easier and less costly compromise for us?

Turns out there was — and we were able to pull this off while still keeping up some of the containerized goodness that we’ve worked on for years. You can imagine that we had ample celebrations during those first two months of 2023! 🎉

---

## Where we go from here

After our k8s on-prem plans had to be discarded, we went on quite a journey to solve the problem for us, in our own way, for our requirements, challenging our assumptions. Needless to say, the learning curve was steep, and this was (and still is) an all-hands-on-deck party for Ops at 37signals, while *still* keeping the lights on for everything else. #TheOpsLife didn’t just stop because of this effort.

But — it’s been fun! And the reduction of complexity and dependencies has only brought benefits so far — not just in day-to-day business, but soon enough also on our cloud bills. We’re quickly marching toward the next application transition right now.

Our apps aren’t the only thing that will come home — we’re exploring search backends, monitoring and so much more. The quest of de-clouding and de-k8sing is far from over, so stay tuned!
