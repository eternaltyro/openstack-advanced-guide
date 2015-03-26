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

Switch user to postgres

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

We could tightly configure the firewall to allow traffic only from the machines that host openstack components plus essential services like DNS and system update servers. 

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
