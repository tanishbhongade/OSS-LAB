# NFS on Ubuntu: Install, Export, and Mount

This guide uses the generic `nfs` mount type (not `nfs4`). It covers a
localhost test and how to allow clients on your LAN later.

## 1. Install the NFS server

On the Ubuntu machine that will share the directory:

``` bash
sudo apt update
sudo apt install nfs-kernel-server nfs-common -y
```

## 2. Create a directory to share

``` bash
sudo mkdir -p /data/nfs-share
sudo chown nobody:nogroup /data/nfs-share
sudo chmod 755 /data/nfs-share
```

For a simple write test, the client must have write permission. You can
temporarily use:

``` bash
sudo chmod 777 /data/nfs-share
```

Use suitable ownership and permissions instead of `777` for a real
deployment.

## 3. Configure the export

Edit the exports file:

``` bash
sudo nano /etc/exports
```

For a localhost-only test, add:

``` text
/data/nfs-share 127.0.0.1(rw,sync,no_subtree_check)
```

Save the file, then apply the export:

``` bash
sudo exportfs -rav
sudo systemctl enable --now nfs-kernel-server
sudo systemctl status nfs-kernel-server
```

Check the exported share:

``` bash
sudo exportfs -v
```

## 4. Mount the share on the same machine

Create a mount point and mount the export using `nfs`:

``` bash
sudo mkdir -p /mnt/nfs-share
sudo mount -t nfs 127.0.0.1:/data/nfs-share /mnt/nfs-share
```

Verify the mount and test it:

``` bash
findmnt /mnt/nfs-share
touch /mnt/nfs-share/test.txt
ls -l /data/nfs-share
```

To unmount it:

``` bash
sudo umount /mnt/nfs-share
```

## 5. Extend access to your LAN later

Find the server's LAN IP:

``` bash
hostname -I
```

For example, if the server is `192.168.1.100` and your trusted LAN is
`192.168.1.0/24`, change the export line in `/etc/exports` to:

``` text
/data/nfs-share 192.168.1.0/24(rw,sync,no_subtree_check)
```

Apply the change:

``` bash
sudo exportfs -rav
```

If UFW is enabled, allow NFS from that LAN only:

``` bash
sudo ufw allow from 192.168.1.0/24 to any port 2049 proto tcp
```

On each Ubuntu client, install the client tools and mount the share:

``` bash
sudo apt update
sudo apt install nfs-common -y
sudo mkdir -p /mnt/nfs-share
sudo mount -t nfs 192.168.1.100:/data/nfs-share /mnt/nfs-share
```

Replace `192.168.1.100` with your server's actual LAN IP.

## 6. Optional: mount automatically at boot

On the client, add this line to `/etc/fstab`:

``` text
192.168.1.100:/data/nfs-share /mnt/nfs-share nfs defaults,_netdev 0 0
```

Test the entry:

``` bash
sudo mount -a
```

## Notes

-   `127.0.0.1` works only when the NFS client and server are the same
    machine.
-   For LAN access, export the share to your trusted client IP or
    subnet, not to everyone.
-   The generic `-t nfs` mount command negotiates an NFS version
    supported by both client and server. It is not the same as
    explicitly specifying `-t nfs4`.
-   If mounting fails, check `sudo exportfs -v`,
    `systemctl status nfs-kernel-server`, and connectivity to TCP port
    `2049`.
