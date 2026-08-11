> These are diagnostic/recovery notes for a specific failure mode during a supported standalone Discourse Docker migration. They are not a replacement for the official migration or HTTPS documentation.

On the new VPS, running:
```
./launcher logs app | tail -n 200
``` 

and see 

```
[emerg] cannot load certificate "/shared/ssl/domain.tld.cer": PEM_read_bio_X509_AUX() failed (SSL: error:0480006C routines::no start line:Expecting: TRUSTED CERTIFICATE)nginx:
```

This indicates that nginx is failing because the certificate file does not contain valid PEM certificate data. In this migration, Docker was listening on ports 80 and 443, but nginx inside app could not start.

- - -

then if i run `ls -lah /var/discourse/shared/standalone/ssl/`

and see

```
total 16K
drwxr-xr-x  2 root root 4.0K Aug  7 15:00 .
drwxr-xr-x 12 root root 4.0K Aug  7 15:00 ..
-rw-r--r--  1 root root    0 Aug  7 16:06 domain.tld.cer
-rw-------  1 root root 3.2K Aug  7 16:06 domain.tld.key
-rw-r--r--  1 root root    0 Aug  7 16:06 domain.tld_ecc.cer
-rw-------  1 root root  227 Aug  7 16:06 domain.tld_ecc.key
```

In this migration, the next step was to preserve the failed SSL/Let’s Encrypt state and let a rebuild attempt to obtain fresh certificates. ( https://meta.discourse.org/t/lets-encrypt-empty-cer-files/135241 ). A normal rebuild with intact Let’s Encrypt state should not result in a new certificate being issued unnecessarily. However, Discourse does invoke `acme.sh --issue` for both RSA and ECDSA certificates during Let’s Encrypt configuration. `acme.sh` normally determines that an existing certificate does not need replacement. If Discourse cannot verify the certificate stored in `/shared/letsencrypt`, the current template explicitly retries issuance with `--force`.

so what you can do is

```
cd /var/discourse/shared/standalone
mv ssl ssl.failed-2026MMDD
```

then 

```
ls -ld letsencrypt
```

If that directory exists as well:

```
mv letsencrypt letsencrypt.failed-2026MMDD
```

then rebuild

```
cd /var/discourse
```

```
./launcher rebuild app
```

then run again 

```
ls -lh /var/discourse/shared/standalone/ssl/
```

and if you still get

```
total 8.0K

-rw-r--r-- 1 root root    0 Aug  7 16:18 domain.tld.cer

-rw------- 1 root root 3.2K Aug  7 16:18 domain.tld.key

-rw-r--r-- 1 root root    0 Aug  7 16:18 domain.tld\_ecc.cer

-rw------- 1 root root  227 Aug  7 16:18 domain.tld\_ecc.key
```

then run

```
cd /var/discourse
```
```
./launcher enter app
```

Stop the normal nginx process:

```
sv stop nginx
```

Start nginx with the temporary Let’s Encrypt configuration:

```
/usr/sbin/nginx -c /etc/nginx/letsencrypt.conf
```

Now request the **RSA certificate**:
```
LE_WORKING_DIR=/shared/letsencrypt DEBUG=1 /shared/letsencrypt/acme.sh --issue -d domain.tld -k 4096 -w /var/www/discourse/public
```
from https://meta.discourse.org/t/set-up-https-support-with-lets-encrypt/40709

- - -

Stop here and check the output.

you may see

```
"detail": "too many certificates (5) already issued for this exact set of identifiers in the last 168h0m0s, retry after 2026-08-08 20:50:14 UTC: see [https://letsencrypt.org/docs/rate-limits/#new-certificates-per-exact-set-of-identifiers](https://letsencrypt.org/docs/rate-limits/#new-certificates-per-exact-set-of-identifiers)"
```

in this case stop the temporary Let’s Encrypt nginx inside the `app` container:

```
/usr/sbin/nginx -c /etc/nginx/letsencrypt.conf -s stop
```

Then leave the container with `logout` or `exit`

- - -

## First recovery attempt: copy the existing certificate files

On the previous/old server, check its old certificates:

```
ls -lh /var/discourse/shared/standalone/ssl/
```

Then inspect the RSA certificate:

```
openssl x509 -in /var/discourse/shared/standalone/ssl/domain.tld.cer -noout -subject -issuer -dates
```

And the ECC certificate:

```
openssl x509 -in /var/discourse/shared/standalone/ssl/domain.tld_ecc.cer -noout -subject -issuer -dates
```

If those are non-zero and `notAfter` is still in the future, go to the new server:

```bash
cd /var/discourse
./launcher stop app
```

Now copy the RSA certificate:

```bash
scp root@old_server_ip:/var/discourse/shared/standalone/ssl/domain.tld.cer /var/discourse/shared/standalone/ssl/
```

RSA key:

```bash
scp root@old_server_ip:/var/discourse/shared/standalone/ssl/domain.tld.key /var/discourse/shared/standalone/ssl/
```

ECC certificate:

```bash
scp root@old_server_ip:/var/discourse/shared/standalone/ssl/domain.tld_ecc.cer /var/discourse/shared/standalone/ssl/
```

ECC key:

```
scp root@old_server_ip:/var/discourse/shared/standalone/ssl/domain.tld_ecc.key /var/discourse/shared/standalone/ssl/
```

Restore the expected ownership and permissions:

```
chown root:root /var/discourse/shared/standalone/ssl/domain.tld*
```
```
chmod 644 /var/discourse/shared/standalone/ssl/*.cer
```
```
chmod 600 /var/discourse/shared/standalone/ssl/*.key
```

Check they're now non-zero:

```
ls -lh /var/discourse/shared/standalone/ssl/
```

Then start the app:

```
cd /var/discourse
```
```
./launcher start app
```

- - -

## If copying `/ssl` alone does not survive an `app` start

You may find that starting `app` replaces the copied certificates with zero-byte files. That is what happened in this migration.

In that case, copying only `/ssl` is insufficient. The Discourse Let’s Encrypt configuration uses the persistent state in `/shared/letsencrypt` when managing the certificates installed into `/shared/ssl`.

The recovery that worked in this migration was therefore to copy the old server's working `/shared/letsencrypt` state and `/shared/ssl` directory together.

On the new server, stop `app`:

```bash
cd /var/discourse
./launcher stop app
```

Preserve the current failed SSL directory:

```bash
mv /var/discourse/shared/standalone/ssl /var/discourse/shared/standalone/ssl.failed-20260807-1636
```

Preserve the current failed Let's Encrypt state:

```bash
mv /var/discourse/shared/standalone/letsencrypt /var/discourse/shared/standalone/letsencrypt.failed-20260807-1636
```

Now copy the **entire working SSL directory from the old server**:

> Note: This documents the recovery procedure that worked on this particular migration. It involves copying existing TLS private keys and Let’s Encrypt state between servers, so both servers should be trusted and under your control. Preserve the existing destination directories before replacing them, and verify the certificates and HTTPS operation afterwards. This is not a replacement for the official Discourse migration or Let’s Encrypt documentation.

```bash
scp -rp root@old_server_ip:/var/discourse/shared/standalone/ssl /var/discourse/shared/standalone/
```

Then copy the **entire working Let's Encrypt directory from the old server**:

```bash
scp -rp root@old_server_ip:/var/discourse/shared/standalone/letsencrypt /var/discourse/shared/standalone/
```

Before starting anything, verify:

```bash
ls -lh /var/discourse/shared/standalone/ssl/
```

The certificate files should now be non-zero. In this migration they were approximately:

```text
5.9K  domain.tld.cer
3.2K  domain.tld.key
4.8K  domain.tld_ecc.cer
227B  domain.tld_ecc.key
```

Then start `app`:

```bash
cd /var/discourse
./launcher start app
```

Wait about 10 seconds and **immediately check whether it remained intact**:

```
ls -lh /var/discourse/shared/standalone/ssl/
```

If they remain non-zero, check the logs again:

```
cd /var/discourse
./launcher logs app | tail -n 200
```

The previous `PEM_read_bio_X509_AUX()` certificate-loading error should no longer appear.

Finally, verify HTTPS:

```
curl -I https://domain.tld
```

In this migration, after copying both the working `/ssl` and `/letsencrypt` directories from the old server, starting app no longer replaced the certificates with zero-byte files, and the instance continued operating normally.
