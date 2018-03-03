The configuration file can typically be found at `~/.ssh/config`.

# Security

## Encryption

You can force your OpenSSH client to use only a list of algorithms. This disallows the use of older or insecure algorithms when connecting to servers, but this may also give problems when connecting to older servers which don't support newer encryption algorithms. For it, we add the following lines:

```
HostKeyAlgorithms ssh-ed25519-cert-v01@openssh.com,ssh-rsa-cert-v01@openssh.com,ssh-ed25519
 
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes256-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512,hmac-sha2-256
KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group-exchange-sha256,diffie-hellman-group14-sha1
```

### Key generation

To ensure maximum protection, we can generate large SSH keys. Because we only allowed the ED25519 and RSA algorithms, we don't need to generate keys with other algorithms, such as ECDSA or DSA.

Additionally, RSA and DSA keys of sizes equal to or lower than 1024 bits are considered weak for today's standards, and 2048 bits is the absolute minimum.

To generate a new RSA key:

```
ssh-keygen -t rsa -b 4096
```

Beyond 4096 there's no real increase in security, but there is an increase in computation time.

To generate a new ED25519 key:

```
ssh-keygen -t ed25519 -b 16384
```

16384 bits is the maximum length for an ED25519 key.

You can (and should) also generate multiple keys: One for each service or machine you use. For example, you can have one for GitHub, another one for your own server, a different one for your Raspberry Pi, etc. To do so, you can specify the location of the key by appending `-f ~/.ssh/<YOUR_DESIRED_PATH>` and `-C` option to append a comment. For example:

```
ssh-keygen -t ed25519 -b 16384 -f ~/.ssh/ed25519_github -C "SSH key for GitHub account PeterPan"
```

After the process is complete, you'll get two keys, one of which is finished in .pub. In the above example, that would be `~/.ssh/ed25519_github.pub`. This is the key you'll have to copy to other servers. The other key, with the same name but without the `.pub` extension is a private key and **should never leave your computer**.

To add the newly generated key, we run:

```
ssh-add ~/.ssh/<YOUR_DESIRED_PATH>
```

ED25519 keys will be at `~/.ssh/id_ed25519(.pub)` by default. RSA keys will be at `~/.ssh/id_rsa(.pub)`. If you have generated a key with a different path, it's a good idea to add it to your SSH agent by running the above command. However, keys saved in default locations will be picked up automatically.

SSH keys, if possible, must be protected by strong passwords. This encrypts the private key and ensures that, even if stolen, the server won't be accessable.

## Hash known hosts

This ensures the `KnownHosts` file is unreadable if leaked, which means attackers won't be able to use the keys stolen to connect to those servers. The solution is to add the following line:

```
HashKnownHosts yes
```

# TODO

- Agent forwarding (https://wiki.mozilla.org/Security/Guidelines/OpenSSH)
- TREZOR hardware key
- Kryptonite key
