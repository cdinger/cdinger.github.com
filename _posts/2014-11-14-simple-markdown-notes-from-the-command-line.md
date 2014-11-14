---
layout: post
title: "Simple markdown note taking—launched from the command line"
lead: "How I ditched Evernote for markdown/vim with a tiny bash script for taking notes."
---

If you're like me, you take notes in various different ways. Evernote. Notational Velocity. Random text files on the desktop—usually named something like `lkajsdflkajsdf.md`. A notebook (those things with paper pages that you operate by dragging a pen across its surface). And if you're really like me, you have notes scattered among all these different repositories. These tools are all great, but none of them have ever stuck for me. And when you go to look for something, you have to look in several places. Blech.

### Less is more

When I started my new job a few months ago, I decided to just to ditch the notebook and Evernote. Returning to a developer role, I found myself spending most of my time with my old friends, vim and the command line.

For simplicity's sake, I started with a directory called `Notes` in my Dropbox that served as a bucket of markdown files. Each file followed the naming convention:

```
YYYYMMDD_title_of_the_note.md
```

I searched existing notes with `grep`. Simple.

### Speeding things up

To speed things up a bit, I plopped an alias in my `.bash_profile` called `note` to `cd` to this directory and pop open MacVim. I find it useful to have NERDTree open alongside the new note because I often want to look back at the last note I took for a standing meeting. This alias sped up getting into the note itself, but I found myself often leaving these in unnamed vim tabs and then closing them without saving. It seemed to be time for a proper bash script.

I also realized that re-typing the name of a recurring meeting was a waste of time and prone to typos. I knew that bash completion could be used here reflect on previous note titles. So when I sit down for my next biweekly check-in with Kemal, I'd love to just type `note kTAB` and have my terminal autocomplete to `note kemal_check_in`.

A bash script would also easily fill in the current date for me. Typing `20141114` before the note title is annoying and something that can be easily grabbed from the `date` command.

### The bash script with completion

So here's what I ended up with:

#### `~/bin/note`
{% highlight bash %}
#!/bin/bash
cd ~/Dropbox/Notes && mvim "$(date +%Y%m%d)_$1.md"
{% endhighlight %}

#### `~/bin/note-completion.sh`
{% highlight bash %}
complete -W "$(ls -A1 ~/Dropbox/Notes/*.md | rev | cut -d'/' -f1 | rev | cut -c10- | cut -d'.' -f1 | uniq | sort )" note
{% endhighlight %}

Completion is initialized in my `.bash_profile` like so:

#### `~/.bash_profile`
{% highlight bash %}
[[ -s "/your/home/path/bin/note-completion.sh" ]] && source "/your/home/path/bin/note-completion.sh"
{% endhighlight %}

Just two lines of bash to toss this into my dotfiles repo so it's always available. For now at least, this seems to works pretty well.

Note that you can easily replace `mvim` with your editor of choice. `subl` or `atom` work just as well.
