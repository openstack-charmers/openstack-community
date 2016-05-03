# Guide to creating a basic Openstack API service charm

This guide will walk the creation of a basic charm for the Openstack
Congress service. [Congress Info](https://wiki.openstack.org/wiki/Congress).
The charm will use prewritten Openstack [layers and interfaces](https://github.com/openstack-charmers)
Once the charm is written it will be compose using [charm tools](https://github.com/juju/charm-tools/).

The Congress service needs to register endpoints with Keystone. It needs a service username and password and it also needs a MySQL backend to store its schema.

# Create the skeleton charm

## Create charm directories

Firstly create a directory for the new charm and manage the charm with git.

```bash
mkdir -p congress/charm
cd congress
git init
```
The top layer of this charm is the Congress specific code this code will live in the charm subdirectory.

```mkdir -p charm/{reactive,lib}```

## Describe the Service and required layer(s)

The new charm needs a basic metadata.yaml to describe what service the charm provides. Edit charm/metadata.yaml

```yaml
name: congress
summary: Policy as a service
description: |
 Congress is an open policy framework for the cloud. With Congress, a cloud
 operator can declare, monitor, enforce, and audit "policy" in a heterogeneous
 cloud environment.
```

The [openstack-api layer](https://github.com/openstack-charmers/charm-layer-openstack-api) defines a series of config options and interfaces which are mostly common accross Openstack API services e.g. including the openstack-api-layer will pull in the Keystone and Mysql interfaces (among others) as well as the charm layers the new Congress charm can leverage. To instruct "charm build" to pull in the openstack-api layer edit layer.yaml:

```yaml
includes: ['layer:openstack-api']
```

# Add Congress configuration

## Define Congress attributes

To define the attributes of Congress, inherit the OpenStackCharm class and set Congress specific variables. This is done on the charm/lib/charm/openstack/congress.py file.

```
from charm.openstack.ip import PUBLIC, INTERNAL, ADMIN
from charm.openstack.charm import OpenStackCharmFactory, OpenStackCharm
from charm.openstack.adapters import OpenStackRelationAdapters

class CongressCharm(OpenStackCharm):
    # Packages the service needs installed
    packages = ['congress-server', 'congress-common', 'python-antlr3', 'python-pymysql']

    # Init services the charm manages
    services = ['congress-server']

    # Standard interface adapters class to use.
    adapters_class = OpenStackRelationAdapters

    # Ports that need exposing.
    default_service = 'congress-api'
    api_ports = {
        'congress-api': {
            PUBLIC: 1789,
            ADMIN: 1789,
            INTERNAL: 1789,
        }
    }

    # Database sync command used to initalise the schema.
    sync_cmd = ['congress-db-manage', '--config-file', '/etc/congress/congress.conf', 'upgrade', 'head']

    # The restart map defines which services should be restarted when a given file changes
    restart_map = {
        '/etc/congress/congress.conf': ['congress-server'],
        '/etc/congress/api-paste.ini': ['congress-server'],
        '/etc/congress/policy.json': ['congress-server'],
    }
```

Finally create a charm factory, this is used to allow a different Charm class to be used for different Openstack releases. In this case the same charm class will be used for all Liberty and above releases.

```
class CongressCharmFactory(OpenStackCharmFactory):

    releases = {
        'liberty': CongressCharm
    }

    first_release = 'liberty'
```

## Add Congress code to react to events

### Install Congress Packages

The reactive framework is going to emit events that the Congress charm can react to. the charm needs to define how its going to react to these events and also raise new events as needed.

The first action a charm needs to do is to install the Congress code. This is by done running the install method from CongressCharm created earlier. Edit charm/reactive/handlers.py ...

```
import charms.reactive as reactive
import charm.openstack.congress as congress
import charmhelpers.contrib.openstack.utils as ch_utils

charm = None

def get_charm():
    # Retrieve a charm instance from the charm factory.
    global charm
    if charm is None:
        charm = congress.CongressCharmFactory.charm(
            release=ch_utils.os_release('congress-common', base='liberty'))
    return charm

@reactive.hook('install')
def install_packages():
    # When install hook fires install charm packages
    get_charm().install()
```

### Configure Congress Relation

At this point the charm could be built and deployed and it would deploy a unit, and install congress. However there is no code to specify how this charm should interact with the services to depend on. For example when joining the database the charm needs to specify the user and database it requires. The following code configures the relations with the dependant services.

```
@reactive.when('amqp.connected')
def setup_amqp_req(amqp):
    # Request username and vhost from rabbit charm
    amqp.request_access(username='congress',
                        vhost='openstack')

@reactive.when('shared-db.connected')
def setup_database(database):
    # Request username and database from database charm
    database.configure('congress', 'congress', hookenv.unit_private_ip())

@reactive.when('identity-service.connected')
def setup_endpoint(keystone):
    # Regester endpoints with keystone and receive service credentials
    charm = get_charm()
    keystone.register_endpoints(charm.service_type,
                                charm.region,
                                charm.public_url,
                                charm.internal_url,
                                charm.admin_url)
```

## Configure Congress

Now that the charm has the relations defined that it needs the Congress charm is in a postion to generate its configuration files.

### Create templates

The charm code searches through the templates directories looking for a directory cooresponding to the Openstack release being installed
or earlier. Since Liberty is the earliest release the charm is supporting a directory called liberty will house the templates.

```mkdir -p templates/liberty
cp <from pkg>/{api-paste.init,policy.json,congress.conf} templates/liberty```

For the moment policy.json and api-paste.ini can be used without modification but congress.conf needs to be updated to be a template with site specific information as well as setting some constants. Taking the congress.conf add variables for the keystone and mysql config:

```
auth_strategy = keystone
...
# List of driver class paths to import. (list value)
drivers = congress.datasources.neutronv2_driver.NeutronV2Driver,congress.datasources.glancev2_driver.GlanceV2Driver,congress.datasources.nova_driver.NovaDriver,congress.datasources.keystone_driver.KeystoneDriver,congress.datasources.ceilometer_driver.CeilometerDriver,congress.datasources.cinder_driver.CinderDriver,congress.datasources.swift_driver.SwiftDriver,congress.datasources.plexxi_driver.PlexxiDriver,congress.datasources.vCenter_driver.VCenterDriver,congress.datasources.murano_driver.MuranoDriver,congress.datasources.ironic_driver.IronicDriver
...
connection = {{ shared_db.uri }}
...
[keystone_authtoken]
{% if identity_service.auth_host -%}
auth_uri = {{ identity_service.service_protocol }}://{{ identity_service.service_host }}:{{ identity_service.service_port }}
auth_url = {{ identity_service.auth_protocol }}://{{ identity_service.auth_host }}:{{ identity_service.auth_port }}
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = {{ identity_service.service_tenant }}
username = {{ identity_service.service_username }}
password = {{ identity_service.service_password }}
{% endif -%}

```

### Render the config

Now the templates and interfaces are in place the configs can be rendered. A side-effect of rendering the configs is that any associated services are restarted. Finally, set the config.complete state this will be used later to trigger other events.

```
@reactive.when('shared-db.available')
@reactive.when('identity-service.available')
def render_congress_config(identity_interface, db_interface):
    # Render the congress config files and communicate that by setting the
    # config.complete state
    charm = congress.CongressCharmFactory.charm(
        interfaces=[identity_interface, db_interface]
    )
    charm.render_all_configs()
    reactive.set_state('config.complete')
```

### Run DB Migration

The DB migration can only be run once the config files are in place since as congress.conf will contain the DB connection information. To achieve this the DB migration is gated on the config.complete being set. Finally set the db.synched event so that this is only run once.

```
@reactive.when('config.complete')
@reactive.when_not('db.synched')
def run_db_migration():
    charm = get_charm()
    charm.db_sync()
    reactive.set_state('db.synched')
    charm.restart_all()
```

## Build and Deploy charm
Build the charm to pull down the interfaces and layers.

```
mkdir build
charm build -obuild charm
```

The build charm can now be deployed with juju

```
cd build
mkdir xenial
cd xenial
ln -s ../trusty/congress
juju deploy local:xenial/congress
juju add-relation congress mysql
juju add-relation congress keystone 
juju add-relation congress rabbitmq-server
```
