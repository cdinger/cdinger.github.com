---
layout: post
title: Using ssh bastion hosts without going mad
lead: "A simple ssh configuration to transparently agent-forward through a bastion host."
---

My employer uses ssh bastion hosts. This means in order to ssh into any server,
I must first connect to a known secure server, or
'[bastion host](http://en.wikipedia.org/wiki/Bastion_host)'.

To deal with the double-connection pain, most people I know maintain a long
list aliases to the servers they most commonly ssh into. For years I too
maintained one of these wild configurations in my dotfiles, changing and
committing it every time we stood up new servers. No more!

### SSH config

My `.ssh/config` now contains these four precious lines:

    # ~/.ssh/config
    Host bastion
      HostName some-bastion-host.umn.edu

    Host *.umn.edu
      ProxyCommand ssh bastion nc %h %p

This configuration will match any host under `umn.edu` and transparently proxy it
through the bastion host.

### Bash completion

I also use `bash-completion` on my Mac (`brew install bash-completion`). This
allows me to auto-complete hostnames that are in my `~/.ssh/known_hosts`.
With bash-completion available, `ssh blah+TAB` auto-completes to
`ssh blah.long-department-name.random-number.umn.edu`.

So in the past, my ssh aliases existed for two reasons: to perform the proxy,
and to shorten crazy-long hostnames. This ssh config and bash-completion make
the need to maintain aliases disappear.
