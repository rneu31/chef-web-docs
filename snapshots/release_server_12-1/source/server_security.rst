

.. tag server_security_11

=====================================================
Security
=====================================================

.. tag server_security_ssl

Configuration of SSL for the Chef server using certificate authority-verified certificates is done by placing the certificate and private key file obtained from the certifying authority in the correct files after the initial configuration of Chef server.

Initial configuration of the Chef server is done automatically using a self-signed certificate to create the certificate and private key files for Nginx.

The locations of the certificate and private key files are

* ``/var/opt/opscode/nginx/ca/FQDN.crt``
* ``/var/opt/opscode/nginx/ca/FQDN.key``

Because the FQDN has already been configured, do the following:

#. Replace the contents of ``/var/opt/opscode/nginx/ca/FQDN.crt`` and ``/var/opt/opscode/nginx/ca/FQDN.key`` with the certifying authority's files.
#. Reconfigure the Chef server:

   .. code-block:: bash

      $ chef-server-ctl reconfigure

#. Restart the Nginx service to load the new key and certificate:

   .. code-block:: bash

      $ chef-server-ctl restart nginx

.. end_tag

.. warning:: The FQDN for the Chef server should not exceed 64 characters when using OpenSSL. OpenSSL requires the ``CN`` in a certificate to be no longer than 64 characters.

.. warning:: By default, the Chef server uses the FQDN to determine the common name (``CN``). If the FQDN of the Chef server is longer than 64 characters, the ``reconfigure`` command will not fail, but an empty certificate file will be created. Nginx will not start if a certificate file is empty.

SSL Certificates
=====================================================
.. tag server_security_ssl_cert_custom

The Chef server can be configured to use SSL certificates by adding the following settings to the server configuration file:

.. list-table::
   :widths: 200 300
   :header-rows: 1

   * - Setting
     - Description
   * - ``nginx['ssl_certificate']``
     - The SSL certificate used to verify communication over HTTPS.
   * - ``nginx['ssl_certificate_key']``
     - The certificate key used for SSL communication.

and then setting their values to define the paths to the certificate and key.

.. end_tag

For example:

.. code-block:: ruby

   nginx['ssl_certificate']  = "/etc/pki/tls/certs/your-host.crt"
   nginx['ssl_certificate_key']  = "/etc/pki/tls/private/your-host.key"

Save the file, and then run the following command:

.. code-block:: bash

   $ sudo chef-server-ctl reconfigure

For more information about the server configuration file, see :doc:`chef-server.rb </config_rb_server>`.

SSL Protocols
-----------------------------------------------------
.. tag server_tuning_nginx

The following settings are often modified from the default as part of the tuning effort for the **nginx** service and to configure the Chef server to use SSL certificates:

``nginx['ssl_certificate']``
   The SSL certificate used to verify communication over HTTPS. Default value: ``nil``.

``nginx['ssl_certificate_key']``
   The certificate key used for SSL communication. Default value: ``nil``.

``nginx['ssl_ciphers']``
   The list of supported cipher suites that are used to establish a secure connection. To favor AES256 with ECDHE forward security, drop the ``RC4-SHA:RC4-MD5:RC4:RSA`` prefix. For example:

   .. code-block:: ruby

      nginx['ssl_ciphers'] =  "HIGH:MEDIUM:!LOW:!kEDH: \
                               !aNULL:!ADH:!eNULL:!EXP: \
                               !SSLv2:!SEED:!CAMELLIA: \
                               !PSK"

``nginx['ssl_protocols']``
   The SSL protocol versions that are enabled. SSL 3.0 is supported by the Chef server; however, SSL 3.0 is an obsolete and insecure protocol. Transport Layer Security (TLS)---TLS 1.0, TLS 1.1, and TLS 1.2---has effectively replaced SSL 3.0, which provides for authenticated version negotiation between the chef-client and Chef server, which ensures the latest version of the TLS protocol is used. For the highest possible security, it is recommended to disable SSL 3.0 and allow all versions of the TLS protocol.  For example:

   .. code-block:: ruby

      nginx['ssl_protocols'] = "TLSv1 TLSv1.1 TLSv1.2"

.. note:: See https://wiki.mozilla.org/Security/Server_Side_TLS for more information about the values used with the ``nginx['ssl_ciphers']`` and ``nginx['ssl_protocols']`` settings.

For example, after copying the SSL certificate files to the Chef server, update the ``nginx['ssl_certificate']`` and ``nginx['ssl_certificate_key']`` settings to specify the paths to those files, and then (optionally) update the ``nginx['ssl_ciphers']`` and ``nginx['ssl_protocols']`` settings to reflect the desired level of hardness for the Chef server:

.. code-block:: ruby

   nginx['ssl_certificate'] = "/etc/pki/tls/private/name.of.pem"
   nginx['ssl_certificate_key'] = "/etc/pki/tls/private/name.of.key"
   nginx['ssl_ciphers'] = "HIGH:MEDIUM:!LOW:!kEDH:!aNULL:!ADH:!eNULL:!EXP:!SSLv2:!SEED:!CAMELLIA:!PSK"
   nginx['ssl_protocols'] = "TLSv1 TLSv1.1 TLSv1.2"

.. end_tag

**Example: Configure SSL Keys for Nginx**

The following example shows how the Chef server sets up and configures SSL certificates for Nginx. The cipher suite used by Nginx :ref:`is configurable <config_rb_server-ssl-protocols>` using the ``ssl_protocols`` and ``ssl_ciphers`` settings.

.. code-block:: ruby

   ssl_keyfile = File.join(nginx_ca_dir, "#{node['private_chef']['nginx']['server_name']}.key")
   ssl_crtfile = File.join(nginx_ca_dir, "#{node['private_chef']['nginx']['server_name']}.crt")
   ssl_signing_conf = File.join(nginx_ca_dir, "#{node['private_chef']['nginx']['server_name']}-ssl.conf")

   unless File.exists?(ssl_keyfile) && File.exists?(ssl_crtfile) && File.exists?(ssl_signing_conf)
     file ssl_keyfile do
       owner 'root'
       group 'root'
       mode '0755'
       content '/opt/opscode/embedded/bin/openssl genrsa 2048'
       not_if { File.exist?(ssl_keyfile) }
     end

     file ssl_signing_conf do
       owner 'root'
       group 'root'
       mode '0755'
       not_if { File.exist?(ssl_signing_conf) }
       content <<-EOH
     [ req ]
     distinguished_name = req_distinguished_name
     prompt = no
     [ req_distinguished_name ]
     C                      = #{node['private_chef']['nginx']['ssl_country_name']}
     ST                     = #{node['private_chef']['nginx']['ssl_state_name']}
     L                      = #{node['private_chef']['nginx']['ssl_locality_name']}
     O                      = #{node['private_chef']['nginx']['ssl_company_name']}
     OU                     = #{node['private_chef']['nginx']['ssl_organizational_unit_name']}
     CN                     = #{node['private_chef']['nginx']['server_name']}
     emailAddress           = #{node['private_chef']['nginx']['ssl_email_address']}
     EOH
     end

     ruby_block 'create crtfile' do
       block do
         r = Chef::Resource::File.new(ssl_crtfile, run_context)
         r.owner 'root'
         r.group 'root'
         r.mode '0755'
         r.content "/opt/opscode/embedded/bin/openssl req -config '#{ssl_signing_conf}' -new -x509 -nodes -sha1 -days 3650 -key '#{ssl_keyfile}'"
         r.not_if { File.exist?(ssl_crtfile) }
         r.run_action(:create)
       end
     end
   end

Chef Analytics
-----------------------------------------------------
.. tag server_security_ssl_cert_custom_analytics

The Chef Analytics server can be configured to use SSL certificates by adding the following settings in the server configuration file:

.. list-table::
   :widths: 200 300
   :header-rows: 1

   * - Setting
     - Description
   * - ``ssl['certificate']``
     - The SSL certificate used to verify communication over HTTPS.
   * - ``ssl['certificate_key']``
     - The certificate key used for SSL communication.

and then setting their values to define the paths to the certificate and key.

For example:

.. code-block:: ruby

   ssl['certificate']  = "/etc/pki/tls/certs/your-host.crt"
   ssl['certificate_key']  = "/etc/pki/tls/private/your-host.key"

Save the file, and then run the following command:

.. code-block:: bash

   $ sudo opscode-analytics-ctl reconfigure

.. end_tag

Knife, chef-client
-----------------------------------------------------
.. tag server_security_ssl_cert_client

Chef server 12 enables SSL verification by default for all requests made to the server, such as those made by knife and the chef-client. The certificate that is generated during the installation of the Chef server is self-signed, which means the certificate is not signed by a trusted certificate authority (CA) that ships with the chef-client. The certificate generated by the Chef server must be downloaded to any machine from which knife and/or the chef-client will make requests to the Chef server.

For example, without downloading the SSL certificate, the following knife command:

.. code-block:: bash

   $ knife client list

responds with an error similar to:

.. code-block:: bash

   ERROR: SSL Validation failure connecting to host: chef-server.example.com ...
   ERROR: OpenSSL::SSL::SSLError: SSL_connect returned=1 errno=0 state=SSLv3 ...

This is by design and will occur until a verifiable certificate is added to the machine from which the request is sent.

.. end_tag

See SSL Certificates for more information about how knife and the chef-client use SSL certificates generated by the Chef server.

Private Certificate Authority
-----------------------------------------------------
If an organization is using an internal certificate authority, then the root certificate will not appear in any ``cacerts.pem`` file that ships by default with operating systems and web browsers. Because of this, no currently deployed system will be able to verify certificates that are issued in this manner. To allow other systems to trust certificates from an internal certificate authority, this root certificate will need to be configured so that other systems can follow the chain of authority back to the root certificate. (An intermediate certificate is not enough becuase the root certificate is not already globally known.)

To use an internal certificate authority, append both the server and root certificates into a single ``.crt`` file. For example:

.. code-block:: bash

   $ cat server.crt root.crt >> /var/opt/opscode/nginx/ca/FQDN.crt

Intermediate Certificates
-----------------------------------------------------
To use an intermediate certificate, append both the server and intermediate certificates into a single ``.crt`` file. For example:

.. code-block:: bash

   $ cat server.crt intermediate.crt >> /var/opt/opscode/nginx/ca/FQDN.crt

Regenerate Certificates
-----------------------------------------------------
.. tag server_security_ssl_cert_regenerate

SSL certificates should be regenerated periodically. This is an important part of protecting the Chef server from vulnerabilities and helps to prevent the information stored on the Chef server from being compromised.

To regenerate SSL certificates:

#. Run the following command:

   .. code-block:: bash

      $ chef-server-ctl stop

#. The Chef server can regenerate them. These certificates will be located in ``/var/opt/opscode/nginx/ca/`` and will be named after the FQDN for the Chef server. To determine the FQDN for the server, run the following command:

   .. code-block:: bash

      $ hostname -f

   Please delete the files found in the ca directory with names like this ``$FQDN.crt`` and ``$FQDN.key``.

#. If your organization has provided custom SSL certificates to the Chef server, the locations of that custom certificate and private key are defined in ``/etc/opscode/chef-server.rb`` as values for the ``nginx['ssl_certificate']`` and ``nginx['ssl_certificate_key']`` settings. Delete the files referenced in those two settings and regenerate new keys using the same authority.

#. Run the following command, Chef server-generated SSL certificates will automatically be created if necessary:

   .. code-block:: bash

      $ chef-server-ctl reconfigure

#. Run the following command:

   .. code-block:: bash

      $ chef-server-ctl start

.. end_tag

Key Rotation
=====================================================
Use the following commands to manage public and private key rotation for users and clients.

add-client-key
-----------------------------------------------------
.. tag ctl_chef_server_add_client_key

Use the ``add-client-key`` subcommand to add a client key.

.. end_tag

**Syntax**

.. tag ctl_chef_server_add_client_key_syntax

This subcommand has the following syntax:

.. code-block:: bash

   $ chef-server-ctl add-client-key ORG_NAME CLIENT_NAME [--public-key-path PATH] [--expiration-date DATE] [--key-name NAME]

.. warning:: All options for this subcommand must follow all arguments.

.. end_tag

**Options**

.. tag ctl_chef_server_add_client_key_options

This subcommand has the following options:

``CLIENT_NAME``
   The name of the client that you wish to add a key for.

``-e DATE`` ``--expiration-date DATE``
   An ISO 8601 formatted string: ``YYYY-MM-DDTHH:MM:SSZ``. For example: ``2013-12-24T21:00:00Z``. If not passed, expiration will default to infinity.

``-k NAME`` ``--key-name NAME``
   String defining the name of your new key for this client. If not passed, it will default to the fingerprint of the public key.

``ORG_NAME``
   The short name for the organization to which the client belongs.

``-p PATH`` ``--public-key-path PATH``
   The location to a file containing valid PKCS#1 public key to be added. If not passed, then the server will generate a new one for you and return the private key to STDOUT.

.. end_tag

add-user-key
-----------------------------------------------------
.. tag ctl_chef_server_add_user_key

Use the ``add-user-key`` subcommand to add a user key.

.. end_tag

**Syntax**

.. tag ctl_chef_server_add_user_key_syntax

This subcommand has the following syntax:

.. code-block:: bash

   $ chef-server-ctl add-user-key USER_NAME [--public-key-path PATH] [--expiration-date DATE] [--key-name NAME]

.. warning:: All options for this subcommand must follow all arguments.

.. end_tag

**Options**

.. tag ctl_chef_server_add_user_key_options

This subcommand has the following options:

``-e DATE`` ``--expiration-date DATE``
   An ISO 8601 formatted string: ``YYYY-MM-DDTHH:MM:SSZ``. For example: ``2013-12-24T21:00:00Z``. If not passed, expiration will default to infinity.

``-k NAME`` ``--key-name NAME``
   String defining the name of your new key for this user. If not passed, it will default to the fingerprint of the public key.

``-p PATH`` ``--public-key-path PATH``
   The location to a file containing valid PKCS#1 public key to be added. If not passed, then the server will generate a new one for you and return the private key to STDOUT.

``USER_NAME``
   The user name for the user for which a key is added.

.. end_tag

delete-client-key
-----------------------------------------------------
.. tag ctl_chef_server_delete_client_key

Use the ``delete-client-key`` subcommand to delete a client key.

.. end_tag

**Syntax**

.. tag ctl_chef_server_delete_client_key_syntax

This subcommand has the following syntax:

.. code-block:: bash

   $ chef-server-ctl delete-client-key ORG_NAME CLIENT_NAME KEY_NAME

.. end_tag

**Options**

.. tag ctl_chef_server_delete_client_key_options

This subcommand has the following arguments:

``ORG_NAME``
   The short name for the organization to which the client belongs.

``CLIENT_NAME``
   The name of the client.

``KEY_NAME``
   The unique name to be assigned to the key you wish to delete.

.. end_tag

delete-user-key
-----------------------------------------------------
.. tag ctl_chef_server_delete_user_key

Use the ``delete-user-key`` subcommand to delete a user key.

.. end_tag

**Syntax**

.. tag ctl_chef_server_delete_user_key_syntax

This subcommand has the following syntax:

.. code-block:: bash

   $ chef-server-ctl delete-user-key USER_NAME KEY_NAME

.. warning:: The parameters for this subcommand must be in the order specified above.

.. end_tag

**Options**

.. tag ctl_chef_server_delete_user_key_options

This subcommand has the following arguments:

``USER_NAME``
   The user name.

``KEY_NAME``
   The unique name to be assigned to the key you wish to delete.

.. end_tag

list-client-key
-----------------------------------------------------
.. tag ctl_chef_server_list_client_keys

Use the ``list-client-keys`` subcommand to list client keys.

.. end_tag

**Syntax**

.. tag ctl_chef_server_list_client_keys_syntax

This subcommand has the following syntax:

.. code-block:: bash

   $ chef-server-ctl list-client-keys ORG_NAME CLIENT_NAME [--verbose]

.. warning::  All options for this subcommand must follow all arguments.

.. end_tag

**Options**

.. tag ctl_chef_server_list_client_keys_options

This subcommand has the following options:

``CLIENT_NAME``
   The name of the client.

``ORG_NAME``
   The short name for the organization to which the client belongs.

``--verbose``
   Use to show the full public key strings in command output.

.. end_tag

list-user-key
-----------------------------------------------------
.. tag ctl_chef_server_list_user_keys

Use the ``list-user-keys`` subcommand to list client keys.

.. end_tag

**Syntax**

.. tag ctl_chef_server_list_user_keys_syntax

This subcommand has the following syntax:

.. code-block:: bash

   $ chef-server-ctl list-user-keys USER_NAME [--verbose]

.. warning:: All options for this subcommand must follow all arguments.

.. end_tag

**Options**

.. tag ctl_chef_server_list_user_keys_options

This subcommand has the following options:

``USER_NAME``
   The user name you wish to list keys for.

``--verbose``
   Use to show the full public key strings in command output.

.. end_tag

**Example**

.. tag ctl_chef_server_list_user_keys_summary

To view a list of user keys (including public key output):

.. code-block:: bash

   $ chef-server-ctl list-user-keys applejack --verbose

Returns:

.. code-block:: bash

   2 total key(s) found for user applejack

   key_name: test-key
   expires_at: Infinity
   public_key:
   -----BEGIN PUBLIC KEY-----
   MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA4q9Dh+bwJSjhU/VI4Y8s
   9WsbIPfpmBpoZoZVPL7V6JDfIaPUkdcSdZpynhRLhQwv9ScTFh65JwxC7wNhVspB
   4bKZeW6vugNGwCyBIemMfxMlpKZQDOc5dnBiRMMOgXSIimeiFtL+NmMXnGBBHDaE
   b+XXI8oCZRx5MTnzEs90mkaCRSIUlWxOUFzZvnv4jBrhWsd/yBM/h7YmVfmwVAjL
   VST0QG4MnbCjNtbzToMj55NAGwSdKHCzvvpWYkd62ZOquY9f2UZKxYCX0bFPNVQM
   EvBQGdNG39XYSEeF4LneYQKPHEZDdqe7TZdVE8ooU/syxlZgADtvkqEoc4zp1Im3
   2wIDAQAB
   -----END PUBLIC KEY-----

   key_name: default
   expires_at: Infinity
   public_key:
   -----BEGIN PUBLIC KEY-----
   MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA4q9Dh+bwJSjhU/VI4Y8s
   9WsbIPfpmBpoZoZVPL7V6JDfIaPUkdcSdZpynhRLhQwv9ScTFh65JwxC7wNhVspB
   4bKZeW6vugNGwCyBIemMfxMlpKZQDOc5dnBiRMMOgXSIimeiFtL+NmMXnGBBHDaE
   b+XXI8oCZRx5MTnzEs90mkaCRSIUlWxOUFzZvnv4jBrhWsd/yBM/h7YmVfmwVAjL
   VST0QG4MnbCjNtbzToMj55NAGwSdKHCzvvpWYkd62ZOquY9f2UZKxYCX0bFPNVQM
   EvBQGdNG39XYSEeF4LneYQKPHEZDdqe7TZdVE8ooU/syxlZgADtvkqEoc4zp1Im3
   2wIDAQAB
   -----END PUBLIC KEY-----

.. end_tag

chef-client Settings
=====================================================
.. tag chef_client_ssl_config_settings

Use following client.rb settings to manage SSL certificate preferences:

.. list-table::
   :widths: 200 300
   :header-rows: 1

   * - Setting
     - Description
   * - ``local_key_generation``
     - Whether the Chef server or chef-client generates the private/public key pair. When ``true``, the chef-client generates the key pair, and then sends the public key to the Chef server. Default value: ``true``.
   * - ``ssl_ca_file``
     - The file in which the OpenSSL key is saved. This setting is generated automatically by the chef-client and most users do not need to modify it.
   * - ``ssl_ca_path``
     - The path to where the OpenSSL key is located. This setting is generated automatically by the chef-client and most users do not need to modify it.
   * - ``ssl_client_cert``
     - The OpenSSL X.509 certificate used for mutual certificate validation. This setting is only necessary when mutual certificate validation is configured on the Chef server. Default value: ``nil``.
   * - ``ssl_client_key``
     - The OpenSSL X.509 key used for mutual certificate validation. This setting is only necessary when mutual certificate validation is configured on the Chef server. Default value: ``nil``.
   * - ``ssl_verify_mode``
     - Set the verify mode for HTTPS requests.

       * Use ``:verify_none`` to do no validation of SSL certificates.
       * Use ``:verify_peer`` to do validation of all SSL certificates, including the Chef server connections, S3 connections, and any HTTPS **remote_file** resource URLs used in the chef-client run. This is the recommended setting.

       Depending on how OpenSSL is configured, the ``ssl_ca_path`` may need to be specified. Default value: ``:verify_peer``.
   * - ``verify_api_cert``
     - Verify the SSL certificate on the Chef server. When ``true``, the chef-client always verifies the SSL certificate. When ``false``, the chef-client uses the value of ``ssl_verify_mode`` to determine if the SSL certificate requires verification. Default value: ``false``.

.. end_tag

.. end_tag

