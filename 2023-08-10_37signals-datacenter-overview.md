# 37signals datacenter overview

**Author:** Eron Nicholson, Director of Operations
**Published:** August 10, 2023
**Source:** <https://dev.37signals.com/37signals-datacenter-overview/>

*During our journey off the cloud, we’ve received a lot of questions about our datacenters. No, we do not run them on our own. I’m here to discuss at a high level what 37signals’ datacenter presence looks like.*

---

## Our datacenters

All of 37signals’ apps that serve millions of users every day are served from one datacenter in Ashburn, VA and one in Chicago, IL. This includes [Basecamp](https://basecamp.com), [HEY](https://hey.com), and our many heritage apps that we have dedicated to run [until the end of the internet](https://37signals.com/policies/until-the-end-of-the-internet/). Both of those facilities are run by [Deft](https://deft.com/), who we have been a customer of for more than a decade.

---

## A bit of history

We have actually been laying the groundwork for our move off the cloud and a long-term future in the datacenter for a few years now. We first installed servers in Chicago with Deft, then ServerCentral, in 2010. In 2012 we added our second site with another provider in Virginia. They were a good partner for years but for a variety of reasons we decided to move a few miles away to Deft’s facility in 2021. At the same time, we knew that our cabs in Chicago were becoming increasingly difficult to work with. They were relatively low power, had two older generations of network gear in them plus the accumulation of 10+ years of wiring. Working with Deft, we designed and built an identical future-proofed footprint in each of their sites with a new, modern network setup. We then moved every one of our servers either between buildings (in Ashburn) or between rooms (in Chicago) over the course of about six months. It was quite the undertaking but we accomplished it with no customer impact.

---

## Our carbon-neutral physical footprint

We have four cabinets in both datacenters, each 48 [rack units](https://en.wikipedia.org/wiki/Rack_unit) tall. We can theoretically support 48 single rack unit servers per cab, but in practice, we need to leave room for networking equipment and cable management. We have a mix of single and double rack unit servers, all provided by [Dell](https://www.dell.com), who has been a great partner of ours for years. At the moment we have somewhere between 20-25 servers in each cab, or about 90 servers in each site. Here’s what the rack layout looks like in Chicago, for instance.

![Chicago Cab Elevation](https://dev.37signals.com/assets/images/37signals-datacenter-overview/cabs.png)

Deft provides us 40kw of power split among the four cabs in each site, so about 10kw per cab. Their facility in Chicago is powered entirely by renewable energy. For Ashburn we [purchase carbon credits](https://m.signalvnoise.com/basecamp-has-offset-our-cumulative-emissions-through-2019/)  each year to offset our non-renewable energy usage there.

---

## Networking

We built our new network with redundancy at each layer to ensure that we can withstand the loss of any single link or single device. Our network topology reflects that.

![Network Diagram](https://dev.37signals.com/assets/images/37signals-datacenter-overview/network-diagram.png)

The internal network is powered by [EVPN](https://en.wikipedia.org/wiki/Ethernet_VPN)/[VXLAN](https://en.wikipedia.org/wiki/Virtual_Extensible_LAN) technology developed to handle very large networks like those run by cloud providers. We have external connections to the internet, our other datacenter and AWS. Each of those utilizes different providers and diverse physical paths to ensure resilience.

---

## Performance and redundancy

We leverage both datacenters for Basecamp 4 and our marketing sites using a technology called BGP Anycast. It means that you access Basecamp using whichever site is fastest for you. I [wrote about it in detail](https://signalvnoise.com/posts/3937-optimizing-site-performance-via-anycast-routing) back in 2015 and we have been using it successfully ever since. Our other critical apps run in both datacenters but in an active/standby capacity, where we can move the site to the other datacenter if catastrophe strikes.

---

## The future

We just finished moving our apps off the cloud and [our apps are faster than ever](https://world.hey.com/dhh/cloud-exit-pays-off-in-performance-too-4c53b697) but we are by no means done improving! Later this year we will convert HEY to actively use both datacenters in the same way that Basecamp 4 does now. Later we will deploy more sites internationally to speed up access for our customers around the world. We will continue to convert our apps that never left the datacenter onto [MRSK](https://mrsk.dev/), using newer and faster hardware. This will both speed up the apps and allow us to decommission dozens of older servers. Our move out of the cloud may be done, but work to make our systems better will continue for years to come. Our team will be sharing more details on how we are building modern apps for the datacenter in the coming months, so stay tuned!
