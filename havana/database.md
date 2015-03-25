## Setting up PostgreSQL

PostgreSQL database needs to be set up for various components of Openstack 

## 1. Install PostgreSQL

On Ubuntu

    sudo apt-get install -y postgresql-9.3

## 2. Configuring PostgreSQL

Set password for postgres user in Ubuntu

    $ sudo passwd postgres
    Enter new UNIX password: 
    Retype new UNIX password:
    passwd: password updated successfully

Switch user to postgres

    $ su postgres
    Password: 

Edit the PostgreSQL config file to adjust listen address and password security setting.

    $ pushd /etc/postgresql/9.3/main/
    $ sed -i 's/#password_encryption/password_encryption/1' postgresql.conf
    $ grep encrypted_password postgresql.conf
    password_encryption = on
    $ sed -i "s/'localhost'/'*'/1" postgresql.conf
    $ sed -i 's/#listen_address/listen_address/1' postgresql.conf

Edit host based authentication to selectively allow connections

