Custom Hostgroup Monitoring with Percent-Based Thresholds

Hello, everyone!

I’d like to share a Nagios plugin I created for monitoring hostgroups. This plugin enables precise monitoring by checking the percentage of hosts in a DOWN state within a hostgroup.

This plugin is particularly useful when working with a host group containing many devices.
If notifications are temporarily disabled to avoid spam caused by frequent status changes (e.g., when a client turns off their devices at night), the plugin allows you to receive a notification in case of a serious issue.
For instance, in the event of a central failure causing a large number of devices to go offline, you will be alerted when too many of them transition to offline mode.
Personally, I have configured critical situations to forward notifications directly to Telegram.

The plugin is easy to integrate into an existing Nagios Core setup.

Key Features:
Percent-Based Thresholds: Set Warning and Critical thresholds as percentages.
Custom Status Messages: Outputs user-friendly messages like OK and BAD for better readability.
Basic Authentication Support: Works with secured Nagios APIs.
JSON Parsing: Utilizes jq for efficient JSON data extraction from statusjson.cgi.
Flexible Parameters: Accepts hostgroup, thresholds, authentication, and base URL as arguments.

Requirements:
curl: For fetching data from the Nagios API.
jq: For parsing JSON responses.

Install dependencies (on Ubuntu/Debian):

sudo apt-get update
sudo apt-get install -y curl jq

Example Usage:

./check_offline_hosts <HOSTGROUP_NAME> <WARNING_%> <CRITICAL_%> [<USER>:<PASS>] [<NAGIOS_URL>]

Example:
./check_offline_hosts MyHostGroup 20 30 nagiosadmin:password "http://127.0.0.1/nagios"

Result:

If 30% or more hosts are DOWN - critical status:
BAD: 3 / 10 hosts are DOWN (30%) in hostgroup 'MyHostGroup'

If 20%-29% of hosts are DOWN - warning status:
BAD: 2 / 10 hosts are DOWN (20%) in hostgroup 'MyHostGroup'

If fewer than 20% of hosts are DOWN - ok status:
OK: All 10 hosts are operational (10% DOWN) in hostgroup 'MyHostGroup'

How It Works:
The plugin retrieves hostgroup information using Nagios’ statusjson.cgi API and:
- Calculates the total number of hosts.
- Determines how many hosts are in a DOWN state (status = 4).
- Computes the percentage of DOWN hosts and compares it to the Warning/Critical thresholds.
- Returns a result formatted for Nagios (OK, WARNING, CRITICAL, UNKNOWN).

Nagios Integration:
Add the command to commands.cfg:

define command{
        command_name    check_hostgroup
        command_line    $USER1$/./check_offline_hosts $ARG1$ $ARG2$ $ARG3$ nagiosadmin:password http://127.0.0.1/nagios
        }

You can also create new hostgroup containing the several hostgroups for better clearance:

Define "virtual host" host for every hostgroup:
Example:

define host {
    host_name           client1
    alias               Client 1 Server that contains a lot of devices
    address             127.0.0.1 #use whatever you want, the check will not use it anyway
    check_command       check_hostgroup!hostgroupname1 20 30 # 20% =< - warining; 30% =< - critical;
    max_check_attempts  3
    check_interval      5
    retry_interval      1
    notification_period 24x7
    notification_options d,u,r
    contact_groups      admins
    use                 generic-host
}

define host {
    host_name           client2
    alias               Client 2 Server that contains a lot of devices
    address             127.0.0.1 #use whatever you want, the check will not use it anyway
    check_command       check_hostgroup!hostgroupname2 20 30 # 20% =< - warining; 30% =< - critical;
    max_check_attempts  3
    check_interval      5
    retry_interval      1
    notification_period 24x7
    notification_options d,u,r
    contact_groups      admins
    use                 generic-host
}

Enjoy!
