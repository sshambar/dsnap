# Useful Utils

This directory contains useful utilities.

## Contents

- **mount_hook** - DSnap hook that will manage dynamic mounts.
  See the file for documentation.
- **rsync_no_vanished** - simple rsync wrapper that removes warnings
  when files are removed during backups.  Place in $PATH and set
  RSYNC_EXE="rsync_no_vanished" in DSnap config.
