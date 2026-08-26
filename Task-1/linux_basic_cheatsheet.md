Linux Basic Commands

## File System Navigation

### pwd
Prints where you currently are in the file system. First thing I run when I open a terminal and forget what directory I left off in.

```
pwd
```

Output looks like `/home/user/projects` - that's your current absolute path.

### cd
Change directory. Obvious one but the shortcuts are worth remembering:

```
cd /var/log        # go to an absolute path
cd projects         # go into a folder relative to where you are
cd ..                # go up one level
cd ../..             # up two levels
cd ~                 # jump straight to your home directory
cd -                 # jump back to whatever directory you were just in (this one's underrated)
```

`cd -` has saved me a bunch of times when I'm bouncing between two folders.

### ls
Lists what's in a directory.

```
ls              # basic listing
ls -l           # long format - permissions, owner, size, date
ls -a           # show hidden files too (the ones starting with a dot)
ls -la          # combine both, this is the one I actually use 90% of the time
ls -lh          # long format but with human readable sizes (K, M, G instead of raw bytes)
```

Honestly just alias `ll` to `ls -la` in your bashrc, everyone ends up doing this eventually.

---

## File & Directory Permissions

Every file has permissions for three groups: the owner, the group, and everyone else. Each of those can have read (r), write (w), execute (x).

Run `ls -l` on a file and you'll see something like:

```
-rwxr-xr--  1 user group  1024 Jan 5 10:00 script.sh
```

Breaking that down: first char is file type (- for regular file, d for directory). Then three sets of three - owner perms, group perms, other perms.

### chmod
Changes permissions.

```
chmod 755 script.sh      # owner: read/write/execute, group: read/execute, others: read/execute
chmod 644 file.txt        # owner: read/write, group: read, others: read
chmod +x script.sh        # just add execute permission, don't touch the rest
chmod -w file.txt         # remove write permission for everyone
chmod u+x script.sh       # add execute just for the user/owner
```

The number system is just adding up read(4) + write(2) + execute(1) for each of the three groups. So 755 = 7 (4+2+1, full perms) for owner, 5 (4+1, read+execute) for group, 5 for others. Took me way too long to actually internalize this instead of just memorizing common combos.

### chown
Changes who owns the file.

```
chown user file.txt              # change the owner
chown user:group file.txt        # change owner AND group at the same time
chown -R user:group folder/      # recursive - apply to a whole folder and everything inside it
```

Usually need sudo for this one unless you already own the file and are just changing group membership you're part of.

---

## Package Management

This section assumes Debian/Ubuntu since that's what apt is for. RPM-based distros use yum/dnf instead but that's a different topic.

### apt
The higher-level tool, handles dependencies for you.

```
sudo apt update                    # refresh the list of available packages, always run this first
sudo apt upgrade                   # actually upgrade installed packages to newer versions
sudo apt install package-name      # install something
sudo apt remove package-name       # uninstall but leave config files behind
sudo apt purge package-name        # uninstall AND wipe config files
sudo apt autoremove                # clean up packages that got installed as dependencies but nothing needs them anymore
apt search keyword                 # search for a package by name/description
apt list --installed               # see everything currently installed
```

`apt update` doesn't actually upgrade anything, it just refreshes apt's local cache of what versions exist. People confuse this with `upgrade` all the time (I definitely did when I started).

### dpkg
Lower level than apt, works directly with `.deb` package files, doesn't resolve dependencies for you.

```
sudo dpkg -i package.deb      # install a local .deb file
dpkg -l                        # list all installed packages
dpkg -L package-name           # list all files a package installed
sudo dpkg -r package-name      # remove a package
dpkg -s package-name           # show status/info about a package
```

If you download a `.deb` file off some website (not from a repo), dpkg is what you install it with. If it complains about missing dependencies afterward, run `sudo apt install -f` to fix that.

---

## Networking Commands

### ifconfig
Shows network interfaces and their info (IP address, MAC address, etc). Technically deprecated in favor of `ip addr` on newer systems but still around on a lot of machines and still what people say out loud.

```
ifconfig                # show all interfaces
ifconfig eth0            # show just one interface
```

If it's not installed, `ip addr show` or `ip a` is the modern replacement.

### ping
Checks if a host is reachable, and roughly how long it takes to get a response.

```
ping google.com          # keeps pinging until you Ctrl+C
ping -c 4 google.com      # send exactly 4 pings and stop
```

Good first troubleshooting step for "is the network even working" before digging into anything more complicated.

### netstat
Shows network connections, listening ports, routing tables. Also technically old (replaced by `ss` these days) but still everywhere.

```
netstat -tuln      # show listening TCP/UDP ports, numeric (don't resolve hostnames, faster)
netstat -a          # show all connections and listening ports
netstat -r          # show the routing table
```

I use this constantly for "what's listening on port 8080" type questions. `ss -tuln` does the same thing if netstat isn't installed.

### traceroute
Shows the path packets take to reach a destination, hop by hop, and how long each hop takes.

```
traceroute google.com
```

Useful when ping works but something still feels slow - traceroute can show you where along the path the delay is actually happening. On some systems it's `tracepath` instead if traceroute isn't installed by default.


