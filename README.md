# shim-wifi
This is a writeup on how to put wifi in a normal ChromeOS RMA Shim.

# How it works
Basically, you just put the linux firmware in the RMA Shim while or after building. It's quite simple, but not documented well in the slightest.
```bash
git clone --depth=1 https://chromium.googlesource.com/chromiumos/third_party/linux-firmware
rm -rf $(find ./linux-firmware/* -not -name "iwlwifi*.ucode")
cp -r ./linux-firmware/* $wherever_your_shim_is/lib/firmware/ 
```
In a typical shim running the SH1MMER-ed rootfs, all you have to do to get the wifi device active is put something along the lines of this in whatever file runs in the shim.
```bash
rm -f /etc/resolv.conf
echo "nameserver 8.8.8.8" > /etc/resolv.conf
mkdir -p /run/dbus
dbus-daemon --system > /dev/null 2>&1
mkdir -p /var/lib
modprobe -r iwlwifi
modprobe iwlwifi
```


<br><br>
# Optionally, here's the entire script from IRS to put wifi in the shim with annotations<br><br>
The following function is to allow the user to change the ip address, gateway, and subnet mask
```bash
changedhcpinfo() {
	read -p "Enter IP address (leave blank to keep: $ip): " input
	ip="${input:-$ip}"
	read -p "Enter Gateway (leave blank to keep: $gateway): " input
	gateway="${input:-$gateway}"
	read -p "Enter Subnet Mask (leave blank to keep: $mask): " input
	mask="${input:-$mask}"
	echo "Using IP: $ip"
	echo "Using Gateway: $gateway"
	echo "Using Subnet Mask: $mask"
	read -p "Confirm these changes? (Y/n): " confirmchanges
}
```

The following function is to automatically get the IP address with wild amounts of jank (This is why i posted the instructions above, in case anyone wants to find a better way to do this)
```bash
manipcon() {
    iface=$(ip -o link show | awk -F': ' '/wl/ {print $2; exit}')
    DHCP_INFO=$(dhcpcd -d -4 -G -K -T $iface 2>&1)
    ip=$(echo "$DHCP_INFO" | grep -oE 'offered [0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | awk '{print $2}')
    gateway=$(echo "$DHCP_INFO" | grep -oE 'offered .* from [0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | awk '{print $NF}')
    if [ -z "$ip" ] || [ -z "$gateway" ]; then
        echo "Failed to get ip/gateway."
	sleep 1
    fi
    firstnum=$(echo "$ip" | cut -d. -f1)
    if [ "$firstnum" = "10" ]; then
        mask="255.0.0.0"
    else
        mask="255.255.255.0"
    fi
	changedhcpinfo
}
```

The following function is to set up wifi (this contains the instructions at the top)
```bash
wifi() {
    rm -f /etc/resolv.conf
    sync
    echo "nameserver 8.8.8.8" > /etc/resolv.conf 
	echo -e "This may take a while!!!"
	mkdir -p /run/dbus
	dbus-daemon --system > /dev/null 2>&1
	mkdir -p /var/lib
	firmware
    read -p "Enter your network SSID: " ssid
    read -p "Enter your network password (leave blank if none): " psk
    iface=$(ip -o link show | awk -F': ' '/wl/ {print $2; exit}')
    ifconfig $iface up || return
    if [ -z "$psk" ]; then
        wpa_supplicant -i $iface -C /run/wpa_supplicant -B -c <(
            cat <<EOF
network={
    ssid="$ssid"
    key_mgmt=NONE
}
EOF
            )
        else
            wpa_supplicant -i $iface -C /run/wpa_supplicant -B -c <(wpa_passphrase "$ssid" "$psk")
        fi
        dhcpcd >/dev/null
        manipcon
    	case "$confirmchanges" in
    		n | N) changedhcpinfo ;;
    		*) ifconfig $iface "$ip" netmask "$mask" up
    		       route add default gw "$gateway" ;;
	    esac
}
```
