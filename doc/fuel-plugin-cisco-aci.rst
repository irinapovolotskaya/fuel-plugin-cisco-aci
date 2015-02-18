..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
Fuel Plugin Cisco ACI specification
===================================

The Cisco Application Policy Infrastructure Controller (Cisco APIC) is the unifying point of automation and management for the Application Centric Infrastructure (ACI) fabric. The Cisco APIC provides centralized access to all fabric information, optimizes the application lifecycle for scale and performance, and supports flexible application provisioning across physical and virtual resources.[1]
This specification describes automation of deployment Cisco ACI with OpenStack.

Problem description
===================

This integration should be supported with the upstream version of Fuel product.
Mirantis Openstack 6.0 release has Pluggable Architecture feature, that prevents developers from bringing any changes to the core product. Instead, the Cisco APIC functionality can be implemented as a plugin for Fuel.[2]

This plugin will let end user install the Mirantis OpenStack with Cisco SDN (software defined network) solution. This  new feature supports 3 types of installation:

* Generic APIC ML2 driver

* GBP module and Mapping driver

* GBP module and APIC driver

Each configuration will be described on its own section.

Proposed change
===============

Right now Fuel supports 4 types of network configurations:

* Neutron with VLAN segmentation (default)

* Neutron with GRE segmentation

* Neutron with VMware NSX

* Legacy Networking (nova-network)

When successfully copied to the Fuel Master node and installed, a new submenu will appear on the Settings tab of the Fuel web UI.
End user will have to select a radiobutton with Use Cases described below.

User Story 1: Generic APIC ML2 driver
---------------------------------------------------

This case will provide availability to configure Neutron for using Cisco SDN solution based on generic upstream ML2 neutron driver [3]. To enable this functionality, the plugin should  support 2 types of configuration:

* with automatic hosts discovery (using lldp)
* static config.

This list describes what software and configuration should be added to corresponding hosts to support User Story 1 with autodiscovery feature enabled(checkbox called “Use lldp” set):

* All hosts will be installed with LLDP package
* All hosts will be installed with pip apicapi package
* All hosts will be installed with neutron-driver-apic-agent package
* All hosts will have these configurations in *<neutron.conf>*:

  ::

    [DEFAULT]
    service_plugins=neutron.services.l3_router.l3_apic.ApicL3ServicePlugin
    core_plugin=neutron.plugins.ml2.plugin.Ml2Plugin

* All hosts will have these configurations in *ml2_conf.ini* file:

  ::

    [ml2]
    type_drivers=local,flat,vlan,gre,vxlan
    tenant_network_types=vlan
    mechanism_drivers=openvswitch,cisco_apic
    [ml2_type_vlan]
    network_vlan_ranges="$physnets_dev:$vlan_range"
    [securitygroup]
    enable_security_group=True
    firewall_driver=neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    [ovs]
    integration_bridge="$integration_bridge"
    bridge_mappings="$physnets_dev:$integration_bridge"
    enable_tunneling=False
    [agent]
    polling_interval=2
    l2_population=False
    arp_responder=False

Where **$integration_bridge** , **$physnets_dev**, **$vlan_range** should be configured through the Fuel web UI in Networks->Neutron L2 Configuration section.

* All hosts will have these configurations in *<ml2_conf_cisco.ini>*:

  ::

    [DEFAULT]
    apic_system_id=openstack
    [ml2_cisco_apic]
    apic_hosts=$apic_hosts
    apic_username=$apic_username
    apic_password=$apic_password
    apic_name_mapping=use_name

Where **$apic_hosts**, **$apic_username**, **$apic_password** - should be configured through Fuel web UI in Settings->Access section.

* All controllers will have neutron-driver-apic-svc-agent package installed
* All hosts *<ml2_config_cisco.ini>* will have [apic_external_network:ext] section, if configured though Fuel web UI.

This list describes what software and configuration should be added to corresponding hosts to support US1 with static config chosen:

* All controllers have pip apicapi installed
* neutron-driver-apic-svc-agent neutron-driver-apic-agent and lldp is not installed
* All configurations are the same as "Auto discovery" way
* On all hosts in *<ml2_config_cisco.ini>* file we will be added example(user-defined) section configured through Fuel web UI.

  ::

    [apic_switch:201]
    compute11,compute21=1/10
    compute12=1/11

* For both cases (autodiscovery and static) configs on controller nodes, *<neutron.conf>* should have admin credentials:

  ::

    [keystone_authtoken]
    admin_user="$admin_username"
    admin_password="$admin_password"
    admin_tenant_name="$admin_tenant"

Where **$admin_username**, **$admin_password** and **$admin_tenant** point to cloud administrator credentials.

User Story 2: GBP module and Mapping driver
-------------------------------------------------------------

This case will provide availability to configure Neutron for using Cisco SDN solution based on Cisco group based policy packages.
This list describes what software and configuration should be added to corresponding hosts to support User Story 2.

* All controllers will have these configurations in *<neutron.conf>*:

  ::

    [DEFAULT]
    service_plugins=neutron.services.
    l3_router.l3_router_plugin.L3RouterPlugin,
    gbpservice.neutron.services.grouppolicy.plugin.GroupPolicyPlugin,
    gbpservice.neutron.services.servicechain.servicechain_plugin.ServiceChainPlugin
    core_plugin=neutron.plugins.ml2.plugin.Ml2Plugin
    [group_policy]
    policy_drivers=implicit_policy,apic
    [servicechain]
    servicechain_drivers = simplechain_driver
    [quotas]
    default_quota = -1
    quota_network = -1
    quota_subnet = -1
    quota_port = -1
    quota_security_group = -1
    quota_security_group_rule = -1
    quota_router = -1
    quota_floatingip = -1

* All controllers will have these configurations in *<ml2_conf.ini>*:

  ::

    [ml2]
    type_drivers=local,flat,vlan,gre,vxlan
    tenant_network_types=vlan
    mechanism_drivers=openvswitch,apic_gbp
    [ml2_type_vlan]
    network_vlan_ranges="$physnets_dev:$vlan_range"
    [securitygroup]
    enable_security_group=True
    firewall_driver=neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    [ovs]
    integration_bridge="$integration_bridge"
    bridge_mappings="$physnets_dev:$integration_bridge"
    enable_tunneling=False
    [agent]
    polling_interval=2
    l2_population=False
    arp_responder=False

Where **$integration_bridge**, **$physnets_dev**, **$vlan_range** - should be configured through Fuel web UI in Networks->Neutron L2 Configuration section.

* All controllers will have these configurations in *<ml2_conf_cisco.ini>*:

  ::

    [DEFAULT]
    apic_system_id=openstack
    [ml2_cisco_apic]
    apic_hosts=$apic_hosts
    apic_username=$apic_username
    apic_password=$apic_password
    apic_name_mapping=use_name

Where **$apic_hosts**, **$apic_username**, **$apic_password** - should be configured through Fuel web UI  in Settings->Access section.

* All controllers will have 4 additional package installed:

  * group-based-policy
  * python-group-based-policy-client
  * group-based-policy-ui
  * group-based-policy-automation

* All controllers will enable heat plugin in *<heat.conf>*

  ::

    [DEFAULT]
    plugin_dirs=/path/to/code/gbpautomation/heat

* All controllers will enable Horizon projects by linking *project.py* file to enabled_dashboards directory.
* All hosts *<ml2_config_cisco.ini>* will have [apic_external_network:ext] section, if configured though Fuel web UI.

User Story 3: GBP module and APIC driver
---------------------------------------------------------

This case will provide availability to configure Neutron for using Cisco SDN solution based on Cisco group based policy packages and APIC Controller.
This list describes what software and configuration should be added to corresponding hosts to support User Story 3.

* All controllers will have 4 additional package installed:

  * group-based-policy
  * python-group-based-policy-client
  * group-based-policy-ui
  * group-based-policy-automation

* All controllers will have these configurations in *<neutron.conf>*:

  ::

    Paste config here

* All controllers will have these configurations in *<ml2_conf.ini>*:

  ::

    Paste config here

* All controllers will have these configurations in *<ml2_conf_cisco.ini>*:

  ::

    Paste config here

* All hosts *<ml2_config_cisco.ini>* will have [apic_external_network:ext] section, if configured though Fuel web UI.

* All controllers have pip apicapi installed

* Do we need lldp for this case ? if yes - install necessary packages on all hosts.


Alternatives
---------------

There are no known alternatives for this plugin, although all steps can be performed manually.

Data model impact
-------------------------

GBP installation type requires additional tables in Neutron database. New scheme will be managed by `gbp-db-manage` command that comes from group-based-policy package.

REST API impact
---------------

None.

Upgrade impact
--------------

Upgrading should be tested explicitly with this plugin installed and APIC controller enabled.

Security impact
---------------

This plugin changes Neutron keystone_authtoken credentials from `neutron` user and `services` tenant to `admin` user and `admin` tenant on controller nodes. This may change in future, but for Juno this must be set to admin values.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Plugin impact
-------------

This plugin should not impact other plugins until they don't modify the same settings for Neutron configuration.

Other deployer impact
---------------------

Developer impact
----------------


Implementation
==============

Assignee(s)
-----------
Primary assignee:
 <nkoshikov@mirantis.com>

Work Items
----------

* Create fuel-plugin-cisco-aci plugin

* Develop Fuel web UI part

* Add puppet support for all configuration cases

* Write documentation (User Guide)

Dependencies
============

* Ubuntu 14.04 support in MOS [4]

* This bug should also be fixed [5]

Testing
========

Plugin should pass tempest framework tests.

Documentation Impact
====================

Reference to this plugin should be added to main Fuel documentation.

References
==========

[1] http://cisco.com/go/apic
[2] http://docs.mirantis.com/openstack/fuel/fuel-6.0/plugin-dev.html
[3] https://blueprints.launchpad.net/neutron/+spec/ml2-cisco-apic-mechanism-driver
[4] https://blueprints.launchpad.net/fuel/+spec/support-ubuntu-trusty
[5] https://bugs.launchpad.net/fuel/+bug/1417994

