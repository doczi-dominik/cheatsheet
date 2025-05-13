
| Prefix | Subnet mask | Wildcard mask |
|--------|-------------|---------------|
| /16 | 255.255.0.0 | 0.0.255.255 |
| /24 | 255.255.255.0 | 0.0.0.255 | 
| /30 | 255.255.255.252 | 0.0.0.3 |


# 1. Configure Basic Device Settings

Hostname | *`(config)#` mode*
```
hostname <HOSTNAME_HERE>
```

Domain name | *`(config#)` mode*
```
ip domain-name <DOMAIN_NAME_HERE>
```

Interface IP addresses | *`(config)#` mode*
```
int <INTF_ID_HERE>
ip add <IPV4_HERE> <SUBNET_MASK_HERE>
exit
```

Access Passwords | *`(config)#` mode*
```
enable secret <PASSWORD_HERE>
```

## 1.1 OSPF

Loopback `Lo0` interface | *`(config)#` mode*
```
int loopback 0
ip add <IPV4_HERE> <SUBNET MASK HERE>
no sh
exit
```

Static default route / Default static route | *`(config)#` mode*
```
ip route 0.0.0.0 <INTF_NAME_OR_IPV4_HERE|example: `lo0`>
```

OSPF Dynamic Routing Protocol | *`(config)#` mode*
```
router ospf 1
network <NETWORK_ADDRESS_HERE> area <AREA_ID_HERE>
default-information originate
exit
```

# 2. SSH Server

> [!WARNING]
> Prerequisites: Hostname, Domain name

Local user / Privileged user | *`(config)#` mode*
```
username <USERNAME_HERE> privilege <PRIVILEGE_HERE|highest: 15> secret <PASSWORD_HERE>
```

RSA Key Generation | *`(config)#` mode*
```
crypto key generate rsa mod <RSA_MOD_HERE|example: 1024>
```

Enable SSH Version 2 | *`(config)#` mode*
```
ip ssh ver 2
```

SSH Timeout and Authentication Retries / Auth Retries | *`(config)# mode`*
```ruby
ip ssh time-out <TIMEOUT_IN_SECONDS|example: 60>
ip ssh authentication-retries <RETRY_COUNT|example: 2>
```

Named ACL / Access List / Access Control List for Restricted SSH Access | *`(config)# mode`*
```
ip access-list standard <ACL_NAME_OR_NUM_HERE|example: SSH_RESTRICT>
permit <NETWORK_ADDR_HERE> <SUBNET_MASK_HERE>
! Repeat above permit command here for multiple networks
deny any
exit
```

VTY Line Config | *`(config)# mode`*
> [!CAUTION]
> Configure an ACL for SSH restriction first OR remove the `access-class` command
```ruby
line vty <VTY_BEGIN_ID|example: 0> <VTY_END_ID|example: 15>
login local
trans in ssh
priv 15
access-class <ACL_NAME_OR_NUM_HERE|example: SSH_RESTRICT> in
exit
```

# 3. AAA RADIUS server

> [!CAUTION]
> Configure an username and password before touching any AAA stuff, or you
> will have to perform *password recovery*.

Enable AAA features | *`(config)# mode`*
```
aaa new-model
```

Connect to RADIUS server | *`(config)#` mode*
```ruby
radius-server host <SERVER_IP_HERE> key <KEY_HERE>
```

RADIUS with fallback to no authentication | *`(config)#` mode*
```ruby
aaa auth login default group radius none
```

RADIUS with Telnet access | *`(config)# mode`
```ruby
aaa auth login <NAME_HERE|example: TELNET_LINES> group radius

line vty <VTY_BEGIN_ID|example: 0> <VTY_END_ID|example: 15>
login auth <NAME_HERE|example: TELNET_LINES>
exit
```

# 4. Administrative Roles / Admin Roles
> [!WARNING]
> Prerequisites: Enable AAA features

Root view configuration | *`(config)#` mode*
```ruby
parser view root
secret <ROOT_SECRET_HERE>
enable view
exit
```

Config rule configuration | *`(config)#` mode*
```ruby
parser view <VIEW_NAME_HERE>
commands exec include <COMMAND_HERE>
! Repeat above commands line here to add multiple commands
exit
```

Assign views to local users / usernames | *`(config)#` mode*
```ruby
username <USERNAME_HERE> view <VIEW_NAME_HERE>
```


# 5. DHCP Server

Exclude an address from DHCP | *`(config)#` mode*
```ruby
ip dhcp excluded-address <IPV4_HERE>
```

Configure DHCP pool | *`(config)#` mode*
```ruby
ip dhcp pool <POOL_NAME>
network <NETWORK_ADDRESS_HERE> <SUBNET_MASK_HERE>
```

**Optional** DHCP pool parameters | *`(config-dhcp)#` mode*
```ruby
dns-server <DNS_SERVER_IP_HERE>
default-router <DEFAULT_GATEWAY_IP_HERE>
domain-name <DOMAIN_NAME_HERE>
```


DHCP Relay / Helper Address | *`(config-if)#` mode*
```ruby
ip helper-address <IP_OF_HELPER_HERE>
```

# 6. IPSec VPN

Define interesting traffic using extended ACLs | *`(config)#` mode*
```ruby
ip access-list extended <ACL_NAME_OR_NUM_HERE|example: VPN_ACL>
permit ip <SOURCE_NETWORK_ADDRESS_HERE> <SOURCE_WILDCARD_MASK_HERE> <DEST_NETWORK_ADDRESS_HERE> <DEST_WILDCARD_MASK_HERE>
```

ISAKMP / IKE Phase 1 Policy | *`(config)#` mode*
```ruby
cry isakmp policy <POLICY_ID_HERE|example: 10>
encr <ENCRYPTION_HERE|example: aes>
hash <HASH_METHOD_HERE|example: sha>
group <DH_GROUP_NUM_HERE|use 21 if supported, 14 if not>
lifetime <LIFETIME_HERE|example: 9000>
```

Pre-shared KEY / PSK | *`(config)#` mode*
```ruby
cry isakmp key <PSK_KEY_HERE|example: cisco12345> address <IPV4_HERE>
```

IPSec Transform Set tag 50 with AES256 ESP transform and SHA hashing | *`(config)#` mode*
```ruby
cry ipsec transform-set SET50 esp-aes 256 esp-sha-hmac
```

Crypto Map Config | *`(config)#` mode*
```ruby
cry map <MAP_NAME_HERE|example: VPN_MAP> <POLICY_ID_HERE|example: 10> ipsec-isakmp
set peer <PEER_IP_HERE>
set transform-set SET50
match address <ACL_NAME_OR_NUM_HERE|example: VPN_ACL>
```

# 7. ZPF / Zone-based Policy Firewall

Defining zones | *`(config)#` mode*
```ruby
zone sec <ZONE_NAME|example: INSIDE>
```

Assign zones to interfaces | *`(config-if)#` mode*
```ruby
zone-member sec <ZONE_NAME|example: INSIDE>
```

Create Class Map to match traffic types | *`(config)#` mode*
```ruby
class-map type inspect <MATCH_TYPE_HERE|either: match-any, match-all> <CLASS_MAP_NAME_HERE>
match protocol <PROTOCOL_HERE|example: https>
! repeat above match command here for multiple protocols
exit
```

Create Policy Map | *`(config)#` mode*
```ruby
policy-map type inspect <POLICY_MAP_NAME_HERE>
 class <CLASS_MAP_NAME_HERE|to use default: class-default>
  <ACTION_HERE|either: drop, inspect>
```

Create Zone Pairs | *`(config)#` mode*

```ruby
zone-pair security <PAIR_NAME_HERE> source <SOURCE_ZONE_NAME_HERE> destination <DEST_ZONE_NAME_HERE>
service-policy type inspect <POLICY_MAP_NAME_HERE>
```

# 8. Layer 2 / L2 / Layer2 Configuration and L2 Security

Configure a switch as root switch | *`(config)#` mode*
```ruby
spanning-tree vlan <VLAN_ID_HERE|example: 10> root primary
```

Disable DTP | *`(config-if)#` mode*
```ruby
switchport nonnegotiate
```

Protect against VTP vulnerabilities | *`(config)#` mode*
```ruby
vtp mode transparent
```

Enable portfast and bdpuguard | *`(config-if)#` mode*
```ruby
spanning-tree portfast
spanning-tree bdpuguard enable
```

Enable root port guard (apply on root switch, facing non-root switches) | *`(config-if)#` mode*
```ruby
spanning-tree guard root
```

Configure port security | *`(config-if)#` mode*
```ruby
switchport port-sec
```

# 9. DHCP Snooping

Enable DHCP snooping | *`(config)#` mode*
```
! Global
ip dhcp snooping

! Per-VLAN snooping
ip dhcp snooping vlan <VLAN_ID_LIST_HERE|example: 10,20>
```

Limit number of DHCP requests | *`(config-if)#` mode*
```ruby
ip dhcp snooping limit rate <NUMBER_OF_REQUESTS|example: 6>
```


