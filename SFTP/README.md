# Linux SFTP Server

1. SSH File Transfer Protocol
2. Encrypted file transfers
3. Authentication using passwords or SSH keys
4. Unlike traditional FTP, SFTP does not require a separate FTP server such as `vsftpd` or `ProFTPD`. An OpenSSH server is sufficient.
5. SFTP uses the same port as SSH - `22`.
6. FTP uses port - `23`.

For this project, I'm going to use two CentOS Stream 9 servers on Oracle VirtualBox. One is the SFTP server, and another one is the client.

`/images/server1.png`

`images/server2.png`

## Installing OpenSSH Server on Server One

For our CentOS server:

```bash
sudo dnf install openssh-server
sudo systemctl enable sshd
sudo systemctl status sshd
```

Check if port `22` is listening:

```bash
ss -tulnp | grep :22
```

`images/status.png`

We can also check whether SFTP is working by connecting to localhost using the command:

```bash
sftp <user>@localhost
```

## SFTP Configuration

This is the SFTP configuration file:

```bash
sudo vi /etc/ssh/sshd_config
```

Look for this line, based on the distro:

```text
Subsystem sftp /usr/libexec/openssh/sftp-server
```

or

```text
Subsystem sftp internal-sftp
```

`internal-sftp` is particularly convenient for chrooted SFTP users.

The `internal-sftp` implementation runs inside `sshd`.

**Note:** Before changing this file, make a backup.

## Create a Dedicated SFTP Group

```bash
sudo groupadd sftpusers
```

## Create an SFTP User

```bash
sudo useradd -g sftpusers -s /sbin/nologin john
sudo passwd john
```

`images/groupanduser.png`

`-s /sbin/nologin` prevents the user from obtaining a normal interactive SSH shell. The user can still use SFTP.

## Create the SFTP Directory

```bash
sudo mkdir -p /sftp/john/upload
```

## Set Ownership and Permissions

The user should be allowed to access the `upload` directory only.

```bash
sudo chown root:root /sftp
sudo chown root:root /sftp/john

sudo chmod 755 /sftp
sudo chmod 755 /sftp/john

sudo chown john:sftpusers /sftp/john/upload
sudo chmod 755 /sftp/john/upload
```

`images/ownerandpermission.png`

## Configure SSH for SFTP-Only Access

Add the following at the end of the configuration file:

```text
Match Group sftpusers
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    X11Forwarding no
    AllowTcpForwarding no
```

### Match Group sftpusers

Everything underneath applies to users belonging to the `sftpusers` group.

### ChrootDirectory /sftp/%u

`%u` is replaced by the username.

For `john`, `/sftp/%u` becomes `/sftp/john`, so `john` can only see the `upload/` directory. He does not see the real server's `/etc/`, `/var/`, etc.

### ForceCommand internal-sftp

This forces the user to use SFTP. Even if the user attempts:

```bash
ssh john@server
```

they won't receive a normal shell.

### X11Forwarding no

Disable X11 forwarding. It is not necessary for SFTP.

### AllowTcpForwarding no

Disable TCP forwarding. This prevents the account from being used as an SSH tunnel.

## Validate the SSH Configuration

```bash
sudo sshd -t
```

If successful, there is no output.

## Restart SSH

```bash
sudo systemctl restart sshd
sudo systemctl status sshd
```

## Configure the Firewall

Check whether `firewalld` is running:

```bash
sudo firewall-cmd --state
```

Allow SSH:

```bash
sudo firewall-cmd --permanent --add-service=ssh
```

Reload:

```bash
sudo firewall-cmd --reload
```

Verify:

```bash
sudo firewall-cmd --list-services
```

You should see:

```text
ssh
```

## Test from the Linux Client

Get the SFTP server IP using:

```bash
ip a
```

On the client, try:

```bash
sftp john@192.168.1.11
```

You'll be prompted for the password.

After authentication:

```text
sftp>
```

Try `pwd`. You should see:

```text
Remote working directory: /
```

List the directory:

```bash
ls
```

You should see:

```text
upload
```

Enter the directory:

```bash
cd upload
```

`images/sftp-client.png`

## Upload a File

Create a file:

```bash
echo "Hello SFTP" > test.txt
```

Connect to the SFTP server:

```bash
sftp john@192.168.1.11
```

Enter the upload directory:

```bash
cd upload
```

Upload the file:

```bash
put test.txt
```

`images/sftp-upload.png`

Verify on the server:

```bash
ls -l /sftp/john/upload/
```

## Download a File

Download a file with a new name:

```bash
get test.txt downloaded.txt
```

`images/sftp-download.png`

## Other Useful Commands

```text
lpwd - show remote working directory
lls - list remote files
put file.txt - upload a file
get file.txt - download a file
rm file.txt - remove a file
rmdir directory - remove a directory
rename old.txt new.txt - rename a remote file
exit - disconnect
```

## Test SSH Shell Access

SSH shell access with `john` is blocked because of `ForceCommand internal-sftp`.

```bash
ssh john@192.168.1.100
```

`images/ssh-verification.png`

## Add Multiple SFTP Users

Suppose you want:

```text
john
mary
backup
vendor1
```

### Create Each User

```bash
sudo useradd -g sftpusers -s /sbin/nologin john
sudo useradd -g sftpusers -s /sbin/nologin mary
sudo useradd -g sftpusers -s /sbin/nologin backup
sudo useradd -g sftpusers -s /sbin/nologin vendor1
```

### Set Passwords if Required

```bash
sudo passwd john
sudo passwd mary
sudo passwd backup
sudo passwd vendor1
```

### Create Directories

```bash
sudo mkdir -p /sftp/john/upload
sudo mkdir -p /sftp/mary/upload
sudo mkdir -p /sftp/backup/upload
sudo mkdir -p /sftp/vendor1/upload
```

### Set Chroot Directory Ownership

```bash
sudo chown root:root /sftp/john
sudo chown root:root /sftp/mary
sudo chown root:root /sftp/backup
sudo chown root:root /sftp/vendor1
```

Then:

```bash
sudo chmod 755 /sftp/john
sudo chmod 755 /sftp/mary
sudo chmod 755 /sftp/backup
sudo chmod 755 /sftp/vendor1
```

### Set Upload Ownership

```bash
sudo chown john:sftpusers /sftp/john/upload
sudo chown mary:sftpusers /sftp/mary/upload
sudo chown backup:sftpusers /sftp/backup/upload
sudo chown vendor1:sftpusers /sftp/vendor1/upload
```

Now each user has their own isolated area:

```text
/sftp
├── john
│   └── upload
├── mary
│   └── upload
├── backup
│   └── upload
└── vendor1
    └── upload
```
