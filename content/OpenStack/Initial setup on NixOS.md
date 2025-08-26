My interest in OpenStack started during a cloud computing course, where it was used for hands-on labs and to illustrate core cloud concepts. Since I prefer to self-host my development environments, by self-host I mean running everything locally so I can experiment and tinker. I thought setting up OpenStack would be straightforward. After all, getting platforms like rke2 up and running is pretty simple. How hard could OpenStack be?😅

When I searched for "OpenStack installation," I landed on [this official guide](https://docs.openstack.org/install-guide/). It’s thorough, but honestly, a bit overwhelming. I prefer hands-on learning over wading through lengthy documentation, just give me something I can run, experiment with, and break, then I’ll figure out the rest. While detailed docs are important (please keep writing them!), I wish there were more streamlined, automated ways to get started for those of us who learn by doing.

Good thing I’m running NixOS (btw 😉). There’s probably a module for OpenStack! If you don’t know Nix, it has module system that makes configuring your system easy, mostly as easy as setting `<thing>.enable = true` and you’re off. For example, running k3s in NixOS is as simple as [`services.k3s.enable = true`](https://search.nixos.org/options?channel=unstable&show=services.k3s.enable). You can always tweak more settings later [through the other options](https://search.nixos.org/options?channel=unstable&query=services.k3s), but most modules have sane defaults. I could go on about NixOS, but let’s get back on topic.

[Searching the official NixOS configurations](https://search.nixos.org/options?channel=unstable&query=openstack), we see something with `zfs` (which I believe is a file-system). Nothing official, so let’s check the community. A quick search leads to [this article](https://cyberus-technology.de/en/articles/simplifying-openstack-with-nixos/), looks promising! They have an [openstack-nix](https://github.com/cobaltcore-dev/openstack-nix) project with the packages and module I need. Awesome, let’s try it!

Let’s add an input to my [.dotfiles](https://github.com/mirza-src/.dotfiles):

```nix title="flake.nix"
{
  # ...
  inputs = {
    # ...
    openstack-nix.url = "github:cobaltcore-dev/openstack-nix";
    # ...
  };
  # ...
}
```

Now, let’s check what modules are available under `modules/default.nix`:
https://github.com/cobaltcore-dev/openstack-nix/blob/e97581816584486555d2647ae2b2aae6b1751197/modules/default.nix#L1-L17

> [!tip]- Tip
> `default.nix` is like `index` files in Nix, meaning they are imported when a _directory import_ is used.

There are three modules: `controllerModule`, `computeModule`, and `testingModules`. We can obviously ignore `testingModules` 🙃. `controllerModule` seems to be a superset of `computeModule` (it takes all the inputs of `computeModule` and more). My best guess: OpenStack is like Kubernetes, you have controller nodes and worker (compute) nodes and the `controller` nodes can also act as `worker` nodes. Let’s see how to use `controllerModule`.
One thing I’ve learned about the Nix ecosystem is that people seem allergic to documentation. They’ll say “the Nix language is self-explanatory” or something along the lines. But it took me a while to get comfortable with Nix and I still struggle sometimes. So either I’m missing some brain cells (I like to think otherwise), or it’s just not true. Anyway, with Nix you have to dive into the source code. Let’s look at the imported file `modules/controller/openstack-controller.nix`:

https://github.com/cobaltcore-dev/openstack-nix/blob/e97581816584486555d2647ae2b2aae6b1751197/modules/controller/openstack-controller.nix#L36

Huh, weird, there’s no `options` section, just `config`. In Nix, `options` lets users customize the module, and `config` is where you implement it (run processes, create files, etc). So, we can’t configure this module, just import it as-is. Not ideal, but let’s try importing it:

```nix title="hosts/proart-p16/default.nix"
{
  pkgs,
  config,
  lib,
  inputs,
  ...
}:
{
  # ...
  imports = [
    # ...
    inputs.openstack-nix.nixosModules.controllerModule
    # ...
  ];
  # ...
}
```

And rebuild the system:

```sh
sudo nixos-rebuild switch --flake .
```

And voilà! It worked as expected i.e. it crashed 💥.

```log
error:
       … while calling the 'seq' builtin
         at /nix/store/bs1sz392mx4ig6l17a6baj8qv1w9pr4s-source/lib/modules.nix:361:18:
          360|         options = checked options;
          361|         config = checked (removeAttrs config [ "_module" ]);
             |                  ^
          362|         _module = checked (config._module);

       … while evaluating a branch condition
         at /nix/store/bs1sz392mx4ig6l17a6baj8qv1w9pr4s-source/lib/modules.nix:297:9:
          296|       checkUnmatched =
          297|         if config._module.check && config._module.freeformType == null && merged.unmatchedDefns != [ ] then
             |         ^
          298|           let

       (stack trace truncated; use '--show-trace' to show the full, detailed trace)

       error: attribute 'controllerModule' missing
       at /nix/store/vlvrih6f3qca07g3ay7gc486zpnwiq7r-source/modules/nixos/openstack.nix:4:5:
            3|   imports = [
            4|     inputs.openstack-nix.nixosModules.controllerModule
             |     ^
            5|   ];
```

BTW, Nix error messages are notoriously confusing, this one is actually one of the _better_ ones I’ve seen.

It looks like `controllerModule` doesn’t exist. That’s odd, this is usually how flakes are structured. Let’s double-check the module’s `flake.nix` just to be sure:

https://github.com/cobaltcore-dev/openstack-nix/blob/e97581816584486555d2647ae2b2aae6b1751197/flake.nix#L12C1-L12C12
It’s easy to miss what’s going on here: `nixosModules = import ./modules { openstackPkgs = packages; };` is in the `outputs` like you’d expect, but then why the error? Turns out, they’re passing the whole attribute set to `flake-utils.lib.eachSystem [ "x86_64-linux" ]`. If you haven’t used `flake-utils` before, here’s the deal, it basically takes all your outputs and slaps a system name (like `x86_64-linux`) in front of every key. So instead of `nixosModules.controllerModule`, you get `nixosModules.x86_64-linux.controllerModule`. That’s why the usual import fails, just gotta use the system-prefixed version and you’re good!

```nix title="hosts/proart-p16/default.nix"
{
  pkgs,
  config,
  lib,
  inputs,
  ...
}:
{
  # ...
  imports = [
    # ...
    inputs.openstack-nix.nixosModules.x86_64-linux.controllerModule
    # ...
  ];
  # ...
}
```

> [!information]-
> This isn’t the usual way modules are exposed in Nix flakes. Per-system attributes are mostly for things like `packages`, modules are supposed to be system-agnostic. If you need different behavior for different systems, you’d use conditionals inside the module, not split them out like this.

Anyway, let’s try rebuilding again:

```sh
sudo nixos-rebuild switch --flake .
```

After a bit of waiting, we’re greeted with even more errors 🥲:

```log
warning: the following units failed: glance-api.service, glance.service, keystone-all.service, placement.service
× glance-api.service - OpenStack Glance API Daemon
     Loaded: loaded (/etc/systemd/system/glance-api.service; enabled; preset: ignored)
     Active: failed (Result: exit-code) since Mon 2025-08-25 19:28:44 CEST; 134ms ago
   Duration: 656ms
 Invocation: 575ff1f1d95a415498444143ebe3d239
    Process: 2477934 ExecStart=/nix/store/69v7i82yws91gpdpmba6p0f8a4503dyv-exec.sh (code=exited, status=99)
   Main PID: 2477934 (code=exited, status=99)
         IP: 0B in, 0B out
         IO: 11M read, 0B written
   Mem peak: 64.1M
        CPU: 544ms

Aug 25 19:28:43 proart-p16 systemd[1]: Started OpenStack Glance API Daemon.
Aug 25 19:28:44 proart-p16 69v7i82yws91gpdpmba6p0f8a4503dyv-exec.sh[2477937]: ERROR: Failed to find some config files: /etc/glance/glance-api-paste.ini
Aug 25 19:28:44 proart-p16 systemd[1]: glance-api.service: Main process exited, code=exited, status=99/n/a
Aug 25 19:28:44 proart-p16 systemd[1]: glance-api.service: Failed with result 'exit-code'.
Aug 25 19:28:44 proart-p16 systemd[1]: glance-api.service: Consumed 544ms CPU time, 64.1M memory peak, 11M read from disk.

× glance.service - OpenStack Glance setup
     Loaded: loaded (/etc/systemd/system/glance.service; enabled; preset: ignored)
     Active: failed (Result: exit-code) since Mon 2025-08-25 19:28:43 CEST; 1s ago
 Invocation: c693aa4bace64e6f8631758a555900fb
    Process: 2477890 ExecStart=/nix/store/zps4l82ch6fb16jdaqp8vjm06scpklgb-glance.sh (code=exited, status=1/FAILURE)
   Main PID: 2477890 (code=exited, status=1/FAILURE)
         IP: 0B in, 0B out
         IO: 41.4M read, 0B written
   Mem peak: 97.7M
        CPU: 512ms

Aug 25 19:28:43 proart-p16 systemd[1]: Starting OpenStack Glance setup...
Aug 25 19:28:43 proart-p16 zps4l82ch6fb16jdaqp8vjm06scpklgb-glance.sh[2477890]: + openstack user create --domain default --password glance glance
Aug 25 19:28:43 proart-p16 zps4l82ch6fb16jdaqp8vjm06scpklgb-glance.sh[2477891]: Failed to discover available identity versions when contacting http://controller:5000/v3. Attempting to parse version from URL.
Aug 25 19:28:43 proart-p16 zps4l82ch6fb16jdaqp8vjm06scpklgb-glance.sh[2477891]: Unable to establish connection to http://controller:5000/v3/auth/tokens: HTTPConnectionPool(host='controller', port=5000): Max retries exceeded with url: /v3/auth/tokens (Caused by NameResolutionError("<urllib3.connection.HTTPConnection object at 0x7f26fd34cb90>: Failed to resolve 'controller' ([Errno -2] Name or service not known)"))
Aug 25 19:28:43 proart-p16 systemd[1]: glance.service: Main process exited, code=exited, status=1/FAILURE
Aug 25 19:28:43 proart-p16 systemd[1]: glance.service: Failed with result 'exit-code'.
Aug 25 19:28:43 proart-p16 systemd[1]: Failed to start OpenStack Glance setup.
Aug 25 19:28:43 proart-p16 systemd[1]: glance.service: Consumed 512ms CPU time, 97.7M memory peak, 41.4M read from disk.

× keystone-all.service - OpenStack Keystone Daemon
     Loaded: loaded (/etc/systemd/system/keystone-all.service; enabled; preset: ignored)
     Active: failed (Result: exit-code) since Mon 2025-08-25 19:28:43 CEST; 2s ago
 Invocation: a85015f8dde34cad834fed8c7ff1b126
    Process: 2474346 ExecStartPre=/nix/store/ywlk00615x0578vy15nmnjij4n4s7vif-unit-script-keystone-all-pre-start/bin/keystone-all-pre-start (code=exited, status=1/FAILURE)
         IP: 0B in, 0B out
         IO: 85.5M read, 8K written
   Mem peak: 184.4M
        CPU: 1.591s

Aug 25 19:27:00 proart-p16 systemd[1]: Starting OpenStack Keystone Daemon...
Aug 25 19:28:43 proart-p16 systemd[1]: keystone-all.service: Control process exited, code=exited, status=1/FAILURE
Aug 25 19:28:43 proart-p16 systemd[1]: keystone-all.service: Failed with result 'exit-code'.
Aug 25 19:28:43 proart-p16 systemd[1]: Failed to start OpenStack Keystone Daemon.
Aug 25 19:28:43 proart-p16 systemd[1]: keystone-all.service: Consumed 1.591s CPU time, 184.4M memory peak, 85.5M read from disk, 8K written to disk.

× placement.service - OpenStack Placement setup
     Loaded: loaded (/etc/systemd/system/placement.service; enabled; preset: ignored)
     Active: failed (Result: exit-code) since Mon 2025-08-25 19:28:44 CEST; 1s ago
 Invocation: 6d66a5a863444df5b971baacdff069d0
    Process: 2477935 ExecStart=/nix/store/kwx2c3vbash22mpk7mxq30kfj0fbp1br-placement.sh (code=exited, status=1/FAILURE)
   Main PID: 2477935 (code=exited, status=1/FAILURE)
         IP: 0B in, 0B out
         IO: 4K read, 0B written
   Mem peak: 55.5M
        CPU: 473ms

Aug 25 19:28:43 proart-p16 systemd[1]: Starting OpenStack Placement setup...
Aug 25 19:28:43 proart-p16 kwx2c3vbash22mpk7mxq30kfj0fbp1br-placement.sh[2477935]: + openstack user create --domain default --password placement placement
Aug 25 19:28:44 proart-p16 kwx2c3vbash22mpk7mxq30kfj0fbp1br-placement.sh[2477936]: Failed to discover available identity versions when contacting http://controller:5000/v3. Attempting to parse version from URL.
Aug 25 19:28:44 proart-p16 kwx2c3vbash22mpk7mxq30kfj0fbp1br-placement.sh[2477936]: Unable to establish connection to http://controller:5000/v3/auth/tokens: HTTPConnectionPool(host='controller', port=5000): Max retries exceeded with url: /v3/auth/tokens (Caused by NameResolutionError("<urllib3.connection.HTTPConnection object at 0x7f82c2750b90>: Failed to resolve 'controller' ([Errno -2] Name or service not known)"))
Aug 25 19:28:44 proart-p16 systemd[1]: placement.service: Main process exited, code=exited, status=1/FAILURE
Aug 25 19:28:44 proart-p16 systemd[1]: placement.service: Failed with result 'exit-code'.
Aug 25 19:28:44 proart-p16 systemd[1]: Failed to start OpenStack Placement setup.
```

So, a few issues: failed requests to `http://controller:5000`, DNS can’t resolve `controller`. This should be `localhost` for us, but I can’t configure the module, so let’s add an entry in `/etc/hosts` (the Nix way):

```nix title="hosts/proart-p16/default.nix"
{
  pkgs,
  config,
  lib,
  inputs,
  ...
}:
{
  # ...
  networking.hosts = {
    "127.0.0.1" = [ "controller" ];
  };
  # ...
}
```

You know the drill: rebuild the system and get greeted by more errors:

```log
warning: the following units failed: glance-api.service, glance.service, keystone-all.service, placement.service
× glance-api.service - OpenStack Glance API Daemon
     Loaded: loaded (/etc/systemd/system/glance-api.service; enabled; preset: ignored)
     Active: failed (Result: exit-code) since Mon 2025-08-25 19:41:50 CEST; 107ms ago
   Duration: 667ms
 Invocation: 3cd716cc9cf64ef882accfb490b0e415
    Process: 2505780 ExecStart=/nix/store/69v7i82yws91gpdpmba6p0f8a4503dyv-exec.sh (code=exited, status=99)
   Main PID: 2505780 (code=exited, status=99)
         IP: 0B in, 0B out
         IO: 5.9M read, 0B written
   Mem peak: 58.8M
        CPU: 581ms

Aug 25 19:41:49 proart-p16 systemd[1]: Started OpenStack Glance API Daemon.
Aug 25 19:41:50 proart-p16 69v7i82yws91gpdpmba6p0f8a4503dyv-exec.sh[2505783]: ERROR: Failed to find some config files: /etc/glance/glance-api-paste.ini
Aug 25 19:41:50 proart-p16 systemd[1]: glance-api.service: Main process exited, code=exited, status=99/n/a
Aug 25 19:41:50 proart-p16 systemd[1]: glance-api.service: Failed with result 'exit-code'.
Aug 25 19:41:50 proart-p16 systemd[1]: glance-api.service: Consumed 581ms CPU time, 58.8M memory peak, 5.9M read from disk.

× glance.service - OpenStack Glance setup
     Loaded: loaded (/etc/systemd/system/glance.service; enabled; preset: ignored)
     Active: failed (Result: exit-code) since Mon 2025-08-25 19:41:49 CEST; 1s ago
 Invocation: 6414520fecae477f861b9b89615e0b37
    Process: 2505754 ExecStart=/nix/store/zps4l82ch6fb16jdaqp8vjm06scpklgb-glance.sh (code=exited, status=1/FAILURE)
   Main PID: 2505754 (code=exited, status=1/FAILURE)
         IP: 720B in, 1.1K out
         IO: 4K read, 0B written
   Mem peak: 54.5M
        CPU: 434ms

Aug 25 19:41:49 proart-p16 systemd[1]: Starting OpenStack Glance setup...
Aug 25 19:41:49 proart-p16 zps4l82ch6fb16jdaqp8vjm06scpklgb-glance.sh[2505754]: + openstack user create --domain default --password glance glance
Aug 25 19:41:49 proart-p16 zps4l82ch6fb16jdaqp8vjm06scpklgb-glance.sh[2505757]: Failed to discover available identity versions when contacting http://controller:5000/v3. Attempting to parse version from URL.
Aug 25 19:41:49 proart-p16 zps4l82ch6fb16jdaqp8vjm06scpklgb-glance.sh[2505757]: Internal Server Error (HTTP 500)
Aug 25 19:41:49 proart-p16 systemd[1]: glance.service: Main process exited, code=exited, status=1/FAILURE
Aug 25 19:41:49 proart-p16 systemd[1]: glance.service: Failed with result 'exit-code'.
Aug 25 19:41:49 proart-p16 systemd[1]: Failed to start OpenStack Glance setup.
Aug 25 19:41:49 proart-p16 systemd[1]: glance.service: Consumed 434ms CPU time, 54.5M memory peak, 4K read from disk, 720B incoming IP traffic, 1.1K outgoing IP traffic.

× keystone-all.service - OpenStack Keystone Daemon
     Loaded: loaded (/etc/systemd/system/keystone-all.service; enabled; preset: ignored)
     Active: failed (Result: exit-code) since Mon 2025-08-25 19:41:49 CEST; 1s ago
 Invocation: 5bcd191c42834014a0d2c62109df04bd
    Process: 2505584 ExecStartPre=/nix/store/ywlk00615x0578vy15nmnjij4n4s7vif-unit-script-keystone-all-pre-start/bin/keystone-all-pre-start (code=exited, status=0/SUCCESS)
    Process: 2505703 ExecStart=/nix/store/s0px540xhxywxbhlbpybfqw2jzqf01h1-keystone-all.sh (code=exited, status=1/FAILURE)
   Main PID: 2505703 (code=exited, status=1/FAILURE)
         IP: 40.4K in, 52.3K out
         IO: 117.2M read, 0B written
   Mem peak: 180M
        CPU: 5.431s

Aug 25 19:41:42 proart-p16 systemd[1]: Starting OpenStack Keystone Daemon...
Aug 25 19:41:46 proart-p16 s0px540xhxywxbhlbpybfqw2jzqf01h1-keystone-all.sh[2505703]: + keystone-manage --config-file /nix/store/0wlxm7z0266939gwa73z9w987rhcia50-keystone.conf bootstrap --bootstrap-password admin --bootstrap-region-id RegionOne
Aug 25 19:41:48 proart-p16 s0px540xhxywxbhlbpybfqw2jzqf01h1-keystone-all.sh[2505703]: + openstack project create --domain default --description 'Service Project' service
Aug 25 19:41:49 proart-p16 s0px540xhxywxbhlbpybfqw2jzqf01h1-keystone-all.sh[2505749]: Failed to discover available identity versions when contacting http://controller:5000/v3. Attempting to parse version from URL.
Aug 25 19:41:49 proart-p16 s0px540xhxywxbhlbpybfqw2jzqf01h1-keystone-all.sh[2505749]: Internal Server Error (HTTP 500)
Aug 25 19:41:49 proart-p16 systemd[1]: keystone-all.service: Main process exited, code=exited, status=1/FAILURE
Aug 25 19:41:49 proart-p16 systemd[1]: keystone-all.service: Failed with result 'exit-code'.
Aug 25 19:41:49 proart-p16 systemd[1]: Failed to start OpenStack Keystone Daemon.
Aug 25 19:41:49 proart-p16 systemd[1]: keystone-all.service: Consumed 5.431s CPU time, 180M memory peak, 117.2M read from disk, 40.4K incoming IP traffic, 52.3K outgoing IP traffic.

× placement.service - OpenStack Placement setup
     Loaded: loaded (/etc/systemd/system/placement.service; enabled; preset: ignored)
     Active: failed (Result: exit-code) since Mon 2025-08-25 19:41:50 CEST; 1s ago
 Invocation: a5d4e58f2e9e4417955afdd6a5fca6ba
    Process: 2505781 ExecStart=/nix/store/kwx2c3vbash22mpk7mxq30kfj0fbp1br-placement.sh (code=exited, status=1/FAILURE)
   Main PID: 2505781 (code=exited, status=1/FAILURE)
         IP: 720B in, 1.1K out
         IO: 4K read, 0B written
   Mem peak: 54.9M
        CPU: 483ms

Aug 25 19:41:49 proart-p16 systemd[1]: Starting OpenStack Placement setup...
Aug 25 19:41:49 proart-p16 kwx2c3vbash22mpk7mxq30kfj0fbp1br-placement.sh[2505781]: + openstack user create --domain default --password placement placement
Aug 25 19:41:50 proart-p16 kwx2c3vbash22mpk7mxq30kfj0fbp1br-placement.sh[2505784]: Failed to discover available identity versions when contacting http://controller:5000/v3. Attempting to parse version from URL.
Aug 25 19:41:50 proart-p16 kwx2c3vbash22mpk7mxq30kfj0fbp1br-placement.sh[2505784]: Internal Server Error (HTTP 500)
Aug 25 19:41:50 proart-p16 systemd[1]: placement.service: Main process exited, code=exited, status=1/FAILURE
Aug 25 19:41:50 proart-p16 systemd[1]: placement.service: Failed with result 'exit-code'.
Aug 25 19:41:50 proart-p16 systemd[1]: Failed to start OpenStack Placement setup.
Aug 25 19:41:50 proart-p16 systemd[1]: placement.service: Consumed 483ms CPU time, 54.9M memory peak, 4K read from disk, 720B incoming IP traffic, 1.1K outgoing IP traffic.
```

At least the errors have changed! domain resolution is working, but whatever’s listening on port `5000` is crashing when accessing `/v3`. Let’s find out what service is on this port. Searching `5000` in the `openstack-nix` repo:

https://github.com/cobaltcore-dev/openstack-nix/blob/e97581816584486555d2647ae2b2aae6b1751197/modules/controller/keystone.nix#L108-L146

There’s an `nginx` reverse proxy running at that port, forwarding to `5001`. I’m no Python expert, but it looks like a `uWSGI` service is listening on `5001`, serving the `keystone` app. Let’s check the uwsgi logs (probably a systemd service):

```sh
journalctl -u uwsgi -o cat -n 50
```

```log
Traceback (most recent call last):
  File "/nix/store/f7cxwxjmwmnyj6lrr3mblddx7spagjzx-python3.12-keystone-26.0.0/bin/.keystone-wsgi-public-wrapped", line 7, in <module>
    from keystone.server.wsgi import initialize_public_application
  File "/nix/store/f7cxwxjmwmnyj6lrr3mblddx7spagjzx-python3.12-keystone-26.0.0/lib/python3.12/site-packages/keystone/server/__init__.py", line 16, in <module>
    from keystone.common import sql
  File "/nix/store/f7cxwxjmwmnyj6lrr3mblddx7spagjzx-python3.12-keystone-26.0.0/lib/python3.12/site-packages/keystone/common/sql/__init__.py", line 15, in <module>
    from keystone.common.sql.core import *  # noqa
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/nix/store/f7cxwxjmwmnyj6lrr3mblddx7spagjzx-python3.12-keystone-26.0.0/lib/python3.12/site-packages/keystone/common/sql/core.py", line 32, in <module>
    from osprofiler import opts as profiler
  File "/nix/store/gqblbc3l1hc4npzqj1z4mndjfz009088-python3.12-osprofiler-4.2.0/lib/python3.12/site-packages/osprofiler/opts.py", line 18, in <module>
    from osprofiler import web
  File "/nix/store/gqblbc3l1hc4npzqj1z4mndjfz009088-python3.12-osprofiler-4.2.0/lib/python3.12/site-packages/osprofiler/web.py", line 16, in <module>
    import webob.dec
  File "/nix/store/fl4s5i7jg8n5nvr29amwn38yhlq19bkq-python3.12-webob-1.8.8/lib/python3.12/site-packages/webob/__init__.py", line 1, in <module>
    from webob.datetime_utils import (  # noqa: F401
    ...<13 lines>...
    )
  File "/nix/store/fl4s5i7jg8n5nvr29amwn38yhlq19bkq-python3.12-webob-1.8.8/lib/python3.12/site-packages/webob/datetime_utils.py", line 18, in <module>
    from webob.compat import (
    ...<4 lines>...
        )
  File "/nix/store/fl4s5i7jg8n5nvr29amwn38yhlq19bkq-python3.12-webob-1.8.8/lib/python3.12/site-packages/webob/compat.py", line 5, in <module>
    from cgi import parse_header
ModuleNotFoundError: No module named 'cgi'
unable to load app 0 (mountpoint='') (callable not found or import error)
*** no app loaded. going in full dynamic mode ***
```

I think we’ve found the culprit: the `cgi` module isn’t installed. So, the requirements weren’t set up correctly in the `keystone` package. Let’s check the `keystone` package.

https://github.com/cobaltcore-dev/openstack-nix/blob/e97581816584486555d2647ae2b2aae6b1751197/packages/keystone.nix

Not much help to me, I’m not a Python expert. It’s getting a bit frustrating, so let’s try some quick fixes. Maybe it’s fixed in a newer version after `26.0.0`? Checking [`keystone` on PyPI](https://pypi.org/project/keystone/), there’s a `27.0.0` available. Let’s try that! Nix is great for this kind of thing.

Man, I’m really starting to _dislike_ this `openstack-nix` flake. Normally, Nix modules running a service have a `service.<thing>.package` option so you can swap out the package, but not here. The `keystone` package is passed as a function argument and used as-is. So, we’ll have to get clever. Luckily, it’s only used once, in `services.uwsgi.instance.vassals.keystone.wsgi-file`, so we can overwrite it with some Nix magic 🪄.

```nix title="hosts/proart-p16/default.nix"
{
  pkgs,
  config,
  lib,
  inputs,
  ...
}:
{
  # ...
  services.uwsgi.instance.vassals.keystone.wsgi-file =
    let
      keystone = inputs.openstack-nix.packages.x86_64-linux.keystone.overrideAttrs rec {
          pname = "keystone";
          version = "27.0.0";
          src = pkgs.fetchPypi {
            inherit pname version;
            sha256 = "sha256-j5Ri/L6Y49UeN/fhgjGD/ESwTDxAhYkjQpVyUckGQlI=";
          };
        };
    in
    lib.mkForce "${keystone}/bin/.keystone-wsgi-public-wrapped";
  # ...
}
```

> [!tip]+
> I cheated a little here, the `sha256` is the hash of the source files, and I got it by rebuilding without the hash and copying the expected hash from the error messages. If there’s a better way, I haven’t found it yet. 😅

```log
error: builder for '/nix/store/x74jyhk48d67a037r5i11h9vl8ksa3cs-python3.12-keystone-26.0.0.drv' failed with exit code 1;
       last 25 log lines:
       > adding 'keystone/trust/schema.py'
       > adding 'keystone/trust/backends/__init__.py'
       > adding 'keystone/trust/backends/base.py'
       > adding 'keystone/trust/backends/sql.py'
       > adding 'keystone/wsgi/__init__.py'
       > adding 'keystone/wsgi/api.py'
       > adding 'keystone-27.0.0.data/data/etc/keystone/sso_callback_template.html'
       > adding 'keystone-27.0.0.data/scripts/keystone-wsgi-admin'
       > adding 'keystone-27.0.0.data/scripts/keystone-wsgi-public'
       > adding 'keystone-27.0.0.dist-info/AUTHORS'
       > adding 'keystone-27.0.0.dist-info/LICENSE'
       > adding 'keystone-27.0.0.dist-info/METADATA'
       > adding 'keystone-27.0.0.dist-info/WHEEL'
       > adding 'keystone-27.0.0.dist-info/entry_points.txt'
       > adding 'keystone-27.0.0.dist-info/pbr.json'
       > adding 'keystone-27.0.0.dist-info/top_level.txt'
       > adding 'keystone-27.0.0.dist-info/RECORD'
       > removing build/bdist.linux-x86_64/wheel
       > Successfully built keystone-27.0.0-py3-none-any.whl
       > Finished creating a wheel...
       > Finished executing pypaBuildPhase
       > Running phase: pythonRuntimeDepsCheckHook
       > Executing pythonRuntimeDepsCheck
       > Checking runtime dependencies for keystone-27.0.0-py3-none-any.whl
       >   - oslo-policy>=4.5.0 not satisfied by version 4.4.0
       For full logs, run:
         nix log /nix/store/x74jyhk48d67a037r5i11h9vl8ksa3cs-python3.12-keystone-26.0.0.drv
error: 1 dependencies of derivation '/nix/store/a096z54n9fn0pqnac5671ny1x0q22hw5-keystone.json.drv' failed to build
error: 1 dependencies of derivation '/nix/store/65d3z3bdrzbldx8nl7kqpghmlbvxcclc-vassals.drv' failed to build
error: 1 dependencies of derivation '/nix/store/g0z8ghxz37qjwk4s5cjv38lj1n9495mp-server.json.drv' failed to build
error: 1 dependencies of derivation '/nix/store/jf5040596y0q089wn5xs7dq8j0x35pza-unit-uwsgi.service.drv' failed to build
error: 1 dependencies of derivation '/nix/store/8y6z5b3rfz6vw6sbigp5igbk726z0vdd-system-units.drv' failed to build
error: 1 dependencies of derivation '/nix/store/drfhfwyx8h51ag9fsfqvb6vgj5p294rp-etc.drv' failed to build
error: 1 dependencies of derivation '/nix/store/vsdggnx4bxpc18mxcpx8g1cmkrags69r-nixos-system-proart-p16-25.11.20250822.f937f8e.drv' failed to build
```

Now `oslo-policy` is outdated, so let’s update it:

```nix title="hosts/proart-p16/default.nix"
{
  pkgs,
  config,
  lib,
  inputs,
  ...
}:
{
  # ...
  services.uwsgi.instance.vassals.keystone.wsgi-file =
    let
      oslo-policy = inputs.openstack-nix.packages.x86_64-linux.oslo-policy.overrideAttrs rec {
        pname = "oslo.policy";
        version = "4.5.0";
        src = pkgs.fetchPypi {
          inherit pname version;
          sha256 = "sha256-Wt5pgfGKM8ORJ0ovAA4ct7ESKHs9hKeUEl9GUGIzPGo=";
        };
      };

      keystone =
        (inputs.openstack-nix.packages.x86_64-linux.keystone.overrideAttrs rec {
          pname = "keystone";
          version = "27.0.0";
          nativeBuildInpus = [ pkgs.python3Packages.legacy-cgi ];
          src = pkgs.fetchPypi {
            inherit pname version;
            sha256 = "sha256-j5Ri/L6Y49UeN/fhgjGD/ESwTDxAhYkjQpVyUckGQlI=";
          };
        }).override
          {
            inherit oslo-policy;
          };
    in
    lib.mkForce "${keystone}/bin/.keystone-wsgi-public-wrapped";
  # ...
}
```

And...

```log
error: builder for '/nix/store/zl6839pkhc8k57yqviyi9yk9hn5j6f4x-python3.12-keystone-26.0.0.drv' failed with exit code 1;
       last 25 log lines:
       >  - Worker 13 (238 tests) => 0:00:55.760241
       >  - Worker 14 (238 tests) => 0:01:00.054763
       >  - Worker 15 (238 tests) => 0:01:06.014923
       >  - Worker 16 (238 tests) => 0:00:53.842821
       >  - Worker 17 (238 tests) => 0:01:01.785706
       >  - Worker 18 (238 tests) => 0:01:05.799870
       >  - Worker 19 (238 tests) => 0:01:04.436807
       >  - Worker 20 (238 tests) => 0:01:09.806450
       >  - Worker 21 (237 tests) => 0:01:05.198050
       >  - Worker 22 (237 tests) => 0:01:08.037577
       >  - Worker 23 (237 tests) => 0:01:10.556906
       > installCheckPhase completed in 1 minutes 23 seconds
       > Running phase: pythonCatchConflictsPhase
       > Found duplicated packages in closure for dependency 'oslo.policy':
       >   oslo.policy 4.5.0 (/nix/store/135sm1zlfaghik96is1g3y2cjd1szclc-python3.12-oslo.policy-4.4.0)
       >     dependency chain:
       >       this derivation: /nix/store/lnnr9583lvjs2fhrvh8ylp2a6cch3jmj-python3.12-keystone-26.0.0
       >       ...depending on: /nix/store/135sm1zlfaghik96is1g3y2cjd1szclc-python3.12-oslo.policy-4.4.0
       >   oslo.policy 4.4.0 (/nix/store/j3gi455sg9pzgghkrv2s3m7cgp38jw3b-python3.12-oslo.policy-4.4.0)
       >     dependency chain:
       >       this derivation: /nix/store/lnnr9583lvjs2fhrvh8ylp2a6cch3jmj-python3.12-keystone-26.0.0
       >       ...depending on: /nix/store/nhc146598rmh72x88smd447vvlxym9vp-python3.12-oslo.upgradecheck-2.4.0
       >       ...depending on: /nix/store/j3gi455sg9pzgghkrv2s3m7cgp38jw3b-python3.12-oslo.policy-4.4.0
       >
       > Package duplicates found in closure, see above. Usually this happens if two packages depend on different version of the same dependency.
       For full logs, run:
         nix log /nix/store/zl6839pkhc8k57yqviyi9yk9hn5j6f4x-python3.12-keystone-26.0.0.drv
error: 1 dependencies of derivation '/nix/store/041a6bjx5k13rmzm03rbvwa4xrampyby-keystone.json.drv' failed to build
error: 1 dependencies of derivation '/nix/store/4jj87cy7s28kp1yigdx3xli8icyf47j3-vassals.drv' failed to build
error: 1 dependencies of derivation '/nix/store/hslgj2srr10jxp1mlz91z80rvx4yj580-server.json.drv' failed to build
error: 1 dependencies of derivation '/nix/store/6kbhnchmddqjaqzfk22r6pn3l84hy0y4-unit-uwsgi.service.drv' failed to build
error: 1 dependencies of derivation '/nix/store/gvby1475qkcp8c12cndwpbasjs1ddsx1-system-units.drv' failed to build
error: 1 dependencies of derivation '/nix/store/6djr5d75w9jqj3w8nbsb2p6zvgpp864c-etc.drv' failed to build
error: 1 dependencies of derivation '/nix/store/6y1sabzccx6dzhif5likfqdkfvspwjm1-nixos-system-proart-p16-25.11.20250822.f937f8e.drv' failed to build
```

...now there are two different versions of `oslo.policy` 😞. Let’s update `oslo.upgradecheck` to use the new one:

```nix title="hosts/proart-p16/default.nix"
{
  pkgs,
  config,
  lib,
  inputs,
  ...
}:
{
  # ...
  services.uwsgi.instance.vassals.keystone.wsgi-file =
    let
      oslo-policy = inputs.openstack-nix.packages.x86_64-linux.oslo-policy.overrideAttrs rec {
        pname = "oslo.policy";
        version = "4.5.0";
        src = pkgs.fetchPypi {
          inherit pname version;
          sha256 = "sha256-Wt5pgfGKM8ORJ0ovAA4ct7ESKHs9hKeUEl9GUGIzPGo=";
        };
      };

      oslo-upgradecheck = inputs.openstack-nix.packages.x86_64-linux.oslo-upgradecheck.override {
        inherit oslo-policy;
      };

      keystone =
        (inputs.openstack-nix.packages.x86_64-linux.keystone.overrideAttrs rec {
          pname = "keystone";
          version = "27.0.0";
          nativeBuildInpus = [ pkgs.python3Packages.legacy-cgi ];
          src = pkgs.fetchPypi {
            inherit pname version;
            sha256 = "sha256-j5Ri/L6Y49UeN/fhgjGD/ESwTDxAhYkjQpVyUckGQlI=";
          };
        }).override
          {
            inherit oslo-policy oslo-upgradecheck;
          };
    in
    lib.mkForce "${keystone}/bin/.keystone-wsgi-public-wrapped";
  # ...
}
```

Ok now the rebuild worked but lets check the service logs again:

```sh
journalctl -u uwsgi -o cat -n 50
```

```log
Traceback (most recent call last):
  File "/nix/store/f7cxwxjmwmnyj6lrr3mblddx7spagjzx-python3.12-keystone-26.0.0/bin/.keystone-wsgi-public-wrapped", line 7, in <module>
    from keystone.server.wsgi import initialize_public_application
  File "/nix/store/f7cxwxjmwmnyj6lrr3mblddx7spagjzx-python3.12-keystone-26.0.0/lib/python3.12/site-packages/keystone/server/__init__.py", line 16, in <module>
    from keystone.common import sql
  File "/nix/store/f7cxwxjmwmnyj6lrr3mblddx7spagjzx-python3.12-keystone-26.0.0/lib/python3.12/site-packages/keystone/common/sql/__init__.py", line 15, in <module>
    from keystone.common.sql.core import *  # noqa
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/nix/store/f7cxwxjmwmnyj6lrr3mblddx7spagjzx-python3.12-keystone-26.0.0/lib/python3.12/site-packages/keystone/common/sql/core.py", line 32, in <module>
    from osprofiler import opts as profiler
  File "/nix/store/gqblbc3l1hc4npzqj1z4mndjfz009088-python3.12-osprofiler-4.2.0/lib/python3.12/site-packages/osprofiler/opts.py", line 18, in <module>
    from osprofiler import web
  File "/nix/store/gqblbc3l1hc4npzqj1z4mndjfz009088-python3.12-osprofiler-4.2.0/lib/python3.12/site-packages/osprofiler/web.py", line 16, in <module>
    import webob.dec
  File "/nix/store/fl4s5i7jg8n5nvr29amwn38yhlq19bkq-python3.12-webob-1.8.8/lib/python3.12/site-packages/webob/__init__.py", line 1, in <module>
    from webob.datetime_utils import (  # noqa: F401
    ...<13 lines>...
    )
  File "/nix/store/fl4s5i7jg8n5nvr29amwn38yhlq19bkq-python3.12-webob-1.8.8/lib/python3.12/site-packages/webob/datetime_utils.py", line 18, in <module>
    from webob.compat import (
    ...<4 lines>...
        )
  File "/nix/store/fl4s5i7jg8n5nvr29amwn38yhlq19bkq-python3.12-webob-1.8.8/lib/python3.12/site-packages/webob/compat.py", line 5, in <module>
    from cgi import parse_header
ModuleNotFoundError: No module named 'cgi'
unable to load app 0 (mountpoint='') (callable not found or import error)
*** no app loaded. going in full dynamic mode ***
```

Updating the source didn't help. So after wasting all that time we are back to square one. Maybe we are missing some python packages in this modules?

uWSGI is running as a systemd service, so let’s try adding the `PYTHONPATH` environment variable to point to the `cgi` library we’re missing. `PYTHONPATH` works like `PATH`, but for additional Python packages. After searching for the `webob` and missing `cgi` issue, I found [this discussion](https://github.com/Pylons/webob/issues/437#issuecomment-1902058169): `cgi` was removed in Python 3.13, but the logs have Python 3.12 splattered everywhere, so the package should still be present. Maybe we’re using `python3.12` libraries, but the runtime interpreter is `python3.13`? Before digging deeper into the version mismatch, I’ll try using the `legacy-cgi` package and see if that resolves the issue.

```nix title="hosts/proart-p16/default.nix"
{
  pkgs,
  config,
  lib,
  inputs,
  ...
}:
{
  # ...
  systemd.services.uwsgi.environment = {
    PYTHONPATH = "${pkgs.python312Packages.legacy-cgi}/lib/python3.12/site-packages:${pkgs.python312Packages.greenlet}/lib/python3.12/site-packages";
  };
  # ...
}
```

Maybe this is my lucky break? Nope 🥲! `journalctl` immediately shows a new error in another package:

```log
Traceback (most recent call last):
  File "/nix/store/i6gx8i6nq7lyzdwzy9sn2ijm476ib666-python3.12-keystone-26.0.0/bin/.keystone-wsgi-public-wrapped", line 7, in <module>
    from keystone.server.wsgi import initialize_public_application
  File "/nix/store/i6gx8i6nq7lyzdwzy9sn2ijm476ib666-python3.12-keystone-26.0.0/lib/python3.12/site-packages/keystone/server/__init__.py", line 16, in <module>
    from keystone.common import sql
  File "/nix/store/i6gx8i6nq7lyzdwzy9sn2ijm476ib666-python3.12-keystone-26.0.0/lib/python3.12/site-packages/keystone/common/sql/__init__.py", line 15, in <module>
    from keystone.common.sql.core import *  # noqa
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/nix/store/i6gx8i6nq7lyzdwzy9sn2ijm476ib666-python3.12-keystone-26.0.0/lib/python3.12/site-packages/keystone/common/sql/core.py", line 39, in <module>
    from keystone.common import driver_hints
  File "/nix/store/i6gx8i6nq7lyzdwzy9sn2ijm476ib666-python3.12-keystone-26.0.0/lib/python3.12/site-packages/keystone/common/driver_hints.py", line 18, in <module>
    from keystone import exception
  File "/nix/store/i6gx8i6nq7lyzdwzy9sn2ijm476ib666-python3.12-keystone-26.0.0/lib/python3.12/site-packages/keystone/exception.py", line 21, in <module>
    import keystone.conf
  File "/nix/store/i6gx8i6nq7lyzdwzy9sn2ijm476ib666-python3.12-keystone-26.0.0/lib/python3.12/site-packages/keystone/conf/__init__.py", line 18, in <module>
    import oslo_messaging
  File "/nix/store/vn7zgnknygqwap0j16hk2w7zs6ca8fqp-python3.12-oslo.messaging-15.0.0/lib/python3.12/site-packages/oslo_messaging/__init__.py", line 16, in <module>
    from .notify import *
  File "/nix/store/vn7zgnknygqwap0j16hk2w7zs6ca8fqp-python3.12-oslo.messaging-15.0.0/lib/python3.12/site-packages/oslo_messaging/notify/__init__.py", line 27, in <module>
    from .listener import *
  File "/nix/store/vn7zgnknygqwap0j16hk2w7zs6ca8fqp-python3.12-oslo.messaging-15.0.0/lib/python3.12/site-packages/oslo_messaging/notify/listener.py", line 140, in <module>
    from oslo_messaging import server as msg_server
  File "/nix/store/vn7zgnknygqwap0j16hk2w7zs6ca8fqp-python3.12-oslo.messaging-15.0.0/lib/python3.12/site-packages/oslo_messaging/server.py", line 27, in <module>
    from oslo_service import service
  File "/nix/store/smyln1yqb7cj03hwb5bxw365wkj6abk8-python3.12-oslo.service-3.6.0/lib/python3.12/site-packages/oslo_service/service.py", line 35, in <module>
    import eventlet
  File "/nix/store/x2s6fmrinxz0k36r6ljnjrd8i3qasm3z-python3.12-eventlet-0.37.0/lib/python3.12/site-packages/eventlet/__init__.py", line 6, in <module>
    from eventlet import convenience
  File "/nix/store/x2s6fmrinxz0k36r6ljnjrd8i3qasm3z-python3.12-eventlet-0.37.0/lib/python3.12/site-packages/eventlet/convenience.py", line 4, in <module>
    from eventlet import greenpool
  File "/nix/store/x2s6fmrinxz0k36r6ljnjrd8i3qasm3z-python3.12-eventlet-0.37.0/lib/python3.12/site-packages/eventlet/greenpool.py", line 4, in <module>
    from eventlet import queue
  File "/nix/store/x2s6fmrinxz0k36r6ljnjrd8i3qasm3z-python3.12-eventlet-0.37.0/lib/python3.12/site-packages/eventlet/queue.py", line 48, in <module>
    from eventlet.event import Event
  File "/nix/store/x2s6fmrinxz0k36r6ljnjrd8i3qasm3z-python3.12-eventlet-0.37.0/lib/python3.12/site-packages/eventlet/event.py", line 1, in <module>
    from eventlet import hubs
  File "/nix/store/x2s6fmrinxz0k36r6ljnjrd8i3qasm3z-python3.12-eventlet-0.37.0/lib/python3.12/site-packages/eventlet/hubs/__init__.py", line 7, in <module>
    from eventlet.support import greenlets as greenlet
  File "/nix/store/x2s6fmrinxz0k36r6ljnjrd8i3qasm3z-python3.12-eventlet-0.37.0/lib/python3.12/site-packages/eventlet/support/__init__.py", line 6, in <module>
    from eventlet.support import greenlets
  File "/nix/store/x2s6fmrinxz0k36r6ljnjrd8i3qasm3z-python3.12-eventlet-0.37.0/lib/python3.12/site-packages/eventlet/support/greenlets.py", line 1, in <module>
    import greenlet
  File "/nix/store/g7k7c5qzsh9agyg4vv664lsb49k8p8fn-python3.12-greenlet-3.2.2/lib/python3.12/site-packages/greenlet/__init__.py", line 29, in <module>
    from ._greenlet import _C_API # pylint:disable=no-name-in-module
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
ModuleNotFoundError: No module named 'greenlet._greenlet'
unable to load app 0 (mountpoint='') (callable not found or import error)
*** no app loaded. going in full dynamic mode ***
```

Don’t think this is the path I want to take. Let’s figure out why `cgi` isn’t available when it should be in `python3.12`. Only one thing comes to mind: The service isn’t running directly, but through `uwsgi`, maybe it’s using a different Python version? Let’s investigate. We know `uwsgi` is running as a systemd unit, so the unit file is a good place to start:

```sh
 systemctl cat uwsgi
```

```systemd
# /etc/systemd/system/uwsgi.service
[Unit]

[Service]
Environment="LOCALE_ARCHIVE=/nix/store/cccr51zb8xj8n5ayb5x6s1cqs3y7r3g7-glibc-locales-2.40-66/lib/locale/locale-archive"
Environment="PATH=/nix/store/ih779chzzag1nm91fgnrndml4mghm3la-coreutils-9.7/bin:/nix/store/3xi6s71d3znq0ivl2r7ypg5rsz71j16h-findutils-4.10.0/bin:/nix/store/k7gv42hpqwh6ghiyl4ava9p5r249x6vn-gnugrep-3.12/bin:/nix/store/ql68miwsgz9094bi3qa7nk17bfwf6h6a-gnused-4.9/bin:/nix/store/wq3ivni0plh7g8xl3my8qr9llh4dy7q4-systemd-257.6/bin:/nix/store/ih779chzzag1nm91fgnrndml4mghm3la-coreutils-9.7/sbin:/nix/store/3xi6s71d3znq0ivl2r7ypg5rsz71j16h-findutils-4.10.0/sbin:/nix/store/k7gv42hpqwh6ghiyl4ava9p5r249x6vn-gnugrep-3.12/sbin:/nix/store/ql68miwsgz9094bi3qa7nk17bfwf6h6a-gnused-4.9/sbin:/nix/store/wq3ivni0plh7g8xl3my8qr9llh4dy7q4-systemd-257.6/sbin"
Environment="TZDIR=/nix/store/ms10flhvgnd1nsfwnb355aw414ijlq8j-tzdata-2025b/share/zoneinfo"
AmbientCapabilities=CAP_SETGID
AmbientCapabilities=CAP_SETUID
AmbientCapabilities=CAP_SETGID
AmbientCapabilities=CAP_SETUID
AmbientCapabilities=CAP_SETUID
AmbientCapabilities=CAP_SETGID
AmbientCapabilities=CAP_SYS_CHROOT
AmbientCapabilities=CAP_SETPCAP
AmbientCapabilities=CAP_CHOWN
CapabilityBoundingSet=CAP_SETGID
CapabilityBoundingSet=CAP_SETUID
CapabilityBoundingSet=CAP_SETGID
CapabilityBoundingSet=CAP_SETUID
CapabilityBoundingSet=CAP_SETUID
CapabilityBoundingSet=CAP_SETGID
CapabilityBoundingSet=CAP_SYS_CHROOT
CapabilityBoundingSet=CAP_SETPCAP
CapabilityBoundingSet=CAP_CHOWN
ExecReload=/nix/store/ih779chzzag1nm91fgnrndml4mghm3la-coreutils-9.7/bin/kill -HUP $MAINPID
ExecStart=/nix/store/ha39hbq2hk4gs4sw83dg9np4jc98h8c3-uwsgi-2.0.30/bin/uwsgi --json /nix/store/a0anawziwh739d4cmdlyd2lsxi7j9kxv-server.json/server.json
ExecStop=/nix/store/ih779chzzag1nm91fgnrndml4mghm3la-coreutils-9.7/bin/kill -INT $MAINPID
Group=nginx
KillSignal=SIGQUIT
NotifyAccess=main
RuntimeDirectory=uwsgi
Type=notify
User=nginx

[Install]
WantedBy=multi-user.target
```

Not a lot of Python stuff in there, but there’s a JSON config file (probably). Let’s check it out:

```sh
cat /nix/store/a0anawziwh739d4cmdlyd2lsxi7j9kxv-server.json/server.json | jq
```

```json
{
  "uwsgi": {
    "emperor": "/nix/store/fbjlyaldd5cz038h6slfp0lzvsiz87nl-vassals"
  }
}
```

Another redirection:

```sh
ls /nix/store/fbjlyaldd5cz038h6slfp0lzvsiz87nl-vassals
```

```log
glance.json  horizon.json  keystone.json
```

maybe we’re finally there 🤞:

```sh
cat /nix/store/fbjlyaldd5cz038h6slfp0lzvsiz87nl-vassals/keystone.json | jq
```

```json
{
  "uwsgi": {
    "buffer-size": 65535,
    "enable-threads": true,
    "env": ["PATH=/nix/store/10vi8lr6093b5n80mmv9mwdrmd38sv9p-python3-3.13.5-env/bin"],
    "http11-socket": "127.0.0.1:5001",
    "immediate-gid": "keystone",
    "immediate-uid": "keystone",
    "lazy-apps": true,
    "master": true,
    "plugins": ["python3"],
    "processes": 4,
    "pyhome": "/nix/store/10vi8lr6093b5n80mmv9mwdrmd38sv9p-python3-3.13.5-env",
    "threads": 4,
    "thunder-lock": true,
    "wsgi-file": "/nix/store/i6gx8i6nq7lyzdwzy9sn2ijm476ib666-python3.12-keystone-26.0.0/bin/.keystone-wsgi-public-wrapped"
  }
}
```

Looks like my hunch was correct! `wsgi-file` points to something built with `python3.12`, but `PATH` and `pyhome` are both created using `python3.13`!

Let’s see if we can change the Python version `uwsgi` uses. There’s no [config option](https://search.nixos.org/options?channel=unstable&query=services.uwsgi) for that, so we’ll need to check the source to see how it uses Python (just click any option and then the "Declared in" link).

Looks like there is a `package` configuration where it is picking the `python3` package, I wonder why it is not visible on the website:
https://github.com/NixOS/nixpkgs/blob/3b9f00d7a7bf68acd4c4abb9d43695afb04e03a5/nixos/modules/services/web-servers/uwsgi.nix#L37

Aha, the option is marked as `internal`:
https://github.com/NixOS/nixpkgs/blob/3b9f00d7a7bf68acd4c4abb9d43695afb04e03a5/nixos/modules/services/web-servers/uwsgi.nix#L102-L105

But we can still use it (I think), it just hides it from the docs. And it’s configured to use the `uwsgi` package:
https://github.com/NixOS/nixpkgs/blob/3b9f00d7a7bf68acd4c4abb9d43695afb04e03a5/nixos/modules/services/web-servers/uwsgi.nix#L242-L244

The `python3` package is a `passthru` (not going into that now) in the `uwsgi` package. Let’s check that package:
https://github.com/NixOS/nixpkgs/blob/3b9f00d7a7bf68acd4c4abb9d43695afb04e03a5/pkgs/by-name/uw/uwsgi/package.nix#L124-L127

...and there’s the `passthru` we expected, taken directly from the inputs:
https://github.com/NixOS/nixpkgs/blob/3b9f00d7a7bf68acd4c4abb9d43695afb04e03a5/pkgs/by-name/uw/uwsgi/package.nix#L1-L25

Great, that means we can `override` it! Let’s do that, making sure to add the plugins as defined in the `uwsgi` module:

```nix title="hosts/proart-p16/default.nix"
{
  pkgs,
  config,
  lib,
  inputs,
  ...
}:
{
  # ...
  services.uwsgi.package = lib.mkForce (
    pkgs.uwsgi.override {
      plugins = config.services.uwsgi.plugins;
      python3 = pkgs.python312;
    }
  );
  # ...
}
```

Let’s check the logs in `journalctl`:

```log
Traceback (most recent call last):
  File "/var/lib/openstack_dashboard/wsgi.py", line 21, in <module>
    from django.core.wsgi import get_wsgi_application
ModuleNotFoundError: No module named 'django'
unable to load app 0 (mountpoint='') (callable not found or import error)
*** no app loaded. going in full dynamic mode ***
Traceback (most recent call last):
  File "/var/lib/openstack_dashboard/wsgi.py", line 21, in <module>
    from django.core.wsgi import get_wsgi_application
ModuleNotFoundError: No module named 'django'
unable to load app 0 (mountpoint='') (callable not found or import error)
*** no app loaded. going in full dynamic mode ***
Traceback (most recent call last):
  File "/var/lib/openstack_dashboard/wsgi.py", line 21, in <module>
    from django.core.wsgi import get_wsgi_application
ModuleNotFoundError: No module named 'django'
unable to load app 0 (mountpoint='') (callable not found or import error)
*** no app loaded. going in full dynamic mode ***
Tue Aug 26 15:00:26 2025 - [emperor] vassal horizon.json is ready to accept requests
Traceback (most recent call last):
  File "/nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/bin/.glance-wsgi-api-wrapped", line 53, in <module>
    application = init_app()
                  ^^^^^^^^^^
  File "/nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/lib/python3.12/site-packages/glance/common/wsgi_app.py", line 112, in init_app
    CONF([], project='glance', default_config_files=config_files)
  File "/nix/store/a7068w9ihmhrfzixx76bhp8pq4k3bzvy-python3.12-oslo.config-9.7.0/lib/python3.12/site-packages/oslo_config/cfg.py", line 2188, in __call__
    raise ConfigFilesNotFoundError(self._namespace._files_not_found)
oslo_config.cfg.ConfigFilesNotFoundError: Failed to find some config files: /etc/glance/glance-api.conf
unable to load app 0 (mountpoint='') (callable not found or import error)
*** no app loaded. going in full dynamic mode ***
Traceback (most recent call last):
  File "/nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/bin/.glance-wsgi-api-wrapped", line 53, in <module>
    application = init_app()
                  ^^^^^^^^^^
  File "/nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/lib/python3.12/site-packages/glance/common/wsgi_app.py", line 112, in init_app
    CONF([], project='glance', default_config_files=config_files)
  File "/nix/store/a7068w9ihmhrfzixx76bhp8pq4k3bzvy-python3.12-oslo.config-9.7.0/lib/python3.12/site-packages/oslo_config/cfg.py", line 2188, in __call__
    raise ConfigFilesNotFoundError(self._namespace._files_not_found)
oslo_config.cfg.ConfigFilesNotFoundError: Failed to find some config files: /etc/glance/glance-api.conf
unable to load app 0 (mountpoint='') (callable not found or import error)
*** no app loaded. going in full dynamic mode ***
Tue Aug 26 15:00:27 2025 - [emperor] vassal glance.json is ready to accept requests
Traceback (most recent call last):
  File "/nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/bin/.glance-wsgi-api-wrapped", line 53, in <module>
    application = init_app()
                  ^^^^^^^^^^
  File "/nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/lib/python3.12/site-packages/glance/common/wsgi_app.py", line 112, in init_app
    CONF([], project='glance', default_config_files=config_files)
  File "/nix/store/a7068w9ihmhrfzixx76bhp8pq4k3bzvy-python3.12-oslo.config-9.7.0/lib/python3.12/site-packages/oslo_config/cfg.py", line 2188, in __call__
    raise ConfigFilesNotFoundError(self._namespace._files_not_found)
oslo_config.cfg.ConfigFilesNotFoundError: Failed to find some config files: /etc/glance/glance-api.conf
unable to load app 0 (mountpoint='') (callable not found or import error)
*** no app loaded. going in full dynamic mode ***
Traceback (most recent call last):
  File "/nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/bin/.glance-wsgi-api-wrapped", line 53, in <module>
    application = init_app()
                  ^^^^^^^^^^
  File "/nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/lib/python3.12/site-packages/glance/common/wsgi_app.py", line 112, in init_app
    CONF([], project='glance', default_config_files=config_files)
  File "/nix/store/a7068w9ihmhrfzixx76bhp8pq4k3bzvy-python3.12-oslo.config-9.7.0/lib/python3.12/site-packages/oslo_config/cfg.py", line 2188, in __call__
    raise ConfigFilesNotFoundError(self._namespace._files_not_found)
oslo_config.cfg.ConfigFilesNotFoundError: Failed to find some config files: /etc/glance/glance-api.conf
unable to load app 0 (mountpoint='') (callable not found or import error)
*** no app loaded. going in full dynamic mode ***
```

Ugh! those errors are fixed, but now the `django` module and `/etc/glance/glance-api.conf` config file are missing. Both errors are in `horizon` and `glance` services. I know `horizon` is just a dashboard (not essential), but not sure what `glance` does. Let’s see why this is happening, let’s check the directory to see if the file exists:

```sh
ls -l /etc/glance/
```

```log
total 8
lrwxrwxrwx 1 root root  59 May 13 23:45 glance-api.conf -> /nix/store/gvlynr2hv3d3v4wdqca55rzasriv1y45-glance-api.conf
lrwxrwxrwx 1 root root 100 May 13 23:45 glance-api-paste.ini -> /nix/store/4rv3acyka0nfk4fykjsijv1qxzf1xmb3-python3.12-glance-29.0.0/etc/glance/glance-api-paste.ini
lrwxrwxrwx 1 root root  97 May 13 23:45 schema-image.json -> /nix/store/4rv3acyka0nfk4fykjsijv1qxzf1xmb3-python3.12-glance-29.0.0/etc/glance/schema-image.json
```

The files exist as `symlinks` to the `nix-store` (usual for Nix). Let’s check the file content:

```sh
cat /etc/glance/glance-api.conf
```

```log
cat: /etc/glance/glance-api.conf: No such file or directory
```

Huh, the file/link exists but can’t be accessed. It has full public access (777), so it’s not a permission issue, maybe a broken symlink?

```sh
ls -l /nix/store/gvlynr2hv3d3v4wdqca55rzasriv1y45-glance-api.conf
```

```log
ls: cannot access '/nix/store/gvlynr2hv3d3v4wdqca55rzasriv1y45-glance-api.conf': No such file or directory
```

Looks like it, lets check how these files are created in the `openstack-nix` module.

https://github.com/cobaltcore-dev/openstack-nix/blob/e97581816584486555d2647ae2b2aae6b1751197/modules/controller/glance.nix#L76-L115

Looks like they’re using `systemd-tmpfiles` to create those files. I’ve never seen that before in Nix, usually you just refer to the Nix store for config files, or if the path isn’t configurable, an FHS environment is created with all the necessary files. Back to the broken `symlinks`, maybe all those system rebuilds changed the actual file in `nix-store`, but `systemd-tmpfiles` hasn’t recreated them? Let’s manually delete and recreate them:

```sh
sudo rm /etc/glance/* || true && sudo systemd-tmpfiles --create && ls -l /etc/glance
```

```log
total 8
lrwxrwxrwx 1 root root  59 Aug 26 16:06 glance-api.conf -> /nix/store/9fa2531pwr4d4863rkvas9hnyg06927v-glance-api.conf
lrwxrwxrwx 1 root root 100 Aug 26 16:06 glance-api-paste.ini -> /nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/etc/glance/glance-api-paste.ini
lrwxrwxrwx 1 root root  97 Aug 26 16:06 schema-image.json -> /nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/etc/glance/schema-image.json
```

Great! It looks like the links have changed, lets restart the `uwsgi` service:

```sh
sudo systemctl restart uwsgi
```

`journalctl` only shows the `django` errors now! 🎉

```log
Traceback (most recent call last):
  File "/var/lib/openstack_dashboard/wsgi.py", line 21, in <module>
    from django.core.wsgi import get_wsgi_application
ModuleNotFoundError: No module named 'django'
unable to load app 0 (mountpoint='') (callable not found or import error)
*** no app loaded. going in full dynamic mode ***
```

Let’s restart the services that were failing before and check for issues:

```sh
sudo systemctl restart glance-api glance keystone-all placement
```

```sh
sudo systemctl status glance-api glance keystone-all placement -o cat
```

```log
● glance-api.service - OpenStack Glance API Daemon
     Loaded: loaded (/etc/systemd/system/glance-api.service; enabled; preset: ignored)
     Active: active (running) since Tue 2025-08-26 16:11:51 CEST; 19min ago
 Invocation: 43b19ac993f442d099c02e5b05c9118d
   Main PID: 114040 (69v7i82yws91gpd)
         IP: 0B in, 0B out
         IO: 620K read, 4K written
      Tasks: 10 (limit: 37101)
     Memory: 129.3M (peak: 130.9M)
        CPU: 8.608s
     CGroup: /system.slice/glance-api.service
             ├─114040 /nix/store/gkwbw9nzbkbz298njbn3577zmrnglbbi-bash-5.3p0/bin/bash /nix/store/69v7i82yws91gpdpmba6p0f8a4503dyv-exec.sh
             ├─114071 /nix/store/dksjvr69ckglyw1k2ss1qgshhcix73p8-python3-3.12.8/bin/python3.12 /nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/bin/.glance-api-wrapped --config-file=/nix>
             ├─114113 /nix/store/dksjvr69ckglyw1k2ss1qgshhcix73p8-python3-3.12.8/bin/python3.12 /nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/bin/.glance-api-wrapped --config-file=/nix>
             ├─114114 /nix/store/dksjvr69ckglyw1k2ss1qgshhcix73p8-python3-3.12.8/bin/python3.12 /nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/bin/.glance-api-wrapped --config-file=/nix>
             ├─114115 /nix/store/dksjvr69ckglyw1k2ss1qgshhcix73p8-python3-3.12.8/bin/python3.12 /nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/bin/.glance-api-wrapped --config-file=/nix>
             ├─114116 /nix/store/dksjvr69ckglyw1k2ss1qgshhcix73p8-python3-3.12.8/bin/python3.12 /nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/bin/.glance-api-wrapped --config-file=/nix>
             ├─114117 /nix/store/dksjvr69ckglyw1k2ss1qgshhcix73p8-python3-3.12.8/bin/python3.12 /nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/bin/.glance-api-wrapped --config-file=/nix>
             ├─114118 /nix/store/dksjvr69ckglyw1k2ss1qgshhcix73p8-python3-3.12.8/bin/python3.12 /nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/bin/.glance-api-wrapped --config-file=/nix>
             ├─114119 /nix/store/dksjvr69ckglyw1k2ss1qgshhcix73p8-python3-3.12.8/bin/python3.12 /nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/bin/.glance-api-wrapped --config-file=/nix>
             └─114120 /nix/store/dksjvr69ckglyw1k2ss1qgshhcix73p8-python3-3.12.8/bin/python3.12 /nix/store/rjkqc167ak0macd65maadwq96mb4zlyq-python3.12-glance-29.0.0/bin/.glance-api-wrapped --config-file=/nix>

Started OpenStack Glance API Daemon.

× glance.service - OpenStack Glance setup
     Loaded: loaded (/etc/systemd/system/glance.service; enabled; preset: ignored)
     Active: failed (Result: exit-code) since Tue 2025-08-26 16:29:52 CEST; 1min 17s ago
 Invocation: 7369ca5d1ec24d4da8ac2b91f0426ee0
    Process: 150042 ExecStart=/nix/store/zps4l82ch6fb16jdaqp8vjm06scpklgb-glance.sh (code=exited, status=1/FAILURE)
   Main PID: 150042 (code=exited, status=1/FAILURE)
         IP: 1K in, 1.1K out
         IO: 0B read, 0B written
   Mem peak: 55.1M
        CPU: 443ms

Starting OpenStack Glance setup...
+ openstack user create --domain default --password glance glance
Bad Gateway (HTTP 502)
glance.service: Main process exited, code=exited, status=1/FAILURE
glance.service: Failed with result 'exit-code'.
Failed to start OpenStack Glance setup.
glance.service: Consumed 443ms CPU time, 55.1M memory peak, 1K incoming IP traffic, 1.1K outgoing IP traffic.

× keystone-all.service - OpenStack Keystone Daemon
     Loaded: loaded (/etc/systemd/system/keystone-all.service; enabled; preset: ignored)
     Active: failed (Result: exit-code) since Tue 2025-08-26 16:29:57 CEST; 1min 13s ago
 Invocation: 090d58d0b69440c9be82c9389725e455
    Process: 150043 ExecStartPre=/nix/store/ywlk00615x0578vy15nmnjij4n4s7vif-unit-script-keystone-all-pre-start/bin/keystone-all-pre-start (code=exited, status=0/SUCCESS)
    Process: 150169 ExecStart=/nix/store/s0px540xhxywxbhlbpybfqw2jzqf01h1-keystone-all.sh (code=exited, status=1/FAILURE)
   Main PID: 150169 (code=exited, status=1/FAILURE)
         IP: 42.6K in, 35.9K out
         IO: 0B read, 0B written
   Mem peak: 105.9M
        CPU: 5.075s

Starting OpenStack Keystone Daemon...
+ keystone-manage --config-file /nix/store/0wlxm7z0266939gwa73z9w987rhcia50-keystone.conf bootstrap --bootstrap-password admin --bootstrap-region-id RegionOne
+ openstack project create --domain default --description 'Service Project' service
Bad Gateway (HTTP 502)
keystone-all.service: Main process exited, code=exited, status=1/FAILURE
keystone-all.service: Failed with result 'exit-code'.
Failed to start OpenStack Keystone Daemon.
keystone-all.service: Consumed 5.075s CPU time, 105.9M memory peak, 42.6K incoming IP traffic, 35.9K outgoing IP traffic.

× placement.service - OpenStack Placement setup
     Loaded: loaded (/etc/systemd/system/placement.service; enabled; preset: ignored)
     Active: failed (Result: exit-code) since Tue 2025-08-26 16:29:52 CEST; 1min 17s ago
 Invocation: 4573c6b0bbf84bab95a9094ce78b6d03
    Process: 150075 ExecStart=/nix/store/kwx2c3vbash22mpk7mxq30kfj0fbp1br-placement.sh (code=exited, status=1/FAILURE)
   Main PID: 150075 (code=exited, status=1/FAILURE)
         IP: 1K in, 1.1K out
         IO: 0B read, 0B written
   Mem peak: 54.3M
        CPU: 421ms

Starting OpenStack Placement setup...
+ openstack user create --domain default --password placement placement
Bad Gateway (HTTP 502)
placement.service: Main process exited, code=exited, status=1/FAILURE
placement.service: Failed with result 'exit-code'.
Failed to start OpenStack Placement setup.
placement.service: Consumed 421ms CPU time, 54.3M memory peak, 1K incoming IP traffic, 1.1K outgoing IP traffic.
```

`glance-api` is finally up and running, but the other services are still failing. At least requests to `http://controller:5000/v3` are working now, so that's progress! The errors are all server-side (5xx), so let's dig into the `uwsgi` logs again and see what's blowing up this time:

```log
2025-08-26 16:29:56.951 109496 ERROR keystone   File "/nix/store/1r0494dxv9ga3gvsn270zdznhys3j4pb-python3.12-sqlalchemy-2.0.34/lib/python3.12/site-packages/sqlalchemy/engine/default.py", line 621, in connect
2025-08-26 16:29:56.951 109496 ERROR keystone     return self.loaded_dbapi.connect(*cargs, **cparams)
2025-08-26 16:29:56.951 109496 ERROR keystone            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
2025-08-26 16:29:56.951 109496 ERROR keystone oslo_db.exception.DBNonExistentDatabase: (sqlite3.OperationalError) unable to open database file
2025-08-26 16:29:56.951 109496 ERROR keystone (Background on this error at: https://sqlalche.me/e/20/e3q8)
2025-08-26 16:29:56.951 109496 ERROR keystone
[pid: 109496|app: 0|req: 18/22] 127.0.0.1 () {32 vars in 474 bytes} [Tue Aug 26 16:29:56 2025] POST /v3/auth/tokens => generated 0 bytes in 8 msecs (HTTP/1.0 500) 0 headers in 0 bytes (0 switches on core 1)
```

Of course the database isn’t working 🤷. Let’s see where it’s configured in the `keystone` module.
Found it:
https://github.com/cobaltcore-dev/openstack-nix/blob/e97581816584486555d2647ae2b2aae6b1751197/modules/controller/keystone.nix#L44-L56

Looks like it’s using `systemd-tmpfiles` again:
https://github.com/cobaltcore-dev/openstack-nix/blob/e97581816584486555d2647ae2b2aae6b1751197/modules/controller/keystone.nix#L82-L106

I’m willing to bet we have some stale `symlinks` again. The config says `mysql`, but the logs suggest `sqlite` database so the config file is probably not being used. Let’s check the `systemd-tmpfiles` config files and delete/recreate all the files once and for all.

```sh
ls -l /etc/tmpfiles.d/
```

```log
total 0
lrwxrwxrwx 1 root root 36 Aug 26 15:22 00-nixos.conf -> /etc/static/tmpfiles.d/00-nixos.conf
lrwxrwxrwx 1 root root 37 Aug 26 15:22 10-glance.conf -> /etc/static/tmpfiles.d/10-glance.conf
lrwxrwxrwx 1 root root 38 Aug 26 15:22 10-horizon.conf -> /etc/static/tmpfiles.d/10-horizon.conf
lrwxrwxrwx 1 root root 34 Aug 26 15:22 10-k3s.conf -> /etc/static/tmpfiles.d/10-k3s.conf
lrwxrwxrwx 1 root root 39 Aug 26 15:22 10-keystone.conf -> /etc/static/tmpfiles.d/10-keystone.conf
lrwxrwxrwx 1 root root 38 Aug 26 15:22 10-neutron.conf -> /etc/static/tmpfiles.d/10-neutron.conf
lrwxrwxrwx 1 root root 35 Aug 26 15:22 10-nova.conf -> /etc/static/tmpfiles.d/10-nova.conf
lrwxrwxrwx 1 root root 40 Aug 26 15:22 10-placement.conf -> /etc/static/tmpfiles.d/10-placement.conf
lrwxrwxrwx 1 root root 36 Aug 26 15:22 10-uwsgi.conf -> /etc/static/tmpfiles.d/10-uwsgi.conf
lrwxrwxrwx 1 root root 43 Aug 26 15:22 graphics-driver.conf -> /etc/static/tmpfiles.d/graphics-driver.conf
lrwxrwxrwx 1 root root 32 Aug 26 15:22 home.conf -> /etc/static/tmpfiles.d/home.conf
lrwxrwxrwx 1 root root 41 Aug 26 15:22 journal-nocow.conf -> /etc/static/tmpfiles.d/journal-nocow.conf
lrwxrwxrwx 1 root root 32 Aug 26 15:22 lvm2.conf -> /etc/static/tmpfiles.d/lvm2.conf
lrwxrwxrwx 1 root root 38 Aug 26 15:22 nix-daemon.conf -> /etc/static/tmpfiles.d/nix-daemon.conf
lrwxrwxrwx 1 root root 37 Aug 26 15:22 portables.conf -> /etc/static/tmpfiles.d/portables.conf
lrwxrwxrwx 1 root root 52 Aug 26 15:22 static-nodes-permissions.conf -> /etc/static/tmpfiles.d/static-nodes-permissions.conf
lrwxrwxrwx 1 root root 35 Aug 26 15:22 systemd.conf -> /etc/static/tmpfiles.d/systemd.conf
lrwxrwxrwx 1 root root 43 Aug 26 15:22 systemd-nologin.conf -> /etc/static/tmpfiles.d/systemd-nologin.conf
lrwxrwxrwx 1 root root 42 Aug 26 15:22 systemd-nspawn.conf -> /etc/static/tmpfiles.d/systemd-nspawn.conf
lrwxrwxrwx 1 root root 39 Aug 26 15:22 systemd-tmp.conf -> /etc/static/tmpfiles.d/systemd-tmp.conf
lrwxrwxrwx 1 root root 31 Aug 26 15:22 tmp.conf -> /etc/static/tmpfiles.d/tmp.conf
lrwxrwxrwx 1 root root 31 Aug 26 15:22 var.conf -> /etc/static/tmpfiles.d/var.conf
lrwxrwxrwx 1 root root 31 Aug 26 15:22 x11.conf -> /etc/static/tmpfiles.d/x11.conf
```

Good thing I can just purge these with a wildcard:

```sh
awk '{print $2}' /etc/tmpfiles.d/10-* | xargs -r sudo rm -rf
```

> [!warning]
> DO NOT RUN THIS COMMAND BLINDLY! I verified it multiple times before running it. It might delete something important!

And recreate them:

```sh
sudo systemd-tmpfiles --create
```

Let’s just restart the machine to let all the services start again (who knows how many services depended on those files I just deleted).

We’re back, let’s check for failed services:

```sh
systemctl list-units --failed
```

```log
  UNIT LOAD ACTIVE SUB DESCRIPTION

0 loaded units listed.
```

No failed services! I have a good feeling about this 🤞. Time to try out the `openstackclient`. Of course, we need credentials for that and some OpenStack services probably do too so we can just extract the credentials from them 🤷.

Turns out, the credentials are configured as environment variables. Here’s what we need:
https://github.com/cobaltcore-dev/openstack-nix/blob/e97581816584486555d2647ae2b2aae6b1751197/modules/controller/openstack-controller.nix#L15-L23

Looks like we can read them from `systemd.services.glance.environment`:
https://github.com/cobaltcore-dev/openstack-nix/blob/e97581816584486555d2647ae2b2aae6b1751197/modules/controller/openstack-controller.nix#L129

```nix title="hosts/proart-p16/default.nix"
{
  pkgs,
  config,
  lib,
  inputs,
  ...
}:
{
  # ...
  environment.variables = config.systemd.services.glance.environment;
  # ...
}
```

After rebuilding, let’s try some `openstack` commands:

```sh
openstack catalog list
```

```log
+----------------------+-----------+-----------------------------------------+
| Name                 | Type      | Endpoints                               |
+----------------------+-----------+-----------------------------------------+
| Identity Service     | identity  | RegionOne                               |
|                      |           |   public: http://controller:5000/v3     |
|                      |           | RegionOne                               |
|                      |           |   admin: http://controller:5000/v3      |
|                      |           | RegionOne                               |
|                      |           |   internal: http://controller:5000/v3   |
|                      |           |                                         |
| Compute Service V2.1 | compute   | RegionOne                               |
|                      |           |   public: http://controller:8774/v2.1   |
|                      |           | RegionOne                               |
|                      |           |   admin: http://controller:8774/v2.1    |
|                      |           | RegionOne                               |
|                      |           |   internal: http://controller:8774/v2.1 |
|                      |           |                                         |
| Image Service        | image     | RegionOne                               |
|                      |           |   public: http://controller:9292        |
|                      |           | RegionOne                               |
|                      |           |   admin: http://controller:9292         |
|                      |           | RegionOne                               |
|                      |           |   internal: http://controller:9292      |
|                      |           |                                         |
| Network Service      | network   | RegionOne                               |
|                      |           |   public: http://controller:9696        |
|                      |           | RegionOne                               |
|                      |           |   admin: http://controller:9696         |
|                      |           | RegionOne                               |
|                      |           |   internal: http://controller:9696      |
|                      |           |                                         |
| Placement Service    | placement | RegionOne                               |
|                      |           |   public: http://controller:8778        |
|                      |           | RegionOne                               |
|                      |           |   admin: http://controller:8778         |
|                      |           | RegionOne                               |
|                      |           |   internal: http://controller:8778      |
|                      |           |                                         |
+----------------------+-----------+-----------------------------------------+
```

```sh
openstack user list
```

```log
+----------------------------------+-----------+
| ID                               | Name      |
+----------------------------------+-----------+
| 361c54bfa09143dfb0c0f802b3f99b62 | admin     |
| 64cb3910fe5e45f191517763e15dda10 | glance    |
| 3a678fd3dd264fe686b40259ec351d23 | placement |
| d3284e6c93764791a6142ec2e1bde502 | neutron   |
| 0e99f051b4a54a9883cb3f9b0b4c7dba | nova      |
+----------------------------------+-----------+
```

Hurray! At least some things are working. I doubt everything is fully functional, but let’s leave that for another day (and fixing the `horizon` service too). 🎉
