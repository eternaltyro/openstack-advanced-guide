## Identity and access management

Keystone takes care of user access management to different OpenStack components. It maintains a role based access control mechanism where roles and privileges are defined and users merely assume roles, thereby inheriting the roles' privileges.'

For example, "Manager" and "Developer" may be the roles present in a system with the manager role holding the privilege to create and delete users and developer role merely being able to access compute and storage components. Then, a user alice whose assigned role is Manager will automatically be able to create new users. A user Bob whose assigned role is Developer will only be able to create and delete new instances. 

## Installing keystone

The following commands install keystone, keystone client and the python bindings for PostgreSQL.

    # apt-get -y install keystone keystone-doc
    # apt-get -y install python-keystone python-psycopg2

## Configuring Keystone

Edit /etc/keystone/keystone.conf

    # vi /etc/keystone/keystone.conf
    connection = postgresql://keystonedba:keystonesecret@10.10.10.2/identity_db

Where 10.10.10.2 is the database node on which PostgreSQL server is listening. 

    # keystone-manage db_sync

### Starting Keystone


## Testing keystone


## Adding Services and Endpoints


## Authenticating against LDAP


## Authenticating against Kerberos


## Keystone Security

- Using SSL to exchange Tokens
- Using a strong admin token
- Using strong database passwords
- Firewall rules

## Using keystone API

    $ sudo apt-get install python-keystoneclient
    $ python
    >>> import keystone
    >>>
