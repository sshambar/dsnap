# DSnap

DSnap is a flexible and easy to configure local file-system rsync
backup manager, and improves on rsync's incremental, space-efficient
backups with History, Backup Sets and Locking.

### Table of Contents

- [Features](#features)
- [Prerequisites](#prerequisites)
- [Usage](#usage)
- [Simple 4-Step Setup](#simple-4-step-setup)
- [Modes of Operation](#modes-of-operation)
- [Backup Source and Target Configuration](#backup-source-and-target-configuration)
- [Target Folders](#target-folders)
- [Backup Sets](#backup-sets)
- [Remote Snap](#remote-snap)
- [Target Locks](#target-locks)
- [Hooks](#hooks)
- [Customization](#customization)
- [Automation](#automation)

## Features

- **History**: Automatically managed point-in-time snapshots.
- **Backup Sets**: All backups sharing a target directory complete or fail
  as a group.
- **Locking**: Per-target locks permits parallel backups while preventing
  conflicting ones.
- **Easy Config**: Per-source rsync filters and options.
- **State**: Backup state reporting.
- **Mirror**: Mirror a backup on a remote host.
- Full customization of configuration file locations and defaults.
- Graceful error handling and easy to understand error messages.
- Optional selection of target(s) for all actions.
- Hooks for pre and post backup actions.
- Dry-run support and debug level tracing.
- Thoroughly tested code.

## Prerequisites

- [Rsync](http://rsync.samba.org/)
- [Bash](http://www.gnu.org/software/bash/)
- Basic Command Line Tools: `touch mv rm mkdir rmdir`
- Optional Tools: `sed find ps nice ionice ssh`

## Usage

~~~~
Usage: dsnap [OPTIONS] MODE [TARGET]...

OPTIONS:
	-c {file} config file location (default: /etc/dsnap/config)
	-h        print short help, twice for detailed help
	-l        rsync logging to stdout (in addition to dsnap.log)
	-n        dry-run, no changes made (useful with -v), twice
		  for fake access
	-r        remote dsnap (to mirror snap.1 from remote hosts)
	-s        sleep a short while before starting backup
	-v        verbose output to stdout, twice for debug logging

MODE:
	list      list most recent backup for each target
	init      initialize new/updated backup entries, creates
	          full backup
	snap      create incremental backup, run once or more daily
	daily | weekly | monthly | yearly
	          manage history, run at the frequency of the mode name
	          (eg: daily)
	unlock    break all target locks, useful during system boot

TARGET:
	If one or more targets are supplied, only backup entries with
	a matching target are processed.  By default, all targets
	are processed.
~~~~

## Simple 4-Step Setup

1. Create the file backup file containing at least one line
(substitute *{..}* values, directories should be full paths):

		$ echo "; {source-dir} ; {target-dir}" > /etc/dsnap/backups

2. "Bless" *{target-dir}* with a DSnap version file:

		$ echo 2 > {target-dir}/dsnap.version

3. Create an initial backup:

		$ dsnap init

4. Add crontab entries to create incremental backups (see [Automation](#automation))

However, DSnap supports a great many options...

## Modes of Operation

Initial full backups are created with the special "init" mode, and is
required when adding/editing backup entries in the "backups" file.

Backups are ordered by date (inter-day snap, daily, weekly etc), with
identical files hard linked together to save space.  DSnap should be
run by cron (or similar) with the following "mode" parameters:

- **snap** - run once or more a day (see **MAX_PER_DAY** below)
- **daily** - run once per day, after snap
- **weekly** - run once per week, after daily
- **monthly** - run once per month, after weekly
- **yearly** - run once per year, after monthly

The "snap" mode creates new incremental backups based on the full
backup created by "init".  All other modes just maintain the history
by renaming directories or removing expired backups (and are
reasonably fast).

The most recent backup time for each target can be listed with the
"list" mode.

If one or more targets are included after the "mode", only backup
entries with matching targets are processed.  By default, all targets
are processed.

## Backup Source and Target Configuration

Files:

	/etc/dsnap/backups
	/etc/dsnap/filters[-{label}]

(see **ETCDIR** in [Customization](#customization) to specify a different
location)

The "backups" file contains entries matching sources with targets.
The format of the file is:

	[ {label} ] ; {source-dir} [ ; {target-dir} [ ; {rsync-options} ] ]

*[..]* indicate optional values, substitute *{..}* values with your own.
Values are separated by semicolons, and empty lines or lines beginning
with '#' are ignored.

- *{source-dir}* is the source directory to backup, and is required.
	If "-r" (remote) is used, {source-dir} is formatted as remote:{remote-dir}
	(matching target on remote machine).
- *{label}* is used to for filter selection and directory names for version
-  2 targets. An empty *{label}* defaults to *{source-dir}*.
- *{target-dir}* is destination directory where all backups are stored.
	An empty *{target-dir}* defaults to **DEFAULT_TARGET_DIR**
- *{rsync-opts}* overrides **DEFAULT_RSYNC_OPTIONS**

> See [Customization](#customization) about **DEFAULT_\*** values.

*{source-dir}* and *{target-dir}* should be full pathnames (starting with '/')
and may contain spaces, special characters, or be double quoted.

> See [Customization](#customization) about **ALLOW_RELATIVE**.

Whenever adding new entries, or updating *{label}* or *{target-dir}*
values, run `dsnap init {target-dir}` to create or update the target.
Without a completed "reference backup," rsync will be unable to link
unchanged files with their existing backup copies, so "snap" fails for
new/updated entries until "init" is successfully run to avoid
inadvertent broken hard links.

> See [Backup Sets](#backup-sets) about grouping backups.

The optional "filters" file contains rsync filters.  The file
"filters-*{label}*" is for label-specific filters. See "man rsync"
(FILTER RULES section) for filter syntax.  The global or per-label file
is used, not both, although one may include the other.

## Target Folders

Files/Folders:

	{target-dir}/dsnap.version
	{target-dir}/snap.1/
	{target-dir}/daily.1/ ...etc

Each target must be "blessed" with the file "dsnap.version" to indicate it
is a valid target, or DSnap will refuse to write to it to avoid overwriting
random directories.  "dsnap.version" must contain the version numbers 1 or 2.
The version determines the directory structure in the backup set.

The backup history is found in folders named after their DSnap modes and
timestamped based on when the backup set under them completed.  The *.{#}*
suffix indicates the backup's age (.2 is older than .1, etc), eg:

	$ ls {target-dir}
	dsnap.version snap.1 snap.2 daily.1 daily.2 ... weekly.1 weekly.2 ...

## Backup Sets

Folders:

	{target-dir}/snap.1/{label}/   ...or...
	{target-dir}/snap.1/{source-dir}/

All entries that share a common *{target-dir}* (including the default) are
processed as a group and called a "backup set."  If a backup for any
entry in a set fails, the whole set fails.  Backups intended to complete
even when others fail should each have their own *{target-dir}*.

Version 1 target sets contain the complete filesystem hierarchy mirroring
*{source-dir}*.  Version 2 sets contain a folder for each *{label}* containing
the hierarchy starting beneath *{source-dir}* (and can be significantly easier
to use).

	$ ls {target-dir}/snap.1
	home-directories source-code media-archive

Version 2 is required to use the **ALLOW\_RELATIVE** option (see
[Customization](#customization)).

## Remote Snap

When dsnap is run with the "-r" option, init and snap work differently.
DSnap will duplicate a backup set from a remote host to the {target-dir}.
{source-dir} in the backup file must take the form remote:{remote-dir}
where {remote-dir} should be target-dir on the remote host, and the
snap.1 directory should exist there.  The {label} is used for
choosing hooks and filters, but is not used as snap.1 will be
duplicated as a complete set.  If only part of the remote snap.1 is
desired, then a filter may be used to select directories/files.

To modify the remote host, ssh options etc, see [Customization](#customization).

For stricter security, **REMOTE_HOST** and **REMOTE_LOCK_HOST** can be different,
and use a unique IdentityFile in ssh_config, and then remote
authorized_keys may be configured as:

	restrict,command="/path/to/rrsync -ro /target/dir" <dsnap-host-key>
	restrict,command="/path/to/dsnap -c /local/config :remote" <dsnap-lock-key>

If you're running DSnap as root, you may also wish to add --numeric-ids
to **DEFAULT_RSYNC_OPTIONS** if local host doesn't use the same user-id
mapping as the remote host.

## Target Locks

Files:

	{target-dir}/dsnap.lock/owner

Before new backup sets are created, or history managed, each target is
locked (the directory "dsnap.lock" created).  Locks are held until the
backup set completes, and history rotation completes (or either fails).
Stale locks are detected and removed by checking lock ownership
("dsnap.lock/owner") or when the lock is older than the
**LOCK_TIMEOUT_HOURS** threshold (see [Customization](#customization)).

The special "unlock" mode attempts to break all existing locks without
checking if they're in use, and is intended to run at system boot (for
example, in rc.local).  When "unlock" is used during boot, scheduled backups
should be run with the "-s" (sleep) parameter to avoid a backup starting
before "unlock" finishes removing all the stale locks.

## Hooks

Files:

	/etc/dsnap/pre-backup(*) [TARGET]...
	/etc/dsnap/post-backup(*) [TARGET]...

(see **HOOKDIR** in [Customization](#customization) to specify a different
location)

Hooks are useful for mount/unmount, or any other housekeeping.  The
"pre-backup" hook is run before backups start, the "post-backup" after they
complete.  If pre-backup fails, the backup will be cancelled unless
pre-backup exits with 24 (in which case backups continue, but DSnap exits
with 24 after it completes).

Hooks receive the same TARGET parameters passed to DSnap (empty if all
targets are being processed).  Also, all the [Customization](#customization)
variables are in the environment (see below), and the current backup mode
is available in $MODE.

> **NOTE:** No locks are held before the hooks are run, so it's possible
> (for example) a pre-backup may be run twice before a post-backup is run if
> backups or rotation are slow enough.  However, all post-backups are run if 
> any pre-backups are attempted.

## Customization

File:

	/etc/dsnap/config

Variables are assigned as '{NAME}={value}' - one per line (quotes optional)
(use the "-c" command-line option to specify a different location)

Global customization and defaults overrides can made in the config file.
The following variables may be set:

### File location configuration

- **DEFAULT_TARGET_DIR** (default: "/opt/backup") - default target if
        not set in "backups"

- **ETCDIR** (default: "/etc/dsnap") - Location of "backups" and "filters"
        (also may be set in DSnap environment)

- **HOOKDIR** (default: **ETCDIR**) - Location of "pre-backup" and
        "post-backup"

- **BACKUPS** (default: **ETCDIR**/backups) - Location of backup list

- **FILTERS** (default: **ETCDIR**/filters) - rsync filter prefix

- **LOGFILE** (default: "/var/log/dsnap/dsnap.log") - Path of rsync log,
        empty/unset to disable rsync logging

- **DEBUGFILE** (optional) - Path of debug log

- **RSYNC_EXE** (default: rsync in $PATH) - Full path of rsync

- **NICE_EXE** (default: nice in $PATH) - Full path of nice

- **IONICE_EXE** (default: ionice in $PATH) - Full path of ionice

- **SSH_EXE** (default: ssh in $PATH) - Full path of ssh (used with "-r")

### Feature configuration

- **DEFAULT_RSYNC_OPTIONS** (default: "-aHS") - Default rsync options.
	If using "-r" you may want to include "--numeric-ids" (requires
	running as root, see "man rsync")

- **MAX_PER_DAY** (default: 2) - Maximum number of "snap" backups to keep.
        (Minimum is 2 so that a full backup is available as snap.1
        after the "daily" mode)

- **MAX_YEARS** (default: 5) Maximum number of yearly backups to keep.
        (Minimum is 1, or disabled by simply not running "yearly" modes)

- **ALLOW_RELATIVE** (default: "no") - Set to "yes" to allow source/target
        directories in "backup" to be relative to the current directory
        when DSnap is run (ie without leading '/').  A relative
        *{target-dir}* requires "dsnap.version" 2, and an effective
        *{label}* not containing '/'

- **LOCK_TIMEOUT_HOURS** (default 24) - Remove target locks even if owner
        appears to exist if lockfile is older than this many hours
        (to avoid possible stale locks).  Requires the utility "find" in
        the path.  Empty/unset to disable timeouts.

- **STARTUP_DELAY** (default 120) - Startup delay when using "-s" (sleep) on
        the command line (suggested for cron entries when using "unlock"
        during system boot, see [Target Locks](#target-locks))

- **REMOTE_HOST** (default: "dsnap-remote") - Remote ssh host config
        for rsync when "-r" is used (see "man ssh")

- **REMOTE_LOCK_HOST** (default: "dsnap-remote") - Remote ssh host config
        for remote dsnap when "-r" is used (so remote dsnap can run hooks
        and aquire locks)

- **SSH_OPTIONS** (optional) - Any options for ssh to **REMOTE_LOCK_HOST**

- **REMOTE_SNAP_EXE** (default: "dsnap") - Location of dsnap on
        **REMOTE_LOCK_HOST** (used with "-r")

- **REMOTE_SNAP_OPTIONS** (optional) - Options to remote dsnap on
        **REMOTE_LOCK_HOST** (used with "-r")

## Automation

To automate incremental backups, add the following to your crontab (see "man
crontab").  DSnap does not need to be run as the root user, but it does need
read access too all the files it is configured to backup.  The timings below
assume incremental backups will complete in less than 5hrs, and removal of
expired backups in less than 15 mins:

	0  */12 * * * root /usr/local/sbin/dsnap -s snap
	45 5    * * * root /usr/local/sbin/dsnap -s daily
	30 5    * * 0 root /usr/local/sbin/dsnap -s weekly
	15 5    1 * * root /usr/local/sbin/dsnap -s monthly
	0  5    1 1 * root /usr/local/sbin/dsnap -s yearly

