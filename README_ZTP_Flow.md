
## SONiC ZTP Execution Flow

The [ZTP process in SONiC](https://github.com/sonic-net/SONiC/blob/master/doc/ztp/ztp.md) follows a structured workflow that determines how configuration artifacts are discovered and applied.

### Service Initialization and Pre-Flight Checks

When the ZTP service starts during the system boot process, it does not immediately look for network provisioning. Instead, it performs a strict sequence of local checks to determine if provisioning is necessary or if it should gracefully exit. To prevent overwriting an active device configuration, the ZTP engine evaluates the following conditions in order of precedence:

- **Administrative Override**: It first checks if ZTP has been administratively disabled (`admin-mode` is False). If so, the service terminates immediately.

- **In-Progress Sessions**: It checks if there is already a ZTP session in progress from a previous boot.

- **Warm Boot**: It reads the boot parameters (`/proc/cmdline`). If the switch is performing a warm boot (a reboot that preserves the data plane forwarding state), ZTP is skipped entirely.

- **Mini-graph Presence**: It checks for the existence of `/etc/sonic/minigraph.xml`. Minigraph is a declarative XML file used in some SONiC environments to generate system configurations. If this file is present, the switch assumes it is already managed, sets the mode to `MANUAL_CONFIG`, and terminates ZTP.

- **Startup Configuration**: Finally, it checks for an existing `config_db.json` file. Crucially, this check is gated by a setting called `monitor-startup-config` (which is enabled by default). If `config_db.json` exists and the monitor setting is true, the switch assumes it is already deployed, skips network discovery, and operates using its existing configuration.

### Local File Check (`Local-FS` Mode)

If the switch passes all the pre-flight checks (meaning it is a fresh boot with no existing configuration or mini-graph), the ZTP engine checks for a pre-staged, local provisioning file before attempting to contact the network. Specifically, it looks for the existence of the file `/host/ztp/ztp_data_local.json`. If this file exists, the switch sets its ZTP source to `local-fs` and entirely bypasses the DHCP network discovery phase. It will then load the JSON file and evaluate its status to begin processing the local instructions.

**USB Provisioning**

The SONiC ZTP codebase does not inherently mount or search removable media. However, USB provisioning is still possible through external processes. If an administrator uses a custom boot script, or if the ONIE installer mounts a plugged-in USB drive and copies the `ztp_data_local.json` file into the `/host/ztp/` directory before the ZTP service starts, the ZTP engine will detect the file just like any other local file and execute it.

### DHCP Discovery

If neither a startup configuration nor a local ZTP file is found, the switch falls back to its network discovery phase. It attempts to obtain provisioning instructions by broadcasting DHCP discovery messages. To ensure it can reach a provisioning server regardless of the network topology, the ZTP service scans across all available interfaces including both out-of-band management ports (eth*) and in-band front-panel interfaces (Ethernet*) using both IPv4 and IPv6.

To help the DHCP server identify the request, the ZTP service includes DHCP Option 77, containing the user-class identifier `SONiC-ZTP`. This identifier allows the server to recognize the requesting client as a SONiC device and respond with the appropriate payload. Additionally, the request includes a client identifier (Option 61) constructed using the device’s specific hardware SKU and serial number, allowing the server to map the physical switch to its unique configuration file.

<img src="pics/ztp-dhcp-discover.png" alt="segment" width="600">

### DHCP Offer

Once a DHCP server receives the discovery request, it responds with a DHCP offer that includes network parameters and potentially the provisioning URL. The switch selects the first interface that successfully receives an offer and treats it as the primary management interface.

<img src="pics/ztp-dhcp-offer.png" alt="segment" width="650">

The switch specifically listens for several DHCP options to locate its configuration payload, including:

- Option 66: TFTP Server
- Option 67: IPv4 Bootfile Name
- Option 59: IPv6 Bootfile URL
- Option 239: Custom Provisioning Script for v4 and v6

### Payload Download and Staging

After confirming the DHCP parameters, the switch contacts the provisioning server and downloads the `ztp.json`.

<img src="pics/tftp-1.png" alt="segment" width="1000">

If the payload is a JSON file, it is stored locally on disk to `/host/ztp/ztp_data.json` so the ZTP engine can process it.

```text
$ cat /host/ztp/ztp_data.json
{
    "ztp": {
        "01-configdb-json": {
            "exit-code": 0,
            "halt-on-failure": false,
            "ignore-result": false,
            "reboot-on-failure": false,
            "reboot-on-success": false,
            "start-timestamp": "2026-03-08 22:24:47 UTC",
            "status": "SUCCESS",
            "timestamp": "2026-03-08 22:25:08 UTC",
            "url": {
                "destination": "/etc/sonic/config_db.json",
                "source": "tftp://10.10.10.2/config_db.json"
            }
        },
        "config-fallback": false,
        "halt-on-failure": false,
        "ignore-result": false,
        "reboot-on-failure": false,
        "reboot-on-success": false,
        "restart-ztp-no-config": true,
        "restart-ztp-on-failure": false,
        "start-timestamp": "2026-03-08 22:24:37 UTC",
        "status": "SUCCESS",
        "timestamp": "2026-03-08 22:25:08 UTC",
        "ztp-json-source": "dhcp-opt67 (eth0)",
        "ztp-json-version": "1.0"
    }
}
```

In SONiC, saving the ZTP payload to `/host/ztp/ztp_data.json` is essential because the operating system utilizes an ephemeral Overlay Filesystem. While normal directories are tied to a temporary read-write layer that gets wiped out during a NOS image upgrade, the `/host` directory serves as a direct, persistent mount to the switch's underlying physical storage. By storing the configuration data here, SONiC ensures that the ZTP payload safely survives the reboots and firmware installations inherent to the provisioning process, allowing the ZTP engine to resume its tasks on the newly installed system without losing its instructions.

The service then parses the JSON structure and creates internal section files representing each configuration task. These section files allow the ZTP engine to track execution progress and maintain state across the provisioning workflow.

```text
$ cat /var/lib/ztp/sections/01-configdb-json/input.json
{
    "01-configdb-json": {
        "halt-on-failure": false,
        "ignore-result": false,
        "reboot-on-failure": false,
        "reboot-on-success": false,
        "status": "BOOT",
        "timestamp": "2026-03-08 22:24:37 UTC",
        "url": {
            "destination": "/etc/sonic/config_db.json",
            "source": "tftp://10.10.10.2/config_db.json"
        }
    }
}
```

### Processing and Artifact Execution

Each configuration section defined in the ZTP JSON file is processed sequentially. The ZTP engine invokes the appropriate **plugin** to execute the action described in the section. Depending on the configuration, this may involve downloading additional files, executing scripts, installing software images, or updating configuration databases.

<img src="pics/tftp-2.png" alt="segment" width="1000">

The exit code returned by the plugin determines whether the step succeeded or failed. An exit code of zero indicates success, while non-zero values indicate an error that may trigger additional error-handling logic.

### Exit Conditions and Restart Logic

Once all sections have been processed, the ZTP service evaluates the overall result. The process is marked as successful only if all required steps complete successfully. If the provisioning process successfully generates a configuration database file, the switch continues operating with the new configuration. If the configuration file is still missing after execution, the ZTP engine restarts network discovery and attempts to retrieve a valid configuration again. Administrators can also manually interrupt the process using CLI commands, in which case the system falls back to a default configuration.

## Generic Structure of `ztp.json`

The `ztp.json` file serves as a structured manifest that defines the tasks the switch must perform during provisioning. The file begins with a mandatory root object named "ztp". If this object is missing, the ZTP engine ignores the file and terminates the provisioning attempt.

Inside this root object, individual provisioning tasks are defined as "configuration sections". Each section describes a specific action such as installing firmware, downloading configuration files, or executing scripts. The ZTP service processes these sections sequentially based on their lexical order. For this reason, administrators typically prefix section names with numeric identifiers to control the execution order (e.g., `01-firmware`, `02-configdb-json`, `03-script`).

Inside each section, you define the parameters for that specific task. You can also include global modifiers like `"ignore-result": true` (which tells ZTP to continue to the next step even if the current one fails) or `"reboot-on-success": true`.

```json
{
  "ztp": {
    "01-firmware": {
      "install": {
        "url": "http://10.0.0.1/sonic-image.bin"
      }
    },
    "02-configdb-json": {
      "url": {
        "source": "tftp://10.0.0.1/config_db.json",
        "destination": "/etc/sonic/config_db.json"
      }
    }
  }
}
```

## ZTP Plugins

While the `ztp.json` file defines the desired provisioning workflow, the actual execution of each task is handled by ZTP plugins. A plugin is an executable component responsible for processing the instructions contained within a configuration section. When the ZTP engine encounters a section, it passes the section’s data to the appropriate plugin, which performs the requested action and returns a status code indicating success or failure.

SONiC includes several predefined plugins that support common provisioning operations. For example, the `configdb-json` plugin installs a new configuration database and reloads the switch configuration, while the `firmware` plugin installs a new SONiC image on an alternate boot partition. Other built-in plugins perform tasks such as verifying network connectivity or configuring SNMP parameters. You can find a list [here](https://github.com/sonic-net/SONiC/blob/master/doc/ztp/ztp.md#323-available-plugins-and-configuration-sections).

In addition to these built-in plugins, SONiC also supports custom plugins defined by the operator. These plugins are typically implemented as scripts that perform environment-specific actions, such as registering the device with an inventory system or installing additional monitoring agents. During the ZTP process, the switch downloads the custom script, executes it with the provided parameters, and evaluates its exit code to determine whether the task completed successfully.

### ZTP Plugin Execution and Security

When the ZTP engine processes a configuration payload, it executes plugins according to the following behavioral and security parameters:

- **Root Privileges and Unrestricted Access**: The core ZTP service operates with full administrative rights (`User=root`, `Group=root` via systemd). Any plugins spawned by the ZTP engine automatically inherit these root privileges. Because there is no sand-boxing, privilege separation, or container isolation involved, plugins have unrestricted access to the underlying host operating system.

- **Execution Method** (Direct vs. Shell): By default, plugins are executed directly (via `exec` rather than a shell environment). This is a security measure designed to prevent shell injection attacks from manipulated provisioning data. However, if a specific task requires a shell environment, administrators can explicitly enable it by adding `"shell": true` to that plugin's section in the ZTP JSON definition.

- **Synchronous Processing**: Plugin execution is strictly synchronous and blocking. The ZTP engine reads the JSON payload sequentially, waiting for each plugin to fully complete its task before moving on to the next section.

- **File Permissions** (Umask): The core ZTP engine operates with a highly restrictive umask of 177. However, to ensure plugins can successfully create necessary files and directories during the provisioning process, they are assigned a distinct, configurable umask (which defaults to a standard 022).

- **Graceful Termination**: If the ZTP service receives a termination signal (such as a system shutdown or service restart), it does not immediately terminate running plugins. Instead, it provides a 60-second grace period, allowing active plugins to finish their tasks cleanly before forcefully terminating them.
