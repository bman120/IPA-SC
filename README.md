# IdM CAC Authentication Guide - PCTE
## Overview
This guide walks through how to enable Smart Card authentication via DoD CAC.
It follows the Red hat IdM Guide for Configuration Smart Card authentication.

## Instructions

### Setup IdM Certificate Trust Store
1. Obtain latest DoD CA Bundle from DISA (cyber.mil) and transfer to IdM. Once on IdM create a directory called `SmartCard` in a privileged location, such as `~/root`. Convert the pcks7 bundle in PEM, using the DISA provided `README` for guidance if needed. Transfer the PEM formatted bundle to the the newly created directory; ensure your working directory is within this new directory context.

2. Verify no expired CAs or Intermediate CAs (iCA) are present.
> Below is a list of the expired CA/iCAs in the latest DISA released DoD Bundle (20220305)
```
Subject: C=US, O=U.S. Government, OU=DoD, OU=PKI, CN=DOD ID CA-43
Subject: C=US, O=U.S. Government, OU=DoD, OU=PKI, CN=DOD ID CA-42
Subject: C=US, O=U.S. Government, OU=DoD, OU=PKI, CN=DOD ID CA-41
Subject: C=US, O=U.S. Government, OU=DoD, OU=PKI, CN=DOD EMAIL CA-44
Subject: C=US, O=U.S. Government, OU=DoD, OU=PKI, CN=DOD EMAIL CA-43
Subject: C=US, O=U.S. Government, OU=DoD, OU=PKI, CN=DOD EMAIL CA-42
Subject: C=US, O=U.S. Government, OU=DoD, OU=PKI, CN=DOD EMAIL CA-41
Subject: C=US, O=U.S. Government, OU=DoD, OU=PKI, CN=DOD ID SW CA-38
Subject: C=US, O=U.S. Government, OU=DoD, OU=PKI, CN=DOD ID SW CA-37
```
You can check for expired certificates using the following command:
`openssl crl2pkcs7 -nocrl -certfile DoD_CAs.pem | openssl pkcs7 -print_certs -text -noout | grep -i " Not After :" -A 1 | grep '2021' -A 1`
For each returned result, remove the correlating certificate from the Bundle, otherwise the setup script will fail during certificate validation.

3. Elevate to root, generate a Kerberos ticket, then execute the following command to generate the setup script:
> **NOT VALIDATED** You may require an Admin level ticket to successfully execute

`ipa-advise config-server-for-smart-card-auth > config-server-for-smart-card-auth.sh`

> 4. "A" is only for IPA server instances that use Apache server version 2.4.36+

4. A) Disable TLS version 1.3 as IPA Smart Card Authentication does not currently support it by adding the following command to the `config-server-for-smart-card-auth.sh` just before restarting the Apache service:
```sh
# Disable TLS 1.3
sed -i 's/^#SSLProtocol/SSLProtocol/' ./ssl.conf
sed -i '/SSLProtocol/ s/$/ -TLSv1.3/' ./ssl.conf
# finally restart apache
systemctl restart httpd.service
```
4. Make script executable with `750 ./config-server-for-smart-card-auth.sh` and execute the script, ensuring you pass the DoD Cert bundle as an option for setup:
`config-server-for-smart-card-auth.sh DoD_CAs.pem`
If successful you should see the following messages at the end:
```sh
CA certificate successfully installed
The ipa-cacert-manage command was successful
Systemwide CA database updated.
Systemwide CA database updated.
The ipa-certupdate command was successful
```

5. Restart the Apache Server:
`systemctl restart httpd`

### Setup User Certificate Mapping
IdM provides multiple options for mapping a certificate to a user.
The efficient way to accomplish this is via the Certificate Issuer and Subject.
You can set the issuer and subject via both WebUI and CLI

#### WebUI Mapping
1. Login to IdM as an User Administrator
2. Select the user you wish to map
3. Select "Certificate Mapping"
4. Select "Subject and Issuer" radio button
5. Put the following into Issuer:
`C=US, O=US, O=U.S. Government, OU=DoD, OU=PKI`
6. Put the followin into Subject:
`CN=LAST.FIRST.M.EDIPI`
> All users with CAC should have provided this info for the GECOS mapping of the for SSO and F5 authentication

#### CLI Mapping
1. Login to IdM via console or SSH as an User Administrator
2. Add a mapping rule for a user with the following command:
`ipa user-add-certmapdata <username> --subject "CN=LAST.FIRST.M.EDIPI,OU=<status>,OU=PKI,OU=DoD,O=U.S. Government,C=US" --issuer "CN=<iCA>,C=US, O=U.S. Government, OU=DoD, OU=PKI"`
> `<status>` will likely be one of `CONTRACTOR`, `CIVILIAN`, `ARMY`, `NAVY`, `COAST GUARD`, `MARINES`, `AIR FORCE`
<br>`<iCA>` will be the intermediate CA CN, such as `CN=DOD ID CA-51`
