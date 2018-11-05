# MikroTik multiple ETECSA logins

### Requirements:
- Mikrotik wireless router
- Mikrotik Level 4 license 

### Add empty authentication method for wireless interfaces
```
/interface wireless security-profiles add authentication-types=wpa-psk,wpa2-psk eap-methods="" management-protection=allowed name=empty supplicant-identity=""
```

### Add ETECSA scanning channels
```
/interface wireless channels add band=5ghz-a/n frequency=5500 list=etecsa name=etecsa1 width=20
```

### Configure your main wifi interface (wlan2 in the example)
```
/interface wireless set [ find default-name=wlan2 ] band=5ghz-onlyn channel-width=20/40mhz-Ce country=australia disabled=no frequency=5500 mac-address=F2:38:38:38:7A:84 radio-name="" scan-list=etecsa1 security-profile=empty ssid=WIFI_ETECSA wireless-protocol=nv2-nstreme-802.11
```

### Add virtual interface (wlan3 in the example) with wlan2 as master interface
```
/interface wireless add disabled=no keepalive-frames=disabled mac-address=F2:38:38:38:7A:99 master-interface=wlan2 mode=station-pseudobridge multicast-buffering=disabled name=wlan3 security-profile=empty ssid=WIFI_ETECSA wds-cost-range=0 wds-default-cost=0 wps-mode=disabled
```

### Add DHCP client to every wifi interface
```
/ip dhcp-client add add-default-route=no dhcp-options=hostname,clientid disabled=no interface=wlan2
/ip dhcp-client add add-default-route=no dhcp-options=hostname,clientid disabled=no interface=wlan3
```
*note:* Probably **wlan2** already has a configured DHCP client so you might get an error. Delete this client and try again.

### Create address lists to separate the traffic
```
/ip firewall address-list add address=10.10.1.174 comment=mycomputer list=gateway1
/ip firewall address-list add address=10.10.2.0/24 comment=network2 list=gateway2
```

### Create mangle rules to make the routing marks
```
/ip firewall mangle add action=mark-routing chain=prerouting comment=gateway1 new-routing-mark=gateway1 src-address-list=gateway1
/ip firewall mangle add action=mark-routing chain=prerouting comment=gateway2 new-routing-mark=gateway2 src-address-list=gateway2
```

### Add NAT masquerade between wifi interfaces
```
/ip firewall nat add action=masquerade chain=srcnat out-interface=all-wireless
```

### Add routing rules for every routing mark (being 10.190.17.1 the gateway ETECSA is using on their DHCP)
```
/ip route add distance=1 gateway=10.190.17.1%wlan2 routing-mark=gateway1
/ip route add distance=1 gateway=10.190.17.1%wlan3 routing-mark=gateway2
```

### Add routing for DNS (being 181.225.231.110 one of the ETECSA DNS and 10.190.17.1 ETECSA's gateway for this connection)
```
/ip route add distance=1 dst-address=181.225.231.110/32 gateway=10.190.17.1
```

### Change the identity of your router so it does not look like MikroTik router
```
/system identity set name=android-ffdg786354635
```

### Add script to call DHCP-client renew to avoid disconnection
- First Check you have only the wifi DHCP clients active on the list:
```
/ip dhcp-client print             
Flags: X - disabled, I - invalid, D - dynamic 
 #   INTERFACE                               USE-PEER-DNS ADD-DEFAULT-ROUTE STATUS        ADDRESS
 0   ;;; defconf
     wlan2                                   yes          no                bound         10.190.17.142/24  
 1   wlan3                                   yes          no                bound         10.190.17.202/24 
```
If not, remove the ones not related to wlan interfaces.

- Create the script:
Here you should modify the values on the for loop according to the amount of DHCP clients (in the example is 2)
```
/system script add name=renew_all_dhcp source=":for i  from=0  to=1  do={/ip dhcp-client renew \$i}"
```

#### Add a scheduler for the previous script
```
/system scheduler add name=renew_dhcp interval=3m on-event=renew_all_dhcp
```
