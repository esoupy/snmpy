SNMPY
=====

SNMPy extends a running [net-snmp](http://www.net-snmp.org) agent with a custom subtree made out of configurable plugin modules. It makes extensive use of `libnetsnmp` C library to implement an AgentX subagent.

Prerequisites
-------------

* Python 2.7+
* Python [yaml](http://pyyaml.org) module
* Working [net-snmp](http://www.net-snmp.org) agent
* [AgentX](http://net-snmp.sourceforge.net/docs/README.agentx.html) enabled

Installation
------------

Install SNMPy as a Debian package by building a `deb`:

    dpkg-buildpackage
    # or
    pdebuild

Install SNMPy using the standard `setuptools` script:

    python setup.py install

Administration
--------------

SNMPy can be run in foreground or as a backgrounded daemon.  It supports logging directly to console, file, or syslog.

```
usage: snmpy [-h] [-f CONFIG_FILE] [-i INCLUDE_DIR] [-t PERSIST_DIR]
           [-r PARENT_ROOT] [-s SYSTEM_ROOT] [-l LOGGER_DEST] [-w HTTPD_PORT]
           [-p [PID_FILE]] [-m [MIB_FILE]] [-d] [-e EXTRA EXTRA]

Modular SNMP AgentX system

optional arguments:
  -h, --help            show this help message and exit
  -f CONFIG_FILE, --config-file CONFIG_FILE
                        system configuration file
  -i INCLUDE_DIR, --include-dir INCLUDE_DIR
                        plugin configuration path
  -t PERSIST_DIR, --persist-dir PERSIST_DIR
                        plugin state persistence path
  -r PARENT_ROOT, --parent-root PARENT_ROOT
                        parent root class name
  -s SYSTEM_ROOT, --system-root SYSTEM_ROOT
                        system root object id
  -l LOGGER_DEST, --logger-dest LOGGER_DEST
                        filename, syslog:<facility>, or console:
  -w HTTPD_PORT, --httpd-port HTTPD_PORT
                        httpd server listens on this port
  -p [PID_FILE], --create-pid [PID_FILE]
                        daemonize and write pidfile
  -m [MIB_FILE], --create-mib [MIB_FILE]
                        display generated mib file and exit
  -d, --debug           enable debug logging
  -e EXTRA EXTRA, --extra EXTRA EXTRA
                        extra key/val pairs for plugins
```

The system starts by reading the global configuration file. Any options provided on the command-line override the config.

* If `-m | --create-mib` was specified, it writes the generated MIB to the specified file (or `stdout` if not set or set to `-`) and exits.
* If `-p | --create-pid` was specified, it daemonizes and writes the PID to the specified file (or `/var/run/snmpy.pid` if not set) and exits.
* It then loads all modules located in the directory specified by `-i | --include-dir` and builds an SNMP tree, initializes the agent, and enters an update loop.
* It also creates an HTTP server that currently only answers to `/mib` GET requests upon which, it will deliver the generated run-time MIB file contents.

Configuration
-------------

SNMPy configuration is [yaml](http://yaml.org) formatted.  It consists of one main file, specified by `-f | --config-file` on the command-line parameter, and many module configuration files located in the directory specified by `-i | --include-dir` command-line parameter or `include_dir` global setting.  These are explained below.

### global settings ###
All command-line parameters, shown above in the help screenshot, have a corresponding global setting.  For example `--parent-root` command-line parameter is `parent_root` global setting and so on.  The main configuration file must have any global settings under the `snmpy_global` top-level item in order to be recognized. i.e.

```yaml
snmpy_global:
    include_dir: '/etc/snmpy/conf.d'
    persist_dir: '/var/lib/snmpy'
    parent_root: 'ucdavis'          # .1.3.6.1.4.1.2021
    system_root: '1123'             # .1.3.6.1.4.1.2021.1123
    logger_dest: 'syslog:local1'
    create_pid:  '/var/run/snmpy.pid'
    httpd_port:  1123
    debug:       False
```

* `include_dir`: location for module configuration files (see module settings below).
* `persist_dir`: location for module persistence data files.
* `parent_root`: SNMP object under which to install the SNMPy subtree.
* `system_root`: SNMP OID where the SNMPy subtree starts. `SNMPY-MIB::snmpyMIB` is rooted here, which can be walked to retreive all data that it manages.
* `logger_dest`: Destination for the SNMPy log.  It can be a `/path/to/file`, `console:` for stdout, or `syslog:<facility>`.
* `httpd_port`: Port to listen for http requests.  This is also used to make sure only one SNMPy process is running on a system.
* `create_pid`: Location of the PID file.  If this is specified, SNMPy will daemonize and run in the background.
* `create_mib`: This shouldn't be speicified in the config file because SNMPy will just write a MIB file and exit.  If a MIB file is needed, use the `-m | --create-mib` command-line parameter, or simply download from a running agent via HTTP.

        curl -s http://localhost:1123/mib # where 1123 is the configured httpd_port

### module settings ###
SNMPy module configuration begins with the filename which must meet these criteria:

1. config file names begin with a system-unique and 0-padded 4 digit number followed by underscore
1. config file names are used for subtree MIB object names
1. config file names end with `.yml` or `.yaml`

For example, with a global config above and a module file `/etc/snmpy/conf.d/0023_dmidecode_system.yml`, SNMP data will be made available via:

* Numeric OID: `.1.3.6.1.4.1.2021.1123.23`
* Symbolic OID: `SNMPY-MIB::snmpyDmidecodeSystem`

Every module configuration file must specify which plugin to use and how often, in minutes, to update.  Optionally, extra plugin-dependant configuration settings may be provided.  These are described below for each currently supported plugin.  At a minimum a module configuration file must contain these lines:

```yaml
module: # one of the plugins described below
period: # refresh time in minutes or "once" for startup collection only
```

Optionally every module may request its state be saved between SNMPy restarts by adding a top-level `retain` boolean key:

```yaml
retain: # true | false
```

Note: retain setting is ignored if `-r | --persist-dir` command-line parameter or `persist_dir` global configuration item is disabled.

Plugins
-------

SNMPy ships with several plugins ready for use, some of which are generic and can be applied toward many different use cases.  Several example configuration modules are available to demonstrate functionality.

### exec_table ###
The `exec_table` plugin provides tabular data from the output results of an executable command. Configuration items which must be specified are:

```yaml
module: exec_table
period: 5

object: '/path/to/command'
parser:
    type: 'regex'
    path:
        - 'First Item Pattern:\s+(?P<item_one>[^\n]+)'
        - 'Second Item Pattern:\s+(?P<item_two>[^\n]+)'
        - '(?P<other_item>(?:true|false) other item)'

table:
    - item_one:   'integer'
    - item_two:   'string'
    - other_item: 'string'
```

* `object`: Full path to executable command to run and parse output from.
* `parser`: Text parser to invoke for this plugin.
    * `type`: Currently the only type supported is `regex`, but `xml` and `json` may be supported in the future.
    * `path`: List of one or more Python [regular expressions](http://docs.python.org/3/library/re.html) that capture named groups which match column definitions (described below). Multiple matches become rows in the resulting SNMP table.
* `table`: defines the columns for this plugin.
    * item names: List of one or more columns each, specifying its type.

See [`interface_info.yml`](https://github.com/mk23/snmpy/blob/agentx/examples/interface_info.yml) example plugin:

    $ curl -s -o snmpy.mib http://localhost:1123/mib
    $ snmpwalk -m +./snmpy.mib -v2c -cpublic localhost SNMPY-MIB::snmpyInterfaceInfo
    SNMPY-MIB::snmpyInterfaceInfoInterface.1 = STRING: "eth0"
    SNMPY-MIB::snmpyInterfaceInfoInterface.2 = STRING: "eth1"
    SNMPY-MIB::snmpyInterfaceInfoSwitchName.1 = STRING: "sw-r07-03c"
    SNMPY-MIB::snmpyInterfaceInfoSwitchName.2 = STRING: "sw-r07-03c"
    SNMPY-MIB::snmpyInterfaceInfoSwitchPort.1 = STRING: "Gi1/0/28"
    SNMPY-MIB::snmpyInterfaceInfoSwitchPort.2 = STRING: "Gi1/0/42"
    SNMPY-MIB::snmpyInterfaceInfoLinkAuto.1 = STRING: "supported/enabled"
    SNMPY-MIB::snmpyInterfaceInfoLinkAuto.2 = STRING: "supported/enabled"
    SNMPY-MIB::snmpyInterfaceInfoLinkSpeed.1 = INTEGER: 1000
    SNMPY-MIB::snmpyInterfaceInfoLinkSpeed.2 = INTEGER: 1000
    SNMPY-MIB::snmpyInterfaceInfoLinkDuplex.1 = STRING: "full duplex mode"
    SNMPY-MIB::snmpyInterfaceInfoLinkDuplex.2 = STRING: "full duplex mode"

### exec_value ###
The `exec_value` plugin provides simple key-value data from the output results of an executable command.  Configuration items which must be specified are:

```yaml
module: exec_value
period: 5

object: '/path/to/command'

items:
    - item_one:
          type:  'integer'
          regex: 'First Item Pattern:\s+(.+?)$'
    - item_two:
          type:  'string'
          regex: 'Second Item Pattern:\s+(.+?)$'
    - other_item:
          type:  'string'
          regex: '((?:true|false) other item)'
```

* `object`: Full path to executable command to run and parse output from.
* `items`: defines key-value pairs for this plugin.
    * item names: List of one or more item definitions.
        * `type`: SNMP type for this item
        * `regex`: Python [regular expressions](http://docs.python.org/3/library/re.html) that captures a group for this item.

See [`dmidecode_bios.yml`](https://github.com/mk23/snmpy/blob/agentx/examples/dmidecode_bios.yml) example plugin:

    $ curl -s -o snmpy.mib http://localhost:1123/mib
    $ snmpwalk -m +./snmpy.mib -v2c -cpublic localhost SNMPY-MIB::snmpyDmidecodeBios
    SNMPY-MIB::snmpyDmidecodeBiosVendor = STRING: "innotek GmbH"
    SNMPY-MIB::snmpyDmidecodeBiosVersion = STRING: "VirtualBox"
    SNMPY-MIB::snmpyDmidecodeBiosRelease = STRING: "12/01/2006"

### file_table ###
The `file_table` plugin provides tabular data from the contents of a file and behaves just like the `exec_table` plugin except the object parameter refers to a file instead of a command.  Configuration items which must be specified are:

```yaml
module: file_table
period: 5

object: '/path/to/file'
parser:
    type: 'regex'
    path:
        - 'First Item Pattern:\s+(?P<item_one>[^\n]+)'
        - 'Second Item Pattern:\s+(?P<item_two>[^\n]+)'
        - '(?P<other_item>(?:true|false) other item)'

table:
    - item_one:   'integer'
    - item_two:   'string'
    - other_item: 'string'
```

* `object`: Full path to a file to read and parse.
* `parser`: Text parser to invoke for this plugin.
    * `type`: Currently the only type supported is `regex`, but `xml` and `json` may be supported in the future.
    * `path`: List of one or more Python [regular expressions](http://docs.python.org/3/library/re.html) that capture named groups which match column definitions (described below). Multiple matches become rows in the resulting SNMP table.
* `table`: defines the columns for this plugin.
    * item names: List of one or more columns each, specifying its type.

### file_value ###
The `file_value` plugin provides simple key-value data from the contents of a file and behaves similarly to the `exec_value` plugin except the object parameter refers to a file instead of a command and optionally enables file metadata.  Configuration items which must be specified are:

```yaml
module: file_value
period: 5

object: '/path/to/file'
use_stat: True # or False
use_text: True # or False

items:
    - item_one:
          type:  'integer'
          regex: 'First Item Pattern:\s+(.+?)$'
    - item_two:
          type:  'string'
          regex: 'Second Item Pattern:\s+(.+?)$'
    - other_item:
          type:  'string'
          regex: '((?:true|false) other item)'
```

* `object`: Full path to a file to read and parse.
* `use_stat`: Toggles file metadata (size, dates, permissions) in the results
* `use_text`: Toggles content parsing. If disabled, `items` section below is ignored.
* `items`: defines key-value pairs for this plugin.
    * item names: List of one or more item definitions.
        * `type`: SNMP type for this item
        * `regex`: Python [regular expressions](http://docs.python.org/3/library/re.html) that captures a group for this item.

See [`puppet_status.yml`](https://github.com/mk23/snmpy/blob/agentx/examples/puppet_status.yml) example plugin:

    $ curl -s -o snmpy.mib http://localhost:1123/mib
    $ snmpwalk -m +./snmpy.mib -v2c -cpublic localhost SNMPY-MIB::snmpyPuppetStatus
    SNMPY-MIB::snmpyPuppetStatusFileName = STRING: "/var/lib/puppet/state/last_run_summary.yaml"
    SNMPY-MIB::snmpyPuppetStatusFileType = STRING: "regular file"
    SNMPY-MIB::snmpyPuppetStatusFileMode = STRING: "0644"
    SNMPY-MIB::snmpyPuppetStatusFileAtime = INTEGER: 200
    SNMPY-MIB::snmpyPuppetStatusFileMtime = INTEGER: 32074
    SNMPY-MIB::snmpyPuppetStatusFileCtime = INTEGER: 32074
    SNMPY-MIB::snmpyPuppetStatusFileNlink = INTEGER: 1
    SNMPY-MIB::snmpyPuppetStatusFileSize = INTEGER: 574
    SNMPY-MIB::snmpyPuppetStatusFileIno = INTEGER: 279301
    SNMPY-MIB::snmpyPuppetStatusFileUid = INTEGER: 0
    SNMPY-MIB::snmpyPuppetStatusFileGid = INTEGER: 0
    SNMPY-MIB::snmpyPuppetStatusRuntime = INTEGER: 27
    SNMPY-MIB::snmpyPuppetStatusSuccess = INTEGER: 1
    SNMPY-MIB::snmpyPuppetStatusFailure = INTEGER: 0

### log_processor ###
The `log_processor` plugin provides simple key-value data from the contents of a constantly-appended log file, and behaves similarly to the `file_value` plugin except it is able to immediately react to new data as well as rotation events.  Configuration items which must be specified are:

```yaml
module: log_processor
period: 1

object: '/path/to/file'

items:
    - item_one:
          type:  'integer'
          regex: 'First Item Pattern:\s+(.+?)$'
    - item_two:
          type:  'string'
          regex: 'Second Item Pattern:\s+(.+?)$'
    - other_item:
          type:  'string'
          regex: '((?:true|false) other item)'
```

* `object`: Full path to a file to tail and parse.
* `items`: defines key-value pairs for this plugin.
    * item names: List of one or more item definitions.
        * `type`: SNMP type for this item
        * `regex`: Python [regular expressions](http://docs.python.org/3/library/re.html) that captures a group for this item.

See [`hbase_balancer.yml`](https://github.com/mk23/snmpy/blob/agentx/examples/hbase_balancer.yml) example plugin:

    $ curl -s -o snmpy.mib http://localhost:1123/mib
    $ snmpwalk -m +./snmpy.mib -v2c -cpublic localhost SNMPY-MIB::snmpyHbaseBalancer
    SNMPY-MIB::snmpyHbaseBalancerEnabled = "true"
    $ echo '[ignored text] BalanceSwitch=false [ignored text]' >> /var/log/hbase/hbase-hbase-master-localhost.log
    $ snmpwalk -m +./snmpy.mib -v2c -cpublic localhost SNMPY-MIB::snmpyHbaseBalancer
    SNMPY-MIB::snmpyHbaseBalancerEnabled = STRING: "false"

### process_info ###
The `process_info` plugin provides per-process information (open files, running threads, consumed memory) on the running system.  It does not need any extra configuration other than module name and refresh period:

```yaml
module: process_info
period: 1
```

See [`process_info.yml`](https://github.com/mk23/snmpy/blob/agentx/examples/process_info.yml) example plugin:

    $ curl -s -o snmpy.mib http://localhost:1123/mib
    $ snmpwalk -m +./snmpy.mib -v2c -cpublic localhost SNMPY-MIB::snmpyProcessInfo | grep '\.1 ='
    SNMPY-MIB::snmpyProcessInfoPid.1 = INTEGER: 4593
    SNMPY-MIB::snmpyProcessInfoPpid.1 = INTEGER: 4592
    SNMPY-MIB::snmpyProcessInfoName.1 = STRING: "python2.7"
    SNMPY-MIB::snmpyProcessInfoFdOpen.1 = INTEGER: 10
    SNMPY-MIB::snmpyProcessInfoFdLimitSoft.1 = INTEGER: 1024
    SNMPY-MIB::snmpyProcessInfoFdLimitHard.1 = INTEGER: 4096
    SNMPY-MIB::snmpyProcessInfoThrRunning.1 = INTEGER: 7
    SNMPY-MIB::snmpyProcessInfoMemResident.1 = Counter64: 15052
    SNMPY-MIB::snmpyProcessInfoMemSwap.1 = Counter64: 0
    SNMPY-MIB::snmpyProcessInfoCtxVoluntary.1 = Counter64: 817
    SNMPY-MIB::snmpyProcessInfoCtxInvoluntary.1 = Counter64: 143

#### `disk_utilization` ####


Development
-----------

### table plugins ###

### value plugins ###

License
-------
[MIT](http://mk23.mit-license.org/2011-2014/license.html)
