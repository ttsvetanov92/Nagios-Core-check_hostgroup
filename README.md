# Custom Hostgroup Monitoring with Percent-Based Thresholds

## Overview

This Nagios plugin enables precise monitoring by checking the percentage of hosts in a DOWN state within a hostgroup.

This plugin is very useful when working with a host group containing many devices. If notifications are temporarily disabled to avoid spam caused by frequent status changes (e.g., when a client turns off their devices at night), the plugin allows you to receive a notification in case of a mass offline devices. For instance, in the event of a central failure causing a large number of devices to go offline, you will be alerted when too many of them transition to offline mode. Personally, I have configured critical situations to forward notifications directly to Telegram.

The plugin is easy to integrate into an existing Nagios Core setup.

---

## Key Features

1. **Percent-Based Thresholds:** Set Warning and Critical thresholds as percentages.  
2. **Custom Status Messages:** Outputs user-friendly messages like OK and BAD for better readability.  
3. **Basic Authentication Support:** Works with secured Nagios APIs.  
4. **JSON Parsing:** Utilizes `jq` for efficient JSON data extraction from `statusjson.cgi`.  
5. **Flexible Parameters:** Accepts hostgroup, thresholds, authentication, and base URL as arguments.

---

## Requirements

- **curl:** For fetching data from the Nagios API.  
- **jq:** For parsing JSON responses.

### Install Dependencies (on Ubuntu/Debian):

```bash
sudo apt-get update
sudo apt-get install -y curl jq
```

```bash
./check_offline_hosts <HOSTGROUP_NAME> <WARNING_%> <CRITICAL_%> [:] [<NAGIOS_URL>]
```
## Example Usage

```bash
./check_offline_hosts <HOSTGROUP_NAME> <WARNING_%> <CRITICAL_%> [:] [<NAGIOS_URL>]
```

```bash
./check_offline_hosts MyHostGroup 20 30 nagiosadmin:password "http://127.0.0.1/nagios"
```
## Result

### Critical (30% or more hosts DOWN):
```bash
BAD: 3 / 10 hosts are DOWN (30%) in hostgroup 'MyHostGroup'
```
### Warning (20%-29% of hosts DOWN):
```bash
BAD: 2 / 10 hosts are DOWN (20%) in hostgroup 'MyHostGroup'
```
### OK (fewer than 20% of hosts DOWN):
```bash
OK: All 10 hosts are operational (10% DOWN) in hostgroup 'MyHostGroup'
```
## How it works 

The plugin retrieves hostgroup information using Nagios’ statusjson.cgi API and:
- Calculates the total number of hosts.
- Determines how many hosts are in a DOWN state (status = 4).
- Computes the percentage of DOWN hosts and compares it to the Warning/Critical thresholds.
- Returns a result formatted for Nagios (OK, WARNING, CRITICAL, UNKNOWN).

## Nagios Integration

### Add the Command to commands.cfg:

```cfg
define command {
    command_name    check_hostgroup
    command_line    /path/to/check_offline_hosts $ARG1$ $ARG2$ $ARG3$ nagiosadmin:password http://127.0.0.1/nagios
}
```
### Create a "Virtual Host" for Each Hostgroup:

```cfg
define host {
    host_name            client1
    alias                Client 1 Server that contains a lot of devices
    address              127.0.0.1
    check_command        check_hostgroup!hostgroupname1 20 30
    max_check_attempts   3
    check_interval       5
    retry_interval       1
    notification_period  24x7
    notification_options d,u,r
    contact_groups       admins
    use                  generic-host
}

define host {
    host_name            client2
    alias                Client 2 Server that contains a lot of devices
    address              127.0.0.1
    check_command        check_hostgroup!hostgroupname2 20 30
    max_check_attempts   3
    check_interval       5
    retry_interval       1
    notification_period  24x7
    notification_options d,u,r
    contact_groups       admins
    use                  generic-host
}
....
....
define host {
    host_name            clientN
.....
.....
}
# A hostgroups for

define hostgroup{
        hostgroup_name                  0-Hostgroups             ; The name of the new hostgroup
        alias                           Hostgroup checker;
        members                         client1, client2 ... clientN
        }
```


