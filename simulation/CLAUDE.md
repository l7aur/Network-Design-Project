# PTBuilder Scripting Rules

PTBuilder is a Cisco Packet Tracer extension. Scripts are JavaScript run inside PT via Extensions > Builder Code Editor.

## Critical Rules

- NO comments of any kind (`//` or `/* */`) — they break the parser
- NO blank lines inside `configureIosDevice` strings
- Use `\n\` (backslash at end of each line) for multiline IOS strings, NOT template literals

## Functions

```javascript
addDevice(name, model, x, y)
addModule(deviceName, slot, moduleModel)
addLink(dev1, port1, dev2, port2, linkType)
configureIosDevice(name, commands)
configurePcIp(name, dhcp, ip, mask, gateway, dns)
```

## configurePcIp signature

```javascript
configurePcIp("PC1", true)                                          // DHCP
configurePcIp("Server1", false, "10.10.10.10", "255.255.255.128", "10.10.10.1", "8.8.8.8")  // static
```

The second argument is NOT an object — positional args only.

## Link Types

| Type | Works | Notes |
|------|-------|-------|
| `straight` | yes | use for everything |
| `fiber` | only on SFP ports | 2960-24TT has no SFP by default — fiber links silently fail |
| `cross` | yes | |
| `wireless` | uncertain | |

## Device Models (confirmed working)

| Model | Type |
|-------|------|
| `"2911"` | Router |
| `"3650-24PS"` | L3 switch (MDF) — ports: GigabitEthernet1/0/1..24 |
| `"2960-24TT"` | L2 switch (IDF) — ports: FastEthernet0/1..24, GigabitEthernet0/1..2 |
| `"PC-PT"` | Workstation |
| `"Server-PT"` | Server |
| `7960` | IP Phone (number, not string) |
| `"AccessPoint-PT"` | Wireless AP |
| `"Laptop-PT"` | Laptop |

## Port Names

| Device | Port examples |
|--------|--------------|
| 2911 router | `GigabitEthernet0/0`, `GigabitEthernet0/1`, `GigabitEthernet0/2` |
| 3650-24PS | `GigabitEthernet1/0/1` .. `GigabitEthernet1/0/24` |
| 2960-24TT | `FastEthernet0/1` .. `FastEthernet0/24`, `GigabitEthernet0/1`, `GigabitEthernet0/2` |
| PC-PT / Server-PT | `FastEthernet0` |
| IP Phone 7960 | `FastEthernet0` |
| AccessPoint-PT | `Port0` (wired), `Port1` (wireless radio) |

## IOS Config Format

```javascript
configureIosDevice("SW1",
"enable\n\
configure terminal\n\
vlan 100\n\
 name HR\n\
interface FastEthernet0/1\n\
 switchport mode access\n\
 switchport access vlan 100\n\
 spanning-tree portfast\n\
 no shutdown\n\
end");
```

## Working Minimal Example

```javascript
addDevice("R1",  "2911",      100, 100);
addDevice("S1",  "2960-24TT", 300, 100);
addDevice("PC1", "PC-PT",     500, 100);

addLink("R1", "GigabitEthernet0/1", "S1",  "GigabitEthernet0/1", "straight");
addLink("S1", "FastEthernet0/1",    "PC1", "FastEthernet0",       "straight");

configureIosDevice("R1",
"enable\n\
configure terminal\n\
interface GigabitEthernet0/1\n\
 ip address 192.168.1.1 255.255.255.0\n\
 no shutdown\n\
end");

configurePcIp("PC1", true);
```

## ASA Firewall (5506-X) Config Pattern

```javascript
configureIosDevice("FW",
"enable\n\
configure terminal\n\
interface GigabitEthernet1/1\n\
 nameif outside\n\
 security-level 0\n\
 ip address 203.0.113.1 255.255.255.252\n\
 no shutdown\n\
interface GigabitEthernet1/2\n\
 nameif inside\n\
 security-level 100\n\
 ip address 10.0.0.1 255.255.255.252\n\
 no shutdown\n\
route inside 10.10.0.0 255.255.0.0 10.0.0.2\n\
access-list INSIDE extended permit ip any any\n\
access-group INSIDE in interface inside\n\
end");
```

- Ports: GigabitEthernet1/1 through GigabitEthernet1/8
- Uses ASA syntax: `nameif`, `security-level`, `access-list extended`, `access-group`
- security-level 100 = inside (trusted), 0 = outside (untrusted)
- Higher security can reach lower by default; opposite needs ACL

## Network Design (this project)

- `R1` (2911): router-on-a-stick, DHCP server, sub-interfaces per VLAN
- `MDF` (3650-24PS): L2 core switch, trunks everything
- `IDF1` (2960-24TT): access switch, access ports per VLAN
- MDF↔IDF1: two straight links for redundancy (RSTP rapid-pvst blocks one, failover is automatic)
- VLANs: 100 HR, 101 Finance, 102 IT_Ops (10.10.100.x/26 each)
- Inter-VLAN routing: done by R1 sub-interfaces (dot1Q encapsulation)

## Build Strategy

Always add incrementally — devices first, then links, then IOS config, then PC IPs.
If something breaks, remove the last addition and confirm the previous state still works.
