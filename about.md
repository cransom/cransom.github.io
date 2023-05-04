# About

You might think of laziness being a bad thing. Many years ago when I was getting my start, one of my coworkers called me
the 'lazeministrator'. Honestly, I can't remember any more if it was a legitimate slight or a complement but I'll take
and own it now because it fits.

At my first ops gig, we ran a service that did site search and some hosting for a number of large and small web
properties. Yep, it existed before Google and it was pretty good. We had better features than they did, including better
customization, better results, more language support, etc. It was an entirely hosted service proudly running on ~100
FreeBSD machines in an Equinix data center in San Jose.

A team of 4 at our largest, we did everything including some interesting networking where every machine was homed on two
separate networks. These networks were completely isolated and we used some fun routing protocol behaviors (starting my
long term love of BGP) and careful DNS tweaks to ensure that we could do network maintenance at any time, fully taking
down a main switch, firewall, or load balancer in order get big fixes or maintenance done.

Atomz, which eventually was swallowed by multiple entities, ultimately ending up in Adobe's hands, was formative in my
career and where I learned an appreciation for everything. Networking, system administration, automation, monitoring, we
did it all. We had bots on IRC to see and acknowledge pages, preen tickets, and more importantly, chat with coworkers
because while it wasn't fully remote, even in the office, everyone was on IRC.

From there, I transitioned on to being a full time network engineer. I worked on switches, routers, and firewalls from
several different vendors. Juniper ended up being my drug of choice for various reasons. One of the things that
networking devices didn't have a particularly good story for though, was automation. Systems folks had a plethora of
choices. CFEngine had been around a while. Puppet and Chef were brand new on the scene and promised all sorts to fancy
features to provision and deploy software to all of your machines.

The network though, maybe you had a Cisco environment and could use CiscoWorks. If you ever wanted to manage something
from a different vendor though, **buzz**, nope. In order to handle that, you basically had to roll your own automation
system and it likely was based around a mountain of home-grown expect scripts. If you had just a little bit of ambition,
hours of the day were saved by bundling up regular changes into a script. Network automation was a killer feature and
interestingly, ever mentioning it to a software developer would cause their eyes to glaze over and you would meet only
blank stares. They could talk your ears off about message busses, event queues, distributed systems, but if you talked
about physical networking, of which a network is all of those things, they lost all interest. An 802.1q VLAN was magic.

So I worked a lot on automation. Mozilla was a big puppet shop and while I was there, I was dabbling, and Juniper was
trying, to make puppet a viable automation platform. They didn't get a lot of buy in and the number of devices that
supported their puppet agent was quite small, so it never really went anywhere. I eventually moved on from Moz and did
some other things and I was dragged into The Cloud.

I worked for a company that had decided to close their physical data center in downtown San Francisco (why would you put
one there?) and move into AWS. It was a multi-year project that was extended multiple times due to difficulty the
software devs getting time to update systems to migrate elsewhere. My purpose as the senior network engineer was more
clear cut though. We needed to have redundant VPNs and occasional bursts of traffic into AWS for data migration. We also
needed to keep the lights on in the data center under a straining infrastructure that would never see an upgrade. I did
some fun things like flipping our big Juniper SRX pair from Active/Passive into Active/Active and routing table hacks to
circumvent the stateful firewalls for particular flows in order to buy some more time.

The last large project I worked on was setting up an automated VPN provisioning system between our VPCs in each region.
It used BGP to work with Amazon and share routing information and also do some firewalling between the zones. With
Ansible and VyOS, I could spin up a new router from scratch that would chat with AWS APIs to configure IPSEC
connections, setup BGP peering, and scale bandwidth between regions without requiring any other manual effort. At the
very, very end, the services got out of the data center and the company was now entirely in the cloud. I had fulfilled
my primary mission by automating myself out of a job because who needs a network engineer in AWS? Follow up spoiler
here. Around 6 months later, Amazon decided to release the feature to have encrypted tunnels between VPCS in different
regions, thus making that work irrelevant. Oh well.

It was at this last place where I was introduced into a piece of technology and methodology that changed my life. I was
hesitant at first, but my bold peers made huge claims that this Nix thing they heard about was something we needed to
integrate. I was, more or less, part time network at this point so I was helping in this Nix adventure. My first release
was nixos-14.12 and we had successfully spun up a small, but important, part of the environment using nixops.

It had it's quirks and the method we used wasn't something you'd want to do now, but I was hooked and NixOS became my
server OS of choice. I wiped my random Ubuntu machines and converted to NixOS. My desktop remains MacOS since nix is
fairly well supported on it but I'm a die hard user and I can't imagine using anything else.

I now roam the land, preaching the greatness of Nix and how you bolt it into your build systems, your development
shells, and into production to help you sleep through the night.


