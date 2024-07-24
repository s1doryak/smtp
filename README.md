SMTP
====

Environment
-----------

    export USER=onegae
    export HOST=onegae.ga
    export SEND=eugenue@onegae.ga
    export DDNS=T2dXcVhQ...jIxMTE5NDg4

DDNS
----

Обновление IP-адреса:

    wget --no-check-certificate -O - https://freedns.afraid.org/dynamic/update.php?${TOKEN}

Сокращённая запись с переадресацией:

    wget -O - onegae.ml

Провайдеры: 

- <https://freedns.afraid.org/>
- <https://freenom.com/>


Планировщик `crontab -e`:

    PATH=/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/sbin:/usr/local/bin

    4,9,14,19,24,29,34,39,44,49,54,59 * * * * wget -O - onegae.ml >> /dev/null 2>&1

Requirements
------------

    echo "postfix postfix/main_mailer_type string Internet site" | sudo debconf-set-selections
    echo "postfix postfix/mailname string ${HOST}" | sudo debconf-set-selections

    sudo apt install postfix postfix-pcre mailutils opendkim opendkim-tools
    
    mkdir -p /var/spool/mail/${USER}
    chown -R ${USER}:mail /var/spool/mail/${USER}

OpenDKIM
--------

Подпись требуется для того, чтобы удостовериться, что сообщение отправлено с IP-адреса, который (как оказывается) соответствует доменой записи.

    sudo --user opendkim mkdir -p /etc/opendkim/${HOST}
    sudo --user opendkim chown -R opendkim:opendkim /etc/opendkim/${HOST}

ECC (not supported):

    sudo --user opendkim openssl genpkey -algorithm ed25519 -out /etc/opendkim/${HOST}/mail.private
    sudo --user opendkim openssl pkey -in /etc/opendkim/${HOST}/mail.private -pubout -out /etc/opendkim/${HOST}/mail.txt
    sudo --user opendkim openssl asn1parse -in /etc/opendkim/onegae.ga/public.pem -offset 12 -noout -out /dev/stdout | openssl base64

RSA:

    sudo --user opendkim opendkim-genkey --directory=/etc/opendkim/${HOST} --bits=2048 --domain=${HOST} --restrict --selector=mail
    sudo --user opendkim cat /etc/opendkim/${HOST}/mail.txt

Права файлов ключей:

    sudo --user opendkim chmod 0700 /etc/opendkim/${HOST}/
    sudo --user opendkim chmod 0600 /etc/opendkim/${HOST}/mail.private

Then:

    cat <<EOF | sudo tee /etc/opendkim.conf
OversignHeaders     From
Canonicalization    simple

LogWhy              yes
Syslog              yes

UMask               007
UserID              opendkim

PidFile             /run/opendkim/opendkim.pid
KeyFile             /etc/opendkim/${HOST}/mail.private
Domain              ${HOST}
Selector            mail
Socket              inet:8891@localhost
TrustAnchorFile     /usr/share/dns/root.key
EOF

Then:

    sudo opendkim-testkey -vvv

DNS records
-----------

    @ IN TXT v=spf1 +mx ~all

    _dmarc IN TXT v=DMARC1; p=none; ruf=mailto:${SEND};
    
    mail._domainkey	IN TXT "v=DKIM1; h=sha256; k=rsa; s=email; ... "

Postfix
-------

Config:

    echo "${HOST}" | sudo tee /etc/hostname
    echo "${HOST}" | sudo tee /etc/mailname

    sudo postconf -e myhostname="${HOST}"
    sudo postconf -e mydestination="${HOST}"
    sudo postconf -e milter_default_action="accept"
    sudo postconf -e milter_protocol="2"
    sudo postconf -e smtpd_milters="inet:localhost:8891"
    sudo postconf -e non_smtpd_milters="inet:localhost:8891"
    sudo postconf -e smtpd_banner="ESMTP"
    sudo postconf -e smtputf8_enable="no"
    sudo postconf -e mail_spool_directory="/var/spool/mail/"
    sudo postconf -e mailbox_command=""
    sudo postconf -e smtpd_recipient_restrictions="permit"
    sudo postconf -e smtpd_helo_restrictions="permit"

SSL
---

Generate SSL certificates:

    sudo mkdir /etc/postfix/ssl/

    sudo openssl ecparam -genkey -name prime256v1 \
        -out /etc/postfix/ssl/ec.pem

    sudo openssl req -new -sha256 -key /etc/postfix/ssl/ec.pem \
        -out /etc/postfix/ssl/csr.csr

    sudo openssl req -x509 -sha256 -days 365 -key /etc/postfix/ssl/ec.pem \
        -in /etc/postfix/ssl/csr.csr \
        -out /etc/postfix/ssl/certificate.pem

Then:

    sudo postconf -e smtpd_use_tls="yes"
    sudo postconf -e smtpd_tls_auth_only="yes"
    sudo postconf -e smtpd_tls_security_level="encrypt"
    sudo postconf -e smtpd_tls_key_file="/etc/postfix/ssl/ec.pem"
    sudo postconf -e smtpd_tls_cert_file="/etc/postfix/ssl/certificate.pem"
    sudo postconf -e smtpd_tls_loglevel="1"
    sudo postconf -e smtpd_tls_received_header="yes"
    sudo postconf -e smtpd_tls_session_cache_timeout="3600s"
    sudo postconf -e tls_random_source="dev:/dev/urandom"

Then:

    sudo postfix check

Run
---

Then:

    sudo service opendkim restart
    sudo service postfix restart

    systemctl status opendkim.service
    systemctl status postfix.service

Then:

    sudo tail -F /var/log/mail.log

Misc
----

Multiple Users:

    sudo postconf -e alias_maps="hash:/etc/aliases"
    sudo postconf -e alias_database="hash:/etc/aliases"

```
cat <<EOF | sudo tee /etc/aliases
eugenue: ${USER}
tatiana: ${USER}
EOF
```

    sudo newaliases

Multiple Domains and Domainless:

    sudo postconf -e virtual_alias_domains="*"
    sudo postconf -e virtual_alias_maps="hash:/etc/postfix/virtual"

    sudo postconf -e mydestination="kamenka.ml, ${HOST}, [$(curl ifconfig.co)]"

```
cat <<EOF | sudo tee /etc/postfix/virtual
13@kamenka.ml ${USER}
EOF
```

Then:

    sudo postmap /etc/postfix/virtual
    sudo postmap -q 13@kamenka.ml hash:/etc/postfix/virtual

Troubleshooting
---------------

Problem:

    postfix/postfix-script: warning: symlink leaves directory: /etc/postfix/./makedefs.out

Solution:

    rm /etc/postfix/makedefs.out
    ln /usr/share/postfix/makedefs.out /etc/postfix/makedefs.out


mutt
----

cat <<EOF | tee ~/.muttrc
set realname = "Танюшка"
set from = "tatiana@onegae.ga"
set folder = /var/spool/mail/$USER
set beep_new
set delete
set quit
set copy = no
set wait_key = no
unset confirmappend
EOF

