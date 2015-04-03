## Identity and access management

Keystone takes care of user access management to different OpenStack components. It maintains a role based access control mechanism where roles and privileges are defined and users merely assume roles, thereby inheriting the roles' privileges.'

For example, `Manager` and `Developer` may be the roles present in a system with the manager role holding the privilege to create and delete users and developer role merely being able to access compute and storage components. Then, a user alice whose assigned role is Manager will automatically be able to create new users. A user Bob whose assigned role is Developer will only be able to create and delete new instances. 

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

## Running Keystone over WSGI

    sudo apt-get install -y apache2
    sudo mkdir -p /var/www/cgi-bin/keystone/

    ( cat | sudo tee /var/www/cgi-bin/keystone/admin /var/www/cgi-bin/keystone/main ) <<EOF
    import logging
    import os

    from paste import deploy

    from keystone.openstack.common import gettextutils
    # NOTE(dstanek): gettextutils.enable_lazy() must be called before
    # gettextutils._() is called to ensure it has the desired lazy #lookup behavior. This includes cases, like keystone.exceptions, #where gettextutils._() is called at import time.
    gettextutils.enable_lazy()

    from keystone.common import dependency
    from keystone.common import environment
    from keystone.common import sql
    from keystone import config
    from keystone.openstack.common import log
    from keystone import service


    CONF = config.CONF

    config.configure()
    sql.initialize()
    config.set_default_for_default_log_levels()

    CONF(project='keystone')
    config.setup_logging()

    environment.use_stdlib()
    name = os.path.basename(__file__)

    if CONF.debug:
            CONF.log_opt_values(log.getLogger(CONF.prog), logging.DEBUG)


            drivers = service.load_backends()

    # NOTE(ldbragst): 'application' is required in this context by WSGI spec.
    # The following is a reference to Python Paste Deploy documentation
    # http://pythonpaste.org/deploy/
    application = deploy.loadapp('config:%s' % config.find_paste_config(),
                             name=name)

    dependency.resolve_future_dependencies()
    EOF


