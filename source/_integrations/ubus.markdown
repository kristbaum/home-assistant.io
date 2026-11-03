---
title: OpenWrt (ubus)
description: Instructions on how to integrate OpenWrt devices into Home Assistant.
ha_categories:
  - presence_location.network_presence
  - computers_network.routers_network
ha_release: 0.7.6
ha_iot_class: Local Polling
ha_config_flow: true
ha_domain: ubus
ha_platforms:
  - device_tracker
ha_integration_type: integration
ha_quality_scale: legacy
related:
  - docs: /integrations/device_tracker/
    title: Device tracker
  - docs: /integrations/person/
    title: Person
---

The **OpenWrt (ubus)** {% term integration %} offers presence detection by looking at the Wi-Fi clients connected to an [OpenWrt](https://openwrt.org/) device. It talks to the device through [ubus](https://openwrt.org/docs/techref/ubus), the OpenWrt system bus, and reads the client list of each wireless access point (`hostapd`).

With this integration, you can use the presence of phones, tablets, and other devices on your Wi-Fi to see who is home and to trigger automations. For example, you can turn the lights off when everyone leaves home.

## Supported devices

The integration works with OpenWrt devices that provide a wireless access point.

{% important %}
Set up the integration on the OpenWrt devices that provide your Wi-Fi, not on devices that only act as routers. If you have several OpenWrt access points, add one integration entry for each of them.
{% endimportant %}

## Prerequisites

Before you set up the integration, prepare your OpenWrt device so Home Assistant can read the list of connected clients. You do this once, on each OpenWrt device you want to add. You need SSH access to the device.

1. Install the ubus RPC packages.
   - On OpenWrt 25.12 and newer:

     ```bash
     apk update
     apk add rpcd-mod-file uhttpd-mod-ubus
     ```

   - On OpenWrt 24.10 and older:

     ```bash
     opkg update
     opkg install rpcd-mod-file uhttpd-mod-ubus
     ```

2. Add a dedicated system user named `hass` for Home Assistant. You can do this in any way you prefer. For example, add these lines to the end of the following files:
   - In `/etc/passwd`, add: `hass:x:10001:10001:hass:/var:/bin/false`
   - In `/etc/shadow`, add: `hass:x:0:0:99999:7:::`
3. Set a password for the `hass` user:

   ```bash
   passwd hass
   ```

4. Allow the `hass` user to log in to ubus. Edit `/etc/config/rpcd` and add the following lines:

   ```text
   config login
           option username 'hass'
           option password '$p$hass'
           list read hass
           list read unauthenticated
           list write hass
   ```

5. Create an access control list (ACL) file at `/usr/share/rpcd/acl.d/hass.json` that gives the `hass` user read access to the data the integration needs:

   ```json
   {
     "hass": {
       "description": "Access role for OpenWrt ubus integration",
       "read": {
         "ubus": {
           "hostapd.*": ["get_clients"],
           "uci": ["get"],
           "dhcp": ["ipv4leases"],
           "file": ["read"]
         },
         "file": {
           "/tmp/dhcp.leases": ["read"]
         }
       },
       "write": {}
     }
   }
   ```

   The entry `/tmp/dhcp.leases` is the OpenWrt default lease file for dnsmasq. If you changed it, run `uci get dhcp.@dnsmasq[0].leasefile` on your device and use the path it returns instead. Otherwise, device names can't be resolved. If dnsmasq is not used on your device, you can skip this.

6. Restart the services:

   ```bash
   /etc/init.d/rpcd restart && /etc/init.d/uhttpd restart
   ```

7. Check that the `file` namespace is registered. The following command should return `file`:

   ```bash
   ubus list | grep file
   ```

8. Add the ACL file to `/etc/sysupgrade.conf`, so it's kept when you upgrade the OpenWrt firmware:

   ```bash
   echo "/usr/share/rpcd/acl.d/hass.json" >> /etc/sysupgrade.conf
   ```

{% include integrations/config_flow.md %}

{% configuration_basic %}
Host:
  description: "The IP address or hostname of your OpenWrt device. For example, `192.168.1.1`."
Username:
  description: "The username of the account you created on your OpenWrt device, for example, `hass`. Use this dedicated user rather than `root`, so Home Assistant only has the access it needs."
Password:
  description: "The password of the account above."
DHCP software:
  description: "The DHCP server running on your OpenWrt device, used to look up device names: **dnsmasq** (the OpenWrt default), **odhcpd**, or **None**. Select **None** if the device doesn't run a DHCP server, for example, if it's a bridged access point. In that case, devices are named by their MAC address."
{% endconfiguration_basic %}

## Migrating from YAML

If you previously set up this integration with a `device_tracker` entry in your {% term "`configuration.yaml`" %} file, Home Assistant imports that configuration automatically the next time it starts. Each OpenWrt device in your YAML configuration becomes its own integration entry.

After the import, a repair issue in {% my repairs title="**Settings** > **System** > **Repairs**" %} asks you to remove the YAML configuration:

1. Remove the `ubus` entries from the `device_tracker` section of your {% term "`configuration.yaml`" %} file.
2. Restart Home Assistant.

YAML configuration for this integration stops working in Home Assistant 2027.5.

If the import fails because the OpenWrt device can't be reached or the credentials are rejected, a repair issue tells you which host failed. Correct the entry in your {% term "`configuration.yaml`" %} file and restart Home Assistant, or set up the integration from the UI instead.

## Supported functionality

The integration creates a {% term "device tracker" %} entity for each Wi-Fi client it sees connected to your OpenWrt device. The entity is **Home** while the client is connected and **Away** when it's not. Each tracked client also gets its own device in Home Assistant, named after its DHCP hostname, or after its MAC address if no hostname is known.

New device tracker entities are disabled by default, because most clients on a network, like TVs or smart plugs, aren't worth tracking. To track a phone or another device:

1. Go to {% my integrations title="**Settings** > **Devices & services**" %} and select **OpenWrt (ubus)**.
2. Select the entities, then select the device tracker you want to use.
3. Select the cogwheel {% icon "mdi:cog" %} and turn on **Enabled**.

To use a device tracker for presence detection, add it to a person under {% my people title="**Settings** > **People**" %}. For details, refer to the [person integration](/integrations/person/).

## Data updates

The integration {% term polling polls %} each OpenWrt device every 30 seconds for the list of connected Wi-Fi clients. Device names are read from the DHCP leases when the integration starts.

## Known limitations

- Only Wi-Fi clients connected to the access points of the OpenWrt device are detected. Wired devices and clients of other access points are not.
- The integration connects to the OpenWrt device over HTTP.

## Troubleshooting

### Enable debug logging

If the integration doesn't behave as expected, enable debug logging to see more detail:

1. Go to {% my integrations title="**Settings** > **Devices & services**" %} and select **OpenWrt (ubus)**.
2. Select the three dots {% icon "mdi:dots-vertical" %} menu and select **Enable debug logging**.
3. Reproduce the problem and wait at least one minute.
4. Select **Disable debug logging**. Your browser downloads the log file.

If you prefer, you can also enable debug logging in your {% term "`configuration.yaml`" %} file:

```yaml
logger:
  logs:
    homeassistant.components.ubus: debug
    aioubus: debug
    openwrt: debug
```

### Symptom: "Invalid authentication" during setup

The setup form shows **Invalid authentication**, or the log shows `Invalid credentials for ubus`.

#### Resolution

- Check the username and password.
- Make sure the `config login` section for the user exists in `/etc/config/rpcd` on the OpenWrt device, then restart `rpcd`.

### Symptom: "Failed to connect" during setup

The setup form shows **Failed to connect**, or the log shows `Error connecting to ubus`.

#### Resolution

- Make sure Home Assistant can reach the OpenWrt device at the host you entered.
- Make sure the `uhttpd-mod-ubus` package is installed and `uhttpd` is running.
- Check whether your OpenWrt device redirects HTTP to HTTPS. Many devices with LuCI do this by default, and answer every HTTP request with a redirect to HTTPS, which the integration can't follow. Run `uci get uhttpd.main.redirect_https` on the device. If it returns `1`, turn off the redirect:

  ```bash
  uci set uhttpd.main.redirect_https=0
  uci commit uhttpd
  /etc/init.d/uhttpd restart
  ```

### Symptom: The log shows "Access denied by ubus"

The integration can log in, but the OpenWrt device refuses the requests.

#### Resolution

1. Check that the ACL file `/usr/share/rpcd/acl.d/hass.json` exists and matches the example in [Prerequisites](#prerequisites).
2. Check that the user name in the ACL file matches the `list read` and `list write` entries in `/etc/config/rpcd`.
3. Restart `rpcd` after any change: `/etc/init.d/rpcd restart`.
4. Check which permissions the `hass` user actually gets. Run the following command on the OpenWrt device, with the password of the `hass` user. The `acls` section of the response should contain the `hostapd.*`, `uci`, `dhcp`, and `file` entries from the ACL file:

   ```bash
   ubus call session login '{"username": "hass", "password": "YOUR_PASSWORD"}'
   ```

5. Check that the objects and methods from the ACL file exist on the device. For example, `ubus -v list "hostapd.*"` should show the `get_clients` method, and `ubus -v list dhcp` should show `ipv4leases` if you use odhcpd.
6. Check the system log on the OpenWrt device for errors from `rpcd` or `uhttpd`:

   ```bash
   logread | grep -E "rpcd|uhttpd"
   ```

### Symptom: The log shows "No hostapd data"

The integration can log in, but no wireless access points are found.

#### Resolution

Check that the OpenWrt device provides Wi-Fi. The following command, run on the device, should list at least one `hostapd` object, for example `hostapd.phy0-ap0`:

```bash
ubus list "hostapd.*"
```

If the list is empty, the device doesn't run a wireless access point, for example, because it only acts as a router. Set up the integration on your OpenWrt access points instead.

### Symptom: Devices are named by their MAC address

#### Resolution

- Check that **DHCP software** matches the DHCP server running on the OpenWrt device. You can only set this when you add the integration, so delete and re-add the integration to change it.
- If you use dnsmasq, check that the lease file path in the ACL file matches the output of `uci get dhcp.@dnsmasq[0].leasefile`.
- Some devices don't share a hostname with the DHCP server. These devices are always named by their MAC address. You can rename them in Home Assistant.

## Removing the integration

This integration follows standard integration removal.

{% include integrations/remove_device_service.md %}

After deleting the integration, you can optionally remove the `hass` user, the `config login` section in `/etc/config/rpcd`, and the ACL file `/usr/share/rpcd/acl.d/hass.json` from your OpenWrt device. Also remove the ACL file path from `/etc/sysupgrade.conf`.
