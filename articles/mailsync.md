# Mutt, IMAP and Mail Synchronization with Runit

I've always been a fan of [mutt](http://mutt.org/) for email. Twenty years ago,
I ran a local mail server, making mutt configuration trivial. Coming back after
a ten-year hiatus when I used Apple's Mail.app, I decided against running
another mail server. While mutt works well in this circumstance, it takes some
work to configure.

Mutt supports IMAP directly, but the experience is painful. The interface is
laggy while it syncs with the remote server. I'm not even sure what happens
when the server is unavailable. A better approach is to let mutt manage a local
mail store and find external programs to sync with the remote mailboxes.

For awhile, I was using [offlineimap](http://www.offlineimap.org/). However,
its reliance on Python 2 and the slow progress on a Python 3 rewrite led me to
[isync](https://isync.sourceforge.io/), which supports multiple accounts,
multiple sync profiles and built-in support for reading passphrases from an
external command.

To send messages, [msmtp](https://marlam.de/msmtp/) is a convenient option.
Like isync, msmtp supports multiple accounts and external passphrase reading.
Together, these allow mutt to be configured with multiple account
"personalities".

There are plenty of guides for configuring mutt, isync and msmtp, so I won't
belabor the issue. However, there are a few key aspects that are worth
documenting.

## Maildir Structure

I separate mail into three accounts, each of which is managed separately by
isync:

    ~/Mail/Work
    ~/Mail/Home
    ~/Mail/Other

Each of these is a separate `MaildirStore` in my isync configuration, with a
corresponding `IMAPStore` configured to point to the remote server. For
password management, I rely on [pass](https://www.passwordstore.org/), which
automatically encrypts text content using GPG. A simple

    PassCmd "/usr/bin/pass path/to/account/credentials"

in each `IMAPStore` section allows isync to discover the password provided that
my GPG key is unlocked. I use `gpg-agent` and a custom script run from the
`pam_exec` PAM module to cache my passphrase.

For msmtp, the configuration is similar, with separate accounts associated with
each of my separate service providers. Again, I rely on pass to read
credentials:

    passwordeval pass path/to/account/credentials


Each account in isync has two associated "channels":

1. `frequent`, which syncs `INBOX`, `Archive`, `Drafts` and `Sent`
2. `occasional`, which syncs every other mailbox.

## IMAP Synchronization

For regular syncing, a
[runit user service](https://docs.voidlinux.org/config/services/user-services.html)
does the job. However, I don't simple time-based synchronization. What I'd
prefer to do is configure a triggering mechanism, then act when the trigger
fires. For this, I use a generic runit service script,
[snooze](https://github.com/leahneukirchen/snooze) and
[inotify-tools](https://github.com/rvoicilas/inotify-tools). By default, I
touch trigger files in my Mail hierarchy:

    ~/Mail/triggers/Work
    ~/Mail/triggers/Home
    ~/Mail/triggers/Other

and then watch for changes (such as those initiated by a simple `touch`) to
launch the sync. The generic run script looks like

    #!/bin/sh
    
    set -e
    
    CHANNEL="${PWD##*-}"
    
    [ -f ./conf ] && . ./conf
    
    # Define a trigger, make sure it exists as a file
    [ -n "$TRIGGER" ] || TRIGGER="${HOME}/Mail/triggers/${CHANNEL}"
    [ -e "$TRIGGER" ] || touch "$TRIGGER"
    [ -f "$TRIGGER" ] || exit 1
    
    # Throttle multiple synchronization attempts
    [ -n "$THROTTLE_RATE" ] || THROTTLE_RATE=60
    
    # Fire synchronization attempts if enough time passes with no trigger
    [ -n "$TIMEOUT" ] || TIMEOUT=900
    
    # Forwarding signals to inotifywait is too awkward without exec...
    # Use file as sempahore to control state transitions between wait and sync
    semaphore="./supervise/mbsync_needed"
    
    if [ ! -f "$semaphore" ]; then
        touch "$semaphore"
        exec inotifywait -qq -e attrib -t $TIMEOUT "$TRIGGER"
        exit 1
    else
        rm -f "$semaphore"
        # Fire the (throttled) sync
        exec snooze -H/1 -M/1 -S/1 -t timefile -T "$THROTTLE_RATE" \
                /bin/sh -c "mbsync -q $CHANNEL 2>&1; touch timefile"
    fi

First, I reuse this script for each account (`CHANNEL`, in the script), so I
gather the name of the account from the service name, which is something like
`mbsync-Work`, `mbsync-Home` or `mbsync-Other`. Then, I determine the location
of the trigger file, a throttle rate in seconds to avoid rapid
resynchronization attempts, and a timeout; a time-based sync will be forced
every `$TIMEOUT` seconds, if trigger-based sync hasn't happened in the
meantime. A `conf` file in the service directory can be used to override the
defaults.

The script operates in one of two modes, wait or sync, depending on the
existence or absence of a "semaphore" file. If the file does not exist, the
service is in "wait" mode; it touches the semaphore to indicate to the next run
that the mode should be switched, then uses `inotifywait` to just wait for the
trigger file to be touched or enough time to pass. When runit restarts the
service, it finds the semaphore file, enters sync mode, removes the semaphore
(to toggle the mode in the next run), and attempts a sync (waiting for the
throttle period as appropriate).

This toggling mode isn't strictly necessary; instead, one could wait on
`inotifywait` without a preceding `exec`, which would allow the script to
continue after the file was touched or the timeout passed. However, in that
instance, the script will not properly forward signals to the `inotifywait`
process, so attempting to bring down the service will leave `inotifywait`
sitting around for nothing. If you care about cleaning up `inotifywait` when
you terminate the service, you can do a big dance with signal handlers in the
shell, or you can employ my mode-switch trick to allow both `inotifywait` and
`snooze` to be exec'ed.

## Firing Triggers

Now that isync runs when triggered (or every fifteen minutes in the absence of
a trigger), let's use the trigger mechanism to make synchronization more
responsive. The first aspect of responsiveness is pushing changes in the local
mail store to the IMAP servers. This is easy to do with a service that wraps
[wendy](http://git.z3bra.org/wendy/log.html), a simple wrapper to inotify
that runs eternally, firing commands whenever file or directory changes are
observed. The runit service for this looks like

    #!/bin/sh
    
    [ -f ./conf ] && . ./conf
    
    # Make sure the trigger file will be in an existing directory
    [ -n "$TRIGGER" ] || TRIGGER="${HOME}/Mail/triggers/${PWD##*-}"
    [ -d "$(dirname "$TRIGGER")" ] || exit 1
    
    # Make sure a watchlist exists
    [ -n "$WATCHLIST" ] || WATCHLIST="${TRIGGER}.watchlist"
    [ -f "$WATCHLIST" ] || exit 1
    
    # When any of the mail directories change, touch the trigger
    exec wendy -m 4034 touch "$TRIGGER" < "$WATCHLIST"

Once again, I use the same script for multiple monitor services, differentiated
by names like `mailmon-Home`, `mailmon-Work` or `mailmon-Other`. Alongside the
trigger directory, I keep a "watchlist" of mailboxes to monitor for changes;
this looks like

    ~/Mail/Home/INBOX/cur
    ~/Mail/Home/INBOX/new
    ~/Mail/Home/INBOX/tmp
    ~/Mail/Home/Archive/cur
    ~/Mail/Home/Archive/new
    ~/Mail/Home/Archive/tmp
    ~/Mail/Home/Drafts/cur
    ~/Mail/Home/Drafts/new
    ~/Mail/Home/Drafts/tmp
    ~/Mail/Home/Sent/cur
    ~/Mail/Home/Sent/new
    ~/Mail/Home/Sent/tmp

for my "Home" account. (In the file, I manually expand `~`; I don't know if
`wendy` will properly expand `~` automatically.) Note that the mailboxes I
monitor correspond to the `frequent` isync channels; changes in other
mailboxes will only be propagated after the sync wait times out.

The `wendy` mode, `4034`, is a bitmask described in its [man
page](https://man.voidlinux.org/wendy.1); expanding this into meaning inotify
events is an exercise for the reader, but it should capture the kinds of file
and directory changes that happen when moving mail around Maildir structures.

With this service running for each account, I have something to touch a sync
trigger any time I move mail around my frequent mailboxes, and the change
should be propagated outward within a minute (based on my throttling
configuration).

## IMAP IDLE: More Responsive Inbound Messaging

To make sure I receive mail more frequently than the 15-minute timeout allows,
I am interested in monitoring mailbox status using IMAP IDLE. There are a few
programs that do this, but the one I use is
[goimapnotify](https://gitlab.com/shackra/goimapnotify). This program connects
to a given IMAP server, registers an IDLE on any number of mailboxes, and waits
for new mail notifications to come through. The relevant portion of its
configuration is

    "passwordCmd": "/usr/bin/pass path/to/account/credentials"
    "onNewMail": "/usr/bin/touch ~/Mail/triggers/<account>"
    "boxes": [
        "INBOX",
	"Archive",
	"Drafts",
	"Sent"
    ]

Again, I rely on pass to manage my credentials, and I touch the appropriate
trigger file when new mail notifications are received. The mailboxes monitored
for events are those from my `frequent` isync group.

The generic runit script takes the form

    #!/bin/sh
    
    set -e
    
    [ -f ./conf ] && . ./conf
    
    [ -n "$THROTTLE_RATE" ] || THROTTLE_RATE=30
    [ -n "$PROFILE" ] || PROFILE="${PWD##*-}"
    
    [ -n "$PROFILE" ] || exit 1
    
    config="${HOME}/.config/imapnotify/${PROFILE}.conf"
    [ -f "$config" ] || exit 1
    
    # Run the watcher, but throttle its restart
    exec snooze -H /1 -M /1 -S /1 -t timefile -T $THROTTLE_RATE \
    		/bin/sh -c "touch timefile; exec goimapnotify -conf '$config' 2>&1"

I'm using the usual name-to-account trick with support for a `conf` override.
This service just launches `goimapnotify` with the right configuration file for
a particular account, but I wrap it in `snooze` to throttle reconnection
attempts if something goes wrong. (Mostly, what goes wrong is a failure to grab
credentials because `gpg-agent` is locked, but if there is a problem
server-side that kicks me off, I don't need to keep reconnecting every second
or two.)

## Mutt Configuration

For each account, I create a `~/.mutt/accound.<account>` configuraton snippet,
which looks something like

    # Set some mailbox locations
    set mbox="+<account>/INBOX"
    set postponed="+<account>/Drafts"
    set record="+<account>/Sent"
    set trash="+<account>/Trash"
    
    # By default, save in Archive
    save-hook . "+<account>/Archive"
    
    set from="User Name <user@account.domain>"
    set sendmail="/usr/bin/msmtp -a <account>"
    set signature="~/.signature.<account>"
    
    color status red white

This sets per-account mailbox locations, signatures, and identities. I also
change the status colors based on the current account, so it is easy to tell
which identity I'm currently using. The `sendmail` command instructs `msmtp` to
use the right SMTP account to send for this identity.

In the main `muttrc`, I use `folder-hook` to make the jump between identities:

    set mbox_type=Maildir
    set folder=~/Mail/

    # Pick a preferred default account
    set spoolfile='+<default-account>/INBOX'
    source ~/.mutt/account.<default-account>

    folder-hook Home/* source ~/.mutt/account.Home
    folder-hook Work/* source ~/.mutt/account.Work
    folder-hook Other/* source ~/.mutt/account.Other

Thus, whenever I change to a mailbox under a specific account, mutt updates its
configuration to use the right identity.

## Conclusion

With a little bit of effort to set up trigger-based IMAP syncing with isync, it
because easy to define arbitrary services that do nothing but touch the
triggers to force a mail sync on demand. This eliminates the need for excessive
polling for new mail but still ensures prompt updates. Combined with a simple
message for reconfiguration on the fly based on the currently displayed
mailbox, it is easy to have sophisticated, responsive IMAP access within mutt
without relying on its more limited native functionality.
