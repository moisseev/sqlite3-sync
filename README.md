# sqlite3-sync
Live SQLite3 database master-slave replication with sqlite3-rdiff using rsync over SSH

The program replicates a local (master) SQLite3 database to a remote one (slave).

## Requirements

On both local and remote sides:
- sqlite3-rdiff (and its dependencies)
- rsync

## Installation

Place the program on the `master` side. Create from the sample and tune your own configuration file:
```sh
> cp sqlite3-sync.conf.sample sqlite3-sync.conf
```
You will also need to create `cache` directories on both sides and configure passwordless SSH login.

Directory layouts example:

```sh
Local master:

/home/user/sqlite3-sync/
├── cache/
├── action*
├── sqlite3-sync*
└── sqlite3-sync.conf

Remote slave:

/home/user/sqlite3-sync/
└── cache/
```

### SSH passwordless login configuration

You are on the local host (master):

* Generate the host keys with the default key file path and an empty passphrase:
 ```sh
 > ssh-keygen
 ```

* Copy the public keys to a remote host's `~/.ssh/authorized_keys`
 ```sh
 > ssh-copy-id -i ~/.ssh/id_rsa.pub [-p port] remote.example.com
 ```

Configuration example:

```
> ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/username/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/username/.ssh/id_rsa.
Your public key has been saved in /home/username/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:L+kApuqWr6RqUzeMdHlhvsnoszBt1PNiy27ggGqfB70 username@local.example.com
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|       o         |
|      + .        |
|   . o.o         |
|  o B.+oS        |
| . BoO +oo       |
|..+o*o=oo..      |
|+B  =Eo+o.       |
|Xo=+..=+.        |
+----[SHA256]-----+
> ssh-copy-id -i ~/.ssh/id_rsa.pub -p 22222 remote.example.com
Password for username@remote.example.com:
```

## Usage

To start replication run the `sqlite3-sync` script.
