[![hacs_badge](https://img.shields.io/badge/HACS-Custom-41BDF5.svg?style=for-the-badge)](https://github.com/hacs/integration).
<!-- [![hacs_badge](https://img.shields.io/badge/HACS-Default-orange.svg)](https://github.com/custom-components/hacs)
![Version](https://img.shields.io/github/v/release/mdeweerd/zha-toolkit)
![Downloads](https://img.shields.io/github/downloads/release/mdeweerd/zha-toolkit/total)
-->
# Highlights

* Read Zigbee attributes into Home Assistant attributes
* Daily ZNP Coordinator backup (See blueprint)
* "Low level" access to most Zigbee commands (read/write/report/cmd/discover)

# Purpose

Using the Home Assistant 'Services' feature, provide direct control over
low level zigbee commands provided in ZHA or Zigpy
that are not otherwise available or too limited for some use cases.

Can also serve as a framework to do local low level coding (the modules are
reloaded on each call).

Provide access to some higher level commands such as ZNP backup (and restore).

Make it easier to perform one-time operations where (some) Zigbee
knowledge is sufficient and avoiding the need to understand the inner
workings of ZHA or Zigpy (methods, quirks, etc).


# Setup

The component files needs to be added to your `custom_components` directory
either manually or using [HACS](https://hacs.xyz/docs/setup/prerequisites)
([Tutorial](https://codingcyclist.medium.com/how-to-install-any-custom-component-from-github-in-less-than-5-minutes-ad84e6dc56ff)).

Then, the integration is only available in Home Assistant after adding 
the next line to `configuration.yaml`, and restarting Home Assistant.
```yaml
zha-toolkit:
```

Before restarting, you may also want to enable debug verbosity.  `zha-toolkit`
isn't verbose when you use it occasionnaly.  As it's a service, there is
no really good way to inform the user about errors other than the log.

Logging will help verify that the commands you send have the desired effect.

Add/update the logger configuration (in the `configuration.yaml` file):
```yaml
logger:
  log:
    custom_components.zha-toolkit: debug
```


You can also change the log configuration dynamically by calling the
`logger.setlevel` service.
Example that sets the debug level for this `zha-toolkit` component and for
zigpy.zcl` (which helps to see some information about actual ZCL frames sent).
This method allows you to enable debug logging only for a limited duration :

```yaml
service: logger.set_level
data: 
    custom_components.zha-toolkit: debug
    zigpy.zcl: debug
```

# Automations

This is a list (of 1) automation:

* DAILY BACKUP OF ZNP DONGLE: [![Open your Home Assistant instance and show the Daily Backup Blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fmdeweerd%2Fzha-toolkit%2Fdev%2Fblueprints%2Fbackup_znp.yaml)


# Using `zha-toolkit`

This components provides a single service (`zha-toolkit.execute`) that
provides several commands (`command` parameter) providing access
to ZHA/Zigbee actions that are not otherwise available.


You can use a service as an action in automations.  So you can send the
commands according to a schedule or other triggers.  For instance, you
could plan a daily backup of your TI-ZNP USB Key configuration.

It will be more common to send a Zigbee command only once: for instance
bind one device to another, set a manufacturer attribute, ... .  
You can perform them using the developer tools.  
The developer tools are handy to test the service first before adding
them to an automation.

Go to Developer Tools > Services in your instance : 
[![Open your Home Assistant instance and show your service developer tools.](https://my.home-assistant.io/badges/developer_services.svg)](https://my.home-assistant.io/redirect/developer_services/).

Choose `zha-toolkit.execute` as the service.  
Enable Yaml entry.  

There are several examples below for different commands.  You can
copy/paste them to start from.

Not all available commands are documented.  The undocumented ones
were in the original repository.  
Some of these undocumented commands seem to be very specific trials
from the original authors.  
Feel free to propose documentation updates.


# Examples

Examples are work in progress and may not be functionnal.

For sleepy devices (on a battery) you may need to wake them up
just after sending the command so that they can receive it.

The 'ieee' address can be the IEEE address, the short network address
(0x1203 for instance), or the entity name (example: "light.tz3000_odygigth_ts0505a_12c90efe_level_light_color_on_off").  Be aware that the network address can change over
time but it is shorter to enter if you know it.


## `scan_device`: Scan a device/Read all attribute values

This operation will get all values for the attributes discovered
on the device.

The result of the scan is written to the `scan` directory located
in the configuration directory of Home Assistant (`config/scan/*_result.txt`).


```yaml
service: zha-toolkit.execute
data:
  ieee: 00:12:4b:00:22:08:ed:1a
  command: scan_device
```

Scan using the entity name:

```yaml
service: zha-toolkit.execute
data:
  command: scan_device
  ieee: light.tz3000_odygigth_ts0505a_12c90efe_level_light_color_on_off
```

## `zdo_scan_now`: Do a topology scan

Runs `topology.scan()`. 

```yaml
service: zha-toolkit.execute
data:
  command: zdo_scan_now
```

## `bind_ieee`: Bind matching cluster to another device

Binds all matching clusters (within the scope of the integrated list)

```yaml
service: zha-toolkit.execute
data:
  ieee: 00:15:8d:00:04:7b:83:69
  command: bind_ieee
  command_data: 00:12:4b:00:22:08:ed:1a

```

## `handle_join`: Handle join - rediscover device

```yaml
service: zha-toolkit.execute
data:
  # Address of the device that joined
  ieee: 00:12:4b:00:22:08:ed:1a
  command: handle_join
  # NWK address of device that joined (must be exact)
  command_data: 0x604e

```

## `attr_read`: Read an attribute value

Read a zigbee attribute value, optionnally write to a state.

```yaml
service: zha-toolkit.execute
data:
  command: attr_read
  ieee: sensor.zigbee_sensor
  # The endpoint is optional - when missing tries to find endpoint matching the cluster
  # endpoint: 1
  cluster: 0xb04
  attribute: 0x50f
  # Optional, state to write the read value to
  state_id: sensor.test
  # Optional, state attribute to write the value to, when missing: writes state itself
  state_attr: option
  # Optional, when true, allows creating the state (if not the state must exist)
  allow_create: True
  # The manufacturer should be set only for manufacturer attributes
  manf: 0x12

```

## `attr_write`: Write(/Read) an attribute value

Write an attribute value to any endpoint/cluster/attribute.

You can provide the numerical value of the attribute id,
or the internal zigpy name (string).

Before and after writing the value is read from the attribute.
If debug logging is active, this will be visible in the `home_assistant.log`.
The last read this can be written to a state.

```yaml
service: zha-toolkit.execute
data:
  ieee: 5c:02:72:ff:fe:92:c2:5d
  command: attr_write
  # Data: Endpoint, Cluster ID, Attribute Id, Attribute Type, Attribute Value, Optional Manf
  command_data: 11,0x0006,0x0000,0x10,
```


Alternate method, not using `command_data` but individual parameters.
(In case `command_data` is used, the more specific paramters override the values)


```yaml
service: zha-toolkit.execute
data:
  command: attr_write
  ieee: 5c:02:72:ff:fe:92:c2:5d
  # The endpoint is optional - when missing tries to find endpoint matching the cluster
  endpoint: 11
  cluster: 0x1706
  attribute: 0x0000
  attr_type: 0x41
  # Example of octet strings (the length is added because of attr_type)
  attr_val:  [41,33,8,45,52,46,50,191,55,57,136,60,100,102,63]
  # Optional manufacturer Id
  manf: 0x1021
  # Optional, state to write the read value to
  state_id: sensor.test
  # Optional, state attribute to write the value to, when missing: writes state itself
  state_attr: option
  # Optional, when true, allows creating the state (if not the state must exist)
  allow_create: True
  # The manufacturer should be set only for manufacturer attributes
  manf: 0x1202
  # You can set the next events to use as a trigger.
  # The event data has the result of the command (currently attr_read, attr_write)
  event_success: my_read_success_trigger_event
  event_fail: my_read_fail_trigger_event
  event_done: my_read_done_trigger_event
  # Settings for attr_write
  # Read attribute before writing it (defaults to True)
  read_before_write: True
  # Read attribute after writing it (defaults to True)
  read_after_write: True
  # Write attribute when the read value matches (defaults to False)
  write_if_equal: False
```


## `conf_report`: Configure reporting

Set the minimum and maximum delay between two reports and
set the level of change required to report a value (before the maximum
delay is expired).

This example configures Temperature reporting on a SonOff SNZB-02 (eWeLink/TH01).
Note that you (may) need to press the button on the thermometer just after
requesting the command (it's a sleepy device and does not wake up often).

After succeeding the configuration, the minimum delay was actually 20s
which is likely the measurement period itself.
The changes were reported when they exceeded 0.10 degrees C.

For sleepy devices, you can add the parameter 'tries' which will retry
until the devices confirms (with success or error)

```yaml
service: zha-toolkit.execute
data:
  command: conf_report
  ieee: 00:12:4b:00:23:b3:da:a5
  # Optional endpoint, when missing will match cluster 
  # endpoint: 1
  cluster: 0x402
  attribute: 0x0000
  min_interval: 60
  max_interval: 300
  reportable_change: 10
  # Optional manufacturer 
  #manf: 0x1204
  # Optional number of configuration attempts
  tries: 3
  # You can set the next events to use as a trigger.
  # The event data has the result of the command (currently attr_read, attr_write)
  event_success: my_conf_success_trigger_event
  event_fail: my_conf_fail_trigger_event
  event_done: my_conf_done_trigger_event
```

Exemple of data available in the event report.

```json
{
    "event_type": "my_conf_done_trigger_event",
    "data": {
        "ieee": "00:12:4b:00:24:42:d1:dc",
        "command": "conf_report",
        "start_time": "2022-01-16T21:56:21.393322+00:00",
        "params": {
            "cmd_id": null,
            "endpoint_id": 1,
            "cluster_id": 513,
            "attr_id": 0,
            "attr_type": null,
            "attr_val": null,
            "min_interval": 60,
            "max_interval": 300,
            "reportable_change": 10,
            "dir": null,
            "manf": null,
            "tries": 3,
            "expect_reply": true,
            "args": [],
            "state_id": "sensor.test",
            "state_attr": null,
            "allow_create": true,
            "event_success": "my_conf_success_trigger_event",
            "event_fail": "my_conf_fail_trigger_event",
            "event_done": "my_conf_done_trigger_event",
            "read_before_write": true,
            "read_after_write": true,
            "write_if_equal": false
        },
        "result_conf": [
            [
                {
                    "status": 0,
                    "direction": null,
                    "attrid": null
                }
            ]
        ]
    },
    "origin": "LOCAL",
    "time_fired": "2022-01-16T21:56:28.248353+00:00",
    "context": {
        "id": "596b9ba7b29d76545295881ea73c5708",
        "parent_id": null,
        "user_id": null
    }
}
```


## `zcl_cmd`: Send a Cluster command

Allows you to send a cluster command.
Also accepts command arguments.

Note:  
There is also the official core service `zha.issue_zigbee_cluster_command`.  You may want to use that instead if it suits your needs.  
The `zha-toolkit` version allows lists of bytes as arg paramters, and has a hack to allow "Add Scene".  It is also easier to adapt than the core that has though release procedures and is not as easily modifiable as a `custom_component`.


```yaml
service: zha-toolkit.execute
data:
  # Device IEEE address - mandatory
  ieee: 5c:02:72:ff:fe:92:c2:5d
  # Service command - mandatory
  command: zcl_cmd
  # Command id - mandatory
  cmd: 0
  # Cluster id - mandatory
  cluster: 1006
  # Endpoint - mandatory
  endpoint: 111
  # Optional: direction (0=to in_cluster (default), 1=to out_cluster),
  dir: 0
  # Optional: expect_reply  (default=true - false when 0 or 'false')
  expect_reply: true 
  # Optional: manf - manufacturer - default : None
  manf: 0x0000
  # Optional: tries - default : 1
  tries: 1
  # Optional (only add when the command requires it): arguments (default=empty)
  args: [ 1, 3, [ 1, 2, 3] ] 

```


### `zcl_cmd` Example: Send `on` command to an OnOff Cluster.

```yaml
service: zha-toolkit.execute
data:
  ieee: 5c:02:72:ff:fe:92:c2:5d
  command: zcl_cmd
  cmd: 1
  cluster: 6
  endpoint: 11
```


### `zcl_cmd` Example: Send `off` command to an OnOff Cluster:

```yaml
service: zha-toolkit.execute
data:
  ieee: 5c:02:72:ff:fe:92:c2:5d
  command: zcl_cmd
  cmd: 0
  cluster: 6
  endpoint: 11
```

### `zcl_cmd` Example: "Store Scene"

```yaml
service: zha-toolkit.execute
data:
  ieee: 5c:02:72:ff:fe:92:c2:5d
  command: zcl_cmd
  cmd: 4
  cluster: 5
  endpoint: 11
  args: [ 2, 5 ]
```

### `zcl_cmd` Example: "Recall Scene"
```yaml
service: zha-toolkit.execute
data:
  ieee: 5c:02:72:ff:fe:92:c2:5d
  command: zcl_cmd
  cmd: 5
  cluster: 5
  endpoint: 11
  args: [ 2, 5 ]
```

Results in (sniffed):
```raw
ZigBee Cluster Library Frame
    Frame Control Field: Cluster-specific (0x01)
        .... ..01 = Frame Type: Cluster-specific (0x1)
        .... .0.. = Manufacturer Specific: False
        .... 0... = Direction: Client to Server
        ...0 .... = Disable Default Response: False
    Sequence Number: 94
    Command: Recall Scene (0x05)
    Payload
        Group ID: 0x0002
        Scene ID: 0x05
```

### `zcl_cmd` Example: "Add Scene"

This example shows that you can provide a list of bytes for an argument:

```yaml
service: zha-toolkit.execute
data:
  ieee: 5c:02:72:ff:fe:92:c2:5d
  command: zcl_cmd
  cmd: 0
  cluster: 5
  endpoint: 11
  args:
    - 2
    - 5
    - 2
    - "Final Example"
    # Two bytes of cluster Id (LSB first), length, attribute value bytes
    #   repeat as needed (inside the list!)
    - [ 0x06, 0x00, 1, 1 ]
```

sniffed as:
```raw
ZigBee Cluster Library Frame
    Frame Control Field: Cluster-specific (0x01)
        .... ..01 = Frame Type: Cluster-specific (0x1)
        .... .0.. = Manufacturer Specific: False
        .... 0... = Direction: Client to Server
        ...0 .... = Disable Default Response: False
    Sequence Number: 76
    Command: Add Scene (0x00)
    Payload, String: Final Example
        Group ID: 0x0002
        Scene ID: 0x05
        Transition Time: 2 seconds
        Length: 13
        String: Final Example
        Extension Set: 06000101
```


## `znp_nvram_backup`: Backup ZNP NVRAM data

The output is written to the customisation directory as `local/nvram_backup.json`
when `command_data` is empty or not provided.  When `command_data` is provided,
it is added just after nvram_backup.

Note: currently under test.


```yaml
service: zha-toolkit.execute
data:
  command: znp_nvram_backup
  # Optional command_data, string added to the basename.
  # With this example the backup is written to `nwk_backup_20220105.json`
  command_data: _20220105
```

## `znp_nvram_restore`: Restore ZNP NVRAM data

Will restore ZNP NVRAM data from `local/nvram_backup.json` where `local`
is a directory in the `zha-toolkit` directory.

Note: currently under test.

For safety, a backup is made of the current network before restoring
`local/nvram_backup.json`.  The name of that backup is according to the format
`local/nvram_backup_YYmmDD_HHMMSS.json`.

```yaml
service: zha-toolkit.execute
data:
  command: znp_nvram_restore
```


## `znp_nvram_reset`: Reset ZNP NVRAM data

Will reset ZNP NVRAM data from `local/nvram_backup.json` where `local`
is a directory in the `zha-toolkit` directory.

Note: currently under test.

For safety, a backup is made of the current network before restoring
`local/nvram_backup.json`.  The name of that backup is according to the format
`local/nvram_backup_YYmmDD_HHMMSS.json`.


```yaml
service: zha-toolkit.execute
data:
  command: znp_nvram_reset
```


## `znp_backup`: Backup ZNP network data 

Used to transfer to another ZNP key later, backup or simply get network key
and other info.

The output is written to the customisation directory as `local/nwk_backup.json`
when `command_data` is empty or not provided.  When `command_data` is provided,
it is added just after nwk_backup.

You can use the blueprint to setup daily backup: [![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fmdeweerd%2Fzha-toolkit%2Fdev%2Fblueprints%2Fbackup_znp.yaml).


The name of that backup is according to the format

```yaml
service: zha-toolkit.execute
data:
  command: znp_backup
  # Optional command_data, string added to the basename.
  # With this example the backup is written to `nwk_backup_20220105.json`
  command_data: _20220105
```

## `znp_restore`: Restore ZNP network data

Will restore network data from `local/nwk_backup.json` where `local`
is a directory in the `zha-toolkit` directory.

Note: currently under test.

For safety, a backup is made of the current network before restoring
`local/nwk_backup.json`.  The name of that backup is according to the format
`local/nwk_backup_YYmmDD_HHMMSS.json`.


A typical use for this is when you migrate from one key to another.

The procedure should be:
1. Backup using the `znp_backup` command in the `zha-toolkit` service.
   Verify that the `nwk_backup.json` file is generated in the `local`
   directory.
2. 
   1. Remove the original Coordinator from your system (e.g., remove the USB key, ...).  
   2. Insert the new Coordinator.
   3. *Only when migrating to a Coordinator with different port/serial path/socket.*  
      Remove/Disable the ZHA Integration from Home Assistant.  
      The alternative is to modify HA’s config file directly to update
      the current integration’s serial path and baudrate
   4. Copy the zigbee.db file (for backup).  
      Moving/renaming it should not be needed.  If you Move or Rename
      the `zigbee.db` the Entity name are lost after the restore 
      (which impacts your automations, UI, etc).
3. 
   1. Restart Home Assistant.
   2. Enable/Add the ZHA Integration to Home Assistant
      (needed if you disabled or removed the ZHA integration in step 2.iii.)
4. Restore using the `znp_restore` command.  
   (If you used a custom file name for the backup then make sure you copy
    it to `nwk_backup.json`).
7. Check the logs (currently the `pre_shutdown` call failed for the first
   successful test, but that is not critical).
8. Restart HA
7. Check that everything is ok.  

**NOTES :**

- Devices may take a while to rejoin the network as the Zigbee
  specification requires them to "back-off" in case of communication
  problems.  
- You may speed up the process by power cycling devices.
- Devices may not be instantly responsive because the zigbee mesh
  needs to be recreated (try the `zdo_scan_now` command to speed
  that up).

(See the [Home Assistant Community Forum](https://community.home-assistant.io/t/zha-custom-service-to-send-custom-zha-commands-extra-functions/373346/33) for a success story.)


```yaml
service: zha-toolkit.execute
data:
  command: znp_restore
  # Optional:
  #  command_data = Counter_increment (for tx).
  #                 defaults to 2500
  command_data: 2500
```


# Credits/Motivation

This project was forked from [Adminiguaga/zha_custom](https://github.com/Adminiuga/zha_custom)
where the "hard tricks" for providing services and accessing ZHA functions were
implemented/demonstrated.  The original codeowners were "[dmulcahey](https://github.com/dmulcahey)"
and "[Adminiuga](https://github.com/adminiuga)".

The initial purpose of this fork was mainly to add custom attribute writes,
custom reporting and more binding possibilities.

The structure was then updated to be compliant with HACS integration so that
the component can be easily added to a Home Assistant setup.

# License

I set the License the same as Home Assistant that has the ZHA component.
The original zha_custom repository does not mention a license.

# Contributing 

## Adding commands/documentation

Feel free to propose documentation of the available commands (not all are documented
above) or propose new commands.

To add a new command, one has to add the `command_handler` to the `__init__.py`
file and the actual command itself to an existing module or a new module if that
is appropriate.
Also add a short description of the command in this `README.md` .


Example of addition to `__init__.py`:

```python
def command_handler_znp_backup(*args, **kwargs):
    """ Backup ZNP network information. """
    from . import znp

    importlib.reload(znp)

    return znp.znp_backup(*args, **kwargs)
```

Anything after `command_handler_` is used to match the `command` parameter
to the service - simply adding a function with such a name "adds" the
 corresponding command.

The `reload` in the code above allows you to change the contents of the
module and test it without having to restart Home Assistant.

The code above imports and reloads `znp.py` and then calls `znp_backup`
in that module.

All methods take the same parameters.  'args' and 'kwargs` do some python
magic to ppropagate a variable number of fixed and named parameters.  But 
in the end the method signature has to look like this:

```python
async def znp_backup(app, listener, ieee, cmd, data, service):
    """ Backup ZNP network information. """

```

Where `app` is the `zigpy` instance and `listener` is the gateway instance.
`ieee`, `cmd` and `data` correspond to the parameters provided to the service.  
You can examine some of the existing code how you can use them.  
Possibly `data` could be more than a string, but that has not been validated for now.

Then you have to import the modules you require in the function - or
add/enable them as imports at the module level.

You can also run `flake8` on your files to find some common basic errors
and provide some code styling consistency.

As far as ZHA and zigpy are concerned, you can find the code for the
ZHA integration at
https://github.com/home-assistant/core/tree/dev/homeassistant/components/zha ,
and the `zigpy` repositories are under https://github.com/zigpy .