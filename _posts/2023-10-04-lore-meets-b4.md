---
title: "lore meets b4"
date: "2023-10-04 04:07PM"
categories: ["workflow"]
tags: ["workflow", "tools"]
---

[b4](https://pypi.org/project/b4/) is a helper utility to work with patches made
available via a public-inbox archive like `lore.kernel.org`. It is written to make
it easier to participate in a patch-based workflows, like those used in the
Linux kernel development.

It can do things like :

- rearrange the patches in proper order
- tally up various follow-up trailers like Reviewed-by, Acked-by, etc
- check if a newer series revision exists and automatically grab it

Starting with version `0.10` b4 can:

- create and manage patch series and cover letters
- track and auto-reroll series revisions
- display range-diffs between revisions
- apply trailers received from reviewers and maintainers
- submit patches without needing a valid SMTP gateway

## Install b4

If you want to try it out, you can install `b4` using:

```bash
pip install --user b4
```

## prep the source tree

Prepare a topical branch where you will be doing your work. We'll be fixing a
typo in arch/arm/boot/dts/aspeed-bmc-opp-lanyang.dts, and we'll base this work
on tag dev-6.1

```bash
git clone https://github.com/openbmc/
b4 prep -n dts-aspeed-typo -f dev-6.5
```

This creates a branch named `b4/dts-aspeed-typo` based of the dev-6.5 tag.It also
add an empty commit for the cover letter which we will edit later.

## Grab a patch to test

If we are not grabbing any patch of others to test/rework on, this step is can be
skipped.

Lets say we want to grab the eSPI driver patch from [here](https://lore.kernel.org/openbmc/20220516005412.4844-4-chiawei_wang@aspeedtech.com/)
all we need is the `Message-ID` field from the email, you can even grab this from
mutt via the command line (or) from the web interface, for the purpose of simplicity
i am grabbing it from the web interface of lore.

```bash
b4 am 20220516005412.4844-4-chiawei_wang@aspeedtech.com
```

The above command generates an mbox file as shown below:

```bash
manojeda@vmlinux$ b4 am 20220516005412.4844-4-chiawei_wang@aspeedtech.com
Grabbing thread from lore.kernel.org/all/20220516005412.4844-4-chiawei_wang%40aspeedtech.com/t.mbox.gz
Analyzing 12 messages in the thread
Checking attestation on all messages, may take a moment...
---
  [PATCH v5 1/4] dt-bindings: aspeed: Add eSPI controller
  [PATCH v5 2/4] MAINTAINER: Add ASPEED eSPI driver entry
  [PATCH v5 3/4] soc: aspeed: Add eSPI driver
  [PATCH v5 4/4] ARM: dts: aspeed: Add eSPI node
---
Total patches: 4
---
Cover: ./v5_20220516_chiawei_wang_arm_aspeed_add_espi_support.cover
 Link: https://lore.kernel.org/r/20220516005412.4844-1-chiawei_wang@aspeedtech.com
 Base: not specified
       git am ./v5_20220516_chiawei_wang_arm_aspeed_add_espi_support.mbx
```

Lets apply the patch onto your branch using:

```bash
git am -3 ./v5_20220516_chiawei_wang_arm_aspeed_add_espi_support.mbx
```

if we get any merge conflicts, lets fix them and then do

```bash
git add <files>
git am --continue
```

## Make the changes and commit

Make the changes to the dts file and commit it using the following commands:

```bash
git add <file>
git commit -s
```

## Edit the cover letter

If you plan to submit a single patch, then the cover letter is not that necessary
and will only be used to track the destination addresses and changelog entries.
You can delete most of the template content and leave just the title and sign-off.

we can edit the cover letter by using the following command:

```bash
b4 prep --edit-cover
```

## Obtain the reviews & maintainers email id's

After we have committed our work, you will want to collect the addresses of people
who should be the ones reviewing it. Running b4 prep --auto-to-cc will invoke
scripts/get_maintainer.pl with the default recommended flags to find out who
should go into the `To:` and `Cc:` headers:

```bash
b4 prep --auto-to-cc
```

## Dry-run and checkpatch

Next, generate the patches using following command and look at their contents to
make sure that everything is looking sane.

```bash
b4 send -o /tmp/mypatch
```

 Good things to check are:

- the From: address
- the To: and Cc: addresses
- general patch formatting
- cover letter formatting (if more than 1 patch in the series)

If everything looks sane, one more recommended step is to run checkpatch.pl from
the top of the kernel tree:

```bash
./scripts/checkpatch.pl /tmp/mypatch/*
```

## Reflect the email to yourself

This is the last step to use before sending off your contribution. Note, that it
will fill out the To: and Cc: headers of all messages with actual recipients,
but `it will NOT actually send mail to them, just to yourself`.

```bash
b4 send --reflect
```

## Finally send the patch

If all your tests are looking good, then you are ready to send your work. Fire off
`b4 send`, review the `Ready to:` section for one final check and either `Ctrl-C`
to get out of it, or hit Enter to submit your work upstream.

```bash
b4 send
```

## Issue with gnome-keyring & ssh session on a VM & b4 send

If you are some one who is using wsl (or) a linux VM for the kernel development
and sending b4 commands from an ssh session(to the linux VM), we need to make
sure that we unlock the gnome-keyring for every ssh session. Otherwise a `b4 send`
would always prompts the password even though we have the gmail app password set
in the keyring. (if you dont understand what this is about , you might wanna look
at [this](https://manojkiraneda.github.io/posts/configure-msmtp/) blog post first
which talks about setting up msmtp as smtp server for sending emails from the
terminal)

There might be more cleaner ways to deal with this problem , but a simple hack
that i could come up with is a simple bash snippet put in my bashrc file

```bash
#!/bin/bash

echo -n 'Unlocking the keyring, please provide the login password : ' >&2
read -s _UNLOCK_PASSWORD || return
killall -q -u "$(whoami)" gnome-keyring-daemon
eval $(echo -n "${_UNLOCK_PASSWORD}" \
           | gnome-keyring-daemon --daemonize --login \
           | sed -e 's/^/export /')
unset _UNLOCK_PASSWORD
echo '' >&2
```

Thanks for reading the post, while this post does not cover every option
