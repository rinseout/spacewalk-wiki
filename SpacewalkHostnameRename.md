# spacewalk-hostname-rename



spacewalk-hostname-rename script is an automated tool for re-configuration of Spacewalk server, in case its hostname or IP has changed.

It takes care of reconfiguration of /etc/rhn/rhn.conf file, SSL certificate, database, jabberd, cobbler, monitoring ... everything what's needed.

*NOTE* Customizations within the rhn.conf file such as pam settings or different path for /var/satellite may need to be manually re-added post reconfiguration. 
## Rename process

To change the hostname or IP of the RHN Satellite:

 * Schedule a maintenance window for the rename process.
 * Change the hostname/IP of the system.
 * Reboot the system to ensure that the new hostname gets active
 * Run the spacewalk-hostname-rename script

The spacewalk-hostname-rename script is an automated tool for reconfiguration of the RHN Satellite server. In the event that you change the ip address and hostname of the RHN Satellite, this script will automatically reconfigure the /etc/rhn/rhn.conf file, SSL certificate, database, jabberd, cobbler, monitoring, and so on.
## Prerequisites

If you've changed the system hostname, make sure you know your SSL CA passphrase. It will be needed by the script.

 
Verify that CA password is working:

    # openssl rsa -in ssl-build/RHN-ORG-PRIVATE-SSL-KEY
    Enter pass phrase for ssl-build/RHN-ORG-PRIVATE-SSL-KEY:
    writing RSA key
    -----BEGIN RSA PRIVATE KEY-----
    cut
    -----END RSA PRIVATE KEY-----
## How to use the script



spacewalk-hostname-rename takes one mandatory argument - IP_ADDRESS - regardless of whether the IP address has changed or not. If there is a need to generate a new SSL certificate, all necessary information will be asked interactivelly, unless it is specified by the options. When the system hostname has not changed, the regeneration of a new SSL server certificate is not necessary. However, if at least one --ssl-* option is specified, certificate generation is forced.


    Usage:
       spacewalk-hostname-rename <IP_ADDRESS> [ --ssl-country=<SSL_COUNTRY> --ssl-state=<SSL_STATE> --ssl-org=<SSL_ORG> --ssl-orgunit=<SSL_ORGUNIT> --ssl-email=<SSL_EMAIL> --ssl-ca-password=<SSL_CA_PASSWORD>]
       spacewalk-hostname-rename { -h | --help }

IP_ADDRESS is the default IP address of the system used mainly for monitoring.

Example usage:


    # spacewalk-hostname-rename <my_ip_address>
    Validating IP ... OK
    =============================================
    hostname: <my_hostname>
    ip: <my_ip_address>
    =============================================
    Stopping rhn-satellite services ... OK
    Testing DB connection ... OK
    Updating /etc/rhn/rhn.conf ... OK
    Actual SSL key pair package: rhn-org-httpd-ssl-key-pair-rlx-2-18.rhndev-1.0-6
    No need to re-generate SSL certificate.
    Regenerating new bootstrap client-config-overrides.txt ... OK
    Updating NOCpulse.ini ... OK
    Updating monitoring data ... OK
    Pushing monitoring scouts ... OK
    Updating other DB entries ... OK
    Changing cobbler settings ... OK
    Changing jabberd settings ... OK
    Starting rhn-satellite services ... OK
## Logging and backup



spacewalk-hostname-rename logs to /var/log/rhn/rhn_hostname_rename.log and all manually changed files get backed up with predefined .rhnbck extension.
## What about the clients?



Check the serverURL value in /etc/sysconfig/rhn/up2date on every client registered to the renamed satellite and change it to the new hostname/IP value. Reconfiguration of RHN proxy servers isn't trivial and it is recommended to recreate the RHN proxy servers.
## Script failures



The most frequent cause of problems is an incorrectly set system hostname. The script checks whether the hostname is correctly set. If not, an error message will inform you of what needs to be fixed. After the problem is resolved, the script can be re-run.
## Where do I find the script?



spacewalk-hostname-rename is a part of *spacewalk-utils* package.