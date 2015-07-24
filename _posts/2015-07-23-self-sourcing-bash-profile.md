---
layout: post
title: Self-sourcing bash profile
lead: "Get ~/.bash_profile changes to self-source."
---

If you're like me, you're constantly popping into `~/.bash_profile` (or it's
friends) and making tweaks. I've always kept an alias for this, `eb`. That
makes the edit quicker, but the obnoxious `source ~/.bash_profile` remains.
Of course you could plop another alias down for the sourcing, but that's kind
of gross. Instead, you can make your `eb` alias source after the edit is
complete:

    alias eb="vim ~/.bash_profile && source ~/.bash_profile"

When you use that alias, you pop into `~/bash_profile`, save your change, and
continue to hack away.
