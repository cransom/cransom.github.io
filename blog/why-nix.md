---
date: 2023-05-01
---
# Nix makes Linux usable

## A little history

I've been a Unixy user for the majority of my life. I started out with a Debian 1.2 disk
from CheapBytes and I fumbled with that until a group of friends told me to get away from that junk
and get onto FreeBSD. Because of that, FreeBSD 4.0 will forever have a place in my heart because the
FreeBSD experience was just right.

It was the first time I was able to successfully compile a kernel, do upgrades, and manage nearly
all of my system in `rc.conf`. Any weirdness in the system was something I could unbreak with a
`make world` and removing `/usr/local` and reinstalling whatever software I had from the ports tree.
My biggest winner and takeaway though, was `rc.conf`.

I became a network engineer and worked predomninantly on Juniper hardware. It too was built on
FreeBSD and using it day to day was so easy. It used a `juniper.conf` and while it wasn't a simple
key-value pair setup that you could interpet in `/bin/sh`, it did make sense in the tree of options
it presented. Under the hood it was XML. On top of the hood, it was something called `yang` and it's
something that many vendors now use. But most importantly, this file is a single source of truth for
how your device worked.

Prior to `juniper.conf`, I was stuck manipulating my switches, routers, and firewalls across vendors
with expect scripts. You could do powerful things with them and change every aspect of them system,
but, it was hard because the basic behavior I was able to impart was always imperative. As in,
"change this setting to X, change that setting to Y". This is similar to other traditional methods
of Unix administration (e.g., puppet and chef) where on each run, they could make a change but were
completely unaware of all of the states that existed before. Most of the time, that's ok. Sometimes
though, it does the wrong thing and triggers Michael Bay style explosions.

`juniper.conf` though, kind of like `rc.conf`, was my single source of truth for the system. It
described users, vlans, interfaces, dns server, dhcp servers, and every other detail of the system
for it to run. Every Linux I had encountered[^1] didn't have this and it made me feel smug being a
network engineer because I could command my device to do exactly what it needed to do without
guessing if users were left behind or if there was some old configuration somehow left on an
interface. Now, I didn't have those popular, fun tools like all of the sysadmins had to manage their
piles of devices but soon those tools lost their shine.

Life changed when I was assaulted with an idea from my coworkers. They found this NixOS project and
proceeded to build a service with it and somehow I was roped into helping support it. I didn't
understand it at first because it was in this weird language that required some kind of
understanding of functional programming. Why would I want this when I have ansible and yaml can
build what I need? After stewing on it, I finally understood. `configuration.nix` was my
`juniper.conf`. At that point, It didn't matter what it was written in. I needed it like oxygen.

## The change

> "When you realize you want to spend the rest of your life with somebody,
> you want the rest of your life to start as soon as possible."

I had a FreeNAS appliance (because ZFS and FreeBSD) which became a NixOS appliance. My Linode VM,
which ran all of services swapped from Ubuntu to NixOS. Because I'm a computer nerd at heart, I have
a bunch of technology at home. At points, it rivaled the complexity of environments at work. When I
made this transition onto a truly declarative configuruation setup, so much of my environment became
more stable and boring. Boring in that, I didn't have to spend nearly as much time keeping things up
to date. I had an OS that could tell me exactly what the upgrade was going to do and if something
was broken, I could roll it back and deal with it then or belay it to another time.

My router, a thinkpad[^2], runs NixOS. A behavior that can get you into trouble when you
are doing small batch, artisinal computing is making changes directly on a machine. If you aren't
careful and keeping track of changes is that it mysteriously starts working somewhere along the way.
You try to repeat it but it didn't work because you changed 5 different files and can't tell which
one made the difference. NixOS, provided you obey the rules of the system, keeps you from doing this to
yourself. Every change you make is to `configuration.nix` and is then applied to the system with
`nixos-rebuild`[^3] which ensures that daemons get restarted, configs are updated, and if you
reboot, everything will keep on trucking. If you goof it up? `--rollback` will fix it.


## The result

There's so much potential energy in Nix and NixOS that drives me insane that it's not more popular
than it is. I fear it's ruined me because it's a topic I can throw into so many different computing
problems that you might have.

* Onboarding takes forever: Nix and direnv solve that.
* the software is hard to build: nix solves that.
* running multiple services on a single machine means we need containers: Nah it doesn't.
* we need containers because building disk images is insane: Stay tuned.



[^1]: This was basically Ubuntu, Redhat, and Centos because that's what everyone used.
[^2]: Powerful, lots of ram and cpu, and it's own on-board keyboard/display when things go real bad.
[^3]: Or whatever change you are making to your system and then deploying it
