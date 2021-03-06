Server security and firewalls
=============================

General
-------

OMERO has been built with security in mind. Various standard security
practices have been adhered to during the development of the server and
client including:

-  Encryption of all passwords between client and server via |SSL|
-  Full encryption of all data when requested via |SSL|
-  User and group based access control
-  Authentication via LDAP
-  Limited visible TCP ports to ease firewalling
-  Use of a higher level language (Java or Python) to limit buffer
   overflows and other security issues associated with native code
-  Escaping and bind variable use in all SQL interactions performed via
   Hibernate

.. note:: The OMERO team treats the security of all components with care and
    attention. If you have a security issue to report, please do not hesitate
    to contact us using our private, secure mailing list as described on the
    :security:`Security <>` page.

Firewall configuration
----------------------

.. _IANA: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xml

Securing your OMERO system with so called *firewalling* or *packet filtering*
can be done quite easily. By default, OMERO clients only need to connect to
two TCP ports for communication with your OMERO.server: 4063 (unsecured) and
4064 (|SSL|). These are the IANA_ assigned ports for the Glacier2 router from
ZeroC_. Both of these values, however, are completely up to you, see
:ref:`security_ssl` below.

Important OMERO ports:

-  **TCP/4063**
-  **TCP/4064**

If you are using OMERO.web, then you will also need to
make your HTTP and HTTPS ports available. These are usually 80 and 443.

Important OMERO.web ports:

-  **TCP/80**
-  **TCP/443**

Example OpenBSD firewall rules
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    block in log on $ext_if from any to <omero_server_ip>
    pass in on $ext_if proto tcp from any to <omero_server_ip> port 4063
    pass in on $ext_if proto tcp from any to <omero_server_ip> port 4064
    pass in on $ext_if proto tcp from any to <omero_server_ip> port 443
    pass in on $ext_if proto tcp from any to <omero_server_ip> port 80

Example Linux firewall rules
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    iptables -P INPUT drop
    iptables -A INPUT -p tcp --dport 4063 -j ACCEPT
    iptables -A INPUT -p tcp --dport 4064 -j ACCEPT
    iptables -A INPUT -p tcp --dport 443 -j ACCEPT
    iptables -A INPUT -p tcp --dport 80 -j ACCEPT
    ...

--------------

Passwords
---------

The passwords stored in the ``password`` table are salted and hashed, so it is
impossible to recover a lost one, instead a new one must be set by an admin.

If the password for the root user is lost, the only way to reset it (in the
absence of other admin accounts) is to manually update the password table. The
``bin/omero`` command can generate the required SQL statement for you::

    $ bin/omero db password
    Please enter password for OMERO root user:
    Please re-enter password for OMERO root user:
    UPDATE password SET hash = 'PJueOtwuTPHB8Nq/1rFVxg==' WHERE experimenter_id  = 0;

Current hashed password::

    $ psql mydatabase -c " select * from password"
     experimenter_id |           hash           
    -----------------+--------------------------
                   0 | Xr4ilOzQ4PCOq3aQ0qbuaQ==
    (1 row)

Change the password using the generated SQL statement::

    $ psql mydatabase -c "UPDATE password SET hash = 'PJueOtwuTPHB8Nq/1rFVxg==' WHERE experimenter_id  = 0;"
    UPDATE 1


Stored data
-----------

The server's binary repository and database contain information that may
be confidential. Afford access only on a limited and necessary basis.
For example, the :ref:`ReadSession warning <ReadSession-warning>` is for
naught if the restricted administrator can read the contents of the
``session`` table.


Java key- and truststores.
---------------------------

If your server is connecting to another server over |SSL|, you may need
to configure a truststore and/or a keystore for the Java process. This
happens, for example, when your LDAP server uses |SSL|. See the :doc:`LDAP
plugin <server-ldap>` for information on how to configure the LDAP
URLs. As with all configuration properties, you will need to restart
your server after changing them.

To do this, you will need to configure several server properties,
similar to the properties you configured during
:doc:`installation <unix/server-installation>`.

-  truststore path

   ::

       bin/omero config set omero.security.trustStore /home/user/.keystore

       A truststore is a database of trusted entities and their
       associated X.509 certificate chains authenticating the
       corresponding public keys. The truststore contains the
       Certificate Authority (CA) certificates and the certificate(s) of
       the other party to which this entity intends to send encrypted
       (confidential) data. This file must contain the public key
       certificates of the CA and the client's public key certificate.


   If you don't have one you can create it using the following:

   ::
       
       openssl s_client -connect {{host}}:{{port}} -prexit < /dev/null | openssl x509 -outform PEM | keytool -import  -alias ldap -storepass {{password}} -keystore {{truststore}} -noprompt

-  truststore password

   ::

       bin/omero config set omero.security.trustStorePassword secret

-  keystore path

   ::

       bin/omero config set omero.security.keyStore /home/user/.mystore

       A keystore is a database of private keys and their associated
       X.509 certificate chains authenticating the corresponding public
       keys.
       
       A keystore is mostly needed if you are doing client-side certificates 
       for authentication against your LDAP server.

-  keystore password

   ::

       bin/omero config set omero.security.keyStorePassword secret

.. _security_ssl:

|SSL|
-----

Especially if you are going to use LDAP authentication to your server,
it is important to encrypt the transport channel between clients and the
Glacier2 router to keep your passwords safe.

By default, all logins to OMERO occur over |SSL| using an anonymous
handshake. After the initial connection, communication is un-encrypted to
speed up image loading. Clients can still request to have all communications
encrypted by clicking on the lock symbol.
An unlocked symbol means that non-password related
activities (i.e. anything other than login and changing your password)
will be unencrypted, and the only critical data which is passed in
the clear is your session id.

Administrators can configure OMERO such that unencrypted connections are
not allowed, and the user's choice will be silently ignored. The |SSL|
and non-SSL ports are configured in the :file:`etc/grid/default.xml`
file and, as described above, default to 4064 and 4063 respectively and
can be modified using the :ref:`ports_configuration` configuration
properties. For instance, to prefix all ports with 1, use
:property:`omero.ports.prefix`::

    $ bin/omero config set omero.ports.prefix 1

You can disable unencrypted connections by redirecting clients to the |SSL|
port using the server property :property:`omero.router.insecure`::

    $ bin/omero config set omero.router.insecure "OMERO.Glacier2/router:ssl -p 4064 -h @omero.host@"

--------------

.. seealso:: :doc:`server-ldap`
