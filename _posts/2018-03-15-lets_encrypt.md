---
title:  Let's encrypt for Apache vhost running on FreeBSD. 
last_modified_at: 2018-06-26 13:20:21 +0200
---

The following is a quick scratch down of how I have configured Let's encrypt on one of the FreeBSD jails I'm hosting (running Apache24).

All of the commands below have been run as root.

1. Start with installing https://github.com/Neilpang/acme.sh using the command:

    ```sh
    curl https://get.acme.sh | sh
    ```

    1.1 Set HOST to your current hostname

      ```sh
      HOST=example.com
      ```

1. Configure Apache with the following entries:

    ```apache
    # /usr/local/etc/apache24/httpd.conf
    Alias /.well-known/acme-challenge/ /usr/local/www/.well-known/acme-challenge/
    <Directory "/usr/local/www/.well-known/acme-challenge/">
        Options None
        AllowOverride None
        ForceType text/plain
        Require all granted
    </Directory>

    # /usr/local/etc/apache24/extra/httpd-ssl.conf
    SSLCertificateFile "/usr/local/etc/apache24/ssl-cert/example.com.cer"
    SSLCertificateKeyFile "/usr/local/etc/apache24/ssl-cert/example.com.key"
    SSLCertificateChainFile "/usr/local/etc/apache24/ssl-cert/fullchain.cer"
    ```

1. Create initial certificates:

    ```sh
    VHOSTS="/usr/local/www/vhosts/"
    mkdir -p /usr/local/www/.well-known/acme-challenge/

    # Enumerate all vhosts.
    hosts="-d ${HOST}"
    for host in `ls ${VHOSTS}|grep -e [a-z0-9]`; do 
      hosts=`echo "${hosts} -d ${host}.$HOST"`
    done

    .acme.sh/acme.sh  --force --issue $hosts --webroot /usr/local/www \
      --cert-file /usr/local/etc/apache24/ssl-cert/${HOST}.cer \
      --key-file /usr/local/etc/apache24/ssl-cert/${HOST}.key \
      --fullchain-file /usr/local/etc/apache24/ssl-cert/fullchain.cer \
      --reloadcmd "service apache24 reload"
    ```

    Hopefully you now have a working site with TLS/SSL certificates.

1. Create a `findVHosts.sh` script to update vhosts automatically and update config file.

    ```sh
    cat <<\EOF | sed s/example.com/$HOST/ >~/.acme.sh/$HOST/findVHosts.sh
    #!/usr/local/bin/bash

    HOST=example.com
    VHOSTS="/usr/local/www/vhosts"

    newHosts=$(find $VHOSTS -d 1 -type d| sed "s#$VHOSTS/##"|awk "{if (\$0 ~ /[a-z0-9]/) printf \$0\".$HOST,\"}"|sed 's/,$//')

    if `grep -q "Le_Alt='$newHosts'" /root/.acme.sh/$HOST/$HOST.conf`; then
      exit 0
    fi

    # Update "Le_Alt" with new certs.
    sed -i bak "s/^Le_Alt.*/Le_Alt=\'$newHosts\'/" /root/.acme.sh/$HOST/$HOST.conf

    "/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" --force

    exit 1
    EOF

    chmod +x ~/.acme.sh/$HOST/findVHosts.sh

    # Update the config-file to run `findVHosts.sh` as pre-hook.
    sed -i .bak "s#^Le_PreHook.*#Le_PreHook='/root/.acme.sh/$HOST/findVHosts.sh'#" /root/.acme.sh/$HOST/$HOST.conf 
    ```

1. Enjoy :).
