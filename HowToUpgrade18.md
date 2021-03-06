# Spacewalk Upgrade Instructions



These are upgrade instruction for upgrading Spacewalk 1.7 to Spacewalk 1.8.

These upgrade instruction apply to Spacewalk installations meeting the following criteria:

  *  Spacewalk 1.7 running on Red Hat Enterprise Linux 5 Server, 6 Server or CentOS/Scientific Linux or Fedora 16.
  *  Your Spacewalk uses either of Oracle 10g (including XE) / Oracle 11g / PostgreSQL 8.4+ as a database backend.
## Archive of older upgrade instructions



 * Spacewalk 1.6 to 1.7 upgrade instructions, are available at [[HowToUpgrade17]]
 * Spacewalk 1.5 to 1.6 upgrade instructions, are available at [[HowToUpgrade16]]

----
## Assumptions



  * For RHEL or CentOS, you will need the base and EPEL repositories enabled for dependencies.
  * For Fedora, your Fedora yum repositories are setup properly.
  * You had set up your yum to point to Spacewalk 1.8 repository. For the repo setup specifics, see HowToInstall#SettingupSpacewalkrepo.
    * In particular, for Fedora, make sure your jpackage repo is setup properly: https://fedorahosted.org/spacewalk/wiki/HowToInstall#Jpackagerepository
## Database and configuration backup




  *  For existing configuration files, create a backup of everything under /etc/sysconfig/rhn /etc/rhn and /etc/jabberd
  *  Backup your SSL build directory, ordinarily /root/ssl-build
  *  For instructions on how to create a backup of your existing Spacewalk database consult either Oracle / PostgreSQL documentation or contact your DBA
## Package upgrade



Your Spacewalk should be stopped at this point.

If you have previously updated your cobbler to version higher than 2.0 from EPEL / Fedora and you want to keep using that version, run:

    # yum install cobbler-epel

If you still use cobbler 2.0, it will be upgraded to cobbler 2.0 shipped with Spacewalk 1.8.

Perform package upgrade using yum:

    # yum upgrade

During the upgrade, you may notice messages printed to the terminal when installing oracle-instantclient-selinux and spacewalk-selinux. These messages are produced by restorecon and do not pose any harm.

Check any `.rpmnew`/`.rpmsave` files that were created during the upgrade for your configuration files and make sure changes to your configuration files are preserved while new content from the distribution files is carried over.

    # yum install rpmconf
    # rpmconf -a
## Schema upgrade



Make sure your Spacewalk server is down:


    # /usr/sbin/spacewalk-service status

Do a backup of your database.

Make sure your database server is running. Run spacewalk-schema-upgrade script to upgrade database schema:


    # /usr/bin/spacewalk-schema-upgrade

Important notes:

  * The above command will inform you whether or not was the schema upgrade successful.
    * Log files from schema upgrade are being put into /var/log/spacewalk/schema-upgrade.
    * Should the schema upgrade fail, investigate, restore from backup, fix the cause (for example, if it failed because of insufficient space in tablespace, extend the tablespaces) and rerun spacewalk-schema-upgrade.
## Upgrade of Spacewalk configuration



  1. Use spacewalk-setup to upgrade Spacewalk configuration. Run as root:


    # spacewalk-setup --disconnected --external-db --upgrade

  2. Restore some of the custom values you might have set previously in /etc/rhn/rhn.conf from the backup of your configuration files, such as:

  *  debug = 3
  *  pam_auth_service = rhn-satellite

  3. Enable (or re-enable) monitoring (and monitoring scout).

  * If you wish to enable (or re-enable) monitoring without enabling monitoring scout, run the following command as root:


    # /usr/share/spacewalk/setup/upgrade/rhn-enable-monitoring.pl

  * If you wish to enable (or re-enable) both monitoring and monitoring scout, run the following command:


    # /usr/share/spacewalk/setup/upgrade/rhn-enable-monitoring.pl --enable-scout
## Restart Spacewalk




    # /usr/sbin/spacewalk-service start