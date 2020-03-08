Continue in adfnis playbook

ANSIBLE PLAYBOOK for MariaDB Galera cluster
===================================================

These roles allow you to automatically setup a MariaDB Galera cluster with sane
default settings. 
Inpsired by https://github.com/olafz/percona-clustercheck, your database proxy/lb (like HAProxy) can healhchecks galera cluster sync in every nodes by calling port 9200 (200: healthy)

Tested in Ubuntu 16.04.

Prerequisites
-------------

The machine from which the playbook is being run needs to have Ansible >= 2.5
installed. For detailed information how to obtain current packages for your
distribution of choice have a look at the
[Ansible documentation](https://docs.ansible.com/ansible/intro_installation.html).

As we're accessing information from a group of hosts within these playbooks we
need to have fact caching enabled in Ansible. To, for example, cache using json
files in you homedirectory you would add the following lines to
``/etc/ansible/ansible.cfg``:

```
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = ~/.ansible/cache
```

For more information about other cache mechanisms have a look at the
[Ansible documentation](https://docs.ansible.com/ansible/playbooks_variables.html#fact-caching)
regarding fact-caching.

Installing requirements
-----------------------

To install the required packages and configure SELinux and the firewall you can
run only the tasks tagged with ``setup``

    ansible-playbook -i galera.hosts galera.yml --tags setup

Configure MariaDB Galera cluster
--------------------------------

To run all further tasks to configure MariaDB Galera cluster and add the
required user for the **S**tate **S**napshot **T**ransfer (SST) run the tags ``config``, ``auth`` and ``healthcheck`` directly.

    ansible-playbook -i galera.hosts galera.yml --tags auth, config, healthcheck

Bootstrapping MariaDB Galera cluster
------------------------------------

Bootstrapping the cluster can be done using a playbook dedicated for
bootstrapping the MariaDB Galera cluster called galera_bootstrap.yml

    ansible-playbook -i galera.hosts galera_bootstrap.yml

It uses the first node in the group ``galera_cluster`` to bootstrap the cluster.

Rolling MariaDB Galera cluster updates
--------------------------------------

The role ``galera_conf`` is set up to allow rolling updates of the configuration
of the Galera cluster. When the variable ``bootstrapped`` is set to ``yes`` the
``mariadb`` service is restarted after a change of the configuration. It is
important to note that the playbook contains the keyword ``serial: 1``, meaning
that the configuration is applied one node at a time so that the will never lose
quorum in the process of applying the new configuration.

    ansible-playbook -i galera.hosts galera_rolling_update.yml

Remove MariaDB Galera cluster
------------------------------------

Remove the cluster can be done using a playbook dedicated for
remove the MariaDB Galera cluster called galera_remove.yml

    ansible-playbook -i galera.hosts galera_remove.yml

Remove all packages and data_dir. 

Configure Database Proxy with HAProxy and Setup healthcheck
------------------------------------
Assume you already have a seperate HAProxy node and run role healthcheck.
Below is a sample configuration for HAProxy. The point of this is that the application will be able to connect to localhost port 3307, so although we are using Galera Cluster with several nodes, the application will see this as a single MySQL server running on localhost.

/etc/haproxy/haproxy.cfg

...
listen galera-cluster 0.0.0.0:3307
  balance leastconn
  option httpchk
  mode tcp
    server node1 1.2.3.4:3306 check port 9200 inter 5000 fastinter 2000 rise 2 fall 2
    server node2 1.2.3.5:3306 check port 9200 inter 5000 fastinter 2000 rise 2 fall 2
    server node3 1.2.3.6:3306 check port 9200 inter 5000 fastinter 2000 rise 2 fall 2 backup
MySQL connectivity is checked via HTTP on port 9200. The clustercheck script is a simple shell script which accepts HTTP requests and checks MySQL on an incoming request. If the Galera Cluster node is ready to accept requests, it will respond with HTTP code 200 (OK), otherwise a HTTP error 503 (Service Unavailable) is returned.

Vagrant support
---------------

For the live demo during the presentation I've been using Vagrant and with the
following instructions you can use the Vagrantfile to perform those steps.

### Prerequisites

As we need support for Vagrant private_network we need Guest Additions
installed in the guest OS. The ``vagrant-vbguest`` plugins automatically
takes care of this.

    vagrant plugin install vagrant-vbguest

### Vagrant setup

By running `vagrant up` all Ansible tasks in the galera.yml playbook will be
run. To bootstrap the cluster afterwards a separate run of the
galera_bootstrap.yml playbook is required.

    PYTHONUNBUFFERED=1 ANSIBLE_FORCE_COLOR=true ANSIBLE_HOST_KEY_CHECKING=false ANSIBLE_SSH_ARGS='-o UserKnownHostsFile=/dev/null -o IdentitiesOnly=yes -o ControlMaster=auto -o ControlPersist=60s' ansible-playbook --connection=ssh --inventory-file=.vagrant/provisioners/ansible/inventory --sudo galera_bootstrap.yml
