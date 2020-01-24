.. _dev-extensions:

Developing extensions for Sphinx
================================

Since many projects will need special features in their documentation, Sphinx is
designed to be extensible on several levels.

This is what you can do in an extension: First, you can add new
:term:`builder`\s to support new output formats or actions on the parsed
documents.  Then, it is possible to register custom reStructuredText roles and
directives, extending the markup.  And finally, there are so-called "hook
points" at strategic places throughout the build process, where an extension can
register a hook and run specialized code.

An extension is simply a Python module.  When an extension is loaded, Sphinx
imports this module and executes its ``setup()`` function, which in turn
notifies Sphinx of everything the extension offers -- see the extension tutorial
for examples.

The configuration file itself can be treated as an extension if it contains a
``setup()`` function.  All other extensions to load must be listed in the
:confval:`extensions` configuration value.

Discovery of builders by entry point
------------------------------------

.. versionadded:: 1.6

:term:`Builder` extensions can be discovered by means of `entry points`_ so
that they do not have to be listed in the :confval:`extensions` configuration
value.

Builder extensions should define an entry point in the ``sphinx.builders``
group. The name of the entry point needs to match your builder's
:attr:`~.Builder.name` attribute, which is the name passed to the
:option:`sphinx-build -b` option. The entry point value should equal the
dotted name of the extension module. Here is an example of how an entry point
for 'mybuilder' can be defined in the extension's ``setup.py``::

    setup(
        # ...
        entry_points={
            'sphinx.builders': [
                'mybuilder = my.extension.module',
            ],
        }
    )

Note that it is still necessary to register the builder using
:meth:`~.Sphinx.add_builder` in the extension's :func:`setup` function.

.. _entry points: https://setuptools.readthedocs.io/en/latest/setuptools.html#dynamic-discovery-of-services-and-plugins

.. _important-objects:

Important objects
-----------------

There are several key objects whose API you will use while writing an
extension. These are:

**Application**
   The application object (usually called ``app``) is an instance of
   :class:`.Sphinx`.  It controls most high-level functionality, such as the
   setup of extensions, event dispatching and producing output (logging).

   If you have the environment object, the application is available as
   ``env.app``.

**Environment**
   The build environment object (usually called ``env``) is an instance of
   :class:`.BuildEnvironment`.  It is responsible for parsing the source
   documents, stores all metadata about the document collection and is
   serialized to disk after each build.

   Its API provides methods to do with access to metadata, resolving references,
   etc.  It can also be used by extensions to cache information that should
   persist for incremental rebuilds.

   If you have the application or builder object, the environment is available
   as ``app.env`` or ``builder.env``.

**Builder**
   The builder object (usually called ``builder``) is an instance of a specific
   subclass of :class:`.Builder`.  Each builder class knows how to convert the
   parsed documents into an output format, or otherwise process them (e.g. check
   external links).

   If you have the application object, the builder is available as
   ``app.builder``.

**Config**
   The config object (usually called ``config``) provides the values of
   configuration values set in :file:`conf.py` as attributes.  It is an instance
   of :class:`.Config`.

   The config is available as ``app.config`` or ``env.config``.

To see an example of use of these objects, refer to
:doc:`../development/tutorials/index`.

.. _build-phases:

Build Phases
------------

One thing that is vital in order to understand extension mechanisms is the way
in which a Sphinx project is built: this works in several phases.

**Phase 0: Initialization**

   In this phase, almost nothing of interest to us happens.  The source
   directory is searched for source files, and extensions are initialized.
   Should a stored build environment exist, it is loaded, otherwise a new one is
   created.

**Phase 1: Reading**

   In Phase 1, all source files (and on subsequent builds, those that are new or
   changed) are read and parsed.  This is the phase where directives and roles
   are encountered by docutils, and the corresponding code is executed.  The
   output of this phase is a *doctree* for each source file; that is a tree of
   docutils nodes.  For document elements that aren't fully known until all
   existing files are read, temporary nodes are created.

   There are nodes provided by docutils, which are documented `in the docutils
   documentation <http://docutils.sourceforge.net/docs/ref/doctree.html>`__.
   Additional nodes are provided by Sphinx and :ref:`documented here <nodes>`.

   During reading, the build environment is updated with all meta- and cross
   reference data of the read documents, such as labels, the names of headings,
   described Python objects and index entries.  This will later be used to
   replace the temporary nodes.

   The parsed doctrees are stored on the disk, because it is not possible to
   hold all of them in memory.

**Phase 2: Consistency checks**

   Some checking is done to ensure no surprises in the built documents.

**Phase 3: Resolving**

   Now that the metadata and cross-reference data of all existing documents is
   known, all temporary nodes are replaced by nodes that can be converted into
   output using components called transforms.  For example, links are created
   for object references that exist, and simple literal nodes are created for
   those that don't.

**Phase 4: Writing**

   This phase converts the resolved doctrees to the desired output format, such
   as HTML or LaTeX.  This happens via a so-called docutils writer that visits
   the individual nodes of each doctree and produces some output in the process.

.. note::

   Some builders deviate from this general build plan, for example, the builder
   that checks external links does not need anything more than the parsed
   doctrees and therefore does not have phases 2--4.

To see an example of application, refer to :doc:`../development/tutorials/todo`.

.. _ext-metadata:

Extension metadata
------------------

.. versionadded:: 1.3

The ``setup()`` function can return a dictionary.  This is treated by Sphinx
as metadata of the extension.  Metadata keys currently recognized are:

* ``'version'``: a string that identifies the extension version.  It is used for
  extension version requirement checking (see :confval:`needs_extensions`) and
  informational purposes.  If not given, ``"unknown version"`` is substituted.
* ``'env_version'``: an integer that identifies the version of env data
  structure if the extension stores any data to environment.  It is used to
  detect the data structure has been changed from last build.  The extensions
  have to increment the version when data structure has changed.  If not given,
  Sphinx considers the extension does not stores any data to environment.
* ``'parallel_read_safe'``: a boolean that specifies if parallel reading of
  source files can be used when the extension is loaded.  It defaults to
  ``False``, i.e. you have to explicitly specify your extension to be
  parallel-read-safe after checking that it is.
* ``'parallel_write_safe'``: a boolean that specifies if parallel writing of
  output files can be used when the extension is loaded.  Since extensions
  usually don't negatively influence the process, this defaults to ``True``.


Extension How-tos
-----------------

This section contains several how-tos associated with developing extensions in
Sphinx.

Defining custom template functions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Sometimes it is useful to define your own function in Python that you wish to
then use in a template. For example, if you'd like to insert a template value
with logic that depends on the user's configuration in the site, or if you'd
like to include non-trivial checks and provide friendly error messages for
incorrect configuration in the template.

To define your own template function, you'll need to define two functions inside
your module, a **parent function** and a **template function**.

First, define the **parent function**, which accepts the
arguments for :event:`html-page-context`.

Within the parent function, define
the **template function** that you'd like to use within Jinja. The template function
should return a string or docutils object that Sphinx uses in the templating process
(it will have access to all of the variables that are passed to the parent function).

At the end of the parent function, add the **template function** to the Sphinx
application's context with ``context['my_func'] = my_func``.

Finally, in your main ``setup`` function, add your custom setup function
as a callback for :event:`html-page-context`.

.. code:: python

   # The parent function
   def setup_my_func(app, pagename, templatename, context, doctree):

      # Define the template function
      def my_func(mystring):
          return "Your string is %s" % mystring

      # Add it to the page's context
      context['my_func'] = my_func

   # Your extension's setup function
   def setup(app):
      app.connect("html-page-context", setup_my_func)

Now, you will have access to this function in jinja like so:

.. code:: jinja

   <div>
   {{ my_func("some string") }}
   </div>

Make an extension depend on another extension
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Sometimes your extension depends on the functionality of another
Sphinx extension. Most Sphinx extensions are activated in a
site's :file:`conf.py` file, but this is not available to you as an extension
developer.

.. module:: sphinx.application

To ensure that another extension is activated as a part of your own extension,
use the :meth:`Sphinx.setup_extension` method. This will
activate another extension at run-time, ensuring that you have access to its
functionality.

For example, the following code activates the "recommonmark" extension:

.. code:: python

   def setup(app):
      app.setup_extension("recommonmark")

.. note::

   Since your extension will depend on another, make sure to include it as a part
   of your extension's installation requirements.

APIs used for writing extensions
--------------------------------

.. toctree::
   :maxdepth: 2

   appapi
   projectapi
   envapi
   builderapi
   collectorapi
   markupapi
   domainapi
   parserapi
   nodes
   logging
   i18n
   utils
   deprecated
