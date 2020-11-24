I2provisioner
=============
A utility used for the automatic provisioning of hosts into icinga2 via the icinga 2 director module API, and configuring the host itself as an icinga2 endpoint.

This script will perform the following:
- Take in commandline arguments about what you would like to name the host you are going to monitor, and which host template you would like to have it set up with (using the templates API key).
- Gather information about the host that you would like to set up as a monitoring endpoint such as hostname, and IP address.
- Connect to the Director API to request a ticket for the host.
- Add the host to icinga2 for monitoring via the Director API ticket.
- Generate an agent provisioning bash script as provided by the developers of icinga2.
- Format that bash script with all the details that have been either gathered or provided.
- Execute the bash script to configure the host as a monitoring endpoint automatically.
- Deploys the Director module configuration.
- Removes the bash script.

Prerequisites
--------------
- Host must be using linux. Windows not supported at this time. (Confirmed working on Ubuntu 14.04-20.04, but should work on other distros)
- icinga2 must be installed on the host you would like to provision into icinga2.
- The host you would like to provision into icinga2 must be able to talk to your icinga2 master on ports 443/tcp, 5665/tcp.
    - It would work with port 80/tcp/http as well, however you would need to adjust the script for that, and I don't recommend sending API credentials over unencrypted connections.
- The icinga2 [director module](https://github.com/Icinga/icingaweb2-module-director) must be installed and configured on the master.
- [Icinga Web 2](https://github.com/Icinga/icingaweb2) must be installed and configured on the master.
- You must create an Icinga Web 2 web user that has the following permissions (I would recommend a dedicated user with no other permissions:
    - General Module Access (for director)
    - director/api
    - director/deploy
    - director/hosts
- You will need an API key for the host template you would like to use:
    1. Select `Icinga Director` in the icinga2 web interface, in the left side bar.
    2. Select `Hosts`
    3. Select `Host Templates`
    4. Select the template you would like to provision with
    5. Select the `Agent` tab
    6. Select `Generate Self Service API key`
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
The above will check if icinga2 has already been configured on the host in question using a grain called `icinga2_installed`, and stop if it has been. If it hasn't been, it will run the provisioning and then set the `icinga2_installed` grain to `True` to avoid re-provisoning. After it has run the provisioning, it will remove the script from the host.

 If you would like to run the script manually on a host, you may do so:
  - `./I2provisioner.py -d host.example.com -k HostTemplateAPIKey`
