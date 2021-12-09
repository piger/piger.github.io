---
title: YubiKey and GPG
date: 2018-03-26
summary: How to configure a YubiKey to be used as a GnuPG smart-card device.
tags:
- yubikey
- gpg
- gpg2
- ssh
- gpg-agent
categories:
- Tutorial
---

This guide explains how to store a GnuPG key into a YubiKey; specifically it will explain how to:

- create a "master secret key"
- create 3 subkeys: 1 used to sign, 1 used to encrypt and 1 used to authenticate (e.g. OpenSSH)
- store the subkeys on a YubiKey
- have the YubiKey act as a U2F device, a Smart Card device for GnuPG and an OTP device

**NOTE**: the secret keys stored on the YubiKey will be protected by a numeric PIN.

## Requirements

- YubiKey 3.0 or above
- gpg1 / gpg2

On OS X:

    brew install gpg pinentry-mac ykpers

Somehow `pinentry` does not work for me and I have to use `pinentry-mac`.

## References

- https://www.jfry.me/articles/2015/gpg-smartcard/

## Configuring the Yubikey

Configure `OTP/U2F/CCID` mode:

    $ ykpersonalize -m86
    Firmware version 3.4.3 Touch level 1541 Program sequence 1

    The USB mode will be set to: 0x86

    Commit? (y/n) [n]: y

Restart the YubiKey (unplug, insert back).

### Creating the GnuPG keys

#### Configure GnuPG

Ensure that you have a sane configuration for gpg (`~/.gnupg/gpg.conf`):

    no-emit-version
    no-comments
    keyid-format 0xlong
    with-fingerprint
    use-agent
    personal-cipher-preferences AES256 AES192 AES CAST5
    personal-digest-preferences SHA512 SHA384 SHA256 SHA224
    cert-digest-algo SHA512
    default-preference-list SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES CAST5 ZLIB BZIP2 ZIP Uncompressed

And for `gpg-agent` (`~/.gnupg/gpg-agent.conf`):

    use-standard-socket
    enable-ssh-support
    pinentry-program /usr/local/bin/pinentry-mac

**NOTE**: the path to `pinentry` could be different on your system.


### Create the first secret key

Create the master secret key:

    $ gpg --gen-key
    gpg (GnuPG) 1.4.19; Copyright (C) 2015 Free Software Foundation, Inc.
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.

    Please select what kind of key you want:
       (1) RSA and RSA (default)
       (2) DSA and Elgamal
       (3) DSA (sign only)
       (4) RSA (sign only)
    Your selection? 4
    RSA keys may be between 1024 and 4096 bits long.
    What keysize do you want? (2048) 4096
    Requested keysize is 4096 bits
    Please specify how long the key should be valid.
             0 = key does not expire
          <n>  = key expires in n days
          <n>w = key expires in n weeks
          <n>m = key expires in n months
          <n>y = key expires in n years
    Key is valid for? (0) 0
    Key does not expire at all
    Is this correct? (y/N) y

    You need a user ID to identify your key; the software constructs the user ID
    from the Real Name, Comment and Email Address in this form:
        "Heinrich Heine (Der Dichter) <heinrichh@duesseldorf.de>"

    Real name: Pippo Goofy
    Email address: pippo@example.com
    Comment:
    You selected this USER-ID:
        "Pippo Goofy <pippo@example.com>"

    Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
    You need a Passphrase to protect your secret key.

    gpg: problem with the agent - disabling agent use
    We need to generate a lot of random bytes. It is a good idea to perform
    some other action (type on the keyboard, move the mouse, utilize the
    disks) during the prime generation; this gives the random number
    generator a better chance to gain enough entropy.
    ............+++++
    ........................+++++
    gpg: key 0xAABBCCDDEEFFGGHH marked as ultimately trusted
    public and secret key created and signed.

    gpg: checking the trustdb
    gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
    gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
    pub   4096R/0xAABBCCDDEEFFGGHH 2015-10-27
          Key fingerprint = AAAA BBBB CCCC DDD EEEE  FFFF GGGG HHHH IIII LLLL
    uid                 [ultimate] Pippo Goofy <pippo@example.com>

    Note that this key cannot be used for encryption.  You may want to use
    the command "--edit-key" to generate a subkey for this purpose.

Save your key in a variable:

    $ export KEY=0xAABBCCDDEEFFGGHH

#### (Optional) Create a Keybase identity

If you plan to use [Keybase](https://keybase.io), add another *uid*:

    $ gpg --edit-key $KEY
    gpg (GnuPG) 1.4.19; Copyright (C) 2015 Free Software Foundation, Inc.
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.

    Secret key is available.

    pub  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never       usage: SC
                                   trust: ultimate      validity: ultimate
    [ultimate] (1). Pippo Goofy <pippo@example.com>

    gpg> adduid
    Real name: Pippo Goofy
    Email address: pgoofy@keybase.io
    Comment:
    You selected this USER-ID:
        "Pippo Goofy <pgoofy@keybase.io>"

    Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o

    You need a passphrase to unlock the secret key for
    user: "Pippo Goofy <pippo@example.com>"
    4096-bit RSA key, ID 0xAABBCCDDEEFFGGHH, created 2015-10-27

    pub  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never       usage: SC
                                   trust: ultimate      validity: ultimate
    [ultimate] (1)  Pippo Goofy <pippo@example.com>
    [ unknown] (2). Pippo Goofy <pgoofy@keybase.io>

    gpg> uid 1

    pub  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never       usage: SC
                                   trust: ultimate      validity: ultimate
    [ultimate] (1)* Pippo Goofy <pippo@example.com>
    [ unknown] (2). Pippo Goofy <pgoofy@keybase.io>

    gpg> primary

    You need a passphrase to unlock the secret key for
    user: "Pippo Goofy <pippo@example.com>"
    4096-bit RSA key, ID 0xAABBCCDDEEFFGGHH, created 2015-10-27


    pub  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never       usage: SC
                                   trust: ultimate      validity: ultimate
    [ultimate] (1)* Pippo Goofy <pippo@example.com>
    [ unknown] (2)  Pippo Goofy <pgoofy@keybase.io>

    gpg> save

### Create the 3 secret keys

Now generate 3 subkeys (1 to encrypt, 1 to sign, 1 for ssh):

    $ gpg --expert --edit-key $KEY
    gpg (GnuPG) 1.4.19; Copyright (C) 2015 Free Software Foundation, Inc.
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.

    Secret key is available.

    pub  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never       usage: SC
                                   trust: ultimate      validity: ultimate
    [ultimate] (1). Pippo Goofy <pippo@example.com>
    [ultimate] (2)  Pippo Goofy <pgoofy@keybase.io>

    gpg> addkey
    Key is protected.

    You need a passphrase to unlock the secret key for
    user: "Pippo Goofy <pippo@example.com>"
    4096-bit RSA key, ID 0xAABBCCDDEEFFGGHH, created 2015-10-27

    gpg: problem with the agent - disabling agent use
    Please select what kind of key you want:
       (3) DSA (sign only)
       (4) RSA (sign only)
       (5) Elgamal (encrypt only)
       (6) RSA (encrypt only)
       (7) DSA (set your own capabilities)
       (8) RSA (set your own capabilities)
    Your selection? 4
    RSA keys may be between 1024 and 4096 bits long.
    What keysize do you want? (2048) 2048
    Requested keysize is 2048 bits
    Please specify how long the key should be valid.
             0 = key does not expire
          <n>  = key expires in n days
          <n>w = key expires in n weeks
          <n>m = key expires in n months
          <n>y = key expires in n years
    Key is valid for? (0)
    Key does not expire at all
    Is this correct? (y/N) y
    Really create? (y/N) y
    We need to generate a lot of random bytes. It is a good idea to perform
    some other action (type on the keyboard, move the mouse, utilize the
    disks) during the prime generation; this gives the random number
    generator a better chance to gain enough entropy.
    .....+++++
    ...+++++

    pub  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never       usage: SC
                                   trust: ultimate      validity: ultimate
    sub  2048R/0x1122334455667788  created: 2015-10-27  expires: never       usage: S
    [ultimate] (1). Pippo Goofy <pippo@example.com>
    [ultimate] (2)  Pippo Goofy <pgoofy@keybase.io>

    gpg> addkey
    Key is protected.

    You need a passphrase to unlock the secret key for
    user: "Pippo Goofy <pippo@example.com>"
    4096-bit RSA key, ID 0xAABBCCDDEEFFGGHH, created 2015-10-27

    Please select what kind of key you want:
       (3) DSA (sign only)
       (4) RSA (sign only)
       (5) Elgamal (encrypt only)
       (6) RSA (encrypt only)
       (7) DSA (set your own capabilities)
       (8) RSA (set your own capabilities)
    Your selection? 6
    RSA keys may be between 1024 and 4096 bits long.
    What keysize do you want? (2048)
    Requested keysize is 2048 bits
    Please specify how long the key should be valid.
             0 = key does not expire
          <n>  = key expires in n days
          <n>w = key expires in n weeks
          <n>m = key expires in n months
          <n>y = key expires in n years
    Key is valid for? (0)
    Key does not expire at all
    Is this correct? (y/N) y
    Really create? (y/N) y
    We need to generate a lot of random bytes. It is a good idea to perform
    some other action (type on the keyboard, move the mouse, utilize the
    disks) during the prime generation; this gives the random number
    generator a better chance to gain enough entropy.
    ..+++++
    .........+++++

    pub  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never       usage: SC
                                   trust: ultimate      validity: ultimate
    sub  2048R/0x1122334455667788  created: 2015-10-27  expires: never       usage: S
    sub  2048R/0x1A2B3C4D5E6F7F8G  created: 2015-10-27  expires: never       usage: E
    [ultimate] (1). Pippo Goofy <pippo@example.com>
    [ultimate] (2)  Pippo Goofy <pgoofy@keybase.io>

    gpg> addkey
    Key is protected.

    You need a passphrase to unlock the secret key for
    user: "Pippo Goofy <pippo@example.com>"
    4096-bit RSA key, ID 0xAABBCCDDEEFFGGHH, created 2015-10-27

    Please select what kind of key you want:
       (3) DSA (sign only)
       (4) RSA (sign only)
       (5) Elgamal (encrypt only)
       (6) RSA (encrypt only)
       (7) DSA (set your own capabilities)
       (8) RSA (set your own capabilities)
    Your selection? 8

    Possible actions for a RSA key: Sign Encrypt Authenticate
    Current allowed actions: Sign Encrypt

       (S) Toggle the sign capability
       (E) Toggle the encrypt capability
       (A) Toggle the authenticate capability
       (Q) Finished

    Your selection? s

    Possible actions for a RSA key: Sign Encrypt Authenticate
    Current allowed actions: Encrypt

       (S) Toggle the sign capability
       (E) Toggle the encrypt capability
       (A) Toggle the authenticate capability
       (Q) Finished

    Your selection? e

    Possible actions for a RSA key: Sign Encrypt Authenticate
    Current allowed actions:

       (S) Toggle the sign capability
       (E) Toggle the encrypt capability
       (A) Toggle the authenticate capability
       (Q) Finished

    Your selection? a

    Possible actions for a RSA key: Sign Encrypt Authenticate
    Current allowed actions: Authenticate

       (S) Toggle the sign capability
       (E) Toggle the encrypt capability
       (A) Toggle the authenticate capability
       (Q) Finished

    Your selection? q
    RSA keys may be between 1024 and 4096 bits long.
    What keysize do you want? (2048)
    Requested keysize is 2048 bits
    Please specify how long the key should be valid.
             0 = key does not expire
          <n>  = key expires in n days
          <n>w = key expires in n weeks
          <n>m = key expires in n months
          <n>y = key expires in n years
    Key is valid for? (0)
    Key does not expire at all
    Is this correct? (y/N) y
    Really create? (y/N) y
    We need to generate a lot of random bytes. It is a good idea to perform
    some other action (type on the keyboard, move the mouse, utilize the
    disks) during the prime generation; this gives the random number
    generator a better chance to gain enough entropy.
    ....+++++
    ..+++++

    pub  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never       usage: SC
                                   trust: ultimate      validity: ultimate
    sub  2048R/0x1122334455667788  created: 2015-10-27  expires: never       usage: S
    sub  2048R/0x1A2B3C4D5E6F7F8G  created: 2015-10-27  expires: never       usage: E
    sub  2048R/0xRXSNIDWKKWXVXGXN  created: 2015-10-27  expires: never       usage: A
    [ultimate] (1). Pippo Goofy <pippo@example.com>
    [ultimate] (2)  Pippo Goofy <pgoofy@keybase.io>

    gpg> save

Export your public key:

    $ gpg -a --export $KEY > pubkey.asc

### Configure the PIN on the YubiKey

Personalize the YubiKey, for example to set the PIN:

    $ gpg2 --card-edit

    Application ID ...: D30973728497183717309737284971837
    Version ..........: 2.0
    Manufacturer .....: Yubico
    Serial number ....: 12345678
    Name of cardholder: [not set]
    Language prefs ...: [not set]
    Sex ..............: unspecified
    URL of public key : [not set]
    Login data .......: [not set]
    Signature PIN ....: forced
    Key attributes ...: 2048R 2048R 2048R
    Max. PIN lengths .: 127 127 127
    PIN retry counter : 3 3 3
    Signature counter : 0
    Signature key ....: [none]
    Encryption key....: [none]
    Authentication key: [none]
    General key info..: [none]

    gpg/card> admin
    Admin commands are allowed

    gpg/card> passwd
    gpg: OpenPGP card no. D2760001240102000006038112920000 detected

    1 - change PIN
    2 - unblock PIN
    3 - change Admin PIN
    4 - set the Reset Code
    Q - quit

    Your selection? 1
    PIN changed.

    1 - change PIN
    2 - unblock PIN
    3 - change Admin PIN
    4 - set the Reset Code
    Q - quit

    Your selection? q

    gpg/card> url
    URL to retrieve public key: https://keybase.io/pgoofy/key.asc

    gpg/card> name
    Cardholder's surname: Goofy
    Cardholder's given name: Pippo

    gpg/card> login
    Login data (account name): pgoofy

    gpg/card> sex
    Sex ((M)ale, (F)emale or space): m

    gpg/card> lang
    Language preferences: en

    gpg/card> list

    Application ID ...: D30973728497183717309737284971837
    Version ..........: 2.0
    Manufacturer .....: Yubico
    Serial number ....: 12345678
    Name of cardholder: Pippo Goofy
    Language prefs ...: en
    Sex ..............: male
    URL of public key : https://keybase.io/pgoofy/key.asc
    Login data .......: pgoofy
    Signature PIN ....: forced
    Key attributes ...: 2048R 2048R 2048R
    Max. PIN lengths .: 127 127 127
    PIN retry counter : 3 3 3
    Signature counter : 0
    Signature key ....: [none]
    Encryption key....: [none]
    Authentication key: [none]
    General key info..: [none]

    gpg/card> quit

### Upload the secret keys to the YubiKey

**IMPORTANT** Backup your secret keys, because the next steps will delete them from the filesystem!

    $ tar -czf /media/BACKUP/gnupg.tgz ~/.gnupg
    $ gpg2 -a --export-secret-key $KEY >> /media/BACKUP/${KEY}.master.key
    $ gpg2 -a --export-secret-subkeys $KEY >> /media/BACKUP/${KEY}.sub.key

Now we upload the keys to the YubiKey while deleting them from the filesystem. Note: the `key`
command work like a *toggle*.

Note about the `keytocard` command:

> Transfer the selected secret subkey (or the primary key if no subkey has been selected) to a
> smartcard. The secret key in the keyring will be replaced by a stub if the key could be stored
> successfully on the card and you use the save command later. Only certain key types may be
> transferred to the card. A sub menu allows you to select on what card to store the key. Note that
> it is not possible to get that key back from the card - if the card gets broken your secret key
> will be lost unless you have a backup somewhere.

    $ gpg2 --edit-key $KEY
    gpg (GnuPG) 2.0.28; Copyright (C) 2015 Free Software Foundation, Inc.
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.

    Secret key is available.

    pub  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never       usage: SC
                                   trust: ultimate      validity: ultimate
    sub  2048R/0x1122334455667788  created: 2015-10-27  expires: never       usage: S
    sub  2048R/0x1A2B3C4D5E6F7F8G  created: 2015-10-27  expires: never       usage: E
    sub  2048R/0xRXSNIDWKKWXVXGXN  created: 2015-10-27  expires: never       usage: A
    [ultimate] (1). Pippo Goofy <pippo@example.com>
    [ultimate] (2)  Pippo Goofy <pgoofy@keybase.io>

    gpg> toggle

    sec  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never
    ssb  2048R/0x1122334455667788  created: 2015-10-27  expires: never
    ssb  2048R/0x1A2B3C4D5E6F7F8G  created: 2015-10-27  expires: never
    ssb  2048R/0xRXSNIDWKKWXVXGXN  created: 2015-10-27  expires: never
    (1)  Pippo Goofy <pippo@example.com>
    (2)  Pippo Goofy <pgoofy@keybase.io>

    gpg> key 1

    sec  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never
    ssb* 2048R/0x1122334455667788  created: 2015-10-27  expires: never
    ssb  2048R/0x1A2B3C4D5E6F7F8G  created: 2015-10-27  expires: never
    ssb  2048R/0xRXSNIDWKKWXVXGXN  created: 2015-10-27  expires: never
    (1)  Pippo Goofy <pippo@example.com>
    (2)  Pippo Goofy <pgoofy@keybase.io>

    gpg> keytocard
    Signature key ....: [none]
    Encryption key....: [none]
    Authentication key: [none]

    Please select where to store the key:
       (1) Signature key
       (3) Authentication key
    Your selection? 1

    You need a passphrase to unlock the secret key for
    user: "Pippo Goofy <pippo@example.com>"
    2048-bit RSA key, ID 0x1122334455667788, created 2015-10-27


    sec  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never
    ssb* 2048R/0x1122334455667788  created: 2015-10-27  expires: never
                         card-no: 0001 12345678
    ssb  2048R/0x1A2B3C4D5E6F7F8G  created: 2015-10-27  expires: never
    ssb  2048R/0xRXSNIDWKKWXVXGXN  created: 2015-10-27  expires: never
    (1)  Pippo Goofy <pippo@example.com>
    (2)  Pippo Goofy <pgoofy@keybase.io>

    gpg> key 1

    sec  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never
    ssb  2048R/0x1122334455667788  created: 2015-10-27  expires: never
                         card-no: 0001 12345678
    ssb  2048R/0x1A2B3C4D5E6F7F8G  created: 2015-10-27  expires: never
    ssb  2048R/0xRXSNIDWKKWXVXGXN  created: 2015-10-27  expires: never
    (1)  Pippo Goofy <pippo@example.com>
    (2)  Pippo Goofy <pgoofy@keybase.io>

    gpg> key 2

    sec  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never
    ssb  2048R/0x1122334455667788  created: 2015-10-27  expires: never
                         card-no: 0001 12345678
    ssb* 2048R/0x1A2B3C4D5E6F7F8G  created: 2015-10-27  expires: never
    ssb  2048R/0xRXSNIDWKKWXVXGXN  created: 2015-10-27  expires: never
    (1)  Pippo Goofy <pippo@example.com>
    (2)  Pippo Goofy <pgoofy@keybase.io>

    gpg> keytocard
    Signature key ....: 7AD1 CE4D E316 2B98 8EB2  98E1 9D29 E296 58EF 3A17
    Encryption key....: [none]
    Authentication key: [none]

    Please select where to store the key:
       (2) Encryption key
    Your selection? 2

    You need a passphrase to unlock the secret key for
    user: "Pippo Goofy <pippo@example.com>"
    2048-bit RSA key, ID 0x1A2B3C4D5E6F7F8G, created 2015-10-27


    sec  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never
    ssb  2048R/0x1122334455667788  created: 2015-10-27  expires: never
                         card-no: 0001 12345678
    ssb* 2048R/0x1A2B3C4D5E6F7F8G  created: 2015-10-27  expires: never
                         card-no: 0001 12345678
    ssb  2048R/0xRXSNIDWKKWXVXGXN  created: 2015-10-27  expires: never
    (1)  Pippo Goofy <pippo@example.com>
    (2)  Pippo Goofy <pgoofy@keybase.io>

    gpg> key 2

    sec  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never
    ssb  2048R/0x1122334455667788  created: 2015-10-27  expires: never
                         card-no: 0001 12345678
    ssb  2048R/0x1A2B3C4D5E6F7F8G  created: 2015-10-27  expires: never
                         card-no: 0001 12345678
    ssb  2048R/0xRXSNIDWKKWXVXGXN  created: 2015-10-27  expires: never
    (1)  Pippo Goofy <pippo@example.com>
    (2)  Pippo Goofy <pgoofy@keybase.io>

    gpg> key 3

    sec  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never
    ssb  2048R/0x1122334455667788  created: 2015-10-27  expires: never
                         card-no: 0001 12345678
    ssb  2048R/0x1A2B3C4D5E6F7F8G  created: 2015-10-27  expires: never
                         card-no: 0001 12345678
    ssb* 2048R/0xRXSNIDWKKWXVXGXN  created: 2015-10-27  expires: never
    (1)  Pippo Goofy <pippo@example.com>
    (2)  Pippo Goofy <pgoofy@keybase.io>

    gpg> keytocard
    Signature key ....: AAAA BBBB CCCC DDD EEEE  FFFF GGGG HHHH IIII LLL1
    Encryption key....: AAAA BBBB CCCC DDD EEEE  FFFF GGGG HHHH IIII LLL2
    Authentication key: [none]

    Please select where to store the key:
       (3) Authentication key
    Your selection? 3

    You need a passphrase to unlock the secret key for
    user: "Pippo Goofy <pippo@example.com>"
    2048-bit RSA key, ID 0xRXSNIDWKKWXVXGXN, created 2015-10-27


    sec  4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never
    ssb  2048R/0x1122334455667788  created: 2015-10-27  expires: never
                         card-no: 0001 12345678
    ssb  2048R/0x1A2B3C4D5E6F7F8G  created: 2015-10-27  expires: never
                         card-no: 0001 12345678
    ssb* 2048R/0xRXSNIDWKKWXVXGXN  created: 2015-10-27  expires: never
                         card-no: 0001 12345678
    (1)  Pippo Goofy <pippo@example.com>
    (2)  Pippo Goofy <pgoofy@keybase.io>

    gpg> save

If `gpg` seems stuck while trying to use the YubiKey, try to gently restart `gpg-agent`.

### Final check and 😌

Now you should see something like this:

    $ gpg2 --card-status
    Application ID ...: D30973728497183717309737284971837
    Version ..........: 2.0
    Manufacturer .....: Yubico
    Serial number ....: 12345678
    Name of cardholder: Pippo Goofy
    Language prefs ...: en
    Sex ..............: male
    URL of public key : https://keybase.io/pgoofy/key.asc
    Login data .......: pgoofy
    Signature PIN ....: forced
    Key attributes ...: 2048R 2048R 2048R
    Max. PIN lengths .: 127 127 127
    PIN retry counter : 3 3 3
    Signature counter : 0
    Signature key ....: AAAA BBBB CCCC DDD EEEE  FFFF GGGG HHHH IIII LLL1
          created ....: 2015-10-27 20:06:12
    Encryption key....: AAAA BBBB CCCC DDD EEEE  FFFF GGGG HHHH IIII LLL2
          created ....: 2015-10-27 20:07:01
    Authentication key: AAAA BBBB CCCC DDD EEEE  FFFF GGGG HHHH IIII LLL3
          created ....: 2015-10-27 20:07:37
    General key info..: pub  2048R/0x1122334455667788 2015-10-27 Pippo Goofy <pippo@example.com>
    sec   4096R/0xAABBCCDDEEFFGGHH  created: 2015-10-27  expires: never
    ssb>  2048R/0x1122334455667788  created: 2015-10-27  expires: never
                          card-no: 0001 12345678
    ssb>  2048R/0x1A2B3C4D5E6F7F8G  created: 2015-10-27  expires: never
                          card-no: 0001 12345678
    ssb>  2048R/0xRXSNIDWKKWXVXGXN  created: 2015-10-27  expires: never
                          card-no: 0001 12345678

To export your public SSH key, run:

    $ SSH_AUTH_SOCK=$HOME/.gnupg/S.gpg-agent.ssh ssh-add -L

and copy the output to `~/.ssh/yubikey.pub` or something similar.

With a sufficient recent version of OpenSSH you can also use this key only for some hosts, with
something like:

    Host example.com
      IdentityFile ~/.ssh/yubikey.pub
      IdentityAgent ~/.gnupg/S.gpg-agent.ssh
