OMERO.scripts advanced topics
=============================

Regular user (non-admin) workflow
---------------------------------

If you are using a server for which you do not have admin access, you
must upload scripts as 'user' scripts, which are not trusted to run on
the server machine. The OMERO scripting service will still execute these
scripts in a similar manner to official 'trusted' scripts but behind the
scenes it uses the client machine to execute the script. This means that
any package imports required by the script should be available on the
client machine.

The first step is to connect to the server and set up the processor on
the client (see diagram, below).

.. figure:: /images/omero-scripting-workflow.png
  :align: center
  :alt: OMERO scripting workflow

  OMERO scripting workflow

-  Install 'Ice' from ZeroC and set the environment
   variables, as described in the
   :doc:`server installation page </sysadmins/unix/server-installation>`.
-  You also need the OMERO server download. Go to the :downloads:`OMERO
   downloads <>` page and get the appropriate server package (version
   must match the server you are connecting to). Unzip the package in a
   suitable location.

In a command line terminal, change into the unzipped OMERO package,
connect to the server and start user processor. For example for host:
openmicroscopy.org.uk and user: will

::

    $ cd Desktop/OMERO.server-5.3.x-icexx-bxx/
    $ bin/omero -s openmicroscopy.org.uk -u will script serve user
    $ password: ......

You should see an output similar to the one below

::

    Created session afdbba21-35dc-462a-ab6e-15cc94f93957 (user-4@openmicroscopy.org.uk:4064). Idle timeout: 10 min. Current group: read-only-1
    2016-10-03 10:12:45,964 INFO  [                    omero.util.Resources] (Thread-2  ) Starting
    2016-10-03 10:12:45,965 INFO  [              omero.processor.ProcessorI] (MainThread) Registering processor %fOr(Up>[ERUV%B8$.N</omero.scripts.serve-fa53ba-3959-4d85-876a-00e8b932eb -t -e 1.0:tcp -h openmicroscopy.org.uk -p 54385
    Press any key to exit...

Now you need to open a new terminal window in order to continue with your workflow. 

If you want to run scripts belonging to another user in the same
collaborative group you need to set up your local user processor to
accept scripts from that user. First, find the ID of the user, then
start the user processor and give it the user's ID:

::

    $ cd Desktop/OMERO.server-5.3.x-icexx-bxx/
    $ bin/omero user list
    $ bin/omero script serve user=5

From this point on, the user and admin workflows are the same, except
for a couple of options that are not available to regular users. Also
see below.

.. note::

    Because non-official scripts do not have a unique path name, you
    will be able to run the upload command multiple times on the same file.
    This will create multiple copies of a file in OMERO and then you will
    have to choose the most recent one (highest ID) if you want to run the
    latest script. It is best to avoid this and use the 'replace' command as
    for official scripts.

To list user scripts:

::

    $ omero script list user      # lists user scripts
     id  | Scripts for user                                                                            
        -----+---------------------------------------------------------------------------------------------
     151 | examples/HelloWorld.py        
     251 | examples/Edit_Descriptions.py

You can list scripts belonging to another user that are available for
you (e.g. you are both in the same collaborative group) by using the
user ID as described above:

::

    $ omero user list
    $ omero script list user=5

User scripts can be run from OMERO.insight. They will be found under 'User
Scripts' in the scripts menu. Remember, for user scripts you will need
to have the User-Processor running.


The iScript service
-------------------

The OMERO.blitz server provides a service called 
:javadoc:`iScript <slice2html/omero/api/IScript.html>` that includes
methods to upload, delete, query and run scripts. To access these methods
a session needs to be created and the script service started. However,
you may find it more convenient to use the command line
``bin/omero script`` or the OMERO.insight client to work with scripts
as described on the :doc:`/developers/scripts/user-guide`.

Scripting service API
---------------------

**The recommended way of working with the scripting service is via the
command line as described on the** :doc:`user-guide`
**page. The information on this page is only useful if you want to access
the Scripting service from your own client-side Python code.**


OMERO clients can upload, edit, list and run scripts on the OMERO server
using the Scripting Service API.

These methods (discussed below) are implemented in
:source:`examples/ScriptingService/adminWorkflow.py`.
This sample script allows these functions to be called from the command
line and can be used as an example for writing your own clients.

Most functions of the adminWorkflow.py script are also implemented in
the OMERO |CLI| described on the :doc:`/developers/scripts/user-guide`,
which is the preferred way of accessing the scripting service for script
writers.

Having downloaded
:source:`examples/ScriptingService/adminWorkflow.py`,
you can get some instructions for using the script by typing:

::

    $ python adminWorkflow.py help

To upload 'official' scripts, use the uploadOfficialScript method of the
scripting service or use the upload command from adminWorkflow.py (you
can omit password and enter it later if you do not want it showing in
your console):

::

    $ python adminWorkflow.py -s server -u username -p password -f script/file/to/upload.py upload

Official scripts must have unique paths. Therefore, the
uploadOfficialScript method will not allow you to overwrite and existing
script. However, the adminWorkflow.py upload command will automatically
use ``scriptService.editScript()`` if the file exists. If you want to
change this behavior, edit the adminWorkflow.py script accordingly.

To get the official scripts available to run, use the ``getScripts()``
method, which returns a list of Original Files (scripts). This code will
produce a list of scripts like the one above.

::

    scripts = scriptService.getScripts()
    for s in scripts:
        print s.id.val, s.path.val + s.name.val 

This can be called from adminWorkflow.py with this command:

::

    $ python adminWorkflow.py -s server -u username -p password list

The script can then be run, using the script ID and passing the script a
map of the input parameters. The adminWorkflow.py script has a 'run'
command that can be used to identify a script by its ID or path/name and
run it. The 'run' command will ask for parameter inputs at the command
line.

::

    $ python adminWorkflow.py -s localhost -u root -p omero -f scriptID run

or

::

    $ python adminWorkflow.py -s localhost -u root -p omero -f omero/figure_scripts/Roi_Figure.py run

You can combine the latter form of this command with the 'upload' option
to upload and run a script at once, useful for script writing and
testing.

::

    $ python adminWorkflow.py -s localhost -u root -p omero -f omero/figure_scripts/Roi_Figure.py upload run

Alternatively, you could edit adminWorkflow.py to 'hard-code' a set of
input parameters for a particular script (this strategy is used by
:source:`examples/ScriptingService/runHelloWorld.py`.
The code below shows a more complex example parameter map. This strategy
might save you time if you want to be able to rapidly run and re-run a
script you are working on. Of course, it is also possible to run scripts
from OMERO.insight!

::

    cNamesMap = omero.rtypes.rmap({'0':omero.rtypes.rstring("DAPI"),
        '1':omero.rtypes.rstring("GFP"), 
        '2':omero.rtypes.rstring("Red"), 
        '3':omero.rtypes.rstring("ACA")})
    blue = omero.rtypes.rstring('Blue')
    red = omero.rtypes.rstring('Red')
    mrgdColoursMap = omero.rtypes.rmap({'0':blue, '1':blue, '3':red})
    map = {
       "Image_IDs": omero.rtypes.rlist(imageIds),   
       "Channel_Names": cNamesMap,
       "Split_Indexes": omero.rtypes.rlist([omero.rtypes.rlong(1),omero.rtypes.rlong(2)]),
       "Split_Panels_Grey": omero.rtypes.rbool(True),
       "Merged_Colours": mrgdColoursMap,
       "Merged_Names": omero.rtypes.rbool(True),
       "Width": omero.rtypes.rint(200),
       "Height": omero.rtypes.rint(200),
       "Image_Labels": omero.rtypes.rstring("Datasets"),
       "Algorithm": omero.rtypes.rstring("Mean_Intensity"),
       "Stepping": omero.rtypes.rint(1),
       "Scalebar": omero.rtypes.rint(10), # will be ignored since no pixelsize set
       "Format": omero.rtypes.rstring("PNG"),
       "Figure_Name": omero.rtypes.rstring("splitViewTest"),
       "Overlay_Colour": omero.rtypes.rstring("Red"),
       "ROI_Zoom":omero.rtypes.rfloat(3),
       "ROI_Label":omero.rtypes.rstring("fakeTest"), # won't be found - but should still work
    }

The results returned from running the script can be queried for script
outputs, including stdout and stderr. The following code queries the
results for an output named 'Message' (also displayed by OMERO.insight)

::

    print results.keys()
    if 'Message' in results:
        print results['Message'].getValue()
    if 'stdout' in results:
        origFile = results['stdout'].getValue()
        print "Script generated StdOut in file:" , origFile.getId().getValue()
    if 'stderr' in results:
        origFile = results['stderr'].getValue()
        print "Script generated StdErr in file:" , origFile.getId().getValue()

This code has been extended in adminWorkflow.py to display any ``StdErr``
and ``StdOut`` generated by the script when it is run.
