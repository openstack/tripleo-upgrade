===============
tripleo-upgrade
===============

This role aims to provide a unified tool for upgrading TripleO based deploments.

Requirements
------------

This role requires:

* An ansible inventory file containing reacheable undercloud and overcloud nodes

* Nodes in the inventory file are placed in groups based on their roles (e.g compute nodes are part of the 'compute' group)

* Repositories containing packages to be upgraded are already installed on undercloud and overcloud nodes (or, for overcloud, define an upgrade_init_command variable)

* The initial overcloud deploy command is placed in a script file located in the path set by the overcloud_deploy_script var. Each option/environment file should be placed on a separate new line, e.g::

    source ~/stackrc
    export THT=/usr/share/openstack-tripleo-heat-templates/

    openstack overcloud deploy --templates $THT \
    -r ~/openstack_deployment/roles/roles_data.yaml \
    -e $THT/environments/network-isolation.yaml \
    -e $THT/environments/network-management.yaml \
    -e $THT/environments/storage-environment.yaml \
    -e ~/openstack_deployment/environments/nodes.yaml \
    -e ~/openstack_deployment/environments/network-environment.yaml \
    -e ~/openstack_deployment/environments/disk-layout.yaml \
    -e ~/openstack_deployment/environments/neutron-settings.yaml \
    --log-file overcloud_deployment.log &> overcloud_install.log

Role Variables
--------------

Available variables are listed below::

    upgrade_noop: false

Only create upgrade scripts without running them::

    update_noop: false

Only create update scripts without running them::

    undercloud_upgrade: false

Run undercloud upgrade::

    containerized_undercloud_upgrade: false

Run containerized undercloud upgrade::

    overcloud_upgrade: false

Run overcloud upgrade::

    undercloud_update: false

Run undercloud update::

    overcloud_update: false

Run overcloud update::

    overcloud_deploy_script: "~/overcloud_deploy.sh"

Validate overcloud after update::

   overcloud_images_validate: false

Location of the initial overcloud deploy script::

    undercloud_upgrade_script: "~/undercloud_upgrade.sh"

Location of the undercloud upgrade script which is going to be generated by this role::

    overcloud_composable_upgrade_script: "~/composable_docker_upgrade.sh"

Location of the upgrade script used in the composable docker upgrade step which is going to be generated by this role::

    overcloud_converge_upgrade_script: "~/converge_docker_upgrade.sh"

Location of the upgrade script used in the converge docker upgrade step which is going to be generated by this role::

    undercloud_rc: "~/stackrc"

Location of the undercloud credentials file::

    overcloud_rc: "~/overcloudrc"

Location of the overcloud credentials file::

    upgrade_workarounds: false

Allows the user to apply known issues workarounds during the upgrade process. The list of patches/commands used for workarounds should be passed via --extra-vars and it should include dictionaries for undercloud/overcloud workarounds::

    use_oooq: false

Set to true when the deployment has been done by tripleo quickstart::

    workload_launch: false

Set to true to launch an instance before starting upgrade. This can be useful for running tests during upgrade such as live migration or floating IP connectivity checks::

    workload_cleanup: false

Set to true to cleanup previously launched workload when update/upgrade finishes::

    external_network_name: "public"

Name of the external network providing floating IPs for instance connectivity. This provides external connectivity and needs to exist beforehand, created by the user::

    workload_image_url: "https://download.cirros-cloud.net/0.5.2/cirros-0.5.2-x86_64-disk.img"

URL of the image used for the workload instance::

    workload_memory: "512"

Amount of memory assigned for the workload instance::

    workload_sriov: false

Set to true to use an SRIOV PF port when workload is created. Notice this will not work with cirros images::

    tripleo_ci: false

Set to true when running the role in the TripleO CI jobs. It avoids losing connectivity to the undercloud by skipping reboot and ssh kill tasks::

    upgrade_init_command: |
        sudo tripleo-repos -b pike current

Bash commands, defines a custom upgrade init to be taken into account during overcloud upgrade.

    update_cell: false

Set it to true to get a multi-cell update.  It changes the way the oc_roles_hosts is calculated.

     l3_agent_connectivity_check: false

When set to true add a vm with attached fip and monitor ping from the undercloud.  If ping loss time is higher than `loss_threshold` seconds or `loss_threshold_percent` in percentage we fail.

     update_loss_threshold: 0

For update run tasks we set a 0 seconds loss threshold by default.

     loss_threshold: 60

Default time is second for ping loss.

     loss_threshold_percent: 1

Failsafe percentage check for loss threashold in percentage

Set to true to enable validations::

    updates_validations: true


Dependencies
------------

None.


Example Playbook
----------------

An example playbook is provided in tests/test.yml::

    - hosts: undercloud
      gather_facts: true
      become: true
      become_method: sudo
      become_user: stack
      roles:
        - tripleo-upgrade


Usage with tripleo Quickstart
-----------------------------

After a successful deployment with OOOQ, you can create the necessary
scripts using this example playbook (duplicate from
./tests/oooq-test.yaml)::

    ---
    - hosts: undercloud
      gather_facts: true
      become: true
      become_method: sudo
      become_user: stack
      roles:
      - { role: tripleo-upgrade, use_oooq: 'true'}


And then you run it like this (adjust the paths to your oooq specific
one)::

   ANSIBLE_SSH_ARGS="-F $(pwd)/ssh.config.ansible" \
     ANSIBLE_CONFIG=$PWD/ansible.cfg \
     ansible-playbook -i hosts -vvv tripleo-upgrade/tests/oooq-test.yaml

This will only create the file (without running the actual upgrade):
 - undercloud_upgrade.sh
 - composable_docker_upgrade.sh
 - overcloud-compute-\*_upgrade_pre.sh
 - overcloud-compute-\*_upgrade.sh
 - overcloud-compute-\*_upgrade_post.sh
 - converge_docker_upgrade.sh

with the correct parameters.

Usage with InfraRed
-------------------

tripleo-upgrade comes preinstalled as an InfraRed plugin.
In order to install it manually, the following InfraRed command should be used::

    infrared plugin add tripleo-upgrade
    # add with a specific revision / branch
    infrared plugin add --revision stable/rocky tripleo-upgrade

After a successful InfraRed overcloud deployment you need to run the following steps to upgrade the deployment:

Symlink roles path::

    ln -s $(pwd)/plugins $(pwd)/plugins/tripleo-upgrade/infrared_plugin/roles

Set up undercloud upgrade repositories::

    infrared tripleo-undercloud \
        --upgrade yes \
        --mirror ${mirror_location} \
        --ansible-args="tags=upgrade_repos"

Set up undercloud update repositories::

    infrared tripleo-undercloud \
        --update-undercloud yes \
        --mirror ${mirror_location} \
        --build latest \
        --version 12 \
        --ansible-args="tags=upgrade_repos"

Upgrade undercloud::

    infrared tripleo-upgrade \
        --undercloud-upgrade yes

Update undercloud::

    infrared tripleo-upgrade \
        --undercloud-update yes

Set up overcloud upgrade repositories::

    infrared tripleo-overcloud \
        --deployment-files virt \
        --upgrade yes \
        --mirror ${mirror_location} \
        --ansible-args="tags=upgrade_collect_info,upgrade_repos"

Set up overcloud update repositories/containers::

    infrared tripleo-overcloud \
        --deployment-files virt \
        --ocupdate True \
        --build latest \
        --ansible-args="tags=update_collect_info,update_undercloud_validation,update_repos,update_prepare_containers"

Upgrade overcloud::

    infrared tripleo-upgrade \
        --overcloud-upgrade yes

Update overcloud::

    infrared tripleo-upgrade \
        --overcloud-update yes

Advanced upgrade options
------------------------

Operator can now specify order of roles to upgrade by using *roles_upgrade_order* variable.

It's the **responsibility** of operator to specify *Controller* role first followed by all other roles.

*roles_upgrade_order* variable expects roles being separated by *;(semicolon)*, for e.g.:

::

    infrared tripleo-upgrade \
        --overcloud-upgrade yes \
        -e 'roles_upgrade_order=ControllerOpenstack;Database;Messaging'

will upgrade ControllerOpenstack group, then Database and finally Messaging.

Multiple roles could be upgraded in parallel, to achieve this they should be separated by *,(comma)*, for e.g:

::

    infrared tripleo-upgrade \
        --overcloud-upgrade yes \
        -e 'roles_upgrade_order=ControllerOpenstack;Database;Messaging'

will upgrade Controller and Database groups in parallel and then continue with Messaging.

Running the role manually from the undercloud
---------------------------------------------
This role can be run manually from the undercloud by doing the following steps:

Note: before starting the upgrade process make sure that both the undercloud
and overcloud nodes have the repositories with upgraded packages set up

Clone this repository
    git clone https://opendev.org/openstack/tripleo-upgrade

Set ansible roles path::
    ANSIBLE_ROLES_PATH=$(pwd)

Create inventory file::
    printf "[undercloud]\nlocalhost  ansible_connection=local" > hosts

Run the playbook including this role::
    ansible-playbook -i hosts tripleo-upgrade/tests/test.yml

=======
License
=======

Apache License 2.0

==================
Author Information
==================

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
