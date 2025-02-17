# 25 Creating the dummy interface

As an OpenCloudOS user, you can create and use dummy network interfaces for debugging and testing. A dummy interface provides a facility to route packets without actually transmitting the packets. It allows you to additionally create a loopback device managed with NetworkManager, allowing inactive SLIP (Serial Line Internet Protocol) addresses to communicate with processes as if they were concrete addresses.

## 25.1 Create a dummy interface with IPv4 and IPv6 addresses using nmcli

You can create dummy interfaces with various settings. This section describes how to create dummy interfaces using IPv4 and IPv6 addresses. When a virtual interface is created, NetworkManager automatically assigns it to the default public firewall domain.

Note that to configure a virtual interface without an IPv4 or IPv6 address, set the ipv4.method and ipv6.method parameters to disabled. Otherwise, IP autoconfiguration will fail and NetworkManager will deactivate the connection and delete the dummy device.

**Flow**

1. Create a dummy interface named `dummy0` with static IPv4 and IPv6 addresses:
    ```
    # nmcli connection add type dummy ifname dummy0 ipv4.method manual ipv4.addresses 192.0.2.1/24 ipv6.method manual ipv6.addresses 2001:db8:2::1/64
    ```
2. Optional: To view the dummy interface, enter:
    ```
    # nmcli connection show
    NAME            UUID                                  TYPE      DEVICE
    enp1s0          db1060e9-c164-476f-b2b5-caec62dc1b05  ethernet    ens3
    dummy-dummy0    
    ```

# 26 Using nmstate-autoconf to Automatically Configure Network State Using LLDP

Network devices can use the Link Layer Discovery Protocol (LLDP) to identify themselves, their capabilities, and their neighbors on a LAN. The `nmstate-autoconf` tool can use this information to automatically configure the local network interface.

## 26.1 Using nmstate-autoconf to automatically configure network interfaces

The nmstate-autoconf tool configures local devices using LLDP to identify VLAN settings attached to switch interfaces.

This procedure assumes the following scenarios, and the switch uses LLDP broadcast VLAN setup:

- The server's `enp1s0` and `enp2s0` interfaces are connected to switch ports configured with `VLAN ID` `100` and `VLAN name` `prod-net`.
- The server's `enp3s0` interface is connected to a switch port configured with `VLAN ID` `200` and `VLAN name` `mgmt-net`.

The `nmstate-autoconf` tool then uses this information to create the following interfaces on the server:

- bond100 - Bond interface with `enp1s0` and `enp2s0` as ports.
- prod-net - VLAN interface on bond100 with `VLAN ID` `100`.
- mgmt-net - VLAN interface on `enp3s0` with `VLAN ID` `200`.

If you connect multiple network interfaces to ports on different switches that LLDP uses to broadcast the same `VLAN ID`, `nmstate-autoconf` creates a bond with these interfaces and configures a common `VLAN ID` on them.

**Prerequisite**

- The `nmstate` package is installed.
- LLDP is enabled on the network switch.
- Ethernet interface is enabled.

**Flow**

1. Enable LLDP on the Ethernet interface:
   1. Create a `~/enable-lldp.yml` YAML file with the following content:
        ```
        interfaces:
        - name: enp1s0
            type: ethernet
            lldp:
            enabled: true
        - name: enp2s0
            type: ethernet
            lldp:
            enabled: true
        - name: enp3s0
            type: ethernet
            lldp:
            enabled: true
        ```
   2. App settings:
        ```
        # nmstatectl apply ~/enable-lldp.yml
        ```

2. Configure the network interface using LLDP:
   1. Optionally, start a dry run to display and verify the YAML configuration generated by nmstate-autoconf:
        ```
        # nmstate-autoconf -d enp1s0,enp2s0,enp3s0
        ---
        interfaces:
        - name: prod-net
        type: vlan
        state: up
        vlan:
            base-iface: bond100
            id: 100
        - name: mgmt-net
        type: vlan
        state: up
        vlan:
            base-iface: enp3s0
            id: 200
        - name: bond100
        type: bond
        state: up
        link-aggregation:
            mode: balance-rr
            port:
            - enp1s0
            - enp2s0
        ```
   2. Use nmstate-autoconf to generate a configuration based on information received from LLDP and apply the settings to the system:
        ```
        # nmstate-autoconf enp1s0,enp2s0,enp3s0
        ```

**Verify**

- Display settings for a single interface
    ```
    # nmstatectl show <interface_name>
    ```

# 27 Using LLDP to Debug Network Configuration Problems

You can use Link Layer Discovery Protocol (LLDP) to debug configuration issues in your network topology. This means that LLDP can report configuration inconsistencies with other hosts or routers and switches.

## 27.1 Debugging Incorrect VLAN Configuration Using LLDP Information

If you configure a switch port to use specified VLANs and hosts are not receiving packets for those VLANs, you can use Link Layer Discovery Protocol (LLDP) to debug the problem. Please perform this process on the host that did not receive the packet.

**Prerequisite**

- The `nmstate` package is installed.
- The switch supports LLDP.
- LLDP is enabled on the neighbor device.

**Flow**

1. Create a `~/enable-LLDP-enp1s0.yml` file with the following content:
    ```
    interfaces:
    - name: enp1s0
        type: ethernet
        lldp:
        enabled: true
    ```
2. Create a `~/enable-LLDP-enp1s0.yml` file with the following command:
    ```
    # nmstatectl apply ~/enable-LLDP-enp1s0.yml
    ```
3. Display LLDP information:
    ```
    # nmstatectl show enp1s0
    - name: enp1s0
    type: ethernet
    state: up
    ipv4:
        enabled: false
        dhcp: false
    ipv6:
        enabled: false
        autoconf: false
        dhcp: false
    lldp:
        enabled: true
        neighbors:
        - - type: 5
            system-name: Summit300-48
        - type: 6
            system-description: Summit300-48 - Version 7.4e.1 (Build 5)
            05/27/05 04:53:11
        - type: 7
            system-capabilities:
            - MAC Bridge component
            - Router
        - type: 1
            _description: MAC address
            chassis-id: 00:01:30:F9:AD:A0
            chassis-id-type: 4
        - type: 2
            _description: Interface name
            port-id: 1/1
            port-id-type: 5
        - type: 127
            ieee-802-1-vlans:
            - name: v2-0488-03-0505
            vid: 488
            oui: 00:80:c2
            subtype: 3
        - type: 127
            ieee-802-3-mac-phy-conf:
            autoneg: true
            operational-mau-type: 16
            pmd-autoneg-cap: 27648
            oui: 00:12:0f
            subtype: 1
        - type: 127
            ieee-802-1-ppvids:
            - 0
            oui: 00:80:c2
            subtype: 2
        - type: 8
            management-addresses:
            - address: 00:01:30:F9:AD:A0
            address-subtype: MAC
            interface-number: 1001
            interface-number-subtype: 2
        - type: 127
            ieee-802-3-max-frame-size: 1522
            oui: 00:12:0f
            subtype: 4
    mac-address: 82:75:BE:6F:8C:7A
    mtu: 1500
    ```
4. Verify the output to make sure the settings match your expected configuration. For example, LLDP information for an interface connected to a switch shows that this host is connected to a switch port using VLAN ID 448:
    ```
    - type: 127
            ieee-802-1-vlans:
            - name: v2-0488-03-0505
            vid: 488
    ```
    If the network configuration for the enp1s0 interface uses a different VLAN ID, modify it accordingly.

# 28 Manually create NetworkManager configuration sets in keyfile format

NetworkManager supports configuration sets stored in keyfile format. However, by default, if you use NetworkManager tools such as nmcli, networking rhel-system-roles, or the nmstate API to manage configuration files, NetworkManager will still use ifcfg-format configuration files.

## 28.1 NetworkManager profiles in keyfile format

NetworkManager uses an INI-style keyfile format when storing connection profiles on disk.

**Example ethernet connection profile in keyfile format**

```
[connection]
id=example_connection
uuid=82c6272d-1ff7-4d56-9c7c-0eb27c300029
type=ethernet
autoconnect=true

[ipv4]
method=auto

[ipv6]
method=auto

[ethernet]
mac-address=00:53:00:8f:fa:66
```

Each section corresponds to a NetworkManager setting name, and most variables in the NetworkManager keyfile have a one-to-one mapping. This means that NetworkManager's properties are stored in the keyfile as variables with the same name and the same format. However, there are some exceptions, mainly to make the keyfile syntax easier to read.

For security reasons, NetworkManager only uses configuration files owned by root and readable and writable by root only, since connection configuration files can contain sensitive information such as private keys and passphrases.

Depending on the purpose of the connection profile, save it in the following directory:

- `/etc/NetworkManager/system-connections/` : Common location for persistent configuration files created by the user, which can also be edited. NetworkManager automatically copies them to `/etc/NetworkManager/system-connections/`.
- `/run/NetworkManager/system-connections/` : Used for temporary configuration files that are automatically deleted when the system is restarted.
- `/usr/lib/NetworkManager/system-connections/` : For pre-deployed immutable configuration files. When you edit such a configuration file using the NetworkManager API, NetworkManager copies the configuration file to persistent or temporary storage.

NetworkManager does not automatically reload configuration files from disk. When you create or update a connection profile in keyfile format, use the nmcli connection reload command to tell NetworkManager about the change.

## 28.2 Create a NetworkManager configuration set in keyfile format

This section describes how to manually create a NetworkManager connection profile in keyfile format.

Note that manually creating or updating configuration files may result in unexpected or non-functioning network configurations. OpenCloudOS recommends that you use NetworkManager tools such as nmcli, the network RHEL system role, or the nmstate API to manage NetworkManager connections.

**Flow**

1. If you created a configuration file for a hardware interface such as an ethernet card, display the MAC address of the interface:
    ```
    # ip address show enp1s0
    2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        link/ether 00:53:00:8f:fa:66 brd ff:ff:ff:ff:ff:ff
    ```

2. Create a connection profile. For example, for a connection configuration file for an Ethernet device using DHCP, create the `/etc/NetworkManager/system-connections/example.nmconnection` file with the following content:
    ```
    [connection]
    id=example_connection
    type=ethernet
    autoconnect=true

    [ipv4]
    method=auto

    [ipv6]
    method=auto

    [ethernet]
    mac-address=00:53:00:8f:fa:66
    ```

You can use any filename with a `.nmconnection` suffix. However, when you later use the nmcli command to manage the connection, you must refer to the connection name set in this connection id variable. When the id variable is omitted, use the filename without `.nmconnection` to refer to the connection.

3. Set permissions on the configuration file so that only the root user can read and update it:
    ```
    # chown root:root /etc/NetworkManager/system-connections/example.nmconnection
    # chmod 600 /etc/NetworkManager/system-connections/example.nmconnection
    ```

4. Reload the connection profile:
    ```
    # nmcli connection reload
    ```

5. Verify that NetworkManager reads the configuration file from the configuration file:
    ```
    # nmcli -f NAME,UUID,FILENAME connection
    NAME                UUID                                  FILENAME
    example-connection  86da2486-068d-4d05-9ac7-957ec118afba  /etc/NetworkManager/system-connections/example.nmconnection
    ...
    ```
    If the command does not show the newly added connection, verify that the file permissions and the syntax you are using in the file are correct.

6. Optional: If you set the autoconnect variable in the configuration file to false, activate the connection:
    ```
    # nmcli connection up example_connection
    ```

**Verify**

1. Show connection profiles:
    ```
    # nmcli connection show example_connection
    ```

2. Display the IP settings for an interface:
    ```
    # ip address show enp1s0
    ```

## 28.3 Migrate NetworkManager configuration set from ifcfg to keyfile format

You can migrate an existing ifcfg connection profile to keyfile format using the nmcli connection migrate command. This way, all connection configuration sets will be in one location and preferred format.

**Prerequisite**

- There are connection configuration sets in ifcfg format in the `/etc/sysconfig/network-scripts/` directory.

**Flow**

- Migrating Connection Profiles
    ```
    # nmcli connection migrate
    Connection 'enp1s0' (43ed18ab-f0c4-4934-af3d-2b3333948e45) successfully migrated.
    Connection 'enp2s0' (883333e8-1b87-4947-8ceb-1f8812a80a9b) successfully migrated.
    ...
    ```

**Verify**

- Verify that the connection profile was successfully migrated
    ```
    # nmcli -f TYPE,FILENAME,NAME connection
    TYPE      FILENAME                                                           NAME
    ethernet  /etc/NetworkManager/system-connections/enp1s0.nmconnection         enp1s0
    ethernet  /etc/NetworkManager/system-connections/enp2s0.nmconnection         enp2s0
    ...
    ```

## 28.4 Using nmcli to Create a Keyfile Connection Profile in Offline Mode

You can use the nmcli --offline connection add command to create various connection profiles in offline mode keyfile format.

Offline mode ensures that nmcli runs without the NetworkManager service to generate keyfile connection profiles via standard output. This feature is useful in the following situations:

- You want to create connection profiles that require a pre-deployment location. For example, in a container image, or an RPM package.
- You need to create connection profiles in environments where the NetworkManager service is not available. For example, when you want to use the chroot tool. Or when you want to create or modify the network configuration of the OpenCloudOS system via the Kickstart %post script.

You can create the following connection profile types:

- Static ethernet connection
- Dynamic Ethernet connection
- Network binding
- Bridge
- VLAN or any supported connection type

Note that manually creating or updating configuration files may result in unexpected or non-functioning network configurations.

**Prerequisite**

- The NetworkManager service has stopped.

**Flow**

1. Creates a new connection profile in keyfile format. For example, for a connection profile for an Ethernet device not using DHCP, run a similar nmcli command:
    ```
    # nmcli --offline connection add type ethernet con-name Example-Connection ipv4.addresses 192.0.2.1/24 ipv4.dns 192.0.2.200 ipv4.method manual > /etc/NetworkManager/system-connections/output.nmconnection
    ```
    
    The connection name specified with the con-name key is saved in the id variable of the generated configuration set. When you later use the nmcli command to manage this connection, specify the connection as follows:
     - If the id variable is not omitted, use the connection name, such as `Example-Connection`.
     - When omitting the id variable, use the filename without the `.nmconnection` suffix, such as output.

2. Set permissions on the configuration file so that only the root user can read and update it:
    ```
    # chmod 600 /etc/NetworkManager/system-connections/output.nmconnection
    # chown root:root /etc/NetworkManager/system-connections/output.nmconnection
    ```

3. Start the NetworkManager service:
    ```
    # systemctl start NetworkManager.service
    ```

4. Optional: If you set the autoconnect variable in the configuration file to false, activate the connection:
    ```
    # nmcli connection up Example-Connection
    ```

**Verify**

1. Verify that the NetworkManager service is running:
    ```
    # systemctl status NetworkManager.service
    ● NetworkManager.service - Network Manager
    Loaded: loaded (/usr/lib/systemd/system/NetworkManager.service; enabled; vendor preset: enabled)
    Active: active (running) since Wed 2022-08-03 13:08:32 CEST; 1min 40s ago
        Docs: man:NetworkManager(8)
    Main PID: 7138 (NetworkManager)
        Tasks: 3 (limit: 22901)
    Memory: 4.4M
    CGroup: /system.slice/NetworkManager.service
            └─7138 /usr/sbin/NetworkManager --no-daemon

    Aug 03 13:08:33 example.com NetworkManager[7138]: <info>  [1659524913.3600] device (vlan20): state change: secondaries -> activated (reason 'none', sys-iface-state: 'assume')
    Aug 03 13:08:33 example.com NetworkManager[7138]: <info>  [1659524913.3607] device (vlan20): Activation: successful, device activated.
    ...
    ```
2. Verify that NetworkManager can read the configuration set from the configuration file:
    ```
    # nmcli -f TYPE,FILENAME,NAME connection
    TYPE      FILENAME                                                    NAME
    ethernet /etc/NetworkManager/system-connections/output.nmconnection Example-Connection
    ethernet  /etc/sysconfig/network-scripts/ifcfg-enp1s0                 enp1s0
    ...
    ```
    If the output does not show newly created connections, verify that the keyfile permissions and the syntax you are using are correct.
3. Show connection profiles:
    ```
    # nmcli connection show Example-Connection
    connection.id:                          Example-Connection
    connection.uuid:                        232290ce-5225-422a-9228-cb83b22056b4
    connection.stable-id:                   --
    connection.type:                        802-3-ethernet
    connection.interface-name:              --
    connection.autoconnect:                 yes
    ...
    ```

# 29 Using netconsole to log kernel information over the network

Using the netconsole kernel module and service of the same name, you can log kernel messages over the network for easy debugging when a disk fails or a serial console is not possible.

## 29.1 Configure the netconsole service to record kernel information to remote hosts

Using the netconsole kernel module, you can log kernel information to a remote syslog service.

**Prerequisite**

- A syslog service, such as rsyslog, is installed on the remote host.
- The remote syslog service is configured to receive log entries from this host.

**Flow**

1. Install the `netconsole-service` package:
    ```
    # yum install netconsole-service
    ```
2. Edit the `/etc/sysconfig/netconsole` file and set the `SYSLOGADDR` parameter to the IP address of the remote host:
    ```
    # SYSLOGADDR=192.0.2.1
    ```
3. Enable and start the netconsole service:
    ```
    # SYSLOGADDR=192.0.2.1
    ```

**Verify**

- Display the /var/log/messages file on the remote syslog server.

# 30 Systemd network objects and services

NetworkManager configures the network during system boot. However, when booting with remote root (/), for example, if the root directory is stored on an iSCSI device, the network settings are applied on the initial RAM disk (initrd) before RHEL starts. For example, if the network configuration is specified on the kernel command line with rd.neednet=1, or if the configuration is specified to mount a remote filesystem, the network settings will be applied on the initrd.

This chapter describes the different targets used when applying network settings, such as the `network`, `network-online`, and `NetworkManager-wait-online` services, and how to configure systemd services to start after the `network-online` service starts.

## 30.1 Differences between network and network-online systemd targets

`systemd` maintains `network` and `network-online` target units. Special units, such as `NetworkManager-wait-online`.service, have `WantedBy=network-online.target` and `Before=network-online.target` parameters. If enabled, these units will start `network-online.target`, and they will delay the `network-online` target until the network is connected.

The `network-online` target starts a service, which adds a longer delay to further execution. systemd will automatically use the `Wants` and `After` parameters of this target unit to add dependencies to all System V (SysV) init script service units that have a Linux Standard Base (LSB) header pointing to the `$network` facility. The LSB header is metadata for the init script. You can use it to specify dependencies. This is similar to systemd targets.

The network target does not significantly delay the execution of the bootstrap process. Reaching the network target means that the service responsible for setting up the network has started. But it does not mean that a network device has been configured. This goal is very important in the process of shutting down the system. For example, if you have a service that comes after the network target during boot, this dependency will be reversed during shutdown. The network will not be disconnected until the service is stopped. All mount units for remote network filesystems automatically start the `network-online` target unit and are sequenced after it.

Note that the `network-online` target unit is only useful during system startup. After the system has finished booting, this target does not track the presence of the network. Therefore, you cannot use network-online to monitor network connections. This target provides a one-time system startup concept.

## 30.2 NetworkManager-wait-online overview

Synchronous traditional network scripts go through all configuration files to set up the device. They apply all network-related configurations and keep the network online.

The NetworkManager-wait-online service waits for the timeout for the network to be configured. This network configuration involves plugging in Ethernet devices, scanning for Wi-Fi devices, and more. NetworkManager automatically activates the appropriate configuration set configured to start automatically. When automatic activation fails due to a DHCP timeout or similar event, NetworkManager may be busy for a certain period of time. Depending on the configuration, NetworkManager retries to activate the same configuration set or a different configuration set.

When startup is complete, all configuration sets are either disconnected or activated successfully. You can configure profiles to automatically connect. Here are some examples of parameters that set a timeout or define when a connection is considered active:

- connection.wait-device-timeout - set the timeout for the driver to detect the device
- ipv4.may-fail 和 ipv6.may-fail - Use an IP address family setting to activate, or whether a specific address family must already be configured.
- ipv4.gateway-ping-timeout - Delay activation.

## 30.3 Configuring systemd services to start after the network is already up

OpenCloudOS installs systemd service files in the `/usr/lib/systemd/system/` directory. This process creates a drop-in for the service file in `/etc/systemd/system/service_name.service.d/`, which is used with the service file in `/usr/lib/systemd/system/` to Start a specific service. If a setting in a put section overlaps a setting in a service file in `/usr/lib/systemd/system/`, it takes precedence.


**Flow**

1. To open a service file in an editor, enter:
    ```
    # systemctl edit service_name
    ```
2. Enter the following and save changes:
    ```
    [Unit]
    After=network-online.target
    ```
3. Enter the following and save changes:
    ```
    # systemctl daemon-reload
    ```