## Identity and access management

Keystone takes care of user access management to different OpenStack components. It maintains a role based access control mechanism where roles and privileges are defined and users merely assume roles, thereby inheriting the roles' privileges.'

For example, `Manager` and `Developer` may be the roles present in a system with the manager role holding the privilege to create and delete users and developer role merely being able to access compute and storage components. Then, a user alice whose assigned role is Manager will automatically be able to create new users. A user Bob whose assigned role is Developer will only be able to create and delete new instances. 

## AuthZ / AuthN

Authentication and Authorization are two different concepts, and understanding them would help much in debugging and troubleshooting. Authentication is establishing the identity of the user. Authorization is the privilege a user has to access and utilize certain resources. Simply put, proving you are who you claim you are is authentication. Authorization is granting permission to access something. 

## Keystone Concepts

Project and Tenants are the same. Domains, users, roles and groups. 

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

    keystone-all
    service keystone start

## Testing keystone


## Adding Services and Endpoints


## Authenticating against LDAP


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

Request Headers: Accept: application/json, X-Auth-Token: mygreatadminsecret
POST Req Headers: Accept: application/json, X-Auth-Token: mygreatadminsecret, Content-Type: application/json

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

## Tokens

Keystone deals two kinds of tokens depending on architecture of the deployment. 

### UUID Tokens

These are simple 32 character hexadecimal tokens.

- A privileged user sends credentials (username:password) along with the scope of the token required to Keystone. 
- Keystone validates the user credentials and role assignments against the backend (PostgreSQL)
- Once authenticated and authorized, Keystone generates a unique 32digit alpha numeric token and sends it back to the user along with a service catalog that the user has access to.
- The service catalog is sent in the response payload and the token is sent as a HTTP response header variable.
- The User then sends the token along with the service request (operation) to an OpenStack service.
- The service validates the token against Keystone ( backend lookup, yada yada all over again )
- Once Keystone gives the go ahead, the service performs the operation and sends the confirmation back to the user.

As you can see, this is network intensive and does not scale well since each request performs multiple database lookups and validations. To make it easier on the network, we could use PKI tokens.

### PKI / PKIZ Tokens

These are signed tokens in CMS format (Cryptographic Message Syntax defined by IETF RFC-5652). 

- A privileged user sends credentials along with scope to keystone
- Keystone validates the credentials against backend
- Keystone then creates a CMS token which is a message digest containing the user role assignments, service catalog and meta data. 
- The token is then cryptographically signed using Keystone's TLS certificate and Key.
- The token is `base64 endcoded` and sent to the user who sends it to the service endpoints along with a service request
- The service uses a cached copy of Keystone's signing cert and verifies / validates the Certificate by itself.
- The service then looks up the revocation list to see if the certificate had been revoked
- Once all validations pass, the service request is carried out and the confirmation sent to the user.

#### Pros: 

- As it is obvious, the PKI token verifications are done offline locally by the services themselves and not by Keystone everytime. This relieves the network of considerable traffic.
- Tokens can be compressed using zlib, called PKIZ Tokens which are smaller than PKI tokens.

#### Cons:

- TLS certs and keys need to be set up. But this can be done using the `keystone-manage pki_setup` command for evaluation purposes.
- The token size is considerably larger than the UUID tokens

### Using Let's Encrypt to create and install certificates (EXPERIMENTAL)

    $ sudo apt-get install lets-encrypt
    $ sudo ./lets-encrypt --text --agree-eula mysecureopenstack.com

keystone commands use v2.0 or v3 depending on what the endpoint is
use wireshark


## Token Exchange

Wireshark pegs 5000 as IPA port `GSM over IP`
35357 is proper `openstack-id` 

Request Token via `POST` to `/v3/auth/tokens` with JSON payload.
response is a JSON with token supplied in header like so: `X-Subject-Token: 72b224a51f7142b1a77334d5c43e6939`
Use token for further stuff
