# sqlite3-sync
Live SQLite3 database master-slave replication with [sqlite3-rdiff](https://github.com/moisseev/sqlite3-rdiff) using rsync over SSH

The program replicates a local (master) SQLite3 database to a remote one (slave) in bandwith-effective manner.

## Requirements

On both local and remote sides:
- [sqlite3-rdiff](https://github.com/moisseev/sqlite3-rdiff) (and its dependencies)
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

* generate the host keys with the default key file path and an empty passphrase:
 ```sh
 > ssh-keygen
 ```

* copy the public keys to a remote host's `~/.ssh/authorized_keys`
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

## Bandwith usage efficiency

Size of a signature file created by the [sqlite3-rdiff](https://github.com/moisseev/sqlite3-rdiff) utility is about `2.6%` of the slave database size. Rsync in conjunction with signature and delta files caching further reduces the transferring data volume. Total data amount transferred on the wire in both directions is about `0.6%` of the slave database size.

## A replication example

Master database sizes:
```diff
 > find /var/db/rspamd \( -name '*am.sqlite' -or -name '*.db' \) -exec du -shc {} +
 119M    /var/db/rspamd/bayes.ham.sqlite
  17M    /var/db/rspamd/fuzzy.db
 105M    /var/db/rspamd/bayes.spam.sqlite
-240M    total
```
Slave database sizes:
```diff
 > find /var/db/rspamd \( -name '*am.sqlite' -or -name '*.db' \) -exec du -shc {} +
  16M    /var/db/rspamd/fuzzy.db
 114M    /var/db/rspamd/bayes.ham.sqlite
 101M    /var/db/rspamd/bayes.spam.sqlite
-232M    total
```
```diff
 > ./sqlite3-sync
 
 [ ---- Slave ---- ] Creating signatures ...
 
 signature /var/db/rspamd/bayes.ham.sqlite bayes.ham.sqlite.sign --table-name % --rows-per-hash 18
 =1              tokenizer
 =1              users
 =1              languages
 =135764         tokens
 signature /var/db/rspamd/bayes.spam.sqlite bayes.spam.sqlite.sign --table-name % --rows-per-hash 18
 =1              tokenizer
 =1              users
 =1              languages
 =109859         tokens
 signature /var/db/rspamd/fuzzy.db fuzzy.db.sign --table-name % --rows-per-hash 18
 =1385           digests
 =14736          shingles
 =1              sources
 
 [ Master <- Slave ] Downloading signatures ...
 
 receiving incremental file list
 bayes.ham.sqlite.sign
-      3,301,376 100%    2.79MB/s    0:00:01 (xfr#1, to-chk=2/3)
 bayes.spam.sqlite.sign
-      2,674,688 100%  301.76kB/s    0:00:08 (xfr#2, to-chk=1/3)
 fuzzy.db.sign
-        401,408 100%  515.11kB/s    0:00:00 (xfr#3, to-chk=0/3)
 
+sent 24,291 bytes  received 1,303,001 bytes  115,416.70 bytes/sec
+total size is 6,377,472  speedup is 4.80
 
 [ --- Master ---  ] Creating deltas ...
 
 delta bayes.ham.sqlite.sign /var/db/rspamd/bayes.ham.sqlite bayes.ham.sqlite.delta --table-name %
 -0      +0      tokenizer
 -0      +0      users
 -1      +3      languages
 -1323   +28150  tokens
 delta bayes.spam.sqlite.sign /var/db/rspamd/bayes.spam.sqlite bayes.spam.sqlite.delta --table-name %
 -0      +0      tokenizer
 -0      +0      users
 -1      +4      languages
 -1146   +25352  tokens
 delta fuzzy.db.sign /var/db/rspamd/fuzzy.db fuzzy.db.delta --table-name %
 -6      +220    digests
 -174    +4728   shingles
 -1      +1      sources
 
 [ Master -> Slave ] Uploading deltas ...
 
 sending incremental file list
 bayes.ham.sqlite.delta
-        835,584 100%  312.09kB/s    0:00:02 (xfr#1, to-chk=2/3)
 bayes.spam.sqlite.delta
-        749,568 100%  205.50kB/s    0:00:03 (xfr#2, to-chk=1/3)
 fuzzy.db.delta
-        172,032 100%  166.34kB/s    0:00:01 (xfr#3, to-chk=0/3)
 
+sent 886,811 bytes  received 3,247 bytes  104,712.71 bytes/sec
+total size is 1,757,184  speedup is 1.97
 
 [ ---- Slave ---- ] Patching databases ...
 
 patch /var/db/rspamd/bayes.ham.sqlite bayes.ham.sqlite.delta /var/db/rspamd/bayes.ham.sqlite --table-name % --multimaster 0
 -0      +0      tokenizer (0 triggers disabled)
 -0      +0      users (0 triggers disabled)
 -1      +3      languages (0 triggers disabled)
 -1323   +28150  tokens (0 triggers disabled)
 patch /var/db/rspamd/bayes.spam.sqlite bayes.spam.sqlite.delta /var/db/rspamd/bayes.spam.sqlite --table-name % --multimaster 0
 -0      +0      tokenizer (0 triggers disabled)
 -0      +0      users (0 triggers disabled)
 -1      +4      languages (0 triggers disabled)
 -1146   +25352  tokens (0 triggers disabled)
 patch /var/db/rspamd/fuzzy.db fuzzy.db.delta /var/db/rspamd/fuzzy.db --table-name % --multimaster 0
 -6      +220    digests (0 triggers disabled)
 -174    +4728   shingles (0 triggers disabled)
 -1      +1      sources (0 triggers disabled)
 >
```