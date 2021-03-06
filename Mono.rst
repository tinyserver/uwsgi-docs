The Mono ASP.NET plugin
=======================

uWSGI 1.9 added support for the Mono platform, expecially for the ASP.NET infrastructure.

The most common way to deploy Mono ASP.NET applications is with the XSP project, a simple webserver gateway
implementing HTTP and FastCGI protocols.

With the Mono plugin you will be able to host asp.net application directly in uWSGI gaining all of its feature in your application for free.

As all of the other uWSGI plugin you can call functions exported from the other languages using the :doc:`RPC` subsystem.

Building uWSGI + Mono
*********************

You can build the mono support as a plugin or in a monolithic build.

A build profile named "mono" is available, making the task pretty simple.

Be sure to have mono installed in your system (you need mono headers, the mcs compiler and obviously the System.Web gac. They are available in standard mono distribution)

On recent Debian/Ubuntu systems a

.. code-block:: sh

    apt-get install build-essential python mono-xsp4 asp.net-examples

(mono-xsp4 will be a trick to install all we need in a single shot, while asp.net examples will be used for testing our setup)

We can build  monolithic uWSGI distribution with Mono embedded:

.. code-block:: sh

   UWSGI_PROFILE=mono make

At the end of the procedure (if all goes well) you will get the path to the uwsgi.dll assembly.

You may want to install it in your GAC (with gacutil -i <path>) to avoid specifying its path every time (this library allows access
to the uWSGI api from Mono applications)

Starting the server
*******************

The Mono plugin has an official modifier1: the 15

.. code-block:: ini

   [uwsgi]
   http = :9090
   http-modifier1 = 15
   mono-app = /usr/share/asp.net-demos
   mono-index = index.asp

The previous setup assumes uwsgi.dll has been installed in the GAC, if it is not your case you can force its path with:

.. code-block:: ini

   [uwsgi]
   http = :9090
   http-modifier1 = 15
   mono-app = /usr/share/asp.net-demos
   mono-index = index.asp
   mono-assembly = /usr/lib/uwsgi/uwsgi.dll

The /usr/share/asp.net-demos is the directory containing the Mono examples asp.net applications.

If starting uwsgi you get an error about not being able to find uwsgi.dll you can enforce a specific search path with

.. code-block:: ini

   [uwsgi]
   http = :9090
   http-modifier1 = 15
   mono-app = /usr/share/asp.net-demos
   mono-index = index.asp
   mono-assembly = /usr/lib/uwsgi/uwsgi.dll
   env = MONO_PATH=/usr/lib/uwsgi/

or you can simply copy uwsgi.dll in the /bin directory of your site directory (/usr/share/asp.net-demos in this case).

The ``mono-index`` option is used to set the file to search when a directory is requested. You can specify it multiple times.


Concurrency and fork() unfriendlyness
**************************************

As the Mono VM is not fork() friendly, a new VM is spawned for each workers. This ensure you can run your application in multiprocessing mode.

Mono has a real solid multithreading support and it works great with the uWSGI threads support.

.. code-block:: ini

   [uwsgi]
   http = :9090
   http-modifier1 = 15
   mono-app = /usr/share/asp.net-demos
   mono-index = index.asp
   mono-assembly = /usr/lib/uwsgi/uwsgi.dll
   env = MONO_PATH=/usr/lib/uwsgi/
  
   master = true
   processes = 4
   threads = 20

With this setup you will spawn 4 processes each with 20 threads. Try to not rely on a single process. Albeit it is a common setup
in the so-called "Enterprise environment", having multiple processes ensure you greater availability (thanks to the master work).
This rule (as an example) applies even to the :doc:`JVM` plugin.

API access
**********

This is a work in progress. Currently only a couple of functions are exported. High precedence will be given to the :doc:`RPC` and Signal subsystem and to the :doc:`Caching` framework

Tricks
******

As always uWSGI tries to optimize (where possibile) the "common" operations of your applications. Serving static files is automatically
accelerated (or offloaded if offloading is enabled) and all of the path resolutions are cached.
