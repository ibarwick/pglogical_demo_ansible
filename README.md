pglogical_demo_ansible
======================

This repository contains a set of Ansible playbooks to easily
create a simple `pglogical` set-up with two or more nodes running
on different ports on localhost.

For more details on 2ndQuadrant's `pglogical`, see:

  http://2ndquadrant.com/en/resources/pglogical/


Requirements
------------

- Ansible ( http://www.ansible.com/ )
- psycopg2
- PostgreSQL 9.5 or 9.6 with the pglogical extension(s) available
  Note: `pglogical_demo_ansible` will not build the pglogical extensions(s);
  see https://2ndquadrant.com/en/resources/pglogical/pglogical-docs/ for details
  on how to do this.


pglogical Configuration
-----------------------

Create a file `your_hostname.yml` in the `host_vars` directory
with the following variables (default values in parentheses)

- `db_superuser` (`postgres`)
- `db_name` (`pgl_demo`)
- `pg_bin`: path to the pglogical-enabled PostgreSQL `bin/` directory
- `base_data_dir`: arbitrary directory for each instance's data files
- `base_log_dir` (`/tmp`): directory for PostgreSQL log files
- `pgl_ports`: list of ports for PostgreSQL instances
- `pgl_provider`: port number of the initial PostgreSQL instance,
  from which the other instances will subscribe from (usually just
  the first entry in `pgl_ports`).

Add `your_hostname` to the `[hosts]` section of `hosts.ini`

The file `host_vars/sample_host.yml` provides a useful template containing the
parameters which will probably need to be configured.

MacPorts users: if Ansible complains about `psycopg2` being missing, it is
probably using the OS X native Python interpreter; set `ansible_python_interpreter`
to point to the MacPorts version.


Create a pglogical cluster
--------------------------

    $ ansible-playbook -i hosts.ini pgl_init.yml

    PLAY [all] *********************************************************************

    TASK [pgl_init : Create data directories] **************************************
    changed: [osaka] => (item=10001)
    changed: [osaka] => (item=10002)
    changed: [osaka] => (item=10003)

    TASK [pgl_init : Perform initdb] ***********************************************
    changed: [osaka] => (item=10001)
    changed: [osaka] => (item=10002)
    changed: [osaka] => (item=10003)

    (...)

    TASK [pgl_init : Create required extensions] ***********************************
    changed: [osaka] => (item=[10001, u'pglogical'])
    changed: [osaka] => (item=[10002, u'pglogical'])
    changed: [osaka] => (item=[10003, u'pglogical'])

    TASK [pgl_init : Create required extensions (9.4 only)] ************************
    skipping: [osaka] => (item=[10001, u'pglogical_origin'])
    skipping: [osaka] => (item=[10002, u'pglogical_origin'])
    skipping: [osaka] => (item=[10003, u'pglogical_origin'])

    TASK [pgl_init : Generate SQL to create provider node] *************************
    ok: [osaka]

    TASK [pgl_init : Generate SQL to create subscriber node] ***********************
    skipping: [osaka] => (item=10001)
    ok: [osaka] => (item=10002)
    ok: [osaka] => (item=10003)

    TASK [pgl_init : Create provider node] *****************************************
    changed: [osaka]

    TASK [pgl_init : Create subscriber nodes] **************************************
    skipping: [osaka] => (item=10001)
    changed: [osaka] => (item=10002)
    changed: [osaka] => (item=10003)

    PLAY RECAP *********************************************************************
    osaka                      : ok=15   changed=13   unreachable=0    failed=0

A test table `pgl_demo` has been pre-created by the scripts; insert a row on
the provider and it will appear on the subscriber(s):

    $ psql -U postgres -d pgl_demo -h localhost -p 10001
    psql (9.6.2)
    Type "help" for help.

    pgl_demo=# INSERT INTO pgl_test values(1,'foo');
    INSERT 0 1
    Time: 5.239 ms
    pgl_demo=# \q

    $ psql -U postgres -d pgl_demo -h localhost -p 10002
    psql (9.6.2)
    Type "help" for help.

    pgl_demo=# SELECT * FROM pgl_test ;
     id | val
    ----+-----
      1 | foo
    (1 row)

    Time: 0.755 ms

Playbooks
---------

Following playbooks have been defined:

- `pgl_init.yml`

  Creates and configures pglogical nodes

- `pgl_destroy.yml`

  Halts existing pglogical nodes and removes the database directories

- `pgl_start.yml`

  Starts previously configured but not running pglogical nodes

- `pgl_stop.yml`

  Stops running pglogical nodes
