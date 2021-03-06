# `p2` Hooks

Hooks are code that may execute at predefined moments in the `p2` preparer workflow. Hooks are registered in a fashion similar to pods, although several key differences exist between pods, intended as applications, and hooks.

Hooks can be declared and deployed using the `p2-hook` utility. Here is an example creating and deploying a hook that sets up the right application users before the application is launched. `p2-hook` can be run on any host with access to the Consul cluster.

```bash
$ echo '#!/bin/bash
useradd -D -d /data/pods/$POD_ID $POD_ID
' > ensure_user
$ chmod o+x ensure_user
$ MANIFEST=$(p2-bin2pod ensure_user | jq '.["manifest_path"]')
$ p2-hook schedule $MANIFEST before_install
```

This hook establishes that the user running your application is present on the host before any launchable begins.

## Hook Constraints

Hooks are run as the user running the preparer. This will be root for most installations. Any future authentication mechanism will be required when scheduling hooks as they permit the rapid deployment of code that will execute as root in your cluster. Needless to say, `p2` is still in development and we do not recommend deploying it in production yet.

Hooks run with time restrictions. After 30 seconds, the preparer will send the hook SIGTERM and proceed with operations. At 60 seconds if the hook is still running, the preparer will send a SIGKILL.

Finally, hooks cannot alter the execution of the preparer, even if they fail. This is a safety feature similar to the timeouts. This prevents a broken hook from preventing deploys across your cluster.

## Fundamental Hooks Design

At its root, `p2`'s hooks are simply a directory of scripts that are executed during the install and launch phases of `p2` launchables. Each directory can be populated independently of `p2`. However, the `p2-preparer` comes with an option to register all pod manifests and their launchables at `/hooks` as scripts that will be symlinked into this directory. The option is on by default.

## Layout

Values in the `/hooks` keyspace are mapped to hooks on disk in the following manner. For a key value of `/hooks/{event}/{manifest}`, every preparer should install a new pod under `/data/hooks/pods/{event}/{manifest}`. The layout of the pod directory is identical to that of normal applications.

The layout of hooks on disk is somewhat complex, owing to the reuse of the pod design. While somewhat complex, this reuse gives hook implementors all the power of standard `p2` ideas. For the most part, clients shouldn't need to understand how `p2` lays out hooks on disk.

Instead of setting up runit services for each launch script, `p2` instead finds each launch file and symlinks it into the appropriate exec directory. For example, for a hook that adds appropriate sudoers entries given a manifest, if the hook script is in a pod identified as `system` under a launchable called `sudoers`, the hook script will be installed at `/data/hooks/pods/before_install/system/sudoers/current/bin/launch` and it will be symlinked to `/data/hooks/exec/before_install/system_sudoers_launch`.

For launch directories instead of launch scripts, each launch script will be installed into the event directory as needed.
