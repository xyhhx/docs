---
slug: provide-authoritative-dns-services-with-nsd-on-fedora-14
deprecated: true
author:
  name: Brett Kaplan
  email: docs@linode.com
description: 'This guide will show you to install and configure NSD, a lightweight and open-source name server to handle authoritative DNS queries on Fedora 14.'
keywords: ["NSD", "DNS", "resolving", "Fedora 14", "networking"]
tags: ["dns","networking","fedora","resolving"]
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
aliases: ['/networking/dns/provide-authoritative-dns-services-with-nsd-on-fedora-14/','/dns-guides/nsd-authoritative-dns-fedora-14/']
modified: 2013-09-25
modified_by:
  name: Linode
published: 2011-01-25
title: Provide Authoritative DNS Services with NSD on Fedora 14
relations:
    platform:
        key: authoritative-dns-nsd
        keywords:
            - distribution: Fedora 14
---

NSD is a lightweight yet full-featured open source name server daemon created to provide an alternative to BIND.

Before beginning, you should be familiar with basic [DNS terminology and records](/docs/guides/dns-overview/). You will also need to ensure that your current Linode plan has enough memory to run the NSD daemon. Use the developer's [memory usage calculator](http://www.nlnetlabs.nl/projects/nsd/nsd-memsize.html) to determine the memory requirement for your NSD deployment.

## Set the Hostname

Before you begin installing and configuring the components described in this guide, please make sure you've followed our instructions for [setting your hostname](/docs/guides/getting-started/#setting-the-hostname). Issue the following commands to make sure it is set properly:

    hostname
    hostname -f

## Install Required Software

Ensure that your package repositories are up to date and that you've installed all available software upgrades by issuing the following commands:

    yum update

Install NSD and configure the daemon to start on boot with the following commands:

    yum install nsd
    chkconfig nsd on

Proceed to configure the daemon.

## Configure NSD

### Configure NSD Service

Edit the `nsd.conf` file to configure the behavior of the NSD service and the hosted DNS zones. The NSD package provides an example configuration file located at `/etc/nsd/nsd.conf` that you may reference. Your file should resemble the following:

{{< file "/etc/nsd/nsd.conf" conf >}}
server:
:   logfile: "/var/log/nsd.log" username: nsd
{{< /file >}}

### Host Zones with NSD

You must specify at least one zone in the `/etc/nsd/nsd.conf` file before NSD will begin serving DNS records. Refer to the following example configuration for proper syntax.

{{< file "/etc/nsd/nsd.conf" conf >}}
zone:
:   name: example.com zonefile: /etc/nsd/example.com.zone

zone:
:   name: example.org zonefile: /etc/nsd/example.org.zone
{{< /file >}}

Once zones are added to the `nsd.conf` file, proceed to create a zone file for each DNS zone.

## Creating Zone Files

Each domain has zone file specified in the `nsd.conf` file. The syntax of an NSD zone file is similar BIND zone files. Refer to the example zone files that follow for syntax, and modify domain names and IP addresses to reflect the needs of your deployment.

{{< file "/etc/nsd/example.com.zone" >}}
$ORIGIN example.com. $TTL 86400

@ IN SOA ns1.example.com. admin.example.com. (
:   2010011801 ; serial number 28800 ; Refresh 7200 ; Retry 864000 ; Expire 86400 ; Min TTL )

NS ns1.example.com.
:   NS ns2.example.com.

    MX 10 mail.example.com.

ns1 IN A 11.22.33.44 ns2 IN A 22.33.44.55 www IN A 77.66.55.44 tomato IN A 77.66.55.44 mail IN A 88.77.66.55 \* IN A 77.66.55.44
{{< /file >}}

{{< file "/etc/nsd/example.org.zone" >}}
$ORIGIN example.org. $TTL 86400

@ IN SOA ns1.example.org. web-admin.example.org. (
:   2009011803 ; serial number 28800 ; Refresh 7200 ; Retry 864000 ; Expire 86400 ; Min TTL )

NS ns1.example.org.
:   NS ns2.example.org.

    MX 10 mail.example.org.

ns1 IN A 11.22.33.44 ns2 IN A 22.33.44.55 www IN A 44.33.22.11 paisano IN A 44.33.22.11 mail IN A 99.88.77.66

pizzapie IN CNAME paisano
{{< /file >}}

Rebuild the NSD database and restart the daemon with following command sequence:

    nsdc rebuild
    /etc/init.d/nsd restart

Test the configuration and functionality of the DNS serve using `dig`, which provides a [command line DNS client](/docs/guides/use-dig-to-perform-manual-dns-queries/). Issue the following command to test the DNS server:

    dig @localhost www.example.org

The output should resemble the following:

    ; <<>> DiG 9.6.1-P2 <<>> @localhost pizzapie.example.org
    ; (1 server found)
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25199
    ;; flags: qr aa rd; QUERY: 1, ANSWER: 2, ORGNAMEITY: 2, ADDITIONAL: 0
    ;; WARNING: recursion requested but not available

    ;; QUESTION SECTION:
    ;pizzapie.example.org.  IN  A

    ;; ANSWER SECTION:
    pizzapie.example.org. 86400 IN  CNAME   paisano.example.org.
    paisano.example.org. 86400  IN  A   44.33.22.11

    ;; ORGNAMEITY SECTION:
    example.org.    86400   IN  NS  ns1.example.org.
    example.org.    86400   IN  NS  ns2.example.org.

    ;; Query time: 18 msec

Congratulations, you have successfully installed NSD!

## Adjusting NSD for Low-Memory Situations

If you are running NSD in a low-memory environment, amending the values of the following directives in your `/etc/nsd/nsd.conf` file will lower your memory and system resource usage.

{{< file "/etc/nsd/nsd.conf" >}}
ip4-only: yes tcp-count: 10 server-count: 1
{{< /file >}}

## More Information

You may wish to consult the following resources for additional information on this topic. While these are provided in the hope that they will be useful, please note that we cannot vouch for the accuracy or timeliness of externally hosted materials.

- [NSD Homepage](http://nlnetlabs.nl/projects/nsd/)
- [NSD Memory Usage Calculator](http://nlnetlabs.nl/projects/nsd/nsd-memsize.html)



