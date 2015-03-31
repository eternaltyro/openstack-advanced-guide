## Setting up PostgreSQL

PostgreSQL database needs to be set up for various components of Openstack. This can be set up in the same node as the controller components in the case of a simple cluster. In a prodution setup, the database would reside in a dedicated machine possibly with high-availability clustering enabled. 

## 1. Install PostgreSQL

On Ubuntu

    sudo apt-get update
    sudo apt-get install -y postgresql-9.3

On RHEL/CentOS

    sudo yum update all
    sudo yum install postgresql

## 2. Configuring PostgreSQL

Set password for postgres user in Ubuntu

    $ sudo passwd postgres
    Enter new UNIX password: 
    Retype new UNIX password:
    passwd: password updated successfully

Switch user to postgres. From now on, everything (except iptables) is done as *postgres* user and NOT ROOT!

    $ su postgres
    Password: 

*Note*: From now on, doing sudo is not necessary for all postgres owned files and services. 

Edit the PostgreSQL config file to adjust listen address and password security setting.

    $ pushd /etc/postgresql/9.3/main/
    $ sed -i 's/#password_encryption/password_encryption/1' postgresql.conf
    $ grep encrypted_password postgresql.conf
    password_encryption = on
    $ sed -i "s/'localhost'/'*'/1" postgresql.conf
    $ sed -i 's/#listen_address/listen_address/1' postgresql.conf

Edit host based authentication to selectively allow connections

    $ vi /etc/pg_hba.conf

Make it similar to the following:
    
    #local   DB     USER        METHOD
    local    all    postgres    peer
    local    all    all         md5

    #host   DB       USER         CIDR          METHOD
    host    all      all          110.0.0.0/16  md5
    host    identity keystonedba  10.1.2.1/32   md5
    host    compute  novadba      10.1.17.0/24  md5
    host    image    glancedba    10.1.2.3/32   md5
    host    blockstore cinderdba  10.1.2.4/32   md5
    host    network  neutrondba   10.1.2.5/32   md5

Where 110.0.0.0/16 is a network from which all traffic is allowed in and 
 - 10.1.2.1/32 is the node on which identity components are installed,
 - 10.1.17.0/24 is the "network" on which nova components are installed,
 - 10.1.2.3/32 is the node on which glance registry is installed,
 - 10.1.2.4/32 is the node on which cinder registry is installed,
 - 10.1.2.5/32 is the network node.

Restart PostgreSQL service

    $ service postgresql restart

## Creating database and roles 

    $ psql
    =# create role openstackdba password 'openstacksecret';
    =# create role novadba password 'novasecret' login INHERIT in role openstackdba;
    =# create role glancedba password 'glancesecret' login INHERIT in role openstackdba;
    =# create role cinderdba password 'cindersecret' login INHERIT in role openstackdba;
    =# create role neutrondba password 'neutronsecret' login INHERIT in role openstackdba;
    =# create role keystonedba password 'keystonesecret' login INHERIT in role openstackdba;
    =# \du
                                   List of roles
      Role name   |                   Attributes                   | Member of 
    --------------+------------------------------------------------+-----------
     cinderdba    |                                                | {openstackdba}
     glancedba    |                                                | {openstackdba}
     keystonedba  |                                                | {openstackdba}
     neutrondba   |                                                | {openstackdba}
     novadba      |                                                | {openstackdba}
     openstackdba |                                                | {openstackdba}
     postgres     | Superuser, Create role, Create DB, Replication | {openstackdba}
    =#
    =# create database identity_db OWNER keystonedba;
    =# create database compute_db OWNER novadba;
    =# create database network_db OWNER neutrondba;
    =# create database blockstore_db OWNER cinderdba;
    =# create database imagestore_db OWNER glancedba;


Note: Swift block storage does not require any database connection.

### If you are setting up database for fun or experimentation, you can safely skip the following paragraphs.

## Securing PostgreSQL

These are to be taken only as guidelines and not expert recommendations. Do your own research before you follow any of these guidelines. If you are running PostgreSQL on production, you MUST NOT rely on the information below to secure your database. You could only use these as points of start. While it is certainly recommended that you use SSL and strong passwords and restrict traffic using firewalls, obviously, securing the database is much more complex than merely using SSL and strong passwords. Some basic stuff given below.

### Configuring server side SSL

SSL can be configured in the server side to encrypt traffic between client and the database server. The connections can be forced to use SSL using host based authentication by using hostssl in pg_hba.conf like so:
    
    hostssl    identity keystonedba  10.1.2.1/32   md5

Ciphers for SSL need to be configured to use strong encryption with preference for TLSv1.2. The following lines specify the SSL ciphers allowed, the SSL certificate and key file locations in postgresql.conf:

    ssl_ciphers = '!SSLv3:!SSLv2:HIGH:MEDIUM:!LOW:!EXP:!MD5:@STRENGTH'
    ssl_cert_file = '/etc/ssl/certs/ssl-cert.pem'
    ssl_key_file = '/etc/ssl/private/ssl-cert.key'

### Configuring Firewall

We could tightly configure the firewall to allow traffic only from the machines that host openstack components plus essential services like DNS and system update servers. Note: Do everything in this section as root. After you are done, switch back to being postgres.

    # iptables -A OUTPUT -p udp -s 10.0.0.2/32 -d DNSHOST/32 --dport 53 -j ACCEPT
    # iptables -A INPUT -p udp -s 0/0 --sport 53 -d 10.0.0.2/32 -j ACCEPT
    # iptables -A OUTPUT -p tcp -s 10.0.0.2/32 -d UPDATE_SERVERS --dport 80 -m state --state NEW, ESTABLISHED -j ACCEPT
    # iptables -A OUTPUT -p tcp -s 10.0.0.2/32 -d UPDATE_SERVERS --dport 443 -m state --state NEW, ESTABLISHED -j ACCEPT
    # iptables -A INPUT -p tcp -s UPDATE_SERVERS --sport 80 -d 10.0.0.2/32 -m state --state ESTABLISHED -j ACCEPT
    # iptables -A INPUT -p tcp -s 110.0.0.0/16 --sport 1024:65535 -d 10.0.0.2/32 --dport 5432 -m state --state NEW,ESTABLISHED -j ACCEPT
    # iptables -A OUTPUT -p tcp -s 10.0.0.2/32 --sport 5432 -d 110.0.0.0/16 --dport 1024:65535 -m state --state ESTABLISHED -j ACCEPT
    # iptables -A INPUT -p tcp -s 10.1.2.1/32 --sport 1024:65535 -d 10.0.0.2/32 --dport 5432 -m state --state NEW,ESTABLISHED -j ACCEPT
    # iptables -A OUTPUT -p tcp -s 10.0.0.2/32 --sport 5432 -d 10.1.2.1/32 --dport 1024:65535 -m state --state ESTABLISHED -j ACCEPT
    # iptables -A INPUT -p tcp -s 10.1.17.0/24 --sport 1024:65535 -d 10.0.0.2/32 --dport 5432 -m state --state NEW,ESTABLISHED -j ACCEPT
    # iptables -A OUTPUT -p tcp -s 10.0.0.2/32 --sport 5432 -d 10.1.17.0/24 --dport 1024:65535 -m state --state ESTABLISHED -j ACCEPT
    # iptables -A INPUT -p tcp -s 10.1.2.3/32 --sport 1024:65535 -d 10.0.0.2/32 --dport 5432 -m state --state NEW,ESTABLISHED -j ACCEPT
    # iptables -A OUTPUT -p tcp -s 10.0.0.2/32 --sport 5432 -d 10.1.2.3/32 --dport 1024:65535 -m state --state ESTABLISHED -j ACCEPT
    # iptables -A INPUT -p tcp -s 10.1.2.4/32 --sport 1024:65535 -d 10.0.0.2/32 --dport 5432 -m state --state NEW,ESTABLISHED -j ACCEPT
    # iptables -A OUTPUT -p tcp -s 10.0.0.2/32 --sport 5432 -d 10.1.2.4/32 --dport 1024:65535 -m state --state ESTABLISHED -j ACCEPT
    # iptables -A INPUT -p tcp -s 10.1.2.5/32 --sport 1024:65535 -d 10.0.0.2/32 --dport 5432 -m state --state NEW,ESTABLISHED -j ACCEPT
    # iptables -A OUTPUT -p tcp -s 10.0.0.2/32 --sport 5432 -d 10.1.2.5/32 --dport 1024:65535 -m state --state ESTABLISHED -j ACCEPT


### Use strong passwords (duh)

Strong passwords are passwords that are extremely difficult to guess with emphasis on 'extremely'. Which means the passwords are going to have to be hard to remember. A good program that you can use to generate strong passwords is apg. I use the switches to generate passwords that don't screw up with the other lines in the file.

    $ apg -a 0 -M NLC -m 8
    CaipAfyar3
    evIggaj5
    fapAjcem7
    BozedTep7
    MeHarjEg8
    FornIocOc4

You could use a password management utility like 'keepassx' to store and manage the passwords. 

## High-Availability

There are several ways of ensuring high availability in PostgreSQL. But the most common method is to setup a streaming replication among servers. In this scheme, there's one active master server and multiple slave replicas. The master ships the Write Ahead Logs (WAL) to a shared location or to a location on the slave hosts, and continuously keep streaming WALs that the slaves use to replicate data. You can promote a slave to a master server in the event a master server goes
down. During replication, a slave can also cater to read queries from users. This is called 'hot standby'. Let us look at how to set up a simple single slave replica setup.

#### ASSUMPTIONS

- Replicas are named replica1, replica2, etc.
- Replicas bear IP address in the 10.1.7.0/24 CIDR
- The replicas and the master have all the hostnames in the /etc/hosts file

## Installing slave replicas

Slave replicas are installed just like the master server. They have appropriate server configuration file and pg_hba.conf file. Any special configuration parameters pertaining to replication for the master or slaves will be discussed below.

## Establishing connectivity between the master and slave servers

We need to establish seamless connectivity between the master and slaves. We need to be able to transfer files via rsync and let the slave servers connect to the master database server over port 5432.

### SSH connectivity

On the master server, create an SSH key.

    $ ssh-keygen
    Generating public/private rsa key pair.
    Enter file in which to save the key (/var/lib/postgresql/.ssh/id_rsa):
    Enter passphrase (empty for no passphrase): 
    Enter same passphrase again: 
    Your identification has been saved in /var/lib/postgresql/.ssh/id_rsa.
    Your public key has been saved in /var/lib/postgresql/.ssh/id_rsa.pub.
    The key fingerprint is:
    75:88:2b:94:47:21:6d:11:c0:c6:b1:f3:cb:af:2e:f8 postgres@master
    The key's randomart image is:
    +--[ RSA 2048]----+
    |     o++=+       |
    |      +=o. .     |
    |     .=.o o .    |
    |     . + o .     |
    |      . S        |
    |       o .       |
    |     .  o        |
    |    . .  .       |
    |     .Eoo..      |
    +-----------------+

Note that we don't supply a passphrase. So just press enter key when prompted for it. Now we need to send it to all the slave replica.

    $ ssh-copy-id -i ~/.ssh/id_rsa.pub postgres@10.1.7.2
    postgres@10.1.7.2's password:

Repeat this for each replica. Now let's set up the host based authentication for the servers. We can authorize individual servers or an entire CIDR depending on requirements Add the following lines to the pg_hba.conf file:

    host    replication    argusfilch    10.1.7.0/24    trust
    host    replication    argusfilch    10.1.7.2/32    trust

This trusts all connections to the database _replication_ from user _argusfilch_ from network 10.1.7.0/24 which is dedicated for the database servers. This could potentially be a security risk and you should consider using md5 in place of trust. Now let us create the user __argusfilch__ which can do replication.

    postgres=# create role argusfilch password 'mrsnorris' LOGIN REPLICATION;

We create a user specifically for replication since this user need not be superuser. We give this user permission to access a special database for replication called... replication! I have named the user _argusfilch_ after the infamous grumpy caretaker from the Harry Potter book series. 

## Preparing the master server

Let us set up replication on the master server. Here, we set the archival level and the command to be executed "to" archive the WALs. Edit the postgresql.conf directory to set the following parameters:

    wal_level = hot_standby
    archive_mode = on
    archive_command = 'rsync -W -az %p postgres@10.1.7.2:/var/lib/postgresql/walarchive/'
    max_wal_senders = 6 
    wal_keep_segments = 100

Where hot_standby means the archival is quicker than usual. archive_command is basically any command that ships the WALs to a location that is easily accessible by the slave servers. In this example, we have used rsync to copy the WAL files to a directory on the slaves postgres home directory. Note: The directory must exists and MUST be empty the first time the command runs. So on the slave, run the following commands to create a directory accessible by postgres user on that server:

    $ mkdir -m 0755 -p /var/lib/postgresql/walarchive
    $ chown -R postgres:postgres /var/lib/postgresql/walarchive

Now let us restart the master PostgreSQL server. You should begin seeing the _walarchive_ directory populated once restart is complete.

    $ pg_ctlcluster 9.3 main restart

## Preparing the slave servers

Login to the slave servers and use the pg_basebackup tool to copy a base backup of the master server into the postgres data directory of all the slaves.

    $ pg_basebackup -D /var/lib/postgresql/9.3/main -F p -X s -h 10.1.7.1 -p 5432 -U argusfilch -w 

We should now have a data directory that is populated. Now that we have a basic point-in-time backup of the master server, we have to let the slave know how to connect to the master server to recieve a continuous stream of data and change logs. For this, we create a file called _recovery.conf_ under the PostgreSQL data directory and populate it like so:

    standby_mode = on
    trigger_file = '/tmp/promoted'
    primary_conninfo = 'host=10.1.7.1 port=5432 user=argusfilch password=mrsnorris'
    restore_command = 'cp /var/lib/postgresql/walarchive/%f "%p"' >> /var/lib/postgresql/restore.log 2>&1
    recovery_target_timeline = 'latest'

There are certain risks to be concerned. One is the plain text password. This can be skipped when using 'trust' in pg_hba.conf of the master server. Second is the data growth of both the WAL files and the log file. The logs need to be rotated. Note: There are some interesting pg tools in /usr/lib/postgresql/9.x/bin.

TODO: WAL Archive cleanup.

Now let us edit the postgresql.conf file. We could copy over the master's postgresql.conf file and modify it if we want. Set the value according to your network speed. 

    hot_standby = on 
    max_standby_archive_delay = 1min   
    max_standby_streaming_delay = 1min
    hot_standby_feedback = on

Restart PostgreSQL on the slave to check if replication is functional. Make some changes in the master server and check if it's being propagated in time. You can also check the replication status using SQL.

    $ pg_ctlcluster 9.3 main restart
    $ psql
    postgres=# \x
    Expanded display is on.
    postgres=# select * from pg_stat_replication;
    -[ RECORD 1 ]----+------------------------------
    pid              | 30417
    usesysid         | 16764
    usename          | argusfilch
    application_name | walreceiver
    client_addr      | 10.1.7.2
    client_hostname  | 
    client_port      | 41615
    backend_start    | 2015-03-31 03:37:19.060136+00
    state            | streaming
    sent_location    | 0/21014358
    write_location   | 0/21014358
    flush_location   | 0/21014358
    replay_location  | 0/21014358
    sync_priority    | 0
    sync_state       | async

## Failover

If the worst has happened and you need to failover to the standby server, you can do so by either creating the file /tmp/promoted as in the recovery.conf file or you can run the following command on the standby.

    $ pg_ctlcluster 9.3 main promote
