---
date: 2023-05-03
---

# deploying disk images at speed

## background

When I started at a modestly sized, AWS hosted web property several years back, I inherited an
environment that was built on Chef. Chef had served the company well enough. It crossed off all the
check boxes that the marketing literature specified like being able to configure your Ubuntu host
with some ruby code and you can provision hosts without touching them.

As I learned more about the environment and worked with it a bit, I started seeing it's flaws. Or
rather, it started screaming about it's flaws via the pager a few mornings every month. Regularly,
the auto-scaling monitor would report that it added a machine but after 15 minutes it was still not
healthy. The monitor had to wait that long because the auto-scaling-group (ASG) was tuned quite
conservatively. The regular process to bring up a new into service took about 10 minutes so it gave
a good bit of run way to soak up spikes while it provisioned more resources. This wasn't great for
timely alerts.

I'm not a Chef specialist and this was several years ago, but it wasn't a particularly exotic
arrangement. The disk image (AMI) was built with packer and it used cloud-init to pull in some
variables about it's environment (production or which staging). Then it reached out to GitHub to
pull down the repository to have Chef bring the system and application code up to the desired
state. This process relied on external resources including an apt repository to enable Datadog[^1].
One time, that repository was just broken. The most memorable outage was a combination of events
where new nodes were added that never became healthy but then the load subsided and the scale-in
mechanism reaped the good nodes, causing a full site melt down because of no healthy instances. It
was a bad day.

## how to improve

This isn't really a Chef problem and people with better Chef smarts than we had could have probably
addressed many of the flaws. We zoomed out to get a larger picture to see what our real problems
were.

### external dependencies

External dependencies when trying to bring up fleets of hosts, multiple times a day, that you cannot
control can cause chaos. We had a smattering of other failures as well like having GitHub go down
would prohibit scaling. Luckily, that didn't happen when it could have hurt us the most, but that
happens.

### long time to provision

The bursts that we received were often from web crawlers that would index older bits of the site
that were not in cache. There were earlier efforts via `robots.txt` and page metadata to try and
minimize it but not everyone plays by the rules. I came up with some helpful throttling modifiers
for our [Varnish](https://varnish-cache.org/) caching layer that also helped significantly.
2016-2020 was also huge in terms of our traffic because of some newsworthy event that happened
that day and sometimes, you just had to scale quickly and sustain it.

### use the tools we have

AWS has so many features but you don't need them all. Auto scaling groups, elastic load balancers,
RDS, and DNS could deliver a significant chunk of the web apps you might encounter on a daily basis.
One thing we didn't want to do is become dependent on containers. It was a volatile time in the
container space with Kubernetes coming around and Docker releasing a product just to re-write it and
re-release something else 6 months later.


## bold choices made

The developer environments (macOS laptops mostly) and repos had been slowly but steadily converted
from using a Homebrew with manual setup steps over to [Nix](https://nixos.org) in order to cut down
the time it took to onboard devs and make our environments repeatable. We were able to convert
much of the documentation for on-boarding from running random brew commands to 'install nix, follow
ssh setup, clone this repository and run the script, done' and you could be ready to go in an
afternoon. Nix was able to ensure all of the ruby (rails!) was setup and we had lock-step parity
between dev and our [build system](https://github.com/NixOS/hydra).

Nix builds software. A disk image with an operating system is just a special format for
holding way more software, right? Applying Nix to the whole machine yields NixOS. Could we take the
principles from building a rails project and incorporate them into a fast provisioning and resilient
disk image?

Well, we did and you could too.

### step 1. make an image

We wanted to keep the list of external dependencies to a minimum. Granted, we are running on AWS so
we assume some things are always going to work because if they don't, AWS is on fire anyway. The
only external call that the machine makes on boot is to the EC2 Metadata service, in order to get
it's credentials to unlock secrets that are sealed via AWS Key Management Service (KMS). If you'd
like an easy solution for secrets, I recommend [sops](https://github.com/mozilla/sops).

This means that every other bit of software and configuration needed to be packed into the disk
image. This was straight forward other than secrets and we needed to deal with those. We took the
opportunity to improve that as well because the previous Chef systems were shipped with the
decryption key which made secrets pointless.

I'm only going to briefly mention that NixOS exists because anything deeper would require volumes of
novels. Let me know if you are interested in a deeper run through of the process. Once you have your
software bundled up into `configuration.nix`, make sure you include the following to ensure the
build command will work:

```
nix imports = [ <nixpkgs/nixos/maintainers/scripts/ec2/amazon-image.nix> ];
```

Then, you are set to build.

```
$ NIXOS_CONFIG=$PWD/mywebserver.nix nix-build '<nixpkgs/nixos>' -A config.system.build.amazonImage <build build build output>
/nix/store/v2pa00hpc5xjsqg23p3ms7s4fgg2lrsh-nixos-amazon-image-23.05pre-git-x86_64-linux
$ ls
/nix/store/v2pa00hpc5xjsqg23p3ms7s4fgg2lrsh-nixos-amazon-image-23.05pre-git-x86_64-linux
nixos-amazon-image-23.05pre-git-x86_64-linux.vhd  nix-support/
```

The speed of this depends on how big your image is and how fast your builder is. If it's part of a
CI system and you are building regularly, much of it should already be in the nix store and cached.

Done. Now let's get it into AWS.

### step 2. image upload

Beware the AWS Snapshot importer service. We started with it because it was the only option available. It is a lengthy and variable
process where you would upload an image to S3 and then call on the AWS snapshot importer service
to turn it into a snapshot. From there, you could then register an AMI from that snapshot. On a good
day, this was between a 2-7 minute process. On the bad ones, it was 45 minutes. The times didn't
depend on image sizes because we could make big ones or small ones and it was the same. It varied
between regions but typically us-east-2 was a little quicker than us-west-2. Sometimes. The hardest
part was the variability and we couldn't know when speeds would pick back up. On the good side,
nothing was broken while this happened, it was just stopping new code from going out.

It is broken enough that I won't even mention it further here because there's a better way.

The new hotness came around when Amazon announced an API service that you could talk to EBS
snapshots directly. This allowed 3rd parties to quickly read and write from disks allowing
interesting tools like backups to work. They also provided
[coldsnap](https://github.com/awslabs/coldsnap), a command line tool to interact with this service
without using the low level API.

This replaces our ~120 lines of bash script to register a disk image and boils it down 2, though you
should do so more error checking here just in case.

```
$ snapshot_id=$(coldsnap --region $region  upload --description "$host-$(date)" --wait
/nix/store/v2pa00hpc5xjsqg23p3ms7s4fgg2lrsh-nixos-amazon-image-23.05pre-git-x86_64-linux/nixos-amazon-image-23.05pre-git-x86_64-linux.vhd)
aws ec2 register-image --output text --region $region --name  "$host-$(date +%s)" --description
"$host ami on $(date)" --architecture x86_64 --ena-support --sriov-net-support simple
--virtualization-type hvm  --tpm-support v2.0 --root-device-name /dev/sda1 --block-device-mappings
"[ { \"DeviceName\": \"/dev/sda1\", \"Ebs\": { \"SnapshotId\": \"$snapshot_id\" } } ]"
```

This happens at up to 500 megabytes per second if your network connection can handle it. If you are
doing this from a local machine, probably ~30 seconds of your time. You'd want to add some more
checks into that and tag your AMI so you know what it is and when you could reclaim it, but, there
it is.


### step 3. deploy and profit?

There is this app code sitting in a disk image, now it needs to run somewhere. This is going to vary
based on how your app needs to behave but at the least, you'll stick your AMI in a LaunchTemplate.
Do you need blue/green deployment? Do you need a canary test? There's [several examples for
Terraform](https://developer.hashicorp.com/terraform/tutorials/aws/blue-green-canary-tests-deployments)
that walk you through typical options.

We didn't choose Terraform to do the ASG juggling due to fairly distant, but painfully memorable,
trauma with Terraform. Instead, we opted to write our own ruby script (it was a ruby shop) to work
with the load balancers and target groups. The script had custom logic for automatic rollback and
alerting in case these new machines didn't pass health checks because bad code can some times make
it this far. In case of unhandled errors, it always failed in a safe manner and while it was in
play, had never triggered any customer facing outages.

Rollbacks are something that I had mentioned earlier but they became a killer feature for certain.
Because all of these new app instances were ephemeral (all cattle here, no pets.), and they were
quick to spin up and down, code rollbacks were done in ~30 seconds. The Chef and Capistrano runs,
which, may or may not have been able to rollback successfully based on the accumulated, shared,
mutable state, took significantly longer.


## result

In the end, we came up with a "cattle-first" solution that was both more reliable and faster than
our previous mechanism. Around 12 minutes after a developer's commit had landed, the code would be live. Keep
in mind that this was still using some old methods and with some tweaks, this time could be cut in
half. This was more than adequate for our deployment frequency which usually was around 2-6 times
per day.

Our deployments were now fully baked and had a single dependency on KMS to unseal secrets. No
failure of an external service (s3, software repos, container registries) would keep our machines
from booting and serving requests.

Since the machines had to do no work other than to start up a rails server, the total time to bring
up a new server was around 45 seconds. Around 15 of that was spent in the kernel and booting. We
could now tune our scaling algorithms to be more efficient. Rather than averaging 30% CPU
utilization, we could now bring them closer to 50%. I tinkered with higher utilization than that but
rails response time started taking a hit.

Since spin up times were now less than 2 minutes, we were more confident in running servers on spot
instances which was a significant savings as well.


[^1]: I really liked Datadog for it's dashboarding and fault analysis. It was missing some things,
    but we generally got along well.
