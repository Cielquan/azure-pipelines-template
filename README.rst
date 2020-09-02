=======================
azure-pipeline-template
=======================

**First configure a github service connection**

It is suggested to use a generic name, such as `github` so forks can also
configure the same.

You can find this in `Project Settings` => `Service connections` in the Azure
Devops dashboard for your project. Project settings is located in the bottom
left corner of the UI as of 2020-09-02. Below I'm using the endpoint name
`github`.

**Now add this to the beginning of the `azure-pipelines.yml`:**

.. code-block:: yaml

    resources:
      repositories:
        - repository: cielquan
          type: github
          endpoint: github
          name: cielquan/azure-pipelines-template
          ref: refs/master

This will make the templates in this repository available in the `cielquan`
namespace. Note the ref allows you to change the template version you want,
you can use ``refs/master`` if you want latest unstable version or
``refs/tags/<TAG>`` to pin the version to an speciffic ``tag``.


job templates
=============

The job templates can be found in the ``jobs`` directory.


`run-tox.yml`
-------------


Logic
~~~~~

This job template will run ``tox`` for a given set of tox environments on given
os.
Features and functionality:

- each specified toxenv target maps to a single Azure Pipelines job, but split over multiple architectures via the
  image matrix (each matrix will set the `image_name` variable to `macOs`, `linux` or `windows`
  depending on the image used)
- make tox available in the job: provision a python (`3.8`) and install a specified tox into that
- provision a python needed for the target tox environment
- provision the target tox environment (create environment, install dependencies)
- invoke the tox target
- if a junit file is found under `.tox\junit.{toxenv}.xml` upload it as test report
- if coverage is requested, run a tox target that should generate the `.tox\coverage.xml` or `.tox\.coverage`
  and upload those as a build artifact (also enqueue a job after all these job succeed to merge the generated
  coverage reports)
- if coverage was requested queue a job that post all toxenv runs will merge all the coverages via a tox target


Steps
~~~~~

This template will:

#. build a test matrix for the given os and architectures (win only)
#. install a python interpreter for tox only
#. install tox
#. install the test target python version
#. run global custom `setup` scripts
#. run job custom `setup` scripts
#. print PATH
#. print python interpreter info
#. print tox environments
#. generate tox environments
#. run tox environments
#. upload junit file from ``.tox/junit.{toxenv}.xml`` `(if present)`
#. upload coverage artifacts from ``.tox/*`` `(if present)`
#. run job custom `tear down` scripts
#. run global custom `tear down` scripts

After the steps above the template will also add a job to merge the uploaded
coverage artifacts and publish them with ``Cobertura``.


Parameters
~~~~~~~~~~

The following parameters can be set at root level:

- ``dependsOn``: List of jobs every ``tox_env`` should depend on.
  Can be overwritten for single ``tox_env``.
- ``default_python``: Version of python to use when non is detectable or set
  (default: 3.8).
- ``additional_variables``: Mapping of additional variables to set for all
  ``tox_env``. Can be overwritten for single ``tox_env``.
- ``checkout_submodules``: If submodules should also be checked out when the
  repository gets checked out (default: true).
- ``tox_version``: Pin the ``tox`` version to use (default: newest available).
- ``tox_python``: Python version to use for tox (default: 3.8).
- ``pre_test``: List of steps to run before invoking every ``tox_env``.
- ``post_test``: List of steps to run after running every ``tox_env``.
- ``tox_envs``: Mapping of ``tox`` environments (key) and their parameters
  (values) to run:

  - ``display_name``: Name to use for the job instead of the ``tox_env`` name.
    Must be set for ``tox_env`` with a '-' in their name, because dashes are
    not allowed in job names.
  - ``dependsOn``: List of jobs this ``tox_env`` should depend on. Overwrites the
    global ``dependsOn``.
  - ``os``: List of os to run this ``tox_env`` on:

    - ``linux`` / ``lin`` - `ubuntu-latest`
    - ``windows`` / ``win`` - `windows-latest`
    - ``macos`` / ``osx`` - `macOS-latest`
    - not set - fallback to `ubuntu-latest`.

  - ``architectures``: List of architectures to run this ``tox_env`` on:

    - ``x64`` - default
    - ``x86`` - only available for ``windows``.

  - ``py_version``: determines python version to use for this ``tox_env``,
    if not set will be derived from the key or fallback to ``default_python``:

    - ``py36`` or starts with ``py36-`` - Python 3.6
    - ``py37`` or starts with ``py37-`` - Python 3.7
    - ``py38`` or starts with ``py38-`` - Python 3.8
    - ``py39`` or starts with ``py39-`` - Python 3.9 latest pre-release
      (only available on linux -- it is installed from
      `deadsnakes <https://github.com/deadsnakes>`_
    - ``pypy3`` or starts with ``pypy3-`` - PyPy 3

  - ``additional_variables``: Mapping of additional variables to set for this
    ``tox_env``. Overwrites the global ``additional_variables``.
  - ``pre_test``: List of steps to run before this ``tox_env``. Runs after the global
    ``pre_test``.
  - ``post_test``: List of steps to run after this ``tox_env``. Runs before the global
    ``post_test``.

- ``coverage``: List of settings used for coverage processing if set:

  - ``with_toxenv``: Name of the ``tox_env`` to do coverage collecting and
    normalizing with. Runs after every ``tox_env`` in ``for_envs`` and as a
    final job ``report_coverage`` (with the *merge-coverage.yml* template)
    after all ``tox_env`` runs finished to merge the coverage data.
  - ``for_envs``: List of ``tox_env`` to collect coverage data from. Referred
    ``tox_env`` must generate ``.tox/.coverage`` and ``.tox/coverage.xml`` files


Example
~~~~~~~

The following example will run the follwing jobs with ``tox`` version *3.15.0*
called via *python 3.7*:

- ``pre_commit`` on *linux* with *python 3.7*
- ``py38`` on all three os and on windows also on *x86*
- ``py39`` on *linux*
- ``pypy3`` on *linux* and *macos*
- ``docs_test_html`` on *linux* with ``default_python`` version *3.6*
- ``docs_test_linkcheck`` on *linux* with ``default_python`` version *3.6*
- ``report_coverage`` on *linux* with ``default_python`` version *3.6* to
  merge the coverage data generated by ``py38``, ``py39`` and ``pypy3``.

use *python 3.7* to call ``tox`` in version* 3.15.0* for

.. code-block:: yaml

    jobs:
      - template: jobs/run-tox.yml@cielquan
        parameters:
          tox_version: '3.15.0'
          tox_python: '3.7'
          default_python: '3.6'
          tox_envs:
            pre-commit:
              display_name: pre_commit
              py_version: '3.7'
            py38:
              os: [linux, windows, macOs]
              architectures: [x86, x64]
            py39: null
            pypy3:
              os: [linux, macOs]
            docs-test-html:
              display_name: docs_test_html
            docs-test-linkcheck:
              display_name: docs_test_linkcheck
          coverage:
            with_toxenv: 'coverage'
            for_envs: [py38, py39, pypy3]


Mentions
========

Inspired by:

- https://github.com/tox-dev/azure-pipelines-template
- https://github.com/asottile/azure-pipeline-templates
