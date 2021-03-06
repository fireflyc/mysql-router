MySQL Harness
=============

MySQL Harness is an extensible framework that handles loading and
unloading of *plugins*. The built-in features are dependency tracking
between plugins, configuration file handling, and support for plugin
life-cycles.

For the avoidance of doubt, this particular copy of the software is
released under the version 2 of the GNU General Public License.

Copyright (c) 2015, 2017, Oracle and/or its affiliates. All rights reserved.

Building
--------

To build the MySQL Harness you use the standard steps to build from
CMake:

    cmake .
    make

If you want to do an out-of-source build, the procedure is:

    mkdir build
    cd build
    cmake <path-to-source>
    make


Building and running the unit tests
-----------------------------------

To build the unit tests:

    cmake <path-to-source> -DENABLE_TESTS=1

To run the tests

    make test


Coverage information
--------------------

To build so that coverage information is generated:

    cmake <path-to-source> -DENABLE_COVERAGE=1

To get coverage information, just run the program or the unit tests
(do not forget to enable the unit tests if you want to run them). Once
you have collected coverage information, you can generate an HTML
report in `<build-dir>/coverage/html` using:

    make coverage-html

There are three variables to control where the information is
collected and where the reports are written:

- `GCOV_BASE_DIR` is a cache variable with the full path to a base
  directory for the coverage information.

  It defaults to `${CMAKE_BUILD_DIR}/coverage`.
  
- `GCOV_INFO_FILE` is a cache varible with the full path to the info
  file for the collected coverage information.

  It defaults to `${GCOV_BASE_DIR}/coverage.info`.
  
- `GCOV_HTML_DIR` is a cache variable with the full path to the
  directory where the HTML coverage report will be generated.

  It defaults to `${GCOV_BASE_DIR}/html`.


Documentation
-------------

Documentation can be built as follows:

    make docs

The documentation is using Doxygen to extract documentation comments
from the source code.

The documentation will be placed in the `doc/` directory under the
build directory. For more detailed information about the code, please
read the documentation rather than rely on this `README`.


Installing
----------

To install the files, use `make install`. This will install the
harness, the harness library, the header files for writing plugins,
and the available plugins that were not marked with `NO_INSTALL` (see
below).

If you want to provide a different install prefix, you can do that by
setting the `CMAKE_INSTALL_PREFIX`:

    cmake . -DCMAKE_INSTALL_PREFIX=~/tmp


Running
-------

To start the harness, you need a configuration file. You can find an
example in `data/main.conf`:

    # Example configuration file

    [DEFAULT]
    logging_folder = /var/log/router
    config_folder = /etc/mysql/router
    plugin_folder = /var/lib/router
    runtime_folder = /var/run/router
    data_folder = /var/lib/router

    [example]
    library = example

    [logger]
    library = logger

The configuration file contain information about all the plugins that
should be loaded when starting and configuration options for each
plugin.  The default section contain configuration options available
in to all plugins.

To run the harness, just provide the configuration file as the only
argument:

    harness /etc/mysql/harness/main.conf

Note that the harness read directories for logging, configuration,
etc. from the configuration file so you have to make sure these are
present and that the section name is used to find the plugin structure
in the shared library (see below).

Typically, the harness will then load plugins from the directory
`/var/lib/harness` and write log files to `/var/log/harness`.


Writing Plugins
---------------

All available plugins are in the `plugins/` directory. There is one
directory for each plugin and it is assumed that it contain a
`CMakeLists.txt` file.

The main `CMakeLists.txt` file provide an `add_harness_plugin`
function that can be used add new plugins.

    add_harness_plugin(<name> [ NO_INSTALL ]
                       INTERFACE <directory>
                       SOURCES <source> ...
                       REQUIRES <plugin> ...)

This function adds a plugin named `<name>` built from the given
sources. If `NO_INSTALL` is provided, it will not be installed with
the harness (useful if you have plugins used for testing, see the
`tests/` directory). Otherwise, the plugin will be installed in the
*root*`/var/lib/harness` directory.

The header files in the directory given by `INTERFACE` are the
interface files to the plugin and shall be used by other plugins
requiring features from this plugin. These header files will be
installed alongside the harness include files and will also be made
available to other plugins while building from source.

### Plugin Directory Structure ###

Similar to the harness, each plugin have two types of files:

* Plugin-internal files used to build the plugin. These include the
  source files and but also header files associated with each source
  file and are stored in the `src` directory of the plugin directory.
* Interface files used by other plugins. These are header files that
  are made available to other plugins and are installed alongside the
  harness installed files, usually under the directory
  `/usr/include/mysql/harness`.

### Application Information Structure ###

The application information structure contain some basic fields
providing information to the plugin. Currently these fields are
provided:

    struct AppInfo {
      const char *program;                 /* Name of the application */
      const char *plugin_folder;           /* Location of plugins */
      const char *logging_folder;          /* Log file directory */
      const char *config_folder;           /* Config file directory */
      const char *runtime_folder;          /* Run file directory */
      const Config* config;                /* Configuration information */
    };


### Plugin Structure ###

To define a new plugin, you have to create an instance of the
`Plugin` structure in your plugin similar to this:

    #include <mysql/harness/plugin.h>
    
    static const char* requires[] = {
      "magic (>>1.0)",
    };

    Plugin example = {
      PLUGIN_ABI_VERSION,
      ARCHITECTURE_DESCRIPTOR,
      "An example plugin",       // Brief description of plugin
      VERSION_NUMBER(1,0,0),     // Version number of the plugin

      // Array of required plugins
      sizeof(requires)/sizeof(*requires),
      requires,

      // Array of plugins that conflict with this one
      0,
      NULL,

      init,
      deinit,
      start,
    };


### Initialization and Cleanup ###

After the plugin is loaded, the `init()` function is called for all
plugins with a pointer to the harness (as defined above) as the only
argument.

The `init()` functions are called in dependency order so that all
`init()` functions in required plugins are called before the `init()`
function in the plugin itself.

Before the harness exits, it will call the `deinit()` function with a
pointer to the harness as the only argument.


### Starting the Plugin ###

After all the plugins have been successfully initialized, a thread
will be created for each plugins that have a `start()` function
defined.

The start function will be called with a pointer to the harness as the
only parameter. When all the plugins return from their `start()`
functions, the harness will perform cleanup and exit.


License
-------

License information can be found in the License.txt file.

MySQL FOSS License Exception
We want free and open source software applications under certain
licenses to be able to use specified GPL-licensed MySQL client
libraries despite the fact that not all such FOSS licenses are
compatible with version 2 of the GNU General Public License.
Therefore there are special exceptions to the terms and conditions
of the GPLv2 as applied to these client libraries, which are
identified and described in more detail in the FOSS License
Exception at
<http://www.mysql.com/about/legal/licensing/foss-exception.html>.

This distribution may include materials developed by third
parties. For license and attribution notices for these
materials, please refer to the documentation that accompanies
this distribution (see the "Licenses for Third-Party Components"
appendix) or view the online documentation at
<http://dev.mysql.com/doc/>.

GPLv2 Disclaimer
For the avoidance of doubt, except that if any license choice
other than GPL or LGPL is available it will apply instead,
Oracle elects to use only the General Public License version 2
(GPLv2) at this time for any software where a choice of GPL
license versions is made available with the language indicating
that GPLv2 or any later version may be used, or where a choice
of which version of the GPL is applied is otherwise unspecified.
