.. _plugins:
.. _`writing-plugins`:

Writing plugins
===============

It is easy to implement `local conftest plugins`_ for your own project
or `pip-installable plugins`_ that can be used throughout many projects,
including third party projects.  Please refer to :ref:`using plugins` if you
only want to use but not write plugins.

A plugin contains one or multiple hook functions. :ref:`Writing hooks <writinghooks>`
explains the basics and details of how you can write a hook function yourself.
``pytest`` implements all aspects of configuration, collection, running and
reporting by calling :ref:`well specified hooks <hook-reference>` of the following plugins:

* builtin plugins: loaded from pytest's internal ``_pytest`` directory.

* :ref:`external plugins <extplugins>`: modules discovered through
  `setuptools entry points`_

* `conftest.py plugins`_: modules auto-discovered in test directories

In principle, each hook call is a ``1:N`` Python function call where ``N`` is the
number of registered implementation functions for a given specification.
All specifications and implementations follow the ``pytest_`` prefix
naming convention, making them easy to distinguish and find.

.. _`pluginorder`:

Plugin discovery order at tool startup
--------------------------------------

``pytest`` loads plugin modules at tool startup in the following way:

* by loading all builtin plugins

* by loading all plugins registered through `setuptools entry points`_.

* by pre-scanning the command line for the ``-p name`` option
  and loading the specified plugin before actual command line parsing.

* by loading all :file:`conftest.py` files as inferred by the command line
  invocation:

  - if no test paths are specified use current dir as a test path
  - if exists, load ``conftest.py`` and ``test*/conftest.py`` relative
    to the directory part of the first test path.

  Note that pytest does not find ``conftest.py`` files in deeper nested
  sub directories at tool startup.  It is usually a good idea to keep
  your ``conftest.py`` file in the top level test or project root directory.

* by recursively loading all plugins specified by the
  ``pytest_plugins`` variable in ``conftest.py`` files


.. _`pytest/plugin`: http://bitbucket.org/pytest-dev/pytest/src/tip/pytest/plugin/
.. _`conftest.py plugins`:
.. _`localplugin`:
.. _`local conftest plugins`:

conftest.py: local per-directory plugins
----------------------------------------

Local ``conftest.py`` plugins contain directory-specific hook
implementations.  Hook Session and test running activities will
invoke all hooks defined in ``conftest.py`` files closer to the
root of the filesystem.  Example of implementing the
``pytest_runtest_setup`` hook so that is called for tests in the ``a``
sub directory but not for other directories::

    a/conftest.py:
        def pytest_runtest_setup(item):
            # called for running each test in 'a' directory
            print("setting up", item)

    a/test_sub.py:
        def test_sub():
            pass

    test_flat.py:
        def test_flat():
            pass

Here is how you might run it::

     pytest test_flat.py --capture=no  # will not show "setting up"
     pytest a/test_sub.py --capture=no  # will show "setting up"

.. note::
    If you have ``conftest.py`` files which do not reside in a
    python package directory (i.e. one containing an ``__init__.py``) then
    "import conftest" can be ambiguous because there might be other
    ``conftest.py`` files as well on your ``PYTHONPATH`` or ``sys.path``.
    It is thus good practice for projects to either put ``conftest.py``
    under a package scope or to never import anything from a
    ``conftest.py`` file.

    See also: :ref:`pythonpath`.


Writing your own plugin
-----------------------

.. _`setuptools`: https://pypi.org/project/setuptools/

If you want to write a plugin, there are many real-life examples
you can copy from:

* a custom collection example plugin: :ref:`yaml plugin`
* builtin plugins which provide pytest's own functionality
* many `external plugins <http://plugincompat.herokuapp.com>`_ providing additional features

All of these plugins implement :ref:`hooks <hook-reference>` and/or :ref:`fixtures <fixture>`
to extend and add functionality.

.. note::
    Make sure to check out the excellent
    `cookiecutter-pytest-plugin <https://github.com/pytest-dev/cookiecutter-pytest-plugin>`_
    project, which is a `cookiecutter template <https://github.com/audreyr/cookiecutter>`_
    for authoring plugins.

    The template provides an excellent starting point with a working plugin,
    tests running with tox, a comprehensive README file as well as a
    pre-configured entry-point.

Also consider :ref:`contributing your plugin to pytest-dev<submitplugin>`
once it has some happy users other than yourself.


.. _`setuptools entry points`:
.. _`pip-installable plugins`:

Making your plugin installable by others
----------------------------------------

If you want to make your plugin externally available, you
may define a so-called entry point for your distribution so
that ``pytest`` finds your plugin module.  Entry points are
a feature that is provided by `setuptools`_. pytest looks up
the ``pytest11`` entrypoint to discover its
plugins and you can thus make your plugin available by defining
it in your setuptools-invocation:

.. sourcecode:: python

    # sample ./setup.py file
    from setuptools import setup

    setup(
        name="myproject",
        packages=["myproject"],
        # the following makes a plugin available to pytest
        entry_points={"pytest11": ["name_of_plugin = myproject.pluginmodule"]},
        # custom PyPI classifier for pytest plugins
        classifiers=["Framework :: Pytest"],
    )

If a package is installed this way, ``pytest`` will load
``myproject.pluginmodule`` as a plugin which can define
:ref:`hooks <hook-reference>`.

.. note::

    Make sure to include ``Framework :: Pytest`` in your list of
    `PyPI classifiers <https://pypi.org/classifiers/>`_
    to make it easy for users to find your plugin.


.. _assertion-rewriting:

Assertion Rewriting
-------------------

One of the main features of ``pytest`` is the use of plain assert
statements and the detailed introspection of expressions upon
assertion failures.  This is provided by "assertion rewriting" which
modifies the parsed AST before it gets compiled to bytecode.  This is
done via a :pep:`302` import hook which gets installed early on when
``pytest`` starts up and will perform this rewriting when modules get
imported.  However, since we do not want to test different bytecode
from what you will run in production, this hook only rewrites test modules
themselves (as defined by the :confval:`python_files` configuration option),
and any modules which are part of plugins.
Any other imported module will not be rewritten and normal assertion behaviour
will happen.

If you have assertion helpers in other modules where you would need
assertion rewriting to be enabled you need to ask ``pytest``
explicitly to rewrite this module before it gets imported.

.. autofunction:: pytest.register_assert_rewrite
    :noindex:

This is especially important when you write a pytest plugin which is
created using a package.  The import hook only treats ``conftest.py``
files and any modules which are listed in the ``pytest11`` entrypoint
as plugins.  As an example consider the following package::

   pytest_foo/__init__.py
   pytest_foo/plugin.py
   pytest_foo/helper.py

With the following typical ``setup.py`` extract:

.. code-block:: python

   setup(..., entry_points={"pytest11": ["foo = pytest_foo.plugin"]}, ...)

In this case only ``pytest_foo/plugin.py`` will be rewritten.  If the
helper module also contains assert statements which need to be
rewritten it needs to be marked as such, before it gets imported.
This is easiest by marking it for rewriting inside the
``__init__.py`` module, which will always be imported first when a
module inside a package is imported.  This way ``plugin.py`` can still
import ``helper.py`` normally.  The contents of
``pytest_foo/__init__.py`` will then need to look like this:

.. code-block:: python

   import pytest

   pytest.register_assert_rewrite("pytest_foo.helper")


Requiring/Loading plugins in a test module or conftest file
-----------------------------------------------------------

You can require plugins in a test module or a ``conftest.py`` file like this:

.. code-block:: python

    pytest_plugins = ["name1", "name2"]

When the test module or conftest plugin is loaded the specified plugins
will be loaded as well. Any module can be blessed as a plugin, including internal
application modules:

.. code-block:: python

    pytest_plugins = "myapp.testsupport.myplugin"

``pytest_plugins`` variables are processed recursively, so note that in the example above
if ``myapp.testsupport.myplugin`` also declares ``pytest_plugins``, the contents
of the variable will also be loaded as plugins, and so on.

.. _`requiring plugins in non-root conftests`:

.. note::
    Requiring plugins using a ``pytest_plugins`` variable in non-root
    ``conftest.py`` files is deprecated.

    This is important because ``conftest.py`` files implement per-directory
    hook implementations, but once a plugin is imported, it will affect the
    entire directory tree. In order to avoid confusion, defining
    ``pytest_plugins`` in any ``conftest.py`` file which is not located in the
    tests root directory is deprecated, and will raise a warning.

This mechanism makes it easy to share fixtures within applications or even
external applications without the need to create external plugins using
the ``setuptools``'s entry point technique.

Plugins imported by ``pytest_plugins`` will also automatically be marked
for assertion rewriting (see :func:`pytest.register_assert_rewrite`).
However for this to have any effect the module must not be
imported already; if it was already imported at the time the
``pytest_plugins`` statement is processed, a warning will result and
assertions inside the plugin will not be rewritten.  To fix this you
can either call :func:`pytest.register_assert_rewrite` yourself before
the module is imported, or you can arrange the code to delay the
importing until after the plugin is registered.


Accessing another plugin by name
--------------------------------

If a plugin wants to collaborate with code from
another plugin it can obtain a reference through
the plugin manager like this:

.. sourcecode:: python

    plugin = config.pluginmanager.get_plugin("name_of_plugin")

If you want to look at the names of existing plugins, use
the ``--trace-config`` option.


.. _registering-markers:

Registering custom markers
--------------------------

If your plugin uses any markers, you should register them so that they appear in
pytest's help text and do not :ref:`cause spurious warnings <unknown-marks>`.
For example, the following plugin would register ``cool_marker`` and
``mark_with`` for all users:

.. code-block:: python

    def pytest_configure(config):
        config.addinivalue_line("markers", "cool_marker: this one is for cool tests.")
        config.addinivalue_line(
            "markers", "mark_with(arg, arg2): this marker takes arguments."
        )


Testing plugins
---------------

pytest comes with a plugin named ``pytester`` that helps you write tests for
your plugin code. The plugin is disabled by default, so you will have to enable
it before you can use it.

You can do so by adding the following line to a ``conftest.py`` file in your
testing directory:

.. code-block:: python

    # content of conftest.py

    pytest_plugins = ["pytester"]

Alternatively you can invoke pytest with the ``-p pytester`` command line
option.

This will allow you to use the :py:class:`testdir <_pytest.pytester.Testdir>`
fixture for testing your plugin code.

Let's demonstrate what you can do with the plugin with an example. Imagine we
developed a plugin that provides a fixture ``hello`` which yields a function
and we can invoke this function with one optional parameter. It will return a
string value of ``Hello World!`` if we do not supply a value or ``Hello
{value}!`` if we do supply a string value.

.. code-block:: python

    import pytest


    def pytest_addoption(parser):
        group = parser.getgroup("helloworld")
        group.addoption(
            "--name",
            action="store",
            dest="name",
            default="World",
            help='Default "name" for hello().',
        )


    @pytest.fixture
    def hello(request):
        name = request.config.getoption("name")

        def _hello(name=None):
            if not name:
                name = request.config.getoption("name")
            return "Hello {name}!".format(name=name)

        return _hello


Now the ``testdir`` fixture provides a convenient API for creating temporary
``conftest.py`` files and test files. It also allows us to run the tests and
return a result object, with which we can assert the tests' outcomes.

.. code-block:: python

    def test_hello(testdir):
        """Make sure that our plugin works."""

        # create a temporary conftest.py file
        testdir.makeconftest(
            """
            import pytest

            @pytest.fixture(params=[
                "Brianna",
                "Andreas",
                "Floris",
            ])
            def name(request):
                return request.param
        """
        )

        # create a temporary pytest test file
        testdir.makepyfile(
            """
            def test_hello_default(hello):
                assert hello() == "Hello World!"

            def test_hello_name(hello, name):
                assert hello(name) == "Hello {0}!".format(name)
        """
        )

        # run all tests with pytest
        result = testdir.runpytest()

        # check that all 4 tests passed
        result.assert_outcomes(passed=4)


additionally it is possible to copy examples for an example folder before running pytest on it

.. code-block:: ini

  # content of pytest.ini
  [pytest]
  pytester_example_dir = .


.. code-block:: python

    # content of test_example.py


    def test_plugin(testdir):
        testdir.copy_example("test_example.py")
        testdir.runpytest("-k", "test_example")


    def test_example():
        pass

.. code-block:: pytest

    $ pytest
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR, inifile: pytest.ini
    collected 2 items

    test_example.py ..                                                   [100%]

    ============================= warnings summary =============================
    test_example.py::test_plugin
      $REGENDOC_TMPDIR/test_example.py:4: PytestExperimentalApiWarning: testdir.copy_example is an experimental api that may change over time
        testdir.copy_example("test_example.py")

    test_example.py::test_plugin
      $PYTHON_PREFIX/lib/python3.8/site-packages/_pytest/compat.py:333: PytestDeprecationWarning: The TerminalReporter.writer attribute is deprecated, use TerminalReporter._tw instead at your own risk.
      See https://docs.pytest.org/en/stable/deprecations.html#terminalreporter-writer for more information.
        return getattr(object, name, default)

    -- Docs: https://docs.pytest.org/en/stable/warnings.html
    ====================== 2 passed, 2 warnings in 0.12s =======================

For more information about the result object that ``runpytest()`` returns, and
the methods that it provides please check out the :py:class:`RunResult
<_pytest.pytester.RunResult>` documentation.




.. _`writinghooks`:

Writing hook functions
======================


.. _validation:

hook function validation and execution
--------------------------------------

pytest calls hook functions from registered plugins for any
given hook specification.  Let's look at a typical hook function
for the ``pytest_collection_modifyitems(session, config,
items)`` hook which pytest calls after collection of all test items is
completed.

When we implement a ``pytest_collection_modifyitems`` function in our plugin
pytest will during registration verify that you use argument
names which match the specification and bail out if not.

Let's look at a possible implementation:

.. code-block:: python

    def pytest_collection_modifyitems(config, items):
        # called after collection is completed
        # you can modify the ``items`` list
        ...

Here, ``pytest`` will pass in ``config`` (the pytest config object)
and ``items`` (the list of collected test items) but will not pass
in the ``session`` argument because we didn't list it in the function
signature.  This dynamic "pruning" of arguments allows ``pytest`` to
be "future-compatible": we can introduce new hook named parameters without
breaking the signatures of existing hook implementations.  It is one of
the reasons for the general long-lived compatibility of pytest plugins.

Note that hook functions other than ``pytest_runtest_*`` are not
allowed to raise exceptions.  Doing so will break the pytest run.



.. _firstresult:

firstresult: stop at first non-None result
-------------------------------------------

Most calls to ``pytest`` hooks result in a **list of results** which contains
all non-None results of the called hook functions.

Some hook specifications use the ``firstresult=True`` option so that the hook
call only executes until the first of N registered functions returns a
non-None result which is then taken as result of the overall hook call.
The remaining hook functions will not be called in this case.

.. _`hookwrapper`:

hookwrapper: executing around other hooks
-------------------------------------------------

.. currentmodule:: _pytest.core



pytest plugins can implement hook wrappers which wrap the execution
of other hook implementations.  A hook wrapper is a generator function
which yields exactly once. When pytest invokes hooks it first executes
hook wrappers and passes the same arguments as to the regular hooks.

At the yield point of the hook wrapper pytest will execute the next hook
implementations and return their result to the yield point in the form of
a :py:class:`Result <pluggy._Result>` instance which encapsulates a result or
exception info.  The yield point itself will thus typically not raise
exceptions (unless there are bugs).

Here is an example definition of a hook wrapper:

.. code-block:: python

    import pytest


    @pytest.hookimpl(hookwrapper=True)
    def pytest_pyfunc_call(pyfuncitem):
        do_something_before_next_hook_executes()

        outcome = yield
        # outcome.excinfo may be None or a (cls, val, tb) tuple

        res = outcome.get_result()  # will raise if outcome was exception

        post_process_result(res)

        outcome.force_result(new_res)  # to override the return value to the plugin system

Note that hook wrappers don't return results themselves, they merely
perform tracing or other side effects around the actual hook implementations.
If the result of the underlying hook is a mutable object, they may modify
that result but it's probably better to avoid it.

For more information, consult the
:ref:`pluggy documentation about hookwrappers <pluggy:hookwrappers>`.

.. _plugin-hookorder:

Hook function ordering / call example
-------------------------------------

For any given hook specification there may be more than one
implementation and we thus generally view ``hook`` execution as a
``1:N`` function call where ``N`` is the number of registered functions.
There are ways to influence if a hook implementation comes before or
after others, i.e.  the position in the ``N``-sized list of functions:

.. code-block:: python

    # Plugin 1
    @pytest.hookimpl(tryfirst=True)
    def pytest_collection_modifyitems(items):
        # will execute as early as possible
        ...


    # Plugin 2
    @pytest.hookimpl(trylast=True)
    def pytest_collection_modifyitems(items):
        # will execute as late as possible
        ...


    # Plugin 3
    @pytest.hookimpl(hookwrapper=True)
    def pytest_collection_modifyitems(items):
        # will execute even before the tryfirst one above!
        outcome = yield
        # will execute after all non-hookwrappers executed

Here is the order of execution:

1. Plugin3's pytest_collection_modifyitems called until the yield point
   because it is a hook wrapper.

2. Plugin1's pytest_collection_modifyitems is called because it is marked
   with ``tryfirst=True``.

3. Plugin2's pytest_collection_modifyitems is called because it is marked
   with ``trylast=True`` (but even without this mark it would come after
   Plugin1).

4. Plugin3's pytest_collection_modifyitems then executing the code after the yield
   point.  The yield receives a :py:class:`Result <pluggy._Result>` instance which encapsulates
   the result from calling the non-wrappers.  Wrappers shall not modify the result.

It's possible to use ``tryfirst`` and ``trylast`` also in conjunction with
``hookwrapper=True`` in which case it will influence the ordering of hookwrappers
among each other.


Declaring new hooks
------------------------

.. currentmodule:: _pytest.hookspec

Plugins and ``conftest.py`` files may declare new hooks that can then be
implemented by other plugins in order to alter behaviour or interact with
the new plugin:

.. autofunction:: pytest_addhooks
    :noindex:

Hooks are usually declared as do-nothing functions that contain only
documentation describing when the hook will be called and what return values
are expected. The names of the functions must start with `pytest_` otherwise pytest won't recognize them.

Here's an example. Let's assume this code is in the ``hooks.py`` module.

.. code-block:: python

    def pytest_my_hook(config):
        """
        Receives the pytest config and does things with it
        """

To register the hooks with pytest they need to be structured in their own module or class. This
class or module can then be passed to the ``pluginmanager`` using the ``pytest_addhooks`` function
(which itself is a hook exposed by pytest).

.. code-block:: python

    def pytest_addhooks(pluginmanager):
        """ This example assumes the hooks are grouped in the 'hooks' module. """
        from my_app.tests import hooks

        pluginmanager.add_hookspecs(hooks)

For a real world example, see `newhooks.py`_ from `xdist <https://github.com/pytest-dev/pytest-xdist>`_.

.. _`newhooks.py`: https://github.com/pytest-dev/pytest-xdist/blob/974bd566c599dc6a9ea291838c6f226197208b46/xdist/newhooks.py

Hooks may be called both from fixtures or from other hooks. In both cases, hooks are called
through the ``hook`` object, available in the ``config`` object. Most hooks receive a
``config`` object directly, while fixtures may use the ``pytestconfig`` fixture which provides the same object.

.. code-block:: python

    @pytest.fixture()
    def my_fixture(pytestconfig):
        # call the hook called "pytest_my_hook"
        # 'result' will be a list of return values from all registered functions.
        result = pytestconfig.hook.pytest_my_hook(config=pytestconfig)

.. note::
    Hooks receive parameters using only keyword arguments.

Now your hook is ready to be used. To register a function at the hook, other plugins or users must
now simply define the function ``pytest_my_hook`` with the correct signature in their ``conftest.py``.

Example:

.. code-block:: python

    def pytest_my_hook(config):
        """
        Print all active hooks to the screen.
        """
        print(config.hook)


.. _`addoptionhooks`:


Using hooks in pytest_addoption
-------------------------------

Occasionally, it is necessary to change the way in which command line options
are defined by one plugin based on hooks in another plugin. For example,
a plugin may expose a command line option for which another plugin needs
to define the default value. The pluginmanager can be used to install and
use hooks to accomplish this. The plugin would define and add the hooks
and use pytest_addoption as follows:

.. code-block:: python

   # contents of hooks.py

   # Use firstresult=True because we only want one plugin to define this
   # default value
   @hookspec(firstresult=True)
   def pytest_config_file_default_value():
       """ Return the default value for the config file command line option. """


   # contents of myplugin.py


   def pytest_addhooks(pluginmanager):
       """ This example assumes the hooks are grouped in the 'hooks' module. """
       from . import hook

       pluginmanager.add_hookspecs(hook)


   def pytest_addoption(parser, pluginmanager):
       default_value = pluginmanager.hook.pytest_config_file_default_value()
       parser.addoption(
           "--config-file",
           help="Config file to use, defaults to %(default)s",
           default=default_value,
       )

The conftest.py that is using myplugin would simply define the hook as follows:

.. code-block:: python

    def pytest_config_file_default_value():
        return "config.yaml"


Optionally using hooks from 3rd party plugins
---------------------------------------------

Using new hooks from plugins as explained above might be a little tricky
because of the standard :ref:`validation mechanism <validation>`:
if you depend on a plugin that is not installed, validation will fail and
the error message will not make much sense to your users.

One approach is to defer the hook implementation to a new plugin instead of
declaring the hook functions directly in your plugin module, for example:

.. code-block:: python

    # contents of myplugin.py


    class DeferPlugin:
        """Simple plugin to defer pytest-xdist hook functions."""

        def pytest_testnodedown(self, node, error):
            """standard xdist hook function.
            """


    def pytest_configure(config):
        if config.pluginmanager.hasplugin("xdist"):
            config.pluginmanager.register(DeferPlugin())

This has the added benefit of allowing you to conditionally install hooks
depending on which plugins are installed.
