# FortiGate IPsec Monitor

This repo contains the needed CLI commands to create a link-monitor for IPsec tunnels on FortiGate devices.  

This is useful for troubleshooting scenarios where IPsec tunnels have sudden communication issues through a tunnel.

## Link-Monitor 

```bash
config system link-monitor
    edit "vpn_monitor"
        set srcintf "VPN_INTERFACE"
        set server "IP_TO_PING"
        set protocol ping 
        set source-ip VPN_INTERFACE_IP
        set update-policy-route disable
        set service-detection enable
    next
end
```

## Trigger

```bash
config system automation-trigger
    edit "IPsec Link Monitor"
        set description "Link-Monitor triggers when status switches from alive -> dead"
        set event-type event-log
        set logid 22932
            config fields
                edit 1 
                set name "name"
                set value "vpn_monitor"
            next
        end
    next
end
```

Referenced LogID [22932](https://docs.fortinet.com/document/fortigate/7.2.8/fortios-log-message-reference/22932) is the event 'LINK MONITOR STATUS WARNING' which emitted when the link-monitors' status switches from `alive` to `dead`.

To only have this trigger regarding a specific link-monitor we filter for the appropriate `name` value in the event.

## Action

The following are two options depending on use-case:

[Option 1](#troubleshooting) for collecting IKE debugs and sending them via mail for troubleshooting purposes.
[Option 2](#flush-tunnel) to flush the tunnel itself in order to 'bounce' it and momentarily resolve the issue.

### Troubleshooting

Collect IKE debugs:

```bash
config system automation-action
    edit "Get VPN debug"
        set action-type cli-script
        set script "get vpn ike gateway
get vpn ipsec tunnel details
get vpn ipsec tunnel name <VPN NAME>

diagnose vpn tunnel list
diagnose vpn tunnel list name <VPN NAME>
diagnose vpn ike crypto"
        set accprofile "super_admin"
    next
end
```

Send results via email:

```bash
config system automation-action
    edit "IPsec Link-Monitor Notification"
        set description "Send the results of the previous CLI commands as mail"
        set action-type email
        set email-to "recipient@email.com"
        set email-subject "%%log.logdesc%%"
        set message "%%results%%"
    next
end
```

### Flush Tunnel

This will 'bounce' the tunnel (phase 1 & 2) to bring it down and up again.

```bash
config system automation-action
    edit "Flush Tunnel"
        set action-type cli-script
        set script "diagnose vpn ike gateway flush name <PHASE 1 NAME>" # OR put "diagnose vpn tunnel flush <PHASE 1 NAME>" (depends on ForiOS version)
        set accprofile "super_admin"
    next
end
```

## Stitch

Combine the trigger with the action(s) which suit your situation.

*Stitch to collect IKE debugs and send via mail*

```bash
config system automation-stitch
    edit "Monitor IPsec Stitch"
        set trigger "IPsec Link Monitor"
        config actions
            edit 1 
                set action "Get VPN debug"
                set required enable
            next
            edit 2 
                set action "IPsec Link-Monitor Notification"
                set delay 10
                set required enable
            next
        end
    next
end
```

*Stitch to flush the tunnel*

```bash

```
