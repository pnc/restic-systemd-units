# restic systemd units

This is a collection of systemd units for managing backups with
[restic][].

[restic]: https://restic.net/

## Installing

You need a daemon user called `restic`. You can create one with:

    sudo adduser --system --no-create-home --disabled-login --group restic

To install the systemd units and associated support files:

    make && sudo make install && sudo systemctl daemon-reload

### What gets installed?

A bunch of systemd units:

    /etc/systemd/system/restic-backup-daily@.timer
    /etc/systemd/system/restic-backup-monthly@.timer
    /etc/systemd/system/restic-backup@.service
    /etc/systemd/system/restic-backup.target
    /etc/systemd/system/restic-backup-weekly@.timer

    /etc/systemd/system/restic-check-daily@.timer
    /etc/systemd/system/restic-check-monthly@.timer
    /etc/systemd/system/restic-check@.service
    /etc/systemd/system/restic-check-weekly@.timer

And this `tmpfiles.d` configuration, which ensures that `/run/restic`
and `/var/cache/restic` exist:

    /etc/tmpfiles.d/restic.conf

And a helper script, used for manually running restic against the
backup profiles used by these units:

    /usr/local/bin/restic-helper

## Configuration

Put environment variables common to all your backups in
`/etc/restic/restic.conf`.  For example, on my system I have:

    # A pointer to your restic password. This can be set globally here
    # or per-backup profile.
    RESTIC_PASSWORD_FILE=/etc/restic/password

Create one or more backup profiles as subdirectories of `/etc/restic`.
For example, if you're going to be backing up `/home`, you might
create `/etc/restic/home` with the following configuration in
`/etc/restic/home/restic.conf`:

    # What to back up?
    BACKUP_DIR=/home

    # Where to store backups?
    RESTIC_REPOSITORY=/mnt/backups

    # Arguments for the backup command
    RESTIC_BACKUP_ARGS="--tag home"

    # Arguments for the forget command
    RESTIC_FORGET_ARGS="--tag home --keep-daily 2 --keep-weekly 2 --keep-monthly 1"

## Enabling backups

Assuming that you have `/etc/restic/home` configured as above, then to
schedule daily backups of `/home`:

    systemctl enable --now restic-backup-daily@home.timer

Or to schedule weekly backups:

    systemctl enable --now restic-backup-weekly@home.timer

`forget` and `prune` are done automatically after the Sunday backup. See
`restic-backup` for the specific logic.

## Scheduling checks

You can schedule a check (see the `restic` docs for details and caveats)
daily using, e.g.:

    sudo systemctl enable --now restic-check-weekly@home.timer

**Note:** You are on your own for setting up a mechanism to be noted
for backup and check failures. Consider adding an `OnFailure` clause
to both `.service` files.

## Running manually

Assuming that you have `/etc/restic/home` configured as above, then to
force a run of the `/home` backup:

    systemctl start --no-block restic-backup@home.service

## Inspecting logs

You can watch the logs for a given backup using:

    journalctl -f -u restic-backup@home.service

or browse the full output with:

    journalctl -u restic-backup@home.service

## Lockfiles

These units use the `flock` binary to serialize multiple instances of
the same template unit.  This means if you start two backups in
parallel...

    systemctl start restic-backup@home restic-backup@database

...they will actually run one after the other instead of running
at the same time.

## Manually running restic commands

The `restic-helper` command reads the environment configuration from
`/etc/restic/restic.conf` and the current directory.  You can use this
if you want to manually run `restic` commands against one of your
backup profiles.  For example:

    # cd /etc/restic/home
    # sudo -u restic restic-helper snapshots
    * reading configuration from /etc/restic/restic.conf
    * reading configuration from ./restic.conf
    password is correct
    ID        Date                 Host    Tags        Directory
    ----------------------------------------------------------------------
    1dc1d283  2018-03-03 00:00:37  myhost  home        /home
    9dbd3830  2018-03-04 00:00:46  myhost  home        /home
    8938235d  2018-03-05 00:00:40  myhost  home        /home
    ----------------------------------------------------------------------
    3 snapshots
