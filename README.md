# Ansible Bind DNS

## Description

A simple ansible role that will ensure the dns records for a particular domain are configured for you bind DNS server.

## How to use

1. Edit the inventory file and update the `dns` group to your DNS server running bind

```
[dns]
<deviceDNS.domain.com>
```

2. Modify the variables in the `{{ playbook_dir }}/bind9/vars/main.yml` file.  Here is an example configuration

```
---
# vars file for bind
domain:
  name: "example.com"
  cidr: "192.168.1.0/24"
  forward_only_records:
    - name: ns
      type: A
      ip: "192.168.1.83"
    - name: router
      type: A
      ip: "192.168.1.1"
  records:
    - name: "device1"
      type: A
      ip: "192.168.1.2"
```

3. run `$ ansible-playbook config-bind9.yml`

4. enter the username and password for the host machine you defined in the inventory file.  Ansible will attempt to ssh into that machine with the username prvided. Resulting in the following command

    `ssh {{ username }}@deviceDNS.domain.com`

    The password is used incase ansible needs elavated permissions to carry out a task. This means your DNS server will need to be configured to allow the above command to work with no password.

Which will produce the following bind configuration files, on the host you defined in the inventory file.

1. `/etc/bind/named.conf.options`

```
    options {
        directory "/var/cache/bind";
        listen-on { any; };
        // If there is a firewall between you and nameservers you want
        // to talk to, you may need to fix the firewall to allow multiple
        // ports to talk.  See http://www.kb.cert.org/vuls/id/800113

        // If your ISP provided one or more IP addresses for stable 
        // nameservers, you probably want to use them as forwarders.  
        // Uncomment the following block, and insert the addresses replacing 
        // the all-0's placeholder.

        forwarders {
            8.8.8.8;
            8.8.4.4;
        };

        //========================================================================
        // If BIND logs error messages about the root key being expired,
        // you will need to update your keys.  See https://www.isc.org/bind-keys
        //========================================================================
        dnssec-validation auto;
        listen-on-v6 { any; };
        allow-query { localhost; 192.168.1.0/24; };
        recursion yes;
    };

```

2. `/etc/bind/named.conf.local`

```
# forward dns zone
zone "example.com" {
    type master;
    file "/etc/bind/db.example.com";
};

# reverse dns zone
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.1";
};

```

3. `/etc/bind/db.example.com`

```
;
; BIND data file for example.com
;
$TTL    604800
@       IN      SOA     example.com. admin.example.com. (
                       1        ; Serial
                      604800   ; Refresh
                       86400   ; Retry
                     2419200   ; Expire
                      604800 ) ; Negative Cache TTL
;
@       IN      NS      ns.example.com.
ns   IN  A   192.168.1.83
router   IN  A   192.168.1.1
device1   IN  A   192.168.1.2
```

4. `/etc/bind/db.192.168.1`

```
;
; BIND reverse data for 192.168.1.0/24
;
$TTL    604800
@       IN      SOA     example.com. admin.example.com. (
                       1        ; Serial
                      604800   ; Refresh
                       86400   ; Retry
                     2419200   ; Expire
                      604800 ) ; Negative Cache TTL
;
@       IN      NS      ns.example.com.
```