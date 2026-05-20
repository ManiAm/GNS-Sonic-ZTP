
## Build SONiC VS Image with ZTP

In the default SONiC build configuration, the ZTP functionality is not always included in the generated image. As a result, attempting to query the ZTP status on such an image returns a message indicating that the feature is unavailable. To use ZTP in a virtual switch environment, the feature must be explicitly enabled during the image build process.

```bash
admin@sonic:~$ show ztp status
ZTP feature unavailable in this image version
```

ZTP support is controlled through the `ENABLE_ZTP` build flag in the sonic-buildimage repository. When this variable is set, the build system includes the ZTP service, scripts, and supporting components in the final SONiC image.

```bash
# 1. Export the variable first so it applies to the entire session
export ENABLE_ZTP=y

# 2. Initialize the environment
make init

# 3. Configure the build
make configure PLATFORM=vs

# 4. Run the final build
SONIC_BUILD_JOBS=8 make target/sonic-vs.img.gz
```

The resulting `sonic-vs.img.gz` image contains the ZTP service and can be imported into environments such as GNS3 for testing automated provisioning workflows. Sonic build process is explained in [here](https://github.com/ManiAm/net-lab-switch-setup/blob/master/docs/Sonic_Build.md).

## Verify ZTP is Active

After importing the SONiC VS image into GNS3 and starting the virtual switch, the ZTP service initializes during the system boot sequence. Log messages displayed in the console indicate that the ZTP service has started and is waiting for the system to reach an operational state before beginning network discovery.

```text
Debian GNU/Linux 12 sonic ttyS0

sonic login: 2026 Mar  8 19:05:14.360745 sonic INFO sonic-ztp[2324]: ZTP service started.
2026 Mar  8 19:05:14.360941 sonic INFO sonic-ztp[2324]: Checking running configuration to load ZTP configuration profile.
2026 Mar  8 19:05:14.678890 sonic INFO sonic-ztp[2795]: Waiting for system online status before continuing ZTP. (This may take 30--120 seconds).
2026 Mar  8 19:05:19.691185 sonic INFO sonic-ztp[2795]: System is ready to respond.
2026 Mar  8 19:05:19.697996 sonic INFO sonic-ztp[2324]: Link up detected for interface eth0
2026 Mar  8 19:05:19.698102 sonic INFO sonic-ztp[2324]: Restarting network discovery after link scan.
2026 Mar  8 19:05:24.362688 sonic INFO sonic-ztp[2324]: Restarted network discovery after link scan.
```

Unlike many SONiC components that run inside Docker containers, the ZTP service runs directly on the host operating system as a systemd service. It is implemented through a combination of shell scripts and Python code that orchestrate the provisioning workflow. The service starts automatically after several core services are initialized, including `interfaces-config`, `rc-local`, and `database` which are early boot components required for network functionality.

The status of the service can be inspected using standard systemd tools:

```bash
admin@sonic:~$ systemctl status ztp.service
● ztp.service - SONiC Zero Touch Provisioning service
     Loaded: loaded (/lib/systemd/system/ztp.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-03-08 19:05:12 UTC; 3min 20s ago
   Main PID: 2322 (sonic-ztp)
      Tasks: 2 (limit: 9400)
     Memory: 15.5M
     CGroup: /system.slice/ztp.service
             ├─2322 /bin/bash /usr/lib/ztp/sonic-ztp
             └─2324 /usr/bin/python3 /usr/lib/ztp/ztp-engine.py
```

Additionally, SONiC provides a CLI command, which displays operational information about the ZTP subsystem, including whether the service is active, the discovery status, and the runtime duration. At this stage the service is active and waiting to discover provisioning information.

```bash
admin@sonic:~$ show ztp status
ZTP Admin Mode : True
ZTP Service    : Active Discovery
Runtime        : 03m 46s
ZTP Status     : Not Started

ZTP Service is active
```
