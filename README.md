*Note: While this documentation is written specifically for [CanSSOC](https://canssoc.ca/) threat intelligence feeds (TF), the instructions can easily be adjusted to work with any feeds, including your custom feeds, assuming they are in one of the formats supported by Juniper SRX. Please see [General Information](#gi) and [Other Feed Sources](#ofs) below.*

<p><br/></p>

# Juniper SRX Integration with CanSSOC Threat Intelligence Feeds

<p><br/></p>

## Table of Contents


* [General Information](#gi)
* [Configuration Example](#ce)
	* [Threat Feeds Ingestion](#tfi)
	* [Threat Feeds Use in Security Policies](#tfuisp)
* [More Information](#mi)
	* [Junos Resources](#jr)
	* [Other Feed Sources](#ofs)

<p><br/></p>

<a name="gi"></a>

## General Information

Juniper SRX supports the following types of IP addresses in feed files:

* Single IP. For example: `192.0.2.0`
* IP range. For example: `192.0.2.0-192.0.2.10`
* CIDR. For example: `192.0.2.0/24`

Each entry occupies one line. Starting in Junos OS Release 19.3R1, IP address ranges do not need to be sorted in ascending order and the value of the IP entries can overlap in the same feed. In Junos OS Releases before 19.3R1, IP address ranges need to be sorted in ascending order and the value of the IP entries cannot overlap in the same feed.

Example of CanSSOC TF that can be integrated with Juniper SRX:

```
https://SERVERNAME/feeds/canssoc_generic_ipv4
https://SERVERNAME/feeds/canssoc_phishing_ipv4
...

```

where:

* `SERVERNAME` is the name of CanSSOC MineMeld server

Juniper SRX integrates with CanSSOC TF via [dynamic address groups](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/ref/statement/edit-security-dynamic-address.html) in security policies.

The following is the minimum recommended `dynamic-address` stanza for integration with CanSSOC TF:

```
dynamic-address {
    feed-server name {
        description description;
        url url;
        feed-name name {
            description description;
            path path;
            update-interval seconds;
            hold-interval seconds;
        }
        update-interval seconds;
        hold-interval seconds;
    }

    address-name name {
        profile {
            feed-name name;
        }
    }
}
```

<p><br/></p>

<a name="ce"></a>

## Configuration Example

<a name="tfi"></a>

### Threat Feeds Ingestion

Use Juniper SRX CLI and enter its configuration mode by typing `configure`, and run the following set-commands:

**1. Define CanSSOC threat feeds server**

```
## CanSSOC TF Server
set security dynamic-address feed-server canssoc description "CanSSOC Threat Feeds" 
set security dynamic-address feed-server canssoc url https://LOGIN:PASSWORD@SERVERNAME
```

Replace `LOGIN` and `PASSWORD` with your CanSSOC MineMeld credentials, and replace `SERVERNAME` with the name of CanSSOC MineMeld server. Please notice the `:` character separating `LOGIN` and `PASSWORD`, and the `@` character separating credentials from `SERVERNAME`. 

Make sure that neither your `LOGIN` nor `PASSWORD` contain any of the following characters:

```
@ # : /
```

Credentials in the `url` definition won't be obfuscated in your Juniper SRX configuration, so make sure you maintain proper access control for your Juniper SRX configuration.

**2. Define threat feeds provided by CanSSOC threat feeds server**

```
## Generic IPv4 Feed
set security dynamic-address feed-server canssoc feed-name canssoc_generic_ipv4_tf description "Generic IPv4" 
set security dynamic-address feed-server canssoc feed-name canssoc_generic_ipv4_tf path /feeds/canssoc_generic_ipv4
set security dynamic-address feed-server canssoc feed-name canssoc_generic_ipv4_tf update-interval 3600 hold-interval 604800

## Phishing IPv4 Feed
set security dynamic-address feed-server canssoc feed-name canssoc_phishing_ipv4_tf description "Phishing IPv4" 
set security dynamic-address feed-server canssoc feed-name canssoc_phishing_ipv4_tf path /feeds/canssoc_phishing_ipv4
set security dynamic-address feed-server canssoc feed-name canssoc_phishing_ipv4_tf update-interval 3600 hold-interval 604800
```

In this example, each feed will update once an hour (`update-interval 3600`) and keep existing feed entries for one week (`hold-interval 604800`) when the update fails. You can adjust the interval values to match your requirements.

**3. Define AddressName-to-ThreatFeed mapping**

```
## Generic IPv4 Address Name
set security dynamic-address address-name canssoc_generic_ipv4 profile feed-name canssoc_generic_ipv4_tf

## Phishing IPv4 Address Name
set security dynamic-address address-name canssoc_phishing_ipv4 profile feed-name canssoc_phishing_ipv4_tf
```

**4. Review and commit new configuration changes**

While still in the configuration mode, run the following command:

```
show security dynamic-address
```

Your `dynamic-address` stanza should look similar to this:

```
feed-server canssoc {
    description "CanSSOC Threat Feeds";
    url "https://LOGIN:PASSWORD@SERVERNAME";
    feed-name canssoc_generic_ipv4_tf {
        description "Generic IPv4";
        path /feeds/canssoc_generic_ipv4;
        update-interval 3600;
        hold-interval 604800;
    }
    feed-name canssoc_phishing_ipv4_tf {
        description "Phishing IPv4";
        path /feeds/canssoc_phishing_ipv4;
        update-interval 3600;
        hold-interval 604800;
    }
}
address-name canssoc_generic_ipv4 {
    profile {
        feed-name canssoc_generic_ipv4_tf;
    }
}
address-name canssoc_phishing_ipv4 {
    profile {
        feed-name canssoc_phishing_ipv4_tf;
    }
}
```

Review and commit your changes, and exit the configuration mode. You can verify the status of the TF ingestion by running the following commands in the CLI operational mode:

```
show security dynamic-address summary
show security dynamic-address
```

With the size of CanSSOC TF, it can take a couple of seconds for the initial TF update.

To manually refresh your SRX with the latest feed data outside of the regular update interval, run the following command in the CLI operational mode:

```
request security dynamic-address update address-name DYNAMIC_ADDRESS_NAME
```
For example:

```
request security dynamic-address update address-name canssoc_generic_ipv4
```

<a name="tfuisp"></a>

### Threat Feeds Use in Security Policies

The dynamic address names mapped to TF can now be used in your firewall security policies. For example:

```
## Block & Log outbound traffic to canssoc_generic_ipv4 and canssoc_phishing_ipv4 addresses
set security policies from-zone Trusted to-zone Internet policy Block_Outbound match source-address any
set security policies from-zone Trusted to-zone Internet policy Block_Outbound match destination-address canssoc_generic_ipv4
set security policies from-zone Trusted to-zone Internet policy Block_Outbound match destination-address canssoc_phishing_ipv4
set security policies from-zone Trusted to-zone Internet policy Block_Outbound match application any
set security policies from-zone Trusted to-zone Internet policy Block_Outbound then reject
set security policies from-zone Trusted to-zone Internet policy Block_Outbound then log session-init

## Block & Log inbound traffic from canssoc_generic_ipv4 addresses
set security policies from-zone Internet to-zone Trusted policy Block_Inbound match source-address canssoc_generic_ipv4
set security policies from-zone Internet to-zone Trusted policy Block_Inbound match destination-address any
set security policies from-zone Internet to-zone Trusted policy Block_Inbound match application any
set security policies from-zone Internet to-zone Trusted policy Block_Inbound then deny
set security policies from-zone Internet to-zone Trusted policy Block_Inbound then log session-init
```

<p><br/></p>

<a name="mi"></a>

## More Information

<a name="jr"></a>

### Junos Resources

* [Security Policies User Guide for Security Devices: dynamic-address](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/ref/statement/edit-security-dynamic-address.html)
* [Logical Systems and Tenant Systems User Guide for Security Devices: dynamic-address](https://www.juniper.net/documentation/us/en/software/junos/logical-system-security/topics/ref/statement/dynamic-address.html)
* [Logical Systems and Tenant Systems User Guide for Security Devices: show security dynamic-address](https://www.juniper.net/documentation/us/en/software/junos/logical-system-security/topics/ref/command/show-security-dynamic-address.html)
* [Dynamic Address Groups in Security Policies](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/topic-map/security-policy-configuration.html#id-dynamic-address-groups-in-security-policies)

<a name="ofs"></a>

### Other Feed Sources

* [Atlassian IP Address Ranges](https://support.atlassian.com/organization-administration/docs/ip-addresses-and-domains-for-atlassian-cloud-products/) (additional steps required to download and process JSON data)
* [AWS IP Address Ranges](https://docs.aws.amazon.com/general/latest/gr/aws-ip-ranges.html) (additional steps required to download and process JSON data)
* [Azure IP Address Ranges](https://www.microsoft.com/en-us/download/details.aspx?id=56519) (additional steps required to download and process JSON data)
* [GeoIP Blocking with IPdeny](https://www.ipdeny.com/ipblocks/)
* [GitHub IP Address Ranges](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-githubs-ip-addresses) (additional steps required to download and process JSON data)
* [Google IP Address Ranges](https://support.google.com/a/answer/10026322) (additional steps required to download and process JSON data)
* [Office 365 IP Address Ranges](https://docs.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges?view=o365-worldwide) (additional steps required to download and process JSON data)
* [TOR Project Exit Node List](https://check.torproject.org/torbulkexitlist)
