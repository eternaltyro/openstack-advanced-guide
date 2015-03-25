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

    #host   DB       USER         CIDR         METHOD
    host    all      all          110.0.0.0/16  md5
    host    identity keystonedba  10.1.2.3/32  md5
    host    compute  novadba      10.1.2.3/32  md5
    host    image    glancedba    10.1.2.3/32  md5
    host    blockstore cinderdba  10.1.2.3/32  md5
    host    network  neutrondba   10.1.2.3/32  md5

Where 110.0.0.0/16 is the main network from which all traffic is allowed in and 10.1.2.3/32 is the node on which all the services are installed.

Restart PostgreSQL service

    $ service postgresql restart

## Creating database and roles 

    $ psql
    =# create role novadba password 'novasecret';
    =# create role glancedba password 'glancesecret';
    =# create role cinderdba password 'cindersecret';
    =# create role neutrondba password 'neutronsecret';
    =# create role keystonedba password 'keystonesecret';
    =# create role openstackdba password 'openstacksecret';
    =# \du
                                   List of roles
      Role name   |                   Attributes                   | Member of 
    --------------+------------------------------------------------+-----------
     cinderdba    | Cannot login                                   | {}
     glancedba    | Cannot login                                   | {}
     keystonedba  | Cannot login                                   | {}
     neutrondba   | Cannot login                                   | {}
     novadba      | Cannot login                                   | {}
     openstackdba | Cannot login                                   | {}
     postgres     | Superuser, Create role, Create DB, Replication | {}

