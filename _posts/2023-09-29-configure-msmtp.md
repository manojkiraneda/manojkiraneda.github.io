---
title: "Send emails from your terminal"
date: "2023-09-29 8:38AM"
categories: ["workflow"]
tags: ["workflow", "msmtp", "smtp-gmail", "send-mail"]
---

In this tutorial, we'll configure everything needed to send emails from the 
terminal. We'll use msmtp, a lightweight SMTP client. For the sake of the example,
we'll use a GMail account, but any other email provider can do. Your OS is
expected to be Ubuntu based, as usual, although it doesn't really matter. We will
also see how to store the credentials for the email account in the system keyring.

## GMail account setup

For a GMail account, there's a bit of configuration to do. For other email
providers, I have no idea, maybe you can just skip this part, or maybe you will
have to go through a similar procedure.

If you want an external program (msmtp in this case) to talk to the GMail servers
on your behalf, and send emails, you can't just use your usual GMail password.
Instead, GMail requires you to generate so-called app passwords, one for each
application that needs to access your GMail account.

This approach has several advantages:

- it will basically work, GMail won't block you because it thinks that you're
  trying to sign in from an unknown device, a weird location or whatever.
- your main GMail password remains secret, you won't have to write it down in
  any configuration file or anywhere else.
- you can change your main GMail password, no breakage, apps will still work as
  each of them use their own passwords.
- you can revoke an app password anytime, without impacting anything else.

So app passwords are a good idea, it just requires a bit of work to set it up.
Let's see what it takes.

First, `2-Step Verification` must be enabled on your GMail account. Visit 
<https://myaccount.google.com/security>, and if that's not the case, enable it.
You'll need to authorize all of your devices (computer(s), phone(s) and so on),
and it can be a bit tedious, granted. But you only have to do it once in a lifetime,
and after it's done, you're left with a more secure account, so it's not that bad,
right?

Enabling the 2-Step Verification will unlock the feature we need: `App passwords`.
Visit <https://myaccount.google.com/apppasswords>, and under `Signing in to Google`,
click `App passwords`, and generate one. An app password is a 16 characters
string, something like `qwertyuiopqwerty`. It's supposed to be used from only
one place, ie. from `ONE application` that is installed on `ONE device`. That's
why it's common to give it a name of the form `application@device`, so in our
case it could be msmtp@laptop, but really it's free form, choose whatever name
suits you, as long as it makes sense to you.

So let's give a name to this app password, write it down for now, and we're done
with the GMail config.

## Send your first email

Time to get started with `msmtp`.

First thing first, installation, trivial:

```bash
sudo apt install msmtp
```

Let's try to send an email. At this point, we did not create any configuration
file for msmtp yet, so we have to provide every details on the command line.

```
# Write a dummy email
cat << EOF > message.txt
From: YOUR_LOGIN@gmail.com
To: SOMEONE_ELSE@SOMEWHERE_ELSE.com
Subject: Cafe Sua Da

Iced-coffee with condensed milk
EOF

# Send it
cat message.txt | msmtp \
    --auth=on --tls=on \
    --host smtp.gmail.com \
    --port 587 \
    --user YOUR_LOGIN \
    --read-envelope-from \
    --read-recipients

# msmtp prompts you for your password:
# this is where goes the app password!
```

Obviously, in this example you should replace the uppercase words with the real
thing, that is, your email login, and real email addresses.

Also, let me insist, you must enter the app password that was generated
previously, not your real GMail password.

And it should work already, this email should have been sent and received by now.

So let me explain quickly what happened here.

In the file message.txt, 
we provided 
From: (the email address of the person sending the email) and 
To: (the destination email address).
 
Then we asked msmtp to re-use those values to set the envelope of the email with
`--read-envelope-from` and `--read-recipients`.

What about the other parameters?

`--auth=on` because we want to authenticate with the server.
`--tls=on` because we want to make sure that the communication with the server is encrypted.
`--host` and `--port` tells where to find the server. If you don't use GMail, adjust that accordingly.
`--user` is obviously your GMail username.

For more details, you should refer to the msmtp documentation.

## Write a configuration file

So we could send an email, that's cool already.

However the command to do that was a bit long, and we don't want to juggle with
all these arguments every time we send an email. So let's write down all of that
into a configuration file.

msmtp supports two locations: `~/.msmtprc` and `~/.config/msmtp/config`, at your
preference. In this tutorial we'll use `~/.msmtprc`:

```
cat << 'EOF' > ~/.msmtprc
defaults
tls on

account gmail
auth on
host smtp.gmail.com
port 587
user YOUR_LOGIN
from YOUR_LOGIN@gmail.com

account default : gmail
EOF
```
And for a quick explanation:

under `defaults` are the default values for all the following accounts.
under `account` are the settings specific to this account, until another account line is found.
finally, the last line defines which account is the default.

All in all it's pretty simple, and it's becoming easier to send an email:

```
# Write a dummy email. Note that the
# header 'From:' is no longer needed,
# it's already in '~/.msmtprc'.
cat << 'EOF' > message.txt
To: SOMEONE_ELSE@SOMEWHERE_ELSE.com
Subject: Flat White

The milky way for coffee
EOF

# Send it
cat message.txt | msmtp \
    --account default \
    --read-recipients
```

Actually, `--account default` is not needed, as it's the default anyway if you
don't provide a `--account` argument. Furthermore `--read-recipients` can be
shortened as `-t`. So we can make it real short now:

```bash
msmtp -t < message.txt
```

At this point, life is good! Except for one thing maybe: we still have to type
the password every time we send an email. Surely it must be possible to avoid
that annoyance...

## Store your password in the system keyring

For this part, we'll make use of the `libsecret` tool to store the password in
the system keyring via the [Secret Service API](). It means that your desktop
environment should implement the Secret Service specification, which is the case
for both GNOME and KDE.

Note that GNOME provides `Seahorse` to have a look at your secrets, KDE has the
`KDE Wallet`. There's also `KeePassXC`, which I have only heard of but never used.
I guess it can be your password manager of choice if you use neither GNOME nor KDE.

Alright! So let's just make sure that the libsecret tools are installed:

```bash
sudo apt install libsecret-tools
```

And now we can store our password in the system keyring with this command:
```bash
secret-tool store --label msmtp \
    host smtp.gmail.com \
    service smtp \
    user YOUR_LOGIN
```

Let's try to send an email again:

```bash
msmtp -t < message.txt
```

No need for a password anymore, msmtp got it from the system keyring!

For more details on how msmtp handle the passwords, and to see what other methods
are supported, refer to the extensive documentation.

## Git Send-Email

Sending emails with git is a common workflow for some projects, like the Linux
kernel. How does git send-email actually send emails? From the git-send-email
manual page:

> the built-in default is to search for sendmail in /usr/sbin, /usr/lib and $PATH
> if such program is available

It is possible to override this default though:

> --smtp-server= [...] Alternatively it can specify a full pathname of a 
> sendmail-like program instead; the program must support the -i option.

So in order to use msmtp here, you'd add a snippet like that to your `~/.gitconfig`
file:

```
[sendemail]
    smtpserver = /usr/bin/msmtp
```

For a full guide, you can also refer to <https://git-send-email.io>.