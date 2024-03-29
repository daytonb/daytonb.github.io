---
layout: post
title: Getting TLS Certs for Services Running on IdM / FreeIPA Clients
---

# Getting TLS Certs for Services Running on IdM / FreeIPA Clients

## Problem Description

I have a set of Linux machines in an Identity Management (aka IdM or FreeIPA) domain.
On one of the clients I want to run multiple webservices with TLS certificates
These certificates should be trusted by all members of the IdM doamin and automatically renewed as necessary.
The entire IdM domain is air-gapped from the Internet, so Let's Encrypt is not really a viable solution.


## Related Materials

When I was trying to figure out how to accomplish this task I came across a blog titles "Generate certificate with SubjectAltName attributes in FreeIPA".
This has some of the elements of the solution I was seeking, but has a few aspects I did not like:
1. That solution force generated a few new "hosts" and "services" in FreeIPA for the alternative names which didn't feel like the _right_ way since these services are running on an existing host not on a new separate host.
2. That solution generates a Certificate Signing Request (CSR) using openssl which means that we'll have to remember to generate a new certificate when the old one expires.


## Solution

We will use certmonger to manage the TLS certificates with SubjectAltName (SAN) entries  for the webservices which will be signed by the IdM certificate authority (CA).


### Assumptions for This Example

For the example steps below, we'll assume the following:
* The host for the webservices is named `webservicehost.airgapped.example`
* We want to get a TLS cert for the service `internalwebapp1.airgapped.example`
* This example happened to be tested using an IdM server running on a Rocky Linux 8 with the webservices host running CentOS 7.


### Actions to Implement the Solution

1. Logged into the IdM web interface as an IdM admin, go to _Identity_ > _Services_ and click on _Add_.
2. In the _Add service_ window that pops up:
   * _Service_: select _HTTP_
   * _Host Name_: select _webservicehost.airgapped.example_
   * Click _Add and edit_
3. In the interface for editing the _HTTP/webservicehost.airgapped.example_ service:
   * In the _Pricipal alias_ area under _Service Settings_, click _Add_ to add a Kerberos pricipal alias.
4. In the _Add Kerberos Pricipal Alias_ window that pops ups:
   * _New kerberos pricipal alias_: `HTTP/internalwebapp1.airgapped.example`
5. On the commandline of `webservicehost.airgapped.example` run the following command and arguments:
   * `sudo ipa-getcert request \`
   * `-k /etc/pki/tls/private/https-internalwebapp1.airgapped.example.key \`
   * `-f /etc/pki/tls/certs/https-internalwebapp1.airgapped.example.pem \`
      * certmonger needs specific SELinux labels on the key and certificate files. The appropriate labels already exist in `/etc/pki/tls/certs/`, so this is the directory I chose for these files.
   * `-G RSA -g 4096 \`
      * Our cert will have a 4096 bit RSA key pair
   * `-K HTTP/webservicehost.airgapped.example \`
      * The Kerberos pricipal that the Kerberos keytab file on the _web service host_ has. Certmonger uses the keytab file to authenticate to IdM and prove its control of the system. Without the _HTTP/internalwebapp1.airgapped.example_ Kerberos pricipal alias though, certmonger couldn't find a keytab to get the CA to sign a TLS cert for `internalwebapp1.airgapped.example`.
   * `-D internalwebapp1.airgapped.example \`
      * SAN entry for our internal web app
   * `-I internalwebapp1 \`
      * a nickname for the certmonger request
   * `-v \`
      * Verbose output for the `ipa-cert request` command
   * `-w`
      * Wait for the request to complete before returning user to the command prompt.
   * In summary that whole command is:
      ```
      sudo ipa-getcert request \
         -k /etc/pki/tls/private/https-internalwebapp1.airgapped.example.key \
         -f /etc/pki/tls/certs/https-internalwebapp1.airgapped.example.pem \
         -G RSA -g 4096 \
         -K HTTP/webservicehost.airgapped.example \
         -D internalwebapp1.airgapped.example \
         -I internalwebapp1 \
         -v \
         -w
      ```


## Using The Certs

I won't cover these details since they're more specific to the app and the IP configurations, but now that we have the certs, we need to configure the app to know where the certificate and key files are.
We also have to ensure that there's a DNS entry for the web apps hostname so computers on the airgapped network can find the IP address for `internalwebapp1.airgapped.example`.
If the web app has the same IP address as the web server host, then this would be a CNAME record to point to the host's A record.
If the web app has its own IP address, the this would just be an A record of its own.


# Adding additional SAN entries later

## Problem Description

You may decide later that you need to add additional SAN entries to the TLS certificate created above.

## Solution

We will have certmonger _resubmit_ rather than _request_ in order to add an additional SubjectAltName (SAN) entry to the certificate created above.

### Assumptions for This Example

For the example steps below, we'll assume the following:
* The host for the webservices is named `webservicehost.airgapped.example`
* We want to get a TLS cert for the service `internalwebapp1.airgapped.example`
* We want to now add an additional SAN entry for the service `internalwebapp2.airgapped.example`
* This example happened to be tested using an IdM server running on a Rocky Linux 8 with the webservices host running CentOS 7.

### Actions to Implement the Solution

1. Logged into the IdM web interface as an IdM admin, go to _Identity_ > _Services_ and click on the _HTTP/webservicehost.airgapped.example_ entry.
3. In the interface for editing the _HTTP/webservicehost.airgapped.example_ service:
   * In the _Pricipal alias_ area under _Service Settings_, click _Add_ to add a Kerberos pricipal alias.
4. In the _Add Kerberos Pricipal Alias_ window that pops ups:
   * _New kerberos pricipal alias_: `HTTP/internalwebapp2.airgapped.example`
5. On the commandline of `webservicehost.airgapped.example` run the following command and arguments:
   * `sudo ipa-getcert resubmit \`
   * `-i internalwebapp1 \`
      * _the_ nickname we created in the original certmonger request
   * `-K HTTP/webservicehost.airgapped.example \`
      * The Kerberos pricipal that the Kerberos keytab file on the _web service host_ has. Certmonger uses the keytab file to authenticate to IdM and prove its control of the system. Without the _HTTP/internalwebapp1.airgapped.example_ and _HTTP/internalwebapp2.airgapped.example_ Kerberos pricipal aliases though, certmonger couldn't find keytabs to get the CA to sign a TLS cert for `internalwebapp1.airgapped.example` and `internalwebapp1.airgapped.example`.
   * `-D internalwebapp1.airgapped.example \`
      * SAN entry for our first internal web app
   * `-D internalwebapp2.airgapped.example \`
      * SAN entry for our additional internal web app
   * `-v \`
      * Verbose output for the `ipa-cert request` command
   * `-w`
      * Wait for the request to complete before returning user to the command prompt.
   * In summary that whole command is:
      ```
      sudo ipa-getcert resubmit \
         -i internalwebapp1 \
         -K HTTP/webservicehost.airgapped.example \
         -D internalwebapp1.airgapped.example \
         -D internalwebapp2.airgapped.example \
         -v \
         -w
      ```


## Using The Certs

I won't cover these details since they're more specific to the app and the IP configurations, but now that we have the certs, we need to configure the app to know where the certificate and key files are.
We also have to ensure that there's a DNS entry for the web apps hostname so computers on the airgapped network can find the IP address for the new web app `internalwebapp2.airgapped.example`.
If the web app has the same IP address as the web server host, then this would be a CNAME record to point to the host's A record.
If the web app has its own IP address, the this would just be an A record of its own.

Importantly, you will probably have to restart the webserver to pick up the changes to the certificate file. For example, on a Red Hat flavored linux this would be `sudo systemctl restart httpd.service`.
