.. _config:

Hammer IR and Meta Variables
============================

Hammer IR
---------

Hammer IR is the primary standardized data exchange format of Hammer. Hammer IR standardizes physical design constraints such as placement constraints and clock constraints. In addition, the Hammer IR also standardizes communication among and to Hammer plugins, including tool control (e.g. loading tools, etc) and configuration options (e.g. number of CPUs).

The hammer-config library
-------------------------

The `hammer-config library <https://github.com/ucb-bar/hammer/tree/master/hammer/config>`_ is the part of Hammer responsible for parsing Hammer YAML/JSON configuration files into Hammer IR. Hammer IR is used for the standardization and interchange of data between the different parts of Hammer and Hammer plugins.

.. note::
   There is a built-in order of precedence, from lowest to highest:

    #) Hammer's ``defaults.yml``
    #) Tool plugin's ``defaults.yml``
    #) Tech plugin's ``defaults.yml``
    #) User's Hammer IR (using ``-p`` on the command line)

    JSON/YAML files specified with ``-p`` later on the command line have higher precedence and keys appearing later if duplicated in a given file also take precedence. In the examples below, "Level #" will be used to denote the level of precedence of the configuration snippet.

The ``get_setting()`` method is available to all Hammer technology/tool plugins and hooks (see :ref:`hooks`).

Basics
------

.. code-block:: yaml

    foo:
        bar:
            adc: "yes"
            dac: "no"

The basic idea of Hammer IR in YAML/JSON format is centered around a hierarchically nested tree of YAML/JSON dictionaries. For example, the above YAML snippet is translated to two variables which can be queried in code - ``foo.bar.adc`` would have ``yes`` and ``foo.bar.dac`` would have ``no``.

Overriding
----------

Hammer IR snippets frequently "override" each other. For example, a technology plugin might provide some defaults which a specific project can override with a project YAML snippet.

For example, if the base snippet contains ``foo: 12345`` and a snippet in a file of higher precedence contains ``foo: 54321``, then ``get_setting("foo")`` would return ``54321``.

Meta actions
--------------

Sometimes it is desirable that variables are not completely overwritten, but instead modified.

For example, say that the technology plugin provides:

.. code-block:: yaml

    vlsi.tech.foobar65.bad_cells: ["NAND4X", "NOR4X"]

And let's say that in our particular project, we find it undesirable to use the ``NAND2X`` and ``NOR2X`` cells. However, if we simply put the following in our project YAML, the references to ``NAND4X`` and ``NOR4X`` disappear and we don't want to have to copy the information from the base plugin, which may change, or which may be proprietary, etc.

.. code-block:: yaml

    vlsi.tech.foobar65.bad_cells: ["NAND2X", "NOR2X"]

The solution is **meta variables**. This lets Hammer's config parser know that instead of simply replacing the base variable, it should do a particular special action. Any config variable can have ``_meta`` suffixed into a new variable with the desired meta action.

In this example, we can use the ``append`` meta action:

.. code-block:: yaml

    vlsi.tech.foobar65.bad_cells: ["NAND2X", "NOR2X"]
    vlsi.tech.foobar65.bad_cells_meta: append

This will yield the desired result of ``["NAND4X", "NOR4X", "NAND2X", "NOR2X"]`` when ``get_setting("vlsi.tech.foobar65.bad_cells")`` is called in the end.

Applying multiple meta actions
------------------------------

Multiple meta actions can be applied sequentially if the ``_meta`` variable is an array. Example:

In Level 1:

.. code-block:: yaml

    foo.flash: yes

In Level 2 (located at /opt/foo):

.. code-block:: yaml

    foo.pipeline: "CELL_${foo.flash}.lef"
    foo.pipeline_meta: ['subst', 'prependlocal']

Result: ``get_setting("foo.pipeline")`` returns ``/opt/foo/CELL_yes.lef``.

Common meta actions
-------------------

* ``append``: append the elements provided to the base list. (See the above ``vlsi.tech.foobar65.bad_cells`` example.)
* ``subst``: substitute variables into a string.

  Base:

  .. code-block:: yaml

    foo.flash: yes

  Meta:

  .. code-block:: yaml

    foo.pipeline: "${foo.flash}man"
    foo.pipeline_meta: subst

  Result: ``get_setting("foo.flash")`` returns ``yesman``

* ``lazysubst``: by default, variables are only substituted from previous configs. Using ``lazysubst`` allows us to defer the substitution until the very end.

  Example without ``lazysubst``:

  Level 1:

  .. code-block:: yaml

    foo.flash: yes

  Level 2:

  .. code-block:: yaml

    foo.pipeline: "${foo.flash}man"
    foo.pipeline_meta: subst

  Level 3:

  .. code-block:: yaml

    foo.flash: no

  Result: ``get_setting("foo.flash")`` returns ``yesman``

  Example with ``lazysubst``:

  Level 1:

  .. code-block:: yaml

    foo.flash: yes

  Level 2:

  .. code-block:: yaml

    foo.pipeline: "${foo.flash}man"
    foo.pipeline_meta: lazysubst

  Level 3:

  .. code-block:: yaml
    
    foo.flash: no

  Result: ``get_setting("foo.flash")`` returns ``noman``

* ``crossref`` - directly reference another setting. Example:

  Level 1:

  .. code-block:: yaml

    foo.flash: yes

  Level 2:

  .. code-block:: yaml

    foo.mob: "foo.flash"
    foo.mob_meta: crossref

  Result: ``get_setting("foo.mob")`` returns ``yes``

* ``transclude`` - transclude the given path. Example:

  Level 1:

  .. code-block:: yaml

    foo.bar: "/opt/foo/myfile.txt"
    foo.bar_meta: transclude

  Result: ``get_setting("foo.bar")`` returns ``<contents of /opt/foo/myfile.txt>``

* ``prependlocal`` - prepend the local path or package resource directory of this config file. Example:

  Level 1 (located at /opt/foo):

  .. code-block:: yaml

    foo.bar: "myfile.txt"
    foo.bar_meta: prependlocal

  Result: ``get_setting("foo.mob")`` returns ``/opt/foo/myfile.txt``

* ``deepsubst`` - like ``subst`` but descends into sub-elements. Example:

  Level 1:

  .. code-block:: yaml

    foo.bar: "123"

  Level 2:

  .. code-block:: yaml

    foo.bar:
      baz: "${foo.bar}45"
      quux: "32${foo.bar}"
    foo.bar_meta: deepsubst

  Result: ``get_setting("foo.bar.baz")`` returns ``12345`` and ``get_setting("foo.bar.baz")`` returns ``32123``

Type Checking
-------------

Any existing configuration file can and should be accompanied with a corresponding configuration types file.
This allows for static type checking of any key when calling ``get_setting``.
The file should contain the same keys as the corresponding configuration file, but can contain the following as values:

- primitive types (``int``, ``str``, etc.)
- collection types (``list``)
- collections of key-value pairs (``list[dict[str, str]]``, ``list[dict[str, list]]``, etc.) These values are turned into custom constraints (e.g. ``PlacementConstraint``, ``PinAssignment``) later in the Hammer workflow, but the key value pairs are not type-checked any deeper.
- optional forms of the above (``Optional[str]``)
- the wildcard ``Any`` type

Hammer will perform the same without a types file, but it is highly recommended to ensure type safety of any future plugins.

Key History
-----------

With the ``ruamel.yaml`` package, Hammer can emit what files have modified any configuration keys in YAML format.
Turning on key history is accomplished with the ``--dump-history`` command-line flag.
The file is named ``{action}-output-history.yml`` and is located in the output folder of the given action.

Example with the file ``test-config.yml``:

  .. code-block:: yaml

    synthesis.inputs:
        input_files: ["foo", "bar"]
        top_module: "z1top.xdc"

    vlsi:
        core:
            technology: "hammer.technology.nop"

            synthesis_tool: "hammer.synthesis.nop"

``test/syn-rundir/syn-output-history.yml`` after executing the command ``hammer-vlsi --dump-history -p test-config.yml --obj_dir test syn``:

  .. code-block:: yaml

    synthesis.inputs.input_files:  # Modified by: test-config.yml
      - LICENSE
      - README.md
    synthesis.inputs.top_module: z1top.xdc # Modified by: test-config.yml

    vlsi.core.technology: nop # Modified by: test-config.yml
    vlsi.core.synthesis_tool: hammer.synthesis.nop # Modified by: test-config.yml

Example with the files ``test-config.yml`` and ``test-config2.yml``, respectively:

  .. code-block:: yaml

    synthesis.inputs:
        input_files: ["foo", "bar"]
        top_module: "z1top.xdc"

    vlsi:
        core:
            technology: "hammer.technology.nop"

            synthesis_tool: "hammer.synthesis.nop"

  .. code-block:: yaml

    par.inputs:
        input_files: ["foo", "bar"]
        top_module: "z1top.xdc"

    vlsi:
        core:
            technology: "${foo.subst}"

            par_tool: "hammer.par.nop"

    foo.subst: "hammer.technology.nop2"

``test/syn-rundir/par-output-history.yml`` after executing the command ``hammer-vlsi --dump-history -p test-config.yml -p test-config2.yml --obj_dir test syn-par``:

  .. code-block:: yaml

    foo.subst: hammer.technology.nop2 # Modified by: test-config2.yml
    par.inputs.input_files:  # Modified by: test-config2.yml
      - foo
      - bar
    par.inputs.top_module: z1top.xdc # Modified by: test-config2.yml
    synthesis.inputs.input_files:  # Modified by: test-config.yml
      - foo
      - bar
    synthesis.inputs.top_module: z1top.xdc # Modified by: test-config.yml
    vlsi.core.technology: hammer.technology.nop2 # Modified by: test-config.yml, test-config2.yml
    vlsi.core.synthesis_tool: hammer.synthesis.nop # Modified by: test-config.yml
    vlsi.core.par_tool: hammer.par.nop # Modified by: test-config2.yml

Key Description Lookup
----------------------

With the ``ruamel.yaml`` package, Hammer can execute the ``info`` action, allowing users to look up the description of most keys.
The comments must be structured like so in order to be read properly:

  .. code-block:: yaml

    foo: bar # this is a comment 
    # this is another comment for the key "foo"

Hammer will take the descriptions from any ``defaults.yml`` files.

  .. code-block::yaml

    foo.bar:
      apple: banana # type of fruit
      price: 2 # price of fruit

Running ``hammer-vlsi -p test-config.yml info`` (assuming the above configuration is in ``defaults.yml``):

  .. code-block::

    foo.bar
    Select from the current level of keys: foo.bar
    
    apple
    price
    Select from the current level of keys: apple
    ----------------------------------------
    Key: foo.bar.apple
    Value: banana
    Description: # type of fruit
    History: ["defaults.yml"]
    ----------------------------------------

    Continue querying keys? [y/n]: y

    foo.bar
    Select from the current level of keys: foo.bar
    
    apple
    price
    Select from the current level of keys: price
    ----------------------------------------
    Key: foo.bar.price
    Value: 2
    Description: # price of fruit
    History: ["defaults.yml"]
    ----------------------------------------

Keys are queried post-resolution of all meta actions, so their values correspond to the project configuration after other actions like ``syn`` or ``par``.

Reference
---------

For a more comprehensive view, please consult the ``hammer.config`` API documentation in its implementation here:

* https://github.com/ucb-bar/hammer/blob/master/hammer/config/config_src.py
* https://github.com/ucb-bar/hammer/blob/master/tests/test_config.py

In ``config_src.py``, most supported meta actions are contained in the ``directives`` list of the ``get_meta_directives`` method.
