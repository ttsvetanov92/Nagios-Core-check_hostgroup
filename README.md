# Custom Hostgroup Monitoring with Percent-Based Thresholds

## Overview

This Nagios plugin provides precise monitoring by checking the percentage of hosts in a DOWN state within a hostgroup.

It is especially useful when managing host groups with many devices. For instance, if notifications are temporarily disabled to prevent spam from frequent status changes (e.g., when a client powers down their devices at night), the plugin ensures you still receive alerts in case of a significant issue. In the event of a central failure causing a large number of devices to go offline, it will notify you when the number of offline devices exceeds a specified threshold. Personally, I have configured it to forward critical notifications directly to Telegram.

The plugin is straightforward to integrate into an existing Nagios Core setup.

---

## Key Features

1. **Percent-Based Thresholds:** Set Warning and Critical thresholds as percentages.  
2. **Basic Authentication Support:** Works with secured Nagios APIs.  
3. **JSON Parsing:** Utilizes `jq` for efficient JSON data extraction from `statusjson.cgi`.  
4. **Flexible Parameters:** Accepts hostgroup, thresholds, authentication, and base URL as arguments.

---

## Requirements

- **curl:** For fetching data from the Nagios API.  
- **jq:** For parsing JSON responses.

### Install Dependencies (on Ubuntu/Debian):

```bash
sudo apt-get update
sudo apt-get install -y curl jq
```

## Example Usage

`check_hostgroup` can be executed directly from the command line without any Nagios configuration. It fetches data from the Nagios API and calculates the status dynamically. Configuration in Nagios is only required for automated monitoring, notifications, and integration into the UI.

```bash
./check_hostgroup <HOSTGROUP_NAME> <WARNING_%> <CRITICAL_%> [:] [<NAGIOS_URL>]
```

```bash
./check_hostgroup MyHostGroup 20 30 nagiosadmin:password "http://127.0.0.1/nagios"
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
In order to integrate `check_hostgroup` into Nagios as a host check, we use a virtual host.
This does not replace the existing hostgroup but allows monitoring the entire hostgroup as a single unit, making it easier to configure alerts and track status changes.
However, if you only need to run the script manually, no configuration is required.

### Installation
Download the shell scrypt and add it to libexec nagios folder

```bash
git clone https://github.com/ttsvetanov92/Nagios-Core-check_hostgroup/
cd Nagios-Core-check_hostgroup
chmod +x check_hostgroup
sudo mv check_hostgroup /usr/local/nagios/libexec
```

### Add the Command to commands.cfg:

```cfg
define command {
    command_name    check_hostgroup
    command_line    /path/to/check_hostgroup $ARG1$ $ARG2$ $ARG3$ nagiosadmin:password http://127.0.0.1/nagios
}
```
###  Option 1: Using a "Virtual Host" to Monitor the Hostgroup as a Single Unit
This method treats the entire hostgroup as if it were a single host in Nagios. It allows host-based alerts and makes it easier to track the status of a group of devices.

**Example of "Virtual Host" for Each Hostgroup:**

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
        hostgroup_name                  0-Hostgroups             ; The name of the Virtual hostgroup
        alias                           Hostgroup checker;
        members                         client1, client2 ... clientN
        }
```

### Option 2: Configuring check_hostgroup as a Service Check
This method runs `check_hostgroup` as a regular service check for a specific host. It allows more flexible alerts but does not simulate a host failure when a group goes down.

**Example of service configuration:**

```cfg
define service {
    use                     generic-service
    host_name               myhost
    service_description     Hostgroup Status Check
    check_command           check_hostgroup!myhostgroup 20 30
    max_check_attempts      3
    check_interval          5
    retry_interval          1
    notification_period     24x7
    notification_options    w,c,r
    contact_groups          admins
}
```


### Notice
In Nagios, host checks are limited to three default statuses:

- UP (the host is reachable),
- DOWN (the host is unreachable),
- UNREACHABLE (the host cannot be reached due to a network issue).

Unlike service checks, which can return OK, WARNING, CRITICAL, and UNKNOWN, host checks do not support the WARNING status by default.

Keep in mind that if you configure a command as a host check, any state intended to represent WARNING will actually be reported as OK, but you will still receive a notification about the status change.
