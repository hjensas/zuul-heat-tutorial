================================
Zuul Heat Tutorial Environment
================================

A tutorial environment for Zuul CI with OpenStack Heat integration. This
tutorial is based on the `Zuul Quick-Start Tutorial
<https://zuul-ci.org/docs/zuul/latest/tutorials/quick-start.html>`_ from the
Zuul project, extended to demonstrate the Heat driver for dynamic OpenStack
resource provisioning.

Prerequisites
=============

* OpenStack cloud with Heat support
* The following resources in your OpenStack cloud:

  * A Glance image named exactly ``ubuntu-noble-server`` (the tutorial
    configuration uses this name; if your image has a different name, you'll
    need to update both ``heat_tutorial.yaml`` and the provider config)
  * Nova flavors: ``m1.large`` (for the lab server) and ``m1.small`` (for Heat
    test stacks)
  * An external network named ``public`` (for floating IPs), or customize the
    template parameters to use your cloud's external network name
  * **If using the Heat template to deploy the lab server**: An existing
    tenant network named ``private``, or customize the ``private_network``
    parameter in ``heat_tutorial.yaml``
  * A Nova keypair for SSH access to the lab server

Deploying the Lab Server
=========================

You can use any VM with Docker and docker-compose installed as your lab server.
For convenience, this repository includes a Heat template that automates the
deployment:

.. code-block:: bash

   openstack stack create -t heat_tutorial.yaml zuul-heat-tutorial

The template will create a VM with:

* Docker and docker-compose pre-installed
* Necessary development tools
* Configured hostname entry in ``/etc/hosts``
* A floating IP for remote access

**If using the Heat template**, get the floating IP and add it to your
workstation's ``/etc/hosts``:

.. code-block:: bash

   LAB_IP=$(openstack stack output show zuul-heat-tutorial floating_ip -f json | jq -r .output_value)
   echo "$LAB_IP  zuul-heat-tutorial.lab.example.com" | sudo tee -a /etc/hosts

**If using your own VM**, ensure it has Docker, docker-compose, and git
installed, then add the hostname entry **inside the VM** (so services can
reference each other):

.. code-block:: bash

   echo "127.0.0.1 zuul-heat-tutorial.lab.example.com" | sudo tee -a /etc/hosts

.. note::

   If you're accessing the VM remotely from your workstation (not the case if
   you have direct console/GUI access), also add the hostname on your
   workstation:

   .. code-block:: bash

      echo "YOUR_VM_IP  zuul-heat-tutorial.lab.example.com" | sudo tee -a /etc/hosts

The hostname ``zuul-heat-tutorial.lab.example.com`` must be used as it's
referenced throughout the configuration files.

Building the Containers
========================

**All steps in this section should be run on the lab server.**

Since the Heat driver is not yet merged into Zuul's main branch, this tutorial
builds Zuul containers from source to include the Heat driver changes. If you
want to use official Zuul images instead, skip to the Configuration section
below.

Clone the Zuul Repository
--------------------------

.. code-block:: bash

   git clone https://opendev.org/zuul/zuul
   cd zuul

Checkout the Heat Driver Review
--------------------------------

Since the Heat driver is not yet merged, you need to fetch and checkout the
review change. This will automatically fetch the latest patchset:

.. code-block:: bash

   # Define the change ID
   CHANGE_ID="989884"

   # Fetch the latest ref string from the OpenDev API
   LATEST_REF=$(curl -s "https://review.opendev.org/changes/$CHANGE_ID/revisions/current/review" | tail -n +2 | jq -r '.current_revision')

   # Fetch and checkout
   git fetch https://review.opendev.org/zuul/zuul "$LATEST_REF" && git checkout FETCH_HEAD
   git switch -c heat-driver

Build Container Images
-----------------------

**Note:** This step is for development and testing of the Heat driver. If
you're using official Zuul container images, skip this section and omit the
``-f compose.local-images.yaml`` flag when running docker compose below.

Build the Zuul container images from source:

.. code-block:: bash

   sudo docker build -f Dockerfile --target zuul-executor  -t "localhost/zuul-ci/zuul-executor:dev" .
   sudo docker build -f Dockerfile --target zuul-launcher  -t "localhost/zuul-ci/zuul-launcher:dev" .
   sudo docker build -f Dockerfile --target zuul-scheduler -t "localhost/zuul-ci/zuul-scheduler:dev" .
   sudo docker build -f Dockerfile --target zuul-web       -t "localhost/zuul-ci/zuul-web:dev" .

Configuration
=============

OpenStack Configuration
-----------------------

The scheduler and launcher containers will mount your hosts OpenStack configuration
directory (``/etc/openstack``) at ``/etc/openstack`` in the container.

Your OpenStack configuration directory should contain:

* ``clouds.yaml`` - OpenStack cloud credentials
* Any CA certificates referenced in ``clouds.yaml``


**If your OpenStack configuration is in a different location**, set the
``OPENSTACK_CONFIG`` environment variable:

.. code-block:: bash

   export OPENSTACK_CONFIG=/path/to/your/openstack/config


Running Docker Compose
=======================

Clone the repository and run docker compose from the ``zuul-heat-tutorial``
directory:

.. code-block:: bash

   cd ..
   git clone https://github.com/hjensas/zuul-heat-tutorial.git
   cd zuul-heat-tutorial

**If you built containers from source**, include the ``compose.local-images.yaml``
file to use your locally-built images:

.. code-block:: bash

   sudo docker compose -f docker-compose.yaml -f compose.local-images.yaml -p zuul-heat-tutorial up -d

**Otherwise**, omit that file:

.. code-block:: bash

   sudo docker compose -f docker-compose.yaml -p zuul-heat-tutorial up -d

**Useful commands:**

.. code-block:: bash

   # View logs
   sudo docker compose -p zuul-heat-tutorial logs -f

   # Stop the services
   sudo docker compose -p zuul-heat-tutorial down

Accessing Services
==================

Once the compose is running, you can access:

* **Zuul Web UI**: http://zuul-heat-tutorial.lab.example.com:9000
* **Gerrit**: http://zuul-heat-tutorial.lab.example.com:8080
* **Build Logs**: http://zuul-heat-tutorial.lab.example.com:8000

----

=========================================
Tutorial: Configuring the Heat Driver
=========================================

This tutorial walks through configuring Zuul with the Heat driver for dynamic
OpenStack resource provisioning via Heat orchestration templates.

The basic Zuul configuration (pipelines, base job, project templates) has
already been set up automatically by the initialization script. This tutorial
focuses on adding the Heat driver configuration and creating Heat-based jobs.

Add Your Gerrit Account
========================

Gerrit is configured in development mode where passwords are not required and
you can become any user.

1. Visit ``http://zuul-heat-tutorial.lab.example.com:8080`` in your browser
2. Click **Sign in** in the top right corner
3. Click **New Account** under Register
4. Don't enter anything in the confirmation dialog, instead click the
   **settings** link at the bottom
5. In the **Profile** section:

   * Enter your workstation username in the **Username** field
   * Enter your full name in the **Full name** field
   * Click **Save Changes**

6. Scroll to **Email Addresses** section:

   * Enter your email address
   * Click **Send Verification** (it will be automatically confirmed in dev
     mode)

7. Scroll to **SSH keys** section:

   * Copy and paste the contents of ``~/.ssh/id_rsa.pub`` (or your SSH public
     key)
   * Click **Add New SSH Key**

8. Click **Reload** in your browser

You're now logged into your personal Gerrit account.

Verify Zuul Configuration
==========================

The initialization script has already bootstrapped the ``zuul-config`` project
with:

* **Check** and **gate** pipelines for code review workflow
* **Base job** configuration with standard playbooks
* **Project templates** for all repositories

The ``zuul-config`` project is now ready to use. Zuul will automatically load
this configuration when it starts.

**Understanding the setup:**

Zuul recognizes two types of projects:

* **Config projects**: Contain Zuul's configuration (trusted, not dynamically
  evaluated) - like ``zuul-config``
* **Untrusted projects**: Normal projects with code and jobs (dynamically
  evaluated) - like ``test1`` and ``test2``

The bootstrap script created three projects with initial configuration:

* ``zuul-config`` - with pipelines, base job, and project templates
* ``test1`` - empty project for testing
* ``test2`` - empty project for testing

You can view the bootstrapped configuration by cloning the repo:

.. code-block:: bash

   cd ..
   git clone http://zuul-heat-tutorial.lab.example.com:8080/zuul-config
   cd zuul-config

The repository contains:

* ``zuul.d/pipelines.yaml`` - check and gate pipelines
* ``zuul.d/projects.yaml`` - project templates
* ``zuul.d/jobs.yaml`` - base job definition
* ``playbooks/base/`` - pre-run, post-run, and cleanup playbooks

Configure Heat Driver
=====================

The Heat driver enables Zuul to provision dynamic OpenStack resources using
Heat Orchestration Templates (HOT).

**Understanding Heat Provider:**

* Uses **Heat Orchestration Templates (HOT)** stored in your project repository
* Each template defines a complete environment (VMs, networks, routers,
  volumes, etc.)
* Each Heat stack = one Zuul gateway node (a bastion host that Zuul connects
  to directly)
* Additional nodes in the same nodeset can be declared as **via-nodes** —
  they appear in the Ansible inventory and are reached through the gateway
  via SSH ProxyJump
* The heat flavor configuration defines **resource limits** that Zuul enforces
* Templates must output ``bastion_ip`` so Zuul can SSH to the gateway host
* Templates that expose private nodes as via-nodes must also output
  ``zuul_via_nodes`` (see the `Zuul nodeset documentation
  <https://zuul-ci.org/docs/zuul/latest/config/nodeset.html>`_ and the
  `Heat driver via-nodes section
  <https://zuul-ci.org/docs/zuul/latest/drivers/heat.html#via-nodes-in-the-inventory>`_)

Verify OpenStack Credentials
-----------------------------

The ``zuul.conf`` file is already configured with a Heat connection named
``heat-cloud`` (lines 60-63). It references ``/etc/openstack/clouds.yaml``
which should be mounted from your ``$OPENSTACK_CONFIG`` directory.

**Important:** Ensure the ``cloud=`` value in ``etc_zuul/zuul.conf`` (line 62)
matches an entry in your ``clouds.yaml`` file. The default is
``cloud=default``. If your cloud has a different name in ``clouds.yaml``,
update the zuul.conf file accordingly and restart the scheduler and launcher
services:

.. code-block:: bash

   # On your lab server (only if you need to change the cloud name)
   cd ~/zuul-heat-tutorial
   # Edit etc_zuul/zuul.conf and change cloud=default to your cloud name
   sudo docker compose -p zuul-heat-tutorial restart scheduler launcher

Heat Template Directories (Pre-configured)
------------------------------------------

The tutorial environment is already configured to recognize Heat templates in
the ``zuul.heat.d/`` directory of the ``test1`` project. This is set in
``etc_zuul/main.yaml`` via the ``heat-template-dirs`` option:

.. code-block:: yaml

           untrusted-projects:
             - test1:
                 heat-template-dirs:
                   - zuul.heat.d

This tells Zuul to fetch Heat template YAML files from ``zuul.heat.d/`` during
the merge phase. Templates added or modified in a proposed change are resolved
correctly because they're fetched from the merged commit state.

Create Nova Keypair for Executor SSH Access
--------------------------------------------

The Heat driver passes a ``zuul_key_name`` parameter to every template so the
gateway and any via-nodes can be configured with the executor's SSH public
key. You need to create a Nova keypair from the executor's nodepool public
key:

.. code-block:: bash

   # On the lab server — create the keypair directly from the Docker volume
   sudo OS_CLOUD=default openstack keypair create --public-key /var/lib/docker/volumes/zuul-heat-tutorial_sshkey/_data/nodepool.pub zuul-nodes

The keypair name (``zuul-nodes``) must match the ``key-name`` value in the-
label configuration below.

Configure Heat Provider in zuul-config
---------------------------------------

Now we'll add the Heat provider configuration to ``zuul-config``.

If you haven't already cloned the ``zuul-config`` repository, clone it now to your
workstation (from any directory where you want to keep your Zuul projects):

.. code-block:: bash

   git clone http://zuul-heat-tutorial.lab.example.com:8080/zuul-config
   cd zuul-config
   git review -s

If you already cloned it earlier, just navigate to the directory:

.. code-block:: bash

   cd zuul-config
   git review -s

Create the Heat provider configuration:

.. code-block:: bash

   cat << 'EOF' > zuul.d/providers.yaml

   # Heat Provider Configuration
   # Heat uses templates stored in project repos to define complete environments

   # Top-level flavor definition (basic attributes)
   - flavor:
       name: tutorial-heat-flavor
       description: "Heat flavor defines resource limits per stack (not a Nova compute flavor)"

   # Top-level image definition for the bastion node
   - image:
       name: bastion-image
       type: cloud

   # Top-level label definition (basic attributes)
   - label:
       name: heat-lab
       flavor: tutorial-heat-flavor
       image: bastion-image
       description: "Label for Heat-based test environments"

   # Section with Heat-specific overrides
   - section:
       name: heat
       connection: heat-cloud
       region: RegionOne
       label-defaults:
         key-name: zuul-nodes
       flavors:
         - name: tutorial-heat-flavor
           max-ram: 32768                      # 32 GiB in MiB (32 * 1024)
           max-cores: 64                       # 64 CPU cores total
           max-instances: 20                   # Up to 20 Nova instances
           max-floating-ips: 5                 # Up to 5 floating IPs
           max-networks: 5                     # Up to 5 Neutron networks
           max-routers: 2                      # Up to 2 Neutron routers
           max-routers-with-external-gw: 2     # Up to 2 routers with external gateway
           max-volumes: 10                     # Up to 10 Cinder volumes
           max-volume-gb: 100                  # 100 GB total volume storage
       images:
         - name: bastion-image
           image-id: ubuntu-noble-server
           username: ubuntu
           connection-type: ssh

   # Provider references section and lists available labels
   - provider:
       name: heat-provider
       section: heat
       resource-limits:
         stacks: 3
       resource-policy:
         allowed-resource-types:
           - OS::Nova::.*
           - OS::Neutron::.*
           - OS::Cinder::.*
           - OS::Heat::CloudConfig
           - OS::Heat::MultipartMime
           - OS::Heat::Value
           - OS::Heat::Delay
           - OS::Heat::RandomString
       labels:
         - name: heat-lab
   EOF

**Note:** Adjust the following to match your OpenStack environment:

* ``image-id: ubuntu-noble-server`` - your Glance image name or UUID
* ``key-name: zuul-nodes`` - must match the Nova keypair you created above
* ``region: RegionOne`` - your OpenStack region name
* Resource limits (``max-ram``, ``max-cores``, etc.) - adjust based on your
  quota and requirements
* ``resource-policy.allowed-resource-types`` - restricts which OpenStack
  resource types can be used in Heat templates (templates using unapproved types
  will be rejected). Supports regex patterns. The patterns above allow all Nova
  compute resources (``OS::Nova::.*``), all Neutron networking resources
  (``OS::Neutron::.*``), all Cinder volume resources (``OS::Cinder::.*``), and
  specific Heat utility resources: ``OS::Heat::CloudConfig`` for cloud-init,
  ``OS::Heat::MultipartMime`` for combining cloud-config parts,
  ``OS::Heat::Value`` for computed values, ``OS::Heat::Delay`` for orchestration
  delays, and ``OS::Heat::RandomString`` for generating random strings
  (passwords, tokens, etc.). Templates using other resource types (e.g.,
  ``OS::Swift::Container``, ``OS::Trove::Instance``, ``OS::Heat::ResourceGroup``)
  will be denied

For detailed documentation on Heat provider configuration and all available
options, see the `Zuul Heat Driver documentation
<https://zuul-ci.org/docs/zuul/latest/drivers/heat.html>`_.

**Understanding the configuration structure:**

Zuul uses a **three-tier** configuration pattern for providers:

1. **Top-level definitions** (flavor, image, label): Basic attributes and
   references

   * ``flavor: tutorial-heat-flavor`` - just the name and description
   * ``image: bastion-image`` - image name and type
   * ``label: heat-lab`` - references the flavor and image by name

2. **Section** (``section: heat``): Driver-specific overrides and connection

   * ``connection: heat-cloud`` - references the connection in ``zuul.conf``
   * ``region`` - the OpenStack region (set at section level, like other
     drivers)
   * ``label-defaults`` - common label attributes applied to all labels (e.g.,
     ``key-name``)
   * ``flavors`` - overrides the top-level flavor with Heat-specific
     attributes (``max-floating-ips``, etc.)
   * ``images`` - adds ``image-id`` (Glance image name/UUID), ``username``,
     and ``connection-type``

3. **Provider** (``provider: heat-provider``): References section and resource
   limits

   * ``section: heat`` - references the section defined above
   * ``resource-limits`` - provider-level limits (e.g., ``stacks: 3``)
   * ``resource-policy`` - optional policy to restrict resource types in templates

     * ``allowed-resource-types`` - list of permitted Heat resource types
       (e.g., ``OS::Nova::Server``, ``OS::Neutron::Net``). Supports regex
       patterns (e.g., ``OS::Nova::.*`` to allow all Nova resources). When
       specified, templates using resource types not matching these patterns
       will be rejected. This provides security by preventing templates from
       creating unexpected resources
     * Alternatively, use ``disallowed-resource-types`` to block specific types
       (but not both)

   * ``labels`` - lists which labels this provider offers

Commit and Merge Heat Provider Configuration
---------------------------------------------

Now commit and upload the Heat provider configuration to Gerrit:

.. code-block:: bash

   git add zuul.d/providers.yaml
   git commit -m "Add Heat provider configuration"
   git review

Approve and merge the change in Gerrit. Zuul will automatically load the new
provider configuration.

Monitor Zuul's scheduler log to verify it loaded the Heat provider:

.. code-block:: bash

   # On the lab server
   sudo docker logs zuul-scheduler -f
   # Look for: "Loaded configuration from ..." and verify no errors for heat-provider

Create a Heat Template
----------------------

The Heat driver requires a Heat template in your project. Let's create a
simple one.

Clone the test1 project to your workstation:

.. code-block:: bash

   cd ..
   git clone http://zuul-heat-tutorial.lab.example.com:8080/test1
   cd test1
   git review -s
   mkdir -p zuul.heat.d

Create a Heat template with a bastion gateway and an internal via-node:

.. code-block:: bash

   cat << 'EOF' > zuul.heat.d/lab-stack.yaml
   heat_template_version: 2015-04-30

   description: >
     Lab environment with bastion gateway and internal node for Zuul CI.
     The bastion is reachable via floating IP; the internal node is on a
     private network. Zuul reaches internal as a via-node (ProxyJump) and
     the bastion can also SSH to internal directly using an ephemeral key.

   parameters:
     zuul_key_name:
       type: string
       description: Cloud keypair name injected by Zuul for executor access
     zuul_username:
       type: string
       default: zuul
       description: Username injected by Zuul for SSH access
     zuul_image:
       type: string
       description: Glance image for all nodes
     zuul_ephemeral_public_key:
       type: string
       description: Ed25519 public key for bastion-to-internal SSH
     zuul_ephemeral_private_key:
       type: string
       hidden: true
       description: Ed25519 private key injected on bastion for internal access
     zuul_external_network:
       type: string
       description: >
         External network for floating IPs and router gateway, auto-discovered
         and injected by the Zuul Heat driver
     custom_parameter:
       type: string
       default: default-value
       description: Custom parameter (demonstrates parameter override)

   resources:
     private_net:
       type: OS::Neutron::Net

     private_subnet:
       type: OS::Neutron::Subnet
       properties:
         network: { get_resource: private_net }
         cidr: 192.168.100.0/24
         dns_nameservers:
           - 8.8.8.8

     router:
       type: OS::Neutron::Router
       properties:
         external_gateway_info:
           network: { get_param: zuul_external_network }

     router_interface:
       type: OS::Neutron::RouterInterface
       properties:
         router: { get_resource: router }
         subnet: { get_resource: private_subnet }

     bastion_write_files:
       type: OS::Heat::CloudConfig
       properties:
         cloud_config:
           write_files:
             - defer: true
               path:
                 str_replace:
                   template: /home/$USER/.ssh/id_ed25519
                   params:
                     $USER: { get_param: zuul_username }
               permissions: '0600'
               owner:
                 str_replace:
                   template: $USER:$USER
                   params:
                     $USER: { get_param: zuul_username }
               content: { get_param: zuul_ephemeral_private_key }

     bastion_init:
       type: OS::Heat::MultipartMime
       properties:
         parts:
           - config: { get_resource: bastion_write_files }

     internal_init:
       type: OS::Heat::CloudConfig
       properties:
         cloud_config:
           ssh_authorized_keys:
             - { get_param: zuul_ephemeral_public_key }

     bastion_port:
       type: OS::Neutron::Port
       properties:
         network: { get_resource: private_net }
         fixed_ips:
           - subnet: { get_resource: private_subnet }
         security_groups:
           - default

     bastion:
       type: OS::Nova::Server
       properties:
         flavor: m1.small
         image: { get_param: zuul_image }
         key_name: { get_param: zuul_key_name }
         networks:
           - port: { get_resource: bastion_port }
         user_data: { get_resource: bastion_init }
         user_data_format: RAW

     internal_node:
       type: OS::Nova::Server
       properties:
         flavor: m1.small
         image: { get_param: zuul_image }
         key_name: { get_param: zuul_key_name }
         networks:
           - network: { get_resource: private_net }
         user_data: { get_resource: internal_init }
         user_data_format: RAW

     bastion_fip:
       type: OS::Neutron::FloatingIP
       properties:
         floating_network: { get_param: zuul_external_network }
         port_id: { get_resource: bastion_port }

   outputs:
     bastion_ip:
       description: Floating IP of bastion host (REQUIRED by Zuul)
       value: { get_attr: [bastion_fip, floating_ip_address] }
     zuul_via_nodes:
       description: Connection details for via-nodes in the nodeset
       value:
         internal:
           host: { get_attr: [internal_node, first_address] }
           user: { get_param: zuul_username }
     internal_node_ip:
       description: Private IP of internal node (for bastion SSH hop testing)
       value: { get_attr: [internal_node, first_address] }
     custom_parameter:
       description: Custom parameter value (demonstrates get_param)
       value: { get_param: custom_parameter }
   EOF

**Important notes about the template:**

* The template demonstrates **two ways** to reach the internal node:

  * **Via-node (ProxyJump)**: Zuul connects from the executor through the
    bastion to ``internal`` using ``zuul_key_name`` on both nodes. Connection
    details come from the ``zuul_via_nodes`` output
  * **Bastion SSH hop**: The bastion receives an ephemeral private key via
    cloud-init and the internal node authorizes the matching public key. Job
    playbooks running on ``bastion`` can ``ssh`` to the internal IP directly

* Both the **bastion** and **internal_node** use ``zuul_key_name`` as their
  Nova ``key_name`` so the executor can reach the via-node. The ephemeral
  keypair is an additional credential for bastion-to-internal access
* **MultipartMime on bastion**: ``defer: true`` on ``bastion_write_files``
  ensures the ephemeral private key is written after cloud-init has created
  the ``.ssh`` directory
* The ``zuul_via_nodes`` output maps the nodeset node name ``internal`` to
  the internal VM's private IP. The Heat driver copies this into
  ``node_properties.via_nodes`` on the gateway node after the stack reaches
  ``CREATE_COMPLETE``
* The Heat driver only passes a ``zuul_*`` parameter when the template
  declares it — undeclared parameters are simply not sent. Declare only the
  ones your template actually uses
* The template **must output** ``bastion_ip`` — this is how Zuul knows where
  to connect
* Adjust the hardcoded ``flavor: m1.small`` to match your OpenStack
  environment

Create a Heat-based Test Job
-----------------------------

First, create a test playbook that validates **both** connectivity paths —
via-node (executor ProxyJump to ``internal``) and bastion SSH hop
(``bastion`` → internal using the ephemeral key):

.. code-block:: bash

  mkdir -p playbooks
  cat << 'EOF' > playbooks/testjob.yaml
  - hosts: bastion
    gather_facts: true
    tasks:
      - name: Show bastion node information
        debug:
          msg: |
            Running on gateway node {{ inventory_hostname }}
            Hostname: {{ ansible_hostname }}

      - name: Display Heat stack outputs
        debug:
          msg: |
            Heat Stack Outputs:
            {{ nodepool.node_properties.heat_stack_outputs | to_nice_yaml }}

      - name: Check if cloud-init is complete on bastion
        command: cloud-init status --wait
        register: cloud_init_status
        ignore_errors: true

      - name: Display cloud-init status
        debug:
          var: cloud_init_status

      - name: Check for ephemeral SSH key on bastion
        stat:
          path: ~/.ssh/id_ed25519
        register: internal_key

      - name: Verify ephemeral key was injected on bastion
        debug:
          msg: "Ephemeral SSH key for bastion-to-internal hop: {{ 'present' if internal_key.stat.exists else 'missing' }}"

      - name: Test SSH from bastion to internal node
        shell: |
          ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
            -i ~/.ssh/id_ed25519 \
            {{ nodepool.node_properties.heat_stack_outputs.zuul_via_nodes.internal.user | default('ubuntu') }}@{{ nodepool.node_properties.heat_stack_outputs.internal_node_ip }} \
            uname -a
        register: bastion_ssh_test
        ignore_errors: true

      - name: Show bastion-to-internal SSH test result
        debug:
          msg: |
            Bastion SSH hop to internal: {{ 'SUCCESS' if bastion_ssh_test.rc == 0 else 'FAILED' }}
            Output: {{ bastion_ssh_test.stdout }}

  - hosts: internal
    gather_facts: true
    tasks:
      - name: Show internal via-node information
        debug:
          msg: |
            Running on via-node {{ inventory_hostname }}
            Hostname: {{ ansible_hostname }}
            Connected from executor through bastion via SSH ProxyJump

      - name: Verify internal node is reachable via via-node
        command: uname -a
        register: via_node_uname

      - name: Show internal node uname (via-node path)
        debug:
          var: via_node_uname.stdout

  - hosts: all
    tasks:
      - name: Confirm all nodeset nodes are reachable
        debug:
          msg: "Successfully reached {{ inventory_hostname }} ({{ ansible_hostname }})"

      - name: Test the Heat stack environment
        shell: |
          echo "Heat stack environment is working!"
          echo "This job ran on a dynamically provisioned Heat stack."
        register: result

      - name: Show result
        debug:
          var: result.stdout_lines
  EOF

Now update ``.zuul.yaml`` to add a Heat-based job that references the
template:

.. code-block:: bash

   cat << 'EOF' >> .zuul.yaml

   # Nodeset using Heat template with a via-node
   - nodeset:
       name: heat-nodeset
       nodes:
         - name: bastion
           label: heat-lab
           template-config:
             heat-template-path: zuul.heat.d/lab-stack.yaml
             heat-parameters:
               custom_parameter: my-custom-value
         - name: internal
           via: bastion

   # Job using Heat resources
   - job:
       name: testjob-heat
       parent: base
       description: Test job that runs on Heat-provisioned environment
       run: playbooks/testjob.yaml
       nodeset: heat-nodeset

   # Add Heat job to project pipelines
   - project:
       check:
         jobs:
           - testjob-heat
       gate:
         jobs:
           - testjob-heat
   EOF

**Understanding the configuration:**

* The **nodeset** defines a gateway node using the ``heat-lab`` label and an
  **internal** via-node reached through the gateway
* Only the gateway node consumes a label slot — the via-node is not
  independently allocated but appears in the Ansible inventory
* ``template-config`` specifies the Heat template path (relative to the
  project root)
* ``heat-parameters`` allows you to pass custom parameter values to the Heat
  template - these override the default values defined in the template's
  ``parameters`` section
* The path must be inside a directory declared via ``heat-template-dirs`` in
  the tenant config
* Zuul will create the entire Heat stack defined in the template
* The Heat template's ``zuul_via_nodes`` output supplies connection details
  for the internal node; Zuul connects to it via SSH ProxyJump through bastion
* The template also injects an ephemeral keypair so playbooks on ``bastion``
  can SSH to the internal IP directly — a separate path from the via-node
* The job playbook validates both paths: ``hosts: internal`` exercises
  via-node/ProxyJump; tasks on ``bastion`` exercise the SSH hop

Upload and Test
---------------

Upload the test1 changes (including the Heat template and playbook):

.. code-block:: bash

   git add .zuul.yaml zuul.heat.d/ playbooks/
   git commit -m "Add Heat-based test job with orchestration template"
   git review

Visit the `Gerrit dashboard <http://zuul-heat-tutorial.lab.example.com:8080/dashboard/self>`_ to see
your change. Zuul will run the ``testjob-heat`` job which creates a Heat stack
(bastion + internal node) and validates both the via-node path (executor →
bastion → internal via ProxyJump) and the bastion SSH hop (bastion → internal
using the ephemeral key).

You can watch the `Zuul status page <http://zuul-heat-tutorial.lab.example.com:9000/t/example-tenant/status>`_ to
see the Heat stack being provisioned.

Understanding Heat Provider Behavior
------------------------------------

When a job uses a Heat-based nodeset:

1. **Zuul Launcher** receives the job request
2. **Heat template** is read from the project repository
   (``zuul.heat.d/lab-stack.yaml``)
3. **Heat API** is called to create a new stack from the template, with
   ``zuul_key_name``, ``zuul_username``, and ``zuul_image`` passed as
   parameters
4. **Stack orchestration** begins — Heat creates all resources defined in the
   template:

   * Networks and subnets
   * Routers and external connectivity
   * Security groups
   * VM instances (including the bastion)
   * Floating IPs
   * Any other resources in the template

5. Zuul waits for the **bastion host** to become accessible via SSH (using the
   ``bastion_ip`` output)
6. **Zuul Executor** builds the Ansible inventory:

   * The gateway node (``bastion``) is connected to directly
   * Via-nodes (``internal``) are added using connection details from
     ``node_properties.via_nodes``, with SSH ProxyJump through the gateway

7. The job playbook runs on all nodes in the nodeset — validating both the
   via-node path (``hosts: internal``) and bastion SSH hop (tasks on
   ``bastion`` using the ephemeral key)
8. After job completion, the **entire Heat stack is deleted**, cleaning up all
   resources

This provides true CI isolation - each job gets a fresh, dedicated environment
orchestrated exactly as defined in your Heat template. The template can define
arbitrarily complex environments with multiple VMs, networks, volumes, etc.

Troubleshooting
---------------

If the Heat job fails to start:

1. **Check launcher logs**: ``sudo docker compose -p zuul-heat-tutorial logs
   launcher``
2. **Verify Heat connection**: Ensure ``clouds.yaml`` is correct and
   accessible in the launcher container
3. **Verify Heat template**: Check the template syntax and ensure it's in the
   correct path (``zuul.heat.d/lab-stack.yaml``)
4. **Check OpenStack resources**:

   * Ensure the image (``image-id`` in section config) exists: ``openstack
     image list``
   * Verify you have quota for the resources defined in the template
5. **Check Heat stack status**: ``openstack stack list`` and ``openstack stack
   show <stack-name>`` to see stack creation errors
6. **Review template outputs**: The template must output ``bastion_ip`` for
   Zuul to connect. Via-nodes additionally require a ``zuul_via_nodes`` output
   whose keys match the via-node names in the nodeset
7. **Via-node SSH failures**: If the ``internal`` play fails but ``bastion``
   succeeds, check executor logs for ProxyJump or missing ``via_nodes`` errors

Common issues:

* Template missing ``bastion_ip`` output
* ``zuul_via_nodes`` output missing or using a key that does not match the
  via-node name in the nodeset (e.g. ``internal``)
* ``image-id`` in provider config doesn't match a Glance image name or UUID
* Nova keypair (``key-name``) doesn't exist or doesn't contain the executor's
  public key (must be authorised on both gateway and via-nodes)
* Insufficient quota for resources defined in template

Next Steps
----------

Now you have Zuul configured with the Heat provider. You can:

* **Use Heat orchestration** for tests requiring:

  * Multiple VMs with private networking
  * Custom network topologies
  * Persistent volumes
  * Multi-tier application stacks
  * Any resources Heat can orchestrate

* Create multiple Heat templates for different test scenarios
* Define resource policies to limit what templates can create

The key advantage of Heat is that your test environment is defined as code
(the Heat template) and can be versioned alongside your tests, ensuring
reproducible and isolated test environments.
