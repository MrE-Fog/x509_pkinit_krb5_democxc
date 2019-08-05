# Why does this repo exist

I doubt this repo will see many changes to it.  Due to the number of
files, a repo seemed simpler then linking multiple gists together.

# What can this repo be used for

While x509 certificates replicate a lot of the same features as Kerberos, it
seems x509 is gaining popularity after GDPR. Not only is x509 in use at my
employer, but also Facebook who wrote a blog post about **Tupperware** and
why the KDC was no longer meeting their needs.

Some platforms like Hadoop **only** support Kerberos and having an integration
between the two systems is useful.

Using `docker-compose up --build -d` will start a single container having 
sample x509 certificates and a running krb5kdc process for the "EXAMPLE.COM"
realm. This is useful for testing a KDC that issues TGTs when presented with
x509 certificates is needed.

# What should this not be used for?

**This should not be used for anything remotely resembling production.**
There are better Docker files that bring up a stand-alone KDC that use
randomly generated passwords.  Not only does this Dockerfile use simple
hard-coded passwords like `abc123`. it doesn't bother protecting files on
disk with permissions or ownership.

# Usage 

After starting the container with docker-compose, one can `exec` a bash shell.

```
docker exec -it kdc /bin/bash
```

and then obtain a TGT for the user named **user1**.

```
kinit -V -X X509_user_identity=FILE:/var/tmp/certs/user1_certs/client.pem,/var/tmp/certs/user1_certs/clientkey.pem user1
```

Output will be verbose because the KRB5_TRACE shell variable is set.

```
unset KRB5_TRACE
```

will return verbosity to normal.

# Personal lessons learned

### OpenSSL

Working with OpenSSL gets easier the more you work with it.  While each submenu
is its own section in `man openssl`.  Online versions have them broken out
separately. 

x509 certificates require additional fields when integrating with kerberos. Keep
this in mind if integrating Kerberos with an existing x509 system and calculating
implementation time.

### Debugging

Setting the environment variable `KRB5_TRACE=/dev/stdout` is really helpful in 
knowing what the kerberos client is doing.  Be careful some outupt can be 
misleading.  It's not clear that retry logic is kicking in after the client
received a pre-auth required message.

```
[31] 1564981537.42655: Response was from master KDC
[31] 1564981537.42656: Received error from KDC: -1765328359/Additional pre-authentication required
[31] 1564981537.42659: Preauthenticating using KDC method data
[31] 1564981537.42660: Processing preauth types: PA-PK-AS-REQ (16), PA-PK-AS-REP_OLD (15), PA-PK-AS-REQ_OLD (14), PA-FX-FAST (136), PA-PKINIT-KX (147), PA-FX-COOKIE (133)
[31] 1564981537.42661: Received cookie: MIT
```

Using tcpdump with wireshark is somewhat helpful.  With it, you will find
messages like `error-code: eRR-PREAUTH-REQUIRED (25)` which resolve to 
meaniful Internet search results.  The numeric error codes are linked below.

Finally if one needs to break out strace, look for missing files. The following output
helped me realize I was missing the pkinit plugin for the krb5kdc process.

```
stat("/usr/lib64/krb5/plugins/preauth/pkinit.so", 0x7ffc68153540) = -1 ENOENT (No such file or directory)
```

### Keytab replacement

How useful this is for replacing keytabs is to be seen.  The private SSL key is
required when requesting a TGT.  For long running tasks that need to automatically
renew the TGT, the risks are similar to having a passwordless keytab.  

# References

[Redhat documentation on configuration Kerberos](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system-level_authentication_guide/configuring_a_kerberos_5_server)

[PKINIT walkthrough by MIT](https://web.mit.edu/kerberos/krb5-1.13/doc/admin/pkinit.html)

[krb.conf settings](https://web.mit.edu/kerberos/krb5-1.17/doc/admin/conf_files/krb5_conf.html)

[Kerberos Encryption Type Numbers](https://www.iana.org/assignments/kerberos-parameters/kerberos-parameters.xhtml)

[OpenSSL manual](https://www.openssl.org/docs/man1.0.2/man1/)

[OpenSSL cheat sheet](https://gist.github.com/davewongillies/7050080)

[Facebook Tupperware](https://code.fb.com/data-center-engineering/tupperware/)

