

=====================================================
chef_gem
=====================================================

.. warning:: ../../includes_notes/includes_notes_chef_gem_vs_gem_package.rst

.. tag resource_package_chef_gem

Use the **chef_gem** resource to install a gem only for the instance of Ruby that is dedicated to the chef-client. When a package is installed from a local file, it must be added to the node using the **remote_file** or **cookbook_file** resources.

The **chef_gem** resource works with all of the same properties and options as the **gem_package** resource, but does not accept the ``gem_binary`` property because it always uses the ``CurrentGemEnvironment`` under which the chef-client is running. In addition to performing actions similar to the **gem_package** resource, the **chef_gem** resource does the following:

* Runs its actions immediately, before convergence, allowing a gem to be used in a recipe immediately after it is installed
* Runs ``Gem.clear_paths`` after the action, ensuring that gem is aware of changes so that it can be required immediately after it is installed

.. end_tag

.. note:: ../../includes_notes/includes_notes_resource_based_on_package.rst

Syntax
=====================================================
A **chef_gem** resource block manages a package on a node, typically by installing it. The simplest use of the **chef_gem** resource is:

.. code-block:: ruby

   chef_gem 'package_name'

which will install the named package using all of the default options and the default action (``:install``).

The full syntax for all of the properties that are available to the **chef_gem** resource is:

.. code-block:: ruby

   chef_gem 'name' do
     clear_sources              TrueClass, FalseClass
     notifies                   # see description
     options                    String
     package_name               String, Array # defaults to 'name' if not specified
     provider                   Chef::Provider::Package::Rubygems
     source                     String
     subscribes                 # see description
     timeout                    String, Integer
     version                    String, Array
     action                     Symbol # defaults to :install if not specified
   end

where

* ``chef_gem`` tells the chef-client to manage a package
* ``'name'`` is the name of the package
* ``:action`` identifies which steps the chef-client will take to bring the node into the desired state
* ``clear_sources``, ``options``, ``package_name``, ``provider``, ``source``, ``timeout``, and ``version`` are properties of this resource, with the Ruby type shown. See "Properties" section below for more information about all of the properties that may be used with this resource.

Actions
=====================================================
.. tag resource_package_chef_gem_actions

This resource has the following actions:

``:install``
   Default. Install a package. If a version is specified, install the specified version of the package.

``:nothing``
   .. tag resources_common_actions_nothing

   Define this resource block to do nothing until notified by another resource to take action. When this resource is notified, this resource block is either run immediately or it is queued up to be run at the end of the chef-client run.

   .. end_tag

``:purge``
   Purge a package. This action typically removes the configuration files as well as the package.

``:reconfig``
   Reconfigure a package. This action requires a response file.

``:remove``
   Remove a package.

``:upgrade``
   Install a package and/or ensure that a package is the latest version.

.. end_tag

Properties
=====================================================
This resource has the following properties:

``compile_time``
   **Ruby Types:** TrueClass, FalseClass

   Controls the phase during which a gem is installed on a node. Set to ``true`` to install a gem while the resource collection is being built (the "compile phase"). Set to ``false`` to install a gem while the chef-client is configuring the node (the "converge phase"). Possible values: ``nil`` (for verbose warnings), ``true`` (to warn once per chef-client run), or ``false`` (to remove all warnings). Recommended value: ``false``.

   .. tag resource_package_chef_gem_attribute_compile_time

   .. This topic is hooked into client.rb topics, starting with 12.1, in addition to the resource reference pages.

   To suppress warnings for cookbooks authored prior to chef-client 12.1, use a ``respond_to?`` check to ensure backward compatibility. For example:

   .. code-block:: ruby

      chef_gem 'aws-sdk' do
        compile_time false if respond_to?(:compile_time)
      end

   .. end_tag

   .. warning:: .. tag notes_chef_gem_chef_sugar

                If you are using ``chef-sugar``---a `community cookbook <https://supermarket.chef.io/cookbooks/chef-sugar>`__---it must be version 3.0.1 (or higher) to use the previous example. If you are using an older version of ``chef-sugar``, the following workaround is required:

                .. code-block:: ruby

                   chef_gem 'gem_name' do
                     compile_time true if Chef::Resource::ChefGem.instance_methods(false).include?(:compile_time)
                   end

                See this `blog post <http://jtimberman.housepub.org/blog/2015/03/20/chef-gem-compile-time-compatibility/>`__ for more background on this behavior.

                .. end_tag

``ignore_failure``
   **Ruby Types:** TrueClass, FalseClass

   Continue running a recipe if a resource fails for any reason. Default value: ``false``.

``notifies``
   **Ruby Type:** Symbol, 'Chef::Resource[String]'

   .. tag resources_common_notification_notifies

   A resource may notify another resource to take action when its state changes. Specify a ``'resource[name]'``, the ``:action`` that resource should take, and then the ``:timer`` for that action. A resource may notifiy more than one resource; use a ``notifies`` statement for each resource to be notified.

   .. end_tag

   .. tag 5_3

   A timer specifies the point during the chef-client run at which a notification is run. The following timers are available:

   ``:delayed``
      Default. Specifies that a notification should be queued up, and then executed at the very end of the chef-client run.

   ``:immediate``, ``:immediately``
      Specifies that a notification should be run immediately, per resource notified.

   .. end_tag

   .. tag resources_common_notification_notifies_syntax

   The syntax for ``notifies`` is:

   .. code-block:: ruby

      notifies :action, 'resource[name]', :timer

   .. end_tag

``options``
   **Ruby Type:** String

   One (or more) additional options that are passed to the command.

``package_name``
   **Ruby Types:** String, Array

   The name of the package. Default value: the ``name`` of the resource block See "Syntax" section above for more information.

``provider``
   **Ruby Type:** Chef Class

   Optional. Explicitly specifies a provider. See "Providers" section below for more information.

``retries``
   **Ruby Type:** Integer

   The number of times to catch exceptions and retry the resource. Default value: ``0``.

``retry_delay``
   **Ruby Type:** Integer

   The retry delay (in seconds). Default value: ``2``.

``source``
   **Ruby Type:** String

   Optional. The path to a package in the local file system.

``subscribes``
   **Ruby Type:** Symbol, 'Chef::Resource[String]'

   .. tag resources_common_notification_subscribes

   A resource may listen to another resource, and then take action if the state of the resource being listened to changes. Specify a ``'resource[name]'``, the ``:action`` to be taken, and then the ``:timer`` for that action.

   .. end_tag

   .. tag 5_3

   A timer specifies the point during the chef-client run at which a notification is run. The following timers are available:

   ``:delayed``
      Default. Specifies that a notification should be queued up, and then executed at the very end of the chef-client run.

   ``:immediate``, ``:immediately``
      Specifies that a notification should be run immediately, per resource notified.

   .. end_tag

   .. tag resources_common_notification_subscribes_syntax

   The syntax for ``subscribes`` is:

   .. code-block:: ruby

      subscribes :action, 'resource[name]', :timer

   .. end_tag

``timeout``
   **Ruby Types:** String, Integer

   The amount of time (in seconds) to wait before timing out.

``version``
   **Ruby Types:** String, Array

   The version of a package to be installed or upgraded.

Providers
=====================================================
.. tag resources_common_provider

Where a resource represents a piece of the system (and its desired state), a provider defines the steps that are needed to bring that piece of the system from its current state into the desired state.

.. end_tag

.. tag resources_common_provider_attributes

The chef-client will determine the correct provider based on configuration data collected by Ohai at the start of the chef-client run. This configuration data is then mapped to a platform and an associated list of providers.

Generally, it's best to let the chef-client choose the provider, and this is (by far) the most common approach. However, in some cases, specifying a provider may be desirable. There are two approaches:

* Use a more specific short name---``yum_package "foo" do`` instead of ``package "foo" do``, ``script "foo" do`` instead of ``bash "foo" do``, and so on---when available
* Use the ``provider`` property within the resource block to specify the long name of the provider as a property of a resource. For example: ``provider Chef::Provider::Long::Name``

.. end_tag

.. tag resource_package_chef_gem_providers

This resource has the following providers:

``Chef::Provider::Package``, ``package``
   When this short name is used, the chef-client will attempt to determine the correct provider during the chef-client run.

``Chef::Provider::Package::Rubygems``, ``chef_gem``
   Can be used with the ``options`` attribute.

.. end_tag

Examples
=====================================================
The following examples demonstrate various approaches for using resources in recipes. If you want to see examples of how Chef uses resources in recipes, take a closer look at the cookbooks that Chef authors and maintains: https://github.com/chef-cookbooks.

**Compile time vs. converge time installation of gems**

.. tag resource_chef_gem_install_for_use_in_recipes

.. To install a gems file for use in a recipe:

To install a gem while the chef-client is configuring the node (the “converge phase”), set the ``compile_time`` property to ``false``:

.. code-block:: ruby

   chef_gem 'right_aws' do
     compile_time false
     action :install
   end

To install a gem while the resource collection is being built (the “compile phase”), set the ``compile_time`` property to ``true``:

.. code-block:: ruby

   chef_gem 'right_aws' do
     compile_time true
     action :install
   end

.. end_tag

**Install MySQL for Chef**

.. tag resource_chef_gem_install_mysql

.. To install MySQL:

.. code-block:: ruby

   execute 'apt-get update' do
     ignore_failure true
     action :nothing
   end.run_action(:run) if node['platform_family'] == 'debian'

   node.set['build_essential']['compiletime'] = true
   include_recipe 'build-essential'
   include_recipe 'mysql::client'

   node['mysql']['client']['packages'].each do |mysql_pack|
     resources("package[#{mysql_pack}]").run_action(:install)
   end

   chef_gem 'mysql'

.. end_tag

