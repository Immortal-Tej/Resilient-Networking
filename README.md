# Resilient Networking

Host1 is connected to Host2 through two layers of switches. In one layer there are two switches, SW3 and SW4, where Host1 is connected via two Ethernet cables - one to SW3 (Ethernet4) and one to SW4 (Ethernet4). SW3 and SW4 are then connected to SW1 (SW3 via Ethernet1, SW4 via Ethernet1 on SW4's side / Ethernet2 on SW1's side), which is then connected to Host2 (Ethernet3). SW3 and SW4 are also connected to each other directly via an Ethernet cable (Ethernet2 on both), which is what MLAG uses as the peer-link.

So we need to create MLAG for SW3 and SW4 - Host1 will see them as one logical switch.

I never touched Host1's `bond0`, it uses its existing interface as it already was.

## Instructions

```bash
ansible-playbook playbook.yml
```

I logged in using `admin`/`admin`, which is configured in `all.yml`.

## How I solved this assignment

- I manually configured the switches first, before writing any Ansible.
- I configured SW3 manually, then SW4 manually, then SW1.
- I tested if Host1 can reach Host2 without any interruptions.
- There was a lot of lag initially when I tested the ping from Host1 to Host2, since the LACP timer was slow.
- So I made the LACP timer on both SW3 and SW4 fast, so that the message sent to Host1 (or any host) is fast - now it's approximately 1 second before a switch dies and traffic goes to the other switch.
- After testing that the traffic is uninterrupted and the recovery is fast, I started writing the Ansible YAML files.

## Layout

I split this into a playbook, roles, and variable files :
- `hosts.yml` - sw3/sw4 grouped as the mlag pair, sw1 separate
- `group_vars/all.yml` - connection settings, vlan list
- `group_vars/mlag_switches.yml` - mlag domain settings, `mlag_ports` list
- `group_vars/sw1_group.yml` - sw1's interface numbers
- `roles/peer_setup` - pairs sw3 and sw4
- `roles/ports` - builds each host's port-channel, loops over `mlag_ports`
- `roles/sw1_uplink` - sw1's side of the bundle, plain LACP

## Scalability

`mlag_ports` in `group_vars/mlag_switches.yml` is a list, host1 is just the first entry. Adding host2 through host20 is just adding entries there - nothing in `roles/ports` changes.

The only constraint is that the `interface` value in each entry has to match the port the host is actually cabled into on sw3 and sw4 - it's not something that gets detected automatically.

## VLANs

`allowed_vlans` in `group_vars/all.yml` is the single place that controls which VLANs are allowed on every host-facing trunk port-channel. It's currently `"1,10,20,30"` - VLAN 1 stays in there and stays native because Host1's `bond0` sends untagged traffic, so dropping it would break basic connectivity.

## Testing

I tested if the MLAG was working properly using the following method:

- First I ensured that the traffic was flowing from SW3, so that if I turned off SW3, the traffic could immediately switch to SW4.
- To actually make sure the traffic was going through SW3 and not still sitting on SW4, I turned off SW4 for a couple of seconds and then turned it back on, and waited one to two minutes so the switch had fully booted.
- I started a continuous ping from Host1 to Host2, then stopped SW3, and verified the ping was not interrupted - it was not interrupted, the flow kept going - and then restarted SW3.
- The same thing was followed for SW4's failure, but SW3 had to fully recover first, so I waited a couple of minutes before starting.
- I turned off SW3 for a couple of seconds and turned it back on, so that the traffic would now flow through SW4.
- I did the same thing again: verified the ping continued to operate, and then restarted the switch.
- This is how I ensured the connection stayed up during a continuous ping even when a switch fails, and I tested this twice to be sure the entire setup actually worked.
- I only used Containerlab to stop and start the switches, not docker directly.
- This whole test was done twice overall - once after I configured the switches manually, and again after I built and ran the Ansible playbook, to make sure the playbook gave the same working result.
