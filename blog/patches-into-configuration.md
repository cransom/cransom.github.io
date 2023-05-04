---
date: 2018-04-05
---
# Turning patches into configuration

## A foreword

I wrote this back in 2018 with the intent to publish but it never made it out. Wildly, 5 years later, the technique would still apply. I don't use znapzend any longer, but, to keep the story updated, the patch is now irrelevant since upstream changed it to a 10 minute timeout.

## Back to the action.

I recently migrated my home storage machine from [FreeNAS](https://www.freenas.org) to a [NixOS](https://www.nixos.org) machine. The FreeNAS machine was perfectly and I would recommend it to anyone, it just lacked some flexibility on services and available software.

FreeNAS has a built in client for ZFS replication and it works well. I needed to replicate that and there are several tools available, though Znapzend (I really hate typing that name ...) was already packaged and has a module. Great! This should be easy. Znapzend uses an app called mbuffer to buffer the writes from `zfs send` into memory and dumps it into `zfs receive` when it's capable of taking more data. It has an option to abort if it hasn't been able to move data for some amount of (configurable) time.

Since I'm using the snapshot replication just for backups, my backup drive has gzip-9 and de-duplication enabled. It's typical, at least for me, that the buffer can stall for 2-3 minutes at a time while ZFS is reading through data on the receiving side while it tracks deduplication data. Once all the reads have completed, it dumps a couple gigs of info through the buffer and carries on.

Znapzend allows you to change the buffer size however the stall timeout is hard coded to 60 seconds. I tried dumping data a few times, but each time, all of my replications failed except for very small snapshots and file systems. Mbuffer reported in the logs that it had stalled and dutifully aborted. The fix was easy to find since Znapzend is just Perl and it shells out to the system for ZFS and mbuffer commands. I found the arguments for mbuffer and changed it from 60 seconds to 15 minutes. I could have also removed it entirely but timeouts are a good thing so that I can know a failure happened.

Uh oh. I made changes to software. I could (and should!) log an issue up stream to make the timeout configurable. This isn't possible for all projects, but it is a good idea and iterative improvements are how good software is made. But how do I manage this going forward?

With NixOS, it's easy. The same file that configures the system to setup services and install software also defines how the software is built. I can add in some non-default options and not have to worry about missing a patch or running old software.

```nix
nixpkgs.overlays = [(
self: super:
  {
    znapzend = super.znapzend.overrideAttrs (oldAttrs: rec {
      postPatch = ''
        substituteInPlace lib/ZnapZend/ZFS.pm --replace '-W 60' '-W 900'
      '';
    });
  }
)];
```

Every time Znapzend is updated, I update my file. It isn't perfect. It could be that other references to `-W 60` could appear in that file at some point and bad things could happen. A better approach could be that I create a unified diff and a patch. If the patch apply fails, my system will fail to build and tell me exactly why and that's even way before it tries to run broken software.


