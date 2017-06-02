The configuration file can be typically found at `/etc/ssh/sshd_config`.

Be advised that any change in the SSH settings of your servers might cause problems connecting to the server, so every time you change any settings, make sure you have a second open connection to the server. OpenSSH will apply these settings for all new connections, but connections made before the configuration change will persist, which will allow you to revert changes in case something doesn't work correctly.

Almost all settings here will require a restart of the SSH daemon. To do so in Linux distributions running systemd, you can issue the command `$ systemctl restart sshd`. Otherwise, consult your distribution's documentation.

Based on the suggestions below, there is a sample SSHd configuration file. This is only a suggestion and should not be copied directly, but used as a resource to writing your own one.

# Security

## Port

It's generally considered a good idea to change the default SSH server port. Most of the scripts trying to perform brute-force attacks will only look for port 22/TCP. I like to set my servers on port 666/TCP. To do so, we must comment the default line (in case we want to reenable it again at a later point in time) and add the appropriate line:

```
#Port 22
Port 666
```

## Encryption

OpenSSH ships by default with RSA, DSA, ECDSA and ED25519 keys. Because DSA is not considered a good option and ECDSA is an NSA implementation, the best choice is to only allow ED25519 keys. This does come with a drawback, though. Some SSH clients don't yet support ED25519, but a 4096 RSA bit key should suffice, so, to disable only DSA and ECDSA and leave RSA for backwards compatibility purposes, comment (or delete) the respective lines, like this:

```
HostKey /etc/ssh/ssh_host_rsa_key
#HostKey /etc/ssh/ssh_host_dsa_key
#HostKey /etc/ssh/ssh_host_ecdsa_key
HostKey /etc/ssh/ssh_host_ed25519_key
```

You can also comment the RSA key if you're sure you're not going to need it.

To generate a 4096 bit RSA host key (there's no real benefit in larger keys), we run:

```
$ ssh-keygen -f /etc/ssh/ssh_host_rsa_key -N -b 4096 -t rsa
```

Now, to generate a 16384 bit ED25519 host key (the maximum size), we run:

```
$ ssh-keygen -f /etc/ssh/ssh_host_ed25519_key -N -b 16384 -t ed25519
```

Additionally, you can add the following lines that will exclude some old algorithms and use an optimised list that will prioritise the safest ones:

```
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes256-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512,hmac-sha2-256
KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group-exchange-sha256,diffie-hellman-group14-sha1
```

To fetch the supported algorithms for a particular OpenSSH version, issue the following commands:

```
$ ssh -Q cipher
$ ssh -Q cipher-auth
$ ssh -Q mac
$ ssh -Q kex
$ ssh -Q key
```

## Password authentication

Since passwords can be sniffed from a keyboard input or brute-forced, I recommend you disable password authentication **(REMEMBER TO SET UP AN SSH KEY OR YOU WILL BE LOCKED OUT OF THE SYSTEM)**. To do so change the line from `yes` to `no`, as follows:

```
PasswordAuthentication no
```

## Permit Root Login

It's generally a better idea to log-in via a standard user and then elevate privileges using said account, either by means of `sudo` or simply by issuing `su -c root` to obtain a root shell, in case `sudo` is not installed and you know the root password. To disable root log-in, we change the line to `no`, like this:

```
PermitRootLogin no
```

(Note: There are other options, apart from `no` for this setting. `yes` simply allows root to log-in. `without-password` means users will only be able to log-in to the local root account through means other than passwords, like SSH key file authentication. `forced-commands-only` enables root only when a command has been specified. This might be useful for backups, even if root login is not allowed otherwise).

## X11 Forwarding

For headless servers (machines without a display attached) which are not expected to be used directly, there is no graphical server running (X11, X.org, etc), so we can disable a feature of SSH that allows to connect to machine remotely in graphical mode. For this, we need to add the line:

```
X11Forwarding no
```

## Max Startups

To prevent server overload, there's a directive called `MaxStartups`. It is typically set to `10:30:60`, works as follows:

- 10: Number of unauthenticated connections accepted at once.
- 30: Percentage chance of dropping connections once the first limit has been passed.
- 60: Maximum number of simultaneous connections accepted.

"Unauthenticated" means a connection made to the server before the authentication stage. For example: If a connection is made successfully and the SSH server asks the user for its password, the connection is considered unauthenticated until the user inputs the correct password and hits the return key for the server to verify.

You can play around with values. I find the following values work very well for me, but if you have a large SSH server, it might not suit you:

```
MaxStartups 2:50:20
```

## Listen Address (optional)

If you have multiple network interfaces, each with its own IP, you might want to listen on a single IP only. For example, to listen on IP 192.168.1.54:

```
ListenAddress 192.168.1.54
```

## Allow groups (optional)

If you have multiple users on your Linux machine, you might not want to provide them all with remote access. For example, you might want to have a user for development purposes only.

For this, we can take advantage of Linux user groups. We can add users to a group called `sshusers`. From now on, only the users in this group will be allowed to log-in remotely:

First, we create the group:

```
$ groupadd sshusers
```

Second, we add the (existing) user to the group (this procedure can be repeated for as many users as we want to add):

```
$ usermod -a -G sshusers <YOUR_USER_HERE>
```

Finally, we add the following line to the SSH configuration file:

```
AllowGroups sshusers
```

## Two-factor authentication (optional)

The following methods combine two authentication mechanisms for increased protection.

### SSH key pair + server-side Password

This method verifies the SSH key, and then asks the client to enter the account password. **Important:** This method requires password authentication to be enabled, so, if you disabled it as per the above recommendations, you'll need to change it to `yes` again.

To enable it, add the following line:

```
AuthenticationMethods publickey,password
```

### Either SSH keys or passwords + TOTP (also known as Google Authenticator)

Follow the guide [here](https://www.linux.com/blog/securing-ssh-two-factor-authentication-using-google-authenticator).

## Privilege separation

This will use kernel sandbox mechanism where possible in unprivileged processes. To do so, add the following line.

```
UsePrivilegeSeparation sandbox
```

## SFTP

In older versions of SSH, there was a separate SFTP server. It's still maintained as the default for retrocompatibility purposes, but the newer alternative is to use `internal-sftp` (as opposed to `/usr/lib/openssh/sftp-server`. To do so, add the line:

```
Subsystem sftp internal-sftp
```

# Other settings

## Timing out

There are two settings that will help time out broken connections. If on an unstable network, or the Internet connection unexpectedly drops, the SSH socket will stay open for a while until it realises the connection is dead and proceeds to close the socket:

```
ClientAliveInterval 10
ClientAliveCountMax 3
```

The first setting controls how often the SSH server sends control packets to the client. In this case, every 10 seconds. The second setting controls how many "rounds" the client has to fail to respond before the connection is considered dead and killed, in this case 3, so 3*10 = 30 seconds. This timer only counts for actual broken connections, so for example you can leave a terminal open and running, as long as the SSH server is reachable, the Keep-Alive packets will reach the server and the connection will not be terminated.

## Setting an information banner

We can make SSH display a banner before authentication, this banner can contain a dissuasory message, or legal information. Both unauthorized and authorized users will get this banner. For this, we edit the contents of a text file (the default is /etc/issue/net) and add the following line:

```
Banner /etc/issue.net
```

# TODO

- Fail2ban
- Port knocking
- Explain TOTP
