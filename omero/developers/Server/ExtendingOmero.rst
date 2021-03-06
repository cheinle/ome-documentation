Extending OMERO.server
======================

.. topic:: Overview

    Despite all the effort put into building OMERO, it will never
    satisfy the requirements of every group. Where we have seen it
    useful to do so, we have created extension points which can be used
    by third-party developers to extend, improve, and adapt OMERO. We
    outline most of these options below as well as some of their
    trade-offs. We are also always interested to hear other possible
    extension points. Please contact the :mailinglist:`ome-devel mailing list <ome-devel/>`
    with any such suggestions.


Existing extension points
-------------------------

To get a feeling for what type of extension points are available, you
might want to take a look at the following pages. Many of them will
point you back to this page for packaging and deploying your new code.

-  :doc:`/developers/Search/FileParsers` - write Java file parsers to
   further extend search
-  :doc:`/developers/Server/LoginAttemptListener` - write a
   Java handler for failed login attempts
-  :doc:`/developers/cli/index` - write drop in Python extensions
   for the command-line
-  :doc:`/developers/scripts/index` - write python scripts to
   process data server-side
-  :doc:`/developers/Server/Ldap` - write a Java authentication
   plugin
-  :doc:`/developers/Server/PasswordProvider` - write a Java
   password backend
-  :doc:`/developers/Modules/Search/Bridges` - write Java Lucene parsers
   to extend search
-  :doc:`/sysadmins/mail` - server email sender (added in OMERO 5.1, no
   developer documentation as yet)
-  :property:`omero.policy.bean` - policy configuration point e.g. for setting
   download restriction policy on users (added in OMERO 5.1, no developer
   documentation as yet)

Extending the Model
-------------------

The OME Data Model and its OMERO representation, the |OmeroModel|,
intentionally draw lines between what metadata can be supported and what
cannot. Though we are always examining new fields for inclusion, it is not
possible to represent everyone's model within OME.

Structured annotations
^^^^^^^^^^^^^^^^^^^^^^

The primary extension point for including external data are the
:doc:`/developers/Model/StructuredAnnotations` (SAs). SAs
are designed as email-like attachments which can be associated with
various core metadata types. In general, they should link to information
outside of the OME model, i.e. information which OMERO clients and
servers do not understand. URLs can point to external data
sources, or XML in a non-OME namespace can be attached.

The primary drawbacks are that the attachments are opaque and cannot be
used in a fine-grain manner.

Code generation
^^^^^^^^^^^^^^^

Since it is prohibitive to model full objects with the SAs, one alternative is
to add types directly to the :doc:`generated code </developers/Model>`. By
adding a file named ``*.ome.xml`` to
:model_sourcedir:`src/main/resources/mappings` and running a full-build, it
is possible to have new objects generated in all
:doc:`/developers/server-blitz` languages. Supported fields include:

-  boolean
-  string
-  long
-  double
-  timestamp
-  links to any other ``ome.model.*`` object, including enumerations

For example:

::

    <types>
      <!-- "named" and "described" are short-cuts to generate the fields "name" and "description" -->
      <type id="ome.model.myextensions.Example" named="true" described="true">
        <required name="valueA" type="boolean"/>  <!-- This is NONNULL -->
        <optional name="valueB" type="long"/>     <!-- This is nullable -->
        <onemany  name="images" type="ome.model.core.Image"/> <!-- A set of images -->
      </type>
    </types>

Collections of primitive values like
``<onemany name="values" type="long"/>`` are not supported. Please see
the existing mapping files for more examples of what can be done.

The primary drawback of code-generating your own types is isolation and
maintenance. Firstly, your installation becomes isolated from the rest
of the OME ecosystem. New types are not understood by other servers and
clients, and cannot be exported or shared. Secondly, you will need to
maintain your own server **and** client builds of the system, since the
provided binary builds would not have your new types.

Measurement results
^^^^^^^^^^^^^^^^^^^

For storing large quantities of only partially structured data, such as
tabular/CSV data with no pre-defined columns, neither the SAs nor the
code-generation extensions are ideal. SAs cannot easily be aggregated,
and code-generation would generate too many types. This is particularly
clear in the storage and management of HCS analysis results.

To solve this problem, we provide the :ref:`OMERO.tables <omerotables>` API
for storing tabular data indexed via Roi, Well, or Image id.

Services
--------

Traditionally, services were added via Java interfaces in the
:common_sourcedir:`src/main/java/ome/api`
package. The creation of such "core" services is described under
:doc:`/developers/Server/HowToCreateAService`. However,
with the introduction of :doc:`/developers/server-blitz`, it is also
possible to write blitz-only services which are defined by a slice
definition under :blitz_sourcedir:`src/main/slice/omero`.

A core service is required when server internal code should also make
use of the interface. Since this is very rarely the case for third-party
developers wanting to extend OMERO, only the creation of blitz services
will be discussed here.

Add a slice definition
^^^^^^^^^^^^^^^^^^^^^^

The easiest possible service definition in slice is:

::

      module example {
        interface NewService {
          void doSomething();
        };
      };

This should be added to any existing or a new :file:`*.ice` file under the
:file:`src/main/slice/omero` directory. After the next ant build, stubs
will be created for all the :doc:`/developers/server-blitz` languages,
i.e.  |OmeroJava|, |OmeroPy|, and |OmeroCpp|.

.. note::

    Once you have gotten your code working, it is most re-usable
    if you can put it all in a single directory under tools/. These
    components also have their ``resources/*.ice`` files turned into code,
    and they can produce their own artifacts which you can distribute
    without modifying the main code base.

Warning: exceptions
^^^^^^^^^^^^^^^^^^^

You will need to think carefully about what exceptions to handle. Ice
(especially |OmeroCpp|) does not handle exceptions
well that are not strictly defined. In general, if you would like to add
your own exception type, feel free to do so, but either 1) subclass
``omero::ServerError`` or 2) add to the appropriate ``throws`` clauses.
And regardless, if you are accessing any internal OMERO API, add
``omero::ServerError`` to your ``throws`` clause.

See :doc:`/developers/Modules/ExceptionHandling` for more
information.

Java implementation using _Disp
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To implement your service, create a class subclassing
"example.\_NewServiceDisp" class which was code-generated. In this
example, the class would be named "NewServiceI" by convention. If this
service needs to make use of any of the internal API, it should do so
via dependency injection. For example, to use IQuery add either:

::

        void setLocalQuery(LocalQuery query) {
            this.query = query;
        }

or

::

        NewServiceI(LocalQuery query) {
            this.query = query;
        }

The next step "Java Configuration" will take care of how those objects
get injected.

Java implementation using _Tie
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Rather than subclassing the ``_Disp`` object, it is also possible to
implement the ``_Tie`` interface for your new service. This allows
wrapping and testing your implementation more easily at the cost of a
little indirection. You can see how such an object is configured in
:blitz_source:`blitz-servantDefinitions <src/main/resources/ome/services/blitz-servantDefinitions.xml#L36>`.

Java configuration
^^^^^^^^^^^^^^^^^^

Configuration in the Java servers takes place via Spring_. One or more files
matching a pattern like ``ome/services/blitz-*.xml`` should be added to your
application.

::

    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
    <beans>

      <bean class="NewServiceI">
        <description>
        This is a simple bean definition in Spring. The description is not necessary.
        </description>
        <constructor-arg ref="internal-ome.api.IQuery"/>
      </bean>

    </beans>

The three patterns which are available are:

-  ``ome/services/blitz-*.xml`` - highest-level objects which have
   access to all the other defined objects.
-  ``ome/services/services-*.xml`` - internal server objects which do
   not have access to ``blitz-*.xml`` objects.
-  ``ome/services/db-*.xml`` - base connection and security objects.
   These will be included in background java process like the index and
   pixeldata handlers.

   .. note::

      :doc:`/developers/Server/PasswordProvider` and similar should
      be included at this level.

See :blitz_sourcedir:`src/main/resources/ome/services`
and :server_sourcedir:`src/main/resources/ome/services`
for all the available objects.

.. _JavaDeployment:

Java deployment
^^^^^^^^^^^^^^^

Finally, these resources should all be added to
``OMERO_DIST/lib/server/extensions.jar``:

-  the code generated classes
-  your ``NewServiceI.class`` file and any related classes
-  your ``ome/service/blitz-*.xml`` file (or other XML)

Non-service beans
^^^^^^^^^^^^^^^^^

In addition to writing your own services, the instructions above can be
used to package any Spring-bean into the OMERO server. For example:

::

    //
    // MyLoginAttemptListener.java
    //
    import ome.services.messages.LoginAttemptMessage;

    import org.springframework.context.ApplicationListener;

    /**
     * Trivial listener for login attempts.
     */

    public class MyLoginAttemptListener implements
            ApplicationListener<LoginAttemptMessage> {

        public void onApplicationEvent(LoginAttemptMessage lam) {
            if (lam.success != null && !lam.success) {
                // Do something
            }
        }

    }

::

    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
    <!--
    //
    // ome/services/blitz-myLoginListener.xml
    //
    -->
    <beans>
      <bean class="myLoginAttemptListener" class="MyLoginAttemptListener">
        <description>
        This listener will be added to the Spring runtime and listen for all LoginAttemptMessages.
        </description>
      </bean>

    </beans>

Putting ``MyLoginAttemptListener.class`` and
``ome/services/blitz-myLoginListener.xml`` into
``lib/server/extensions.jar`` is enough to activate your code:

::

    ~/example $ ls -1
    MyLoginListener.class
    MyLoginListener.java
    lib
    ...
    ~/example $ jar cvf lib/server/extensions.jar MyLoginListener.class ome/services/blitz-myLoginListener.xml
    added manifest
    adding: MyLoginListener.class(in = 0) (out= 0)(stored 0%)
    adding: ome/services/blitz-myLoginListener.xml(in = 0) (out= 0)(stored 0%)

Servers
-------

With the |OmeroGrid| infrastructure, it is possible to have your own
processes managed by the OMERO infrastructure. For example, at some
sites, `NGINX <https://www.nginx.com/resources/wiki/>`_ is started to
host |OmeroWeb|. Better integration is possible however, if your server
also uses the Ice_ remoting framework.

One way or the other, to have your server started, monitored, and
eventually shutdown by |OmeroGrid|, you will need
to add it to the "application descriptor" for your site. When using:

::

      bin/omero admin start

the application descriptor used is :file:`etc/grid/default.xml`.
The ``<application>`` element contains various ``<node>``\ s. Each node
is a single daemon process that can start and stop other processes.
Inside the nodes, you can either directly add a ``<server>`` element, or
in order to reuse your description, you can use a ``<server-instance>``
which must refer to a ``<server-template>``.

To clarify with an example, if you have a simple
application which should watch for newly created Images and send you an
email: ``mail_on_import.py``, you could add this in either of the following
ways:

Server element
^^^^^^^^^^^^^^

::

      <node name="my-emailer-node">  <!-- this could also be an existing node, but it must be unique -->
        <server id="my-emailer-server" exe="/home/josh/mail_on_import.py" activation="always">
          <env>${PYTHONPATH}</env>
          <!-- The adapter name must also be unique -->
          <adapter name="MyAdapter" register-process="true" endpoints="tcp"/>
        </server>
      </node>

Server-template and server-instance elements
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

      <server-template id="emailer-template">  <!-- must also be unique -->
        <property name="user"/>
        <server id="emailer-server-${user}" exe="/home/${user}/mail_on_import.py" activation="always">
          <env>${PYTHONPATH}</env>
          <adapter name="MyAdapter" register-process="true" endpoints="tcp"/>
        </server>
      </server-template>

      <node name="our-emailer-node">
        <server-instance id="emailer-template" user="ann">
        <server-instance id="emailer-template" user="ann">
      </node>

.. seealso::
    :ome-devel:`[ome-devel] model description driven code generation <2009-July/001332.html>`
