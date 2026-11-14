# Nextcloud Files Backup

One nightly, read-only rsync pull from the Nextcloud AIO server to a private
staging directory on a separate Linux host. Borg then stores an encrypted,
versioned archive in one dedicated repository. The job runs at 03:00 Berlin time
and catches up after downtime. Retention keeps all archives from the last seven
days plus daily, weekly and monthly history under Borg's retention rules.

This backs up the current contents of one account's `files/` directory, including
hidden files and empty directories. Older Borg archives can recover files
deleted or changed after this backup starts. Nextcloud's existing trash, version
history, shares, app state, database and configuration are outside the scope.
Recovery is to ordinary files or a freshly set-up Nextcloud account.

## 1. Confirm the source

[Borgstage](https://github.com/marcschlienger/borgstage) documents AIO with the
data directory at `/data/ncdata`, one primary user, server-side encryption
disabled and no external storage or group folders. Verify these on the live
host. Confirm the account ID and the AIO container's host mount, then compare
`/data/ncdata/ACCOUNT_ID/files` with the web UI.
The design assumes ordinary files on local storage, with no Nextcloud end-to-end
encryption for the folders being backed up. Rsync skips symlinks and special
files. Pulling a live directory can capture files from different moments during
the run. If the account setup changes, review coverage before relying on this job.

On the Linux backup host, install Bash, rsync, OpenSSH client, Borg **1.4.x** and
util-linux (`flock`), plus curl if you set `PING_URL`. Allocate room for one
complete staged copy. Choose a new offsite repository (the example uses Hetzner
Storage Box), or a local repository on separate storage. The Nextcloud server
needs a maintained rsync package including `rrsync`.

As root on the backup host, create two dedicated SSH keys (the second is only
for a remote repository):

```sh
install -d -m 0700 /root/.ssh
ssh-keygen -t ed25519 -N '' -f /root/.ssh/id_ed25519_nextcloud_files
ssh-keygen -t ed25519 -N '' -f /root/.ssh/id_ed25519_nextcloud_files_store
```

On the source, find the actual `rrsync` path with `command -v rrsync`. Add this
line to the authorized keys of the account that can read the whole source tree.
Replace the command path, account ID and public key. The example assumes the
forced-command root SSH setup documented in Borgstage is already in place.
Keep the existing SSH policy. A dedicated source account with sufficient
read/traverse permissions can also be used by changing `User` in the SSH alias.

```text
restrict,command="/usr/bin/rrsync -ro /data/ncdata/ACCOUNT_ID/files" ssh-ed25519 PUBLIC_KEY
```

Add the relevant entries from `ssh-config.example` to the backup host's
`/root/.ssh/config`, fill in the host names and verify and install the host keys
in `/root/.ssh/known_hosts` out of band. On the backup host, `ssh nextcloud-files-source id`
should be rejected by rrsync, while this should list the expected top-level files:

```sh
rsync --list-only -e ssh nextcloud-files-source:./
```

For Hetzner, enable SSH on the Storage Box, use a dedicated subaccount and install
the repository SSH public key on that account. Provider-side snapshots, when
available, protect against repository deletion using backup-host credentials.
See [Hetzner's SSH and Borg setup](https://docs.hetzner.com/storage/storage-box/access/access-ssh-rsync-borg/).

## 2. Install and run once

Run these commands as root on the backup host from this project directory.
The block is safe to repeat when upgrading: the last two commands refuse to
overwrite an existing config or passphrase and report `cannot overwrite existing
file`, which is expected.

```sh
install -m 0755 nextcloud-files-backup /usr/local/sbin/
install -m 0644 nextcloud-files-backup.service nextcloud-files-backup.timer /etc/systemd/system/
systemctl daemon-reload
(umask 077; set -C; cat nextcloud-files-backup.conf.example > /etc/nextcloud-files-backup.conf)
(umask 077; set -C; openssl rand -base64 32 > /etc/nextcloud-files-backup.passphrase)
```

Set the SSH alias and repository in the installed config. The remote path
`borg-1.4` is for Hetzner. For a different SSH host use its Borg 1.4 executable
name or remove the override if the default is correct. For a local repository,
use an absolute path outside the staging directory and remove the override.
Keep the passphrase somewhere accessible independently of the backup host.

```sh
set -a
. /etc/nextcloud-files-backup.conf
set +a
borg init --encryption=repokey-blake2 --make-parent-dirs "$BORG_REPO"
borg key export "$BORG_REPO" /root/nextcloud-files-borg-key
systemctl start nextcloud-files-backup.service
journalctl -u nextcloud-files-backup.service -n 80 --no-pager
borg list "$BORG_REPO"
```

Keep the exported key securely with your recovery materials. After the restore
test below succeeds, enable the timer:

```sh
systemctl enable --now nextcloud-files-backup.timer
systemctl list-timers nextcloud-files-backup.timer
```

An empty source stops the run before the pull, so the staging copy is kept. A
failed pull stops it before Borg; the one exception is rsync exit 24, which
only reports files that vanished from the live source mid-pull. A nonzero
Borg create exit stops retention. The lock prevents overlapping runs on the
same staging directory. If the tree is intentionally emptied, the job will
report failure and preserve the existing history. Inspect the service journal
after failures. Repeat the sample restore periodically and check the repository
with `borg check "$BORG_REPO"` from a shell with the config loaded.

Nothing in the job itself alerts you when a night fails. To get an alert, set
`PING_URL` in the config to the ping URL of a dead-man's-switch monitor, for
example a Healthchecks check. The job requests it once, after Borg has finished
successfully, so a failed, skipped or never-started run shows up as a missing
ping. Schedule the check for 03:00 Europe/Berlin with a grace time longer than a
normal run. The URL acts as a password: keep it in the config, not in the
repository. A failed ping is reported in the journal but does not fail the
backup; leave `PING_URL` unset to disable it.

## 3. Restore a sample, a folder, or everything

On the backup host or a recovery host with Borg 1.4, load the repo settings
(as above), or recreate them with your recovery credentials. From a new
private working directory, pick an archive from `borg list "$BORG_REPO"`
and inspect the stored paths:

```sh
restore_dir=$(mktemp -d)
cd "$restore_dir"
borg list "$BORG_REPO::ARCHIVE"
borg extract "$BORG_REPO::ARCHIVE" 'files/path/to/a/known-file.pdf'
```

Replace the example path with a real file and compare its contents or SHA-256
hash against the original. To extract everything, omit the file argument.
Copy the extracted files where you need them, or upload them through Nextcloud
to a fresh account or a recovery folder in your existing account. Test one
upload and open it from Nextcloud before restoring a large batch.

References: [Borg 1.4 quick start](https://borgbackup.readthedocs.io/en/1.4.0/quickstart.html),
[rrsync manual](https://download.samba.org/pub/rsync/rrsync.1).
