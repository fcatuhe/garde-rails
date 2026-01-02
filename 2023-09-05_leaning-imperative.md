# Leaning imperative

**Author:** Arman Jindal, Summer Intern 2023, Ops
**Published:** September 5, 2023
**Source:** <https://dev.37signals.com/leaning-imperative/>

*Imperative infrastructure tools have trade-offs, but they help us directly and explicitly manage our infrastructure. We’ve embraced these tools and the philosophy they encourage. We are much better for it.*

---

When we migrated off the cloud, we not only cut costs and improved app performance — we also changed our tools. We got rid of [Kubernetes](https://kubernetes.io/), built [Kamal](https://kamal-deploy.org/) — an open-source, remote server, container deployment tool — and modernized our [Chef](https://www.chef.io/) server management setup. See our blog post [de-cloud and de-k8s — bringing our apps back home](https://dev.37signals.com/bringing-our-apps-back-home/) for more details.

These tools, unlike their “cloud native” counterparts, are *imperative* rather than *declarative*. I pointed out this contrarian move to [Eron](/author/eron), our Director of Ops, during my interview for the [summer internship](https://www.linkedin.com/posts/jason-fried_internships-summer-23-37signals-activity-7018316400838524929-RjRk) position. **I asked him how he thinks about imperative vs. declarative infrastructure tools.** He told me he’s still thinking about it!

I got the internship. It was incredible to join the world-class 37signals Ops team at the tail end of an [exciting journey off the cloud](https://37signals.com/podcast/leaving-the-cloud-the-finale/). As part of the Ops team and with their mentorship, I re-engineered our internal docs, re-wrote parts of the Chef infrastructure, and deployed an internal app with Kamal. Based on these experiences, this blog post answers the question I asked Eron during our interview.

---

## Are declarative infrastructure tools simpler?

In a declarative approach, you specify the end state and rely on the tool to create and maintain it (Kubernetes/Terraform). In an imperative approach, you specify a sequence of commands that creates the end state (Chef/Kamal). Proponents of declarative infrastructure tools often claim these tools are “simpler” than their imperative counterparts. True, declarative tools can do more with less code and are another abstraction on top of hardware. But less code and more abstraction aren’t always better. Their broad, unqualified claim of simplicity isn’t true. Simpler for who? Simpler to do what? Simpler at what scale? They weren’t “simpler” for us.

### Explicit over implicit

Imperative tooling encourages us to be more explicit. It’s like showing our work in a maths problem. Sure, it may take more time and space, but understanding, fixing and improving code is a lot faster because of it.

Declarative tools are, *by design*, implicit. It’s not your concern how the tool creates the final state, but that you specified it correctly. This becomes more complicated with more moving parts. Even the most hardened Kubernetes experts might only sometimes know precisely how it changes state under the hood. Hiding complexity “under the hood” when it needs to be managed is technical debt we don’t want in our critical infrastructure.

Here is an example from our modern Chef setup that creates a Kernel Virtual Machine (KVM) guest on a KVM host server.

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

Is it simple? Depends on who you ask. It’s definitely explicit. I can read the man page of `virt-install` and dig into each flag. If I converg Chef on a remote server and it fails, the error messages tells me exactly what failed and where. Declarative tools often don’t.

However, just because we use an imperative tool doesn’t mean our code will be explicit. Our ongoing Chef rewrite from “V1” to “V2” demonstrates a more explicit use of Chef. *Chef V1* was designed so that we could reuse cookbooks. As it grew it became more implicit and interdependent. More variables were passed through node attributes, the configuration for a single server had to be hunted down in multiple cookbooks, and upgrades were challenging because of interdependencies. Moving on-prem, meant relying more on Chef. So we re-wrote it with an entirely different philosophy.

Here’s a quote from our internal docs about the *why* behind our Chef modernization from V1 to V2:
- Upgrade to a modern and supported Chef client and server pair.
- Massively reduce the complexity of the various layers of Chef cookbooks, attributes, data bags and roles. Make things easier to understand. You should be able to understand most of a given config from a single screen, think Dockerfile.
- Drastically reduce the amount of code required to maintain our cookbooks for the next 5+ years. Making them easier to upgrade into future distros.
- Decouple the apps and systems from one-size-fits-all shared cookbooks to purpose driven cookbooks, allowing us to upgrade systems in isolation rather than having to carry every upgrade or change through every app.
- Reevaluate all the assumptions made over the past decade. Our Chef infra has its origins in code written over 13 years ago, and we’ve carried a lot of assumptions forward since then. It’s time to look at every piece of the system and add a comment about why it deserves to be there.

In my PR for a Chef V2 re-write, the [wonderful style nitpicks](https://dev.37signals.com/minding-the-small-stuff-in-pr-reviews/) from [Matthew Kent](https://github.com/mdkent) and [Silvia Uberti](https://github.com/Ladybiss) (our Chef V2 master-minds) are all to be more explicit: more breadcrumbs, reduce unneeded variable assignments, and write out config files in-line instead of with templates. The point is to have all the information we need to understand a cookbook staring you in the face.

One of Matt’s high-level comments was, “imagine you read this 3 years from now with little prior context.” Eron asks us to write code for the bleary-eyed 3 a.m. versions of ourselves that just got woken up by a PagerDuty alert. Both the 3 years and 3 a.m. versions will appreciate **very** explicit code.

This Chef modernization demonstrates a shift in the way we use our tools and in our philosophy. We now joke that, sometimes, it’s good to repeat yourself!

### It’s all about that state (‘bout that state…)

Imperative and declarative tools also create a different relationship to state.

Imperative tools often set up the state of a server through a sequence of commands, whereas declarative add “state reconciliation.” Kubernetes is a clear example. This is great, except when it isn’t.

The Reddit Ops team shared an experience that illustrates this point in their [excellent write-up on their PiDay outage](https://www.reddit.com/r/RedditEng/comments/11xx5o0/you_broke_reddit_the_piday_outage/). Following a **forced** Kubernetes upgrade(!), a misconfigured node selector in one of their [Calico](https://docs.tigera.io/calico/latest/about/) files used the old node terminology of “master” instead of “control-node.” This misconfigured node created all kinds of unexpected behavior in their cluster. The “state magic” (in this case of the “dark arts” variety) that tried to reconcile a misconfigured state obfuscated the problem to the engineers. Engineers had to reason about a system that was trying to change itself.

On the other hand, imperative tools lack this “state magic” and, by attempting less, help us separate concerns. A major simplifying factor for our “Chef V2” is that we use Docker. Instead of preparing each server for the app, we can lean on Docker to handle that app-specific complexity. The right combination of imperative tools helps us manage state more directly and explicitly.

### Static over dynamic

A benefit of declarative tooling (especially in the cloud) is auto-scaling. Infrastructure can grow (and shrink) dynamically in response to usage. It’s great, except we don’t need it. **The “manual” work of resizing is a feature, not a bug, of our infrastructure.**

Given how long we’ve been running our apps and our fairly predictable workloads, we know how to size our VMs. We have plenty of spare capacity; if they need more, we can always give it to them.

For example, we thinly provision our KVM guests. This means they can expand their resources on the physical host when needed. We configure appropriate limits to avoid runaway usage and have alerts for this. We then dig into why a KVM needs more resources.

Finally, because our servers live in a managed data center (see Eron’s [blog post](https://dev.37signals.com/37signals-datacenter-overview/) for more details) we know how much resources we need. We purchase hardware in response to our growing needs.

These helpful frictions make sure we understand the state of our system, keep our resources in check, and that we (the humans on the Ops team) are aware of what is going on.

---

## A mixed reality

The line between imperative and declarative can be blurry. Docker is an interesting example. The Docker file is imperative because the order of the commands matters, but it offers a DSL that works at a higher level of abstraction (i.e., pulling images). `Docker compose` operates more declaratively on Docker images. We aren’t polemically for or against imperative tooling.

The imperative vs. declarative distinction helps us explore trade-offs when choosing a tool. It isn’t about categorizing tools pedantically.  Many tools mix the two approaches in interesting ways. Ultimately, the right tool hides complexity that distracts us from the complexity we need to manage. That’s a team/org specific choice.

Maybe one day we can just declare complex infrastructure into being, and it will always JustWork™. That day isn’t today. There is hubris in thinking these systems will never break. [Software has bugs](https://37signals.com/podcast/software-has-bugs/) and infrastructure isn’t an exception. We don’t want to hide critical complexity to make things “more simple” in the short run.

In addition to being skeptical of claims of simplicity, we should also be skeptical of the many tools and services that help manage the “complex cloud.” Often the complexity is artificial and unnecessary. We went down a rabbit hole of using [EC2 spot instances to cut costs](https://aws.amazon.com/ec2/spot/). We shouldn’t have had these costs in the first place! That said, we’ve taken full advantage of improvements in infrastructure-as-code and containerization technologies, many of which were designed for the cloud. Just because a tool is “cloud native” doesn’t mean it can’t be used on-prem. We should be wary of what the cloud claims as its own and the complexity that it creates.

---

## Conclusion

At [37signals](https://37signals.com), we aren’t afraid to question “best practices” to find what works for us. The Ops team is no different. We tried Docker and loved it. The cloud and Kubernetes, not so much. So we moved our on-prem, adopted imperative infrastructure tools, and embraced a philosophy they encourage.

Though explicitly managing state and “manually” balancing workloads across our infrastructure requires more up-front work and verbose code, imperative tools and their philosophy help us directly manage our complex infrastructure. Our apps are more reliable and performant, and we (the humans on the Ops team) sleep better, too! That’s priceless.
