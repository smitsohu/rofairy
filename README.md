# rofairy
rofairy is a small proof-of-concept tool, to showcase how administrative tools that require write permission can be used in a read-only root filesystem. Its primary purpose is to enable traditional package managers like `apt` to function when system directories like `/boot` or `/usr` are in a read-only state.

Unlike previous approaches,[^1][^2] rofairy allows the read-only restriction to be lifted only as necessary and only for the wrapped application, while all other processes in the system continue to operate within a read-only environment. This approach eliminates potential issues that arise when read-write filesystems are remounted back as read-only.

## Usage
Just prefix your application with `rofairy` and specify which directories should be writable and which directories should be read-only.

```
rofairy [options] application [application arguments|options]

Modify read-only restrictions per application.

Options:
 -r,  --readonly PATH      - remount PATH read-only
 -ra, --readonly-all PATH  - remount PATH read-only,
                             including all submounts
 -w,  --readwrite PATH     - remount PATH read-write
 -wa, --readwrite-all PATH - remount PATH read-write,
                             including all submounts
 -f,  --file FILE          - load instructions from FILE;
                             if path is relative
                             search FILE in /etc/rofairy.d
 -h,  --help               - display this help and exit
 -s,  --sync               - sync all filesystems that
                             were remounted read-write
```

### Example: Package management in Debian/Ubuntu
Create an executable script `/usr/local/bin/rofairy.dpkg` that wraps `dpkg`:

```
#!/bin/sh
exec /usr/local/bin/rofairy -wa /boot -wa /etc -wa /opt -wa /usr -ra /usr/local -s /usr/bin/dpkg "$@"
```

Create a new file `/etc/apt/apt.conf.d/90rofairy.dpkg.conf`:

```
Dir::Bin::dpkg "/usr/local/bin/rofairy.dpkg";
```

Now you can use `apt` in a read-only root filesystem just as usual.

## Installation
Download the script and make it executable:

```
DEST="/usr/local/bin/rofairy"
curl https://raw.githubusercontent.com/smitsohu/rofairy/main/bin/rofairy | sudo tee "$DEST"
sudo chmod 0755 "$DEST"
```

## How does this work
In Linux, filesystems are equipped with two read-only attributes: one associated with the mount point and the other with the superblock. The mount point attribute determines whether file modifications are allowed within a specific mount point. Conversely, the superblock attribute is a system-wide setting that impacts all mount points across all namespaces.

To enable write access to a file, both the mount point and superblock read-only flags must be unset. rofairy clears the superblock flag as needed and does not restore it. This way read-only access is controlled solely at the mount point level, allowing files to be read-only in the current mount namespace while becoming writable in a child mount namespace.

### But I want to restore the read-only superblock
Easy. Run `sudo mount -o remount,ro <any filesystem mountpoint>` after you are done.

## Limitations
Some limitations apply while rofairy or any of its descendants are running:

- Processes with `sys_admin` and `sys_chroot` capability can enter the writable mount namespace by utilizing `setns` system calls.
- A set of processes, typically limited to those with `sys_ptrace` capability, has permission to access the writable mount namespace by following the symbolic links `/proc/[pid]/root` or `/proc/[pid]/cwd`, or they might be able to gain access by issuing `ptrace` system calls. For detailed information, refer to the `ptrace` manual.

## References
[^1]: Debian Wiki: [ReadonlyRoot](https://web.archive.org/web/20230227222836/https://wiki.debian.org/ReadonlyRoot)
[^2]: Red Hat Customer Portal: [What is the impact of mounting /usr in read-only mode on RHEL](https://web.archive.org/web/20230701151203/https://access.redhat.com/solutions/2146081)
