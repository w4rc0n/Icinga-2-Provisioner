I2provisioner
=============
A utility used for the automatic provisioning of hosts into icinga2 via the director module

Prerequisites
- Host must be using linux. Windows not supported at this time. (Confirmed working on Ubuntu 14.04-20.04, but should work on others)
- icinga2 must be installed on the host you would like to provision into icinga2.
- The host you would like to provision into icinga2 must be able to talk to your icinga2 master on port 443/tcp.
- The icinga2 [director module](https://github.com/Icinga/icingaweb2-module-director) must be installed and configured on the master.
- You must create an icinga2 API user that has the following permissions:
    - General Module Access (for director)
    - director/api
    - director/deploy
    - director/hosts
- You must have the following python libraries installed:
    - socket
    - requests
    - argparse
    - subprocess
 
Usage
-------------------------
You will need to adjust the script for your use case, specifically these variables:
```
# Adjust these variables to your environment. This variable should be your CA master node, as well as your configuration master.
master = 'https://yourmaster.com'
masterip = 'ip address of the above host'
# If you have a second master set up in an HA cluster, provide those details below like the above. If you do not have a second master, leave them as is.
master2 = ''
masterip2 = ''
# Your api information
apiuser = 'your_api_user'
apipass = 'your_api_password'
```
This script should be executed on the host that you would like to provision into icinga2 for monitoring. You can automate the deployment of this script using a configuration manager such as SaltStack like so (be sure to place your host template API key below):
```
{%- set icinga2_installed = salt['grains.get']('icinga2_installed') %}

{%- if ( not icinga2_installed ) %}

icinga2_provision:
  file.managed:
    - name: /usr/local/sbin/I2provision.py
    - source: salt://icinga2/files/I2provision.py
    - mode: 700
    - user: root
    - group: root
  cmd.run:
    - name: /usr/local/sbin/I2provision.py -d {{ grains['fqdn'] }} -k <your host template API key here>
    - require:
        - file: icinga2_provision

icinga2_cleanup:
  file.absent:
    - name: /usr/local/sbin/I2provision.py
    - require:
      - cmd: icinga2_provision

icinga2_installed:
   grains.present:
    - name: icinga2_installed
    - value: True
    - require:
        - file: icinga2_cleanup

{%- endif %}
```
The above will check if icinga2 has already been configured on the host in question using a grain called `icinga2_installed`, and stop if it has been. If it hasn't been, it will run the provisioning and then set the `icinga2_installed` grain to `True` to avoid re-provisoning.

 If you would like to run the script manually on a host, you may do so:
  - `python3 I2provisioner.py -d host.example.com -k HostTemplateAPIKey`
