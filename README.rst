=======================
azure-pipeline-template
=======================

+---------------+----------------------------------------------------------------------+
| **General**   | |maintenance| |license|                                              |
+---------------+----------------------------------------------------------------------+
| **Pipeline**  | |azure_pipeline|                                                     |
+---------------+----------------------------------------------------------------------+
| **Tools**     | |tox|                                                                |
+---------------+----------------------------------------------------------------------+
| **VC**        | |vcs| |gpg| |semver| |pre-commit|                                    |
+---------------+----------------------------------------------------------------------+
| **Github**    | |gh_release| |gh_commits_since| |gh_last_commit|                     |
|               +----------------------------------------------------------------------+
|               | |gh_stars| |gh_forks| |gh_contributors| |gh_watchers|                |
+---------------+----------------------------------------------------------------------+


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

*Added in version 0.1.0.*

**Logic**

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


**Steps**

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


**Parameters**

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


**Example**

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


`publish-pypi-poetry.yml`
-------------------------

*Added in version 0.2.0.*

**Logic**

This job template will use `poetry <https://python-poetry.org/>`_ to build
and publish the Python package (both sdist and wheel) to PyPI or a custom
repository.


**Parameters**

The following parameters can be set at root level:

- ``python_version``: Python version to use (default: 3.8).
- ``dependsOn``: List of jobs this job should depend on.
- ``custom_repository``: Boolean for using a custom repository over PyPI
  (default: false)


**Pipeline variables**

For this job to work credentials for the target repository are needed. They
are served via Pipline Variables, which you have to set in the pipelines
Web-UI settings
(`see here for help. <https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=classic%2Cbatch#set-variables-in-pipeline>`_).

If you want to publish to PyPI (*which is the default*) you have to set either:

- ``POETRY_PYPI_TOKEN_PYPI`` as a **secret variable**

or 

- ``POETRY_HTTP_BASIC_PYPI_USERNAME`` as a **non-secret variable** and
- ``POETRY_HTTP_BASIC_PYPI_PASSWORD`` as a **secret variable**

If you want to publish to a custom repository you have to set:

- ``POETRY_REPOSITORIES_CUSTOM_URL`` as a **non-secret variable**

and for the credentials you have to set (similar to PyPI) either:

- ``POETRY_PYPI_TOKEN_CUSTOM`` as a **secret variable**

or 

- ``POETRY_HTTP_BASIC_CUSTOM_USERNAME`` as a **non-secret variable** and
- ``POETRY_HTTP_BASIC_CUSTOM_PASSWORD`` as a **secret variable**


**NOTE**
Currently there are issues with the token variables not being recognized by
poetry as is should. As a workaround for `PyPI <https://pypi.org/>`_ and
`TestPyPI <https://test.pypi.org/>`_ you can set the username to ``__token__``
and the password to the token including the ``pypi-`` at the beginning.


**Example**

This example builds and publishes the package to PyPI.org after the jobs
``report_coverage``, ``pre_commit`` and ``docs`` ran successfully.

.. code-block:: yaml

    - ${{ if startsWith(variables['Build.SourceBranch'], 'refs/tags/') }}:
      - template: jobs/publish-pypi-poetry.yml@cielquan
        parameters:
          dependsOn: [report_coverage, pre_commit, docs]


Mentions
========

Inspired by:

- https://github.com/tox-dev/azure-pipelines-template
- https://github.com/asottile/azure-pipeline-templates


Disclaimer
==========

No active maintenance is intended for this project.
You may leave an issue if you have a questions, bug report or feature request,
but I cannot promise a quick response time.


.. .############################### LINKS ###############################


.. General

.. |maintenance| image:: https://img.shields.io/badge/No%20Maintenance%20Intended-X-red.svg?style=flat-square
    :target: http://unmaintained.tech/
    :alt: Maintenance - not intended

.. |license| image:: https://img.shields.io/github/license/Cielquan/azure-pipelines-template.svg?style=flat-square&label=License
    :alt: License
    :target: https://github.com/Cielquan/azure-pipelines-template/blob/master/LICENSE.rst

.. |black| image:: https://img.shields.io/badge/Code%20Style-black-000000.svg?style=flat-square
    :alt: Code Style - Black
    :target: https://github.com/psf/black


.. Pipeline

.. |azure_pipeline| image:: https://img.shields.io/azure-devops/build/cielquan/a333a3f3-daef-4f27-a8af-c82feeb2df36/4?style=flat-square&logo=azure-pipelines&label=Azure%20Pipelines
    :target: https://dev.azure.com/cielquan/azure-pipelines-template/_build/latest?definitionId=4&branchName=master
    :alt: Azure DevOps builds


.. Tools

.. |poetry| image:: https://img.shields.io/badge/Packaging-poetry-brightgreen.svg?style=flat-square
    :target: https://python-poetry.org/
    :alt: Poetry

.. |tox| image:: https://img.shields.io/badge/Automation-tox-brightgreen.svg?style=flat-square
    :target: https://tox.readthedocs.io/en/latest/
    :alt: tox

.. |pytest| image:: https://img.shields.io/badge/Test%20framework-pytest-brightgreen.svg?style=flat-square
    :target: https://docs.pytest.org/en/latest/
    :alt: Pytest


.. VC

.. |vcs| image:: https://img.shields.io/badge/VCS-git-orange.svg?style=flat-square&logo=git
    :target: https://git-scm.com/
    :alt: VCS

.. |gpg| image:: https://img.shields.io/badge/GPG-signed-blue.svg?style=flat-square&logo=gnu-privacy-guard
    :target: https://gnupg.org/
    :alt: Website

.. |semver| image:: https://img.shields.io/badge/Versioning-semantic-brightgreen.svg?style=flat-square
    :alt: Versioning - semantic
    :target: https://semver.org/

.. |pre-commit| image:: https://img.shields.io/badge/pre--commit-enabled-brightgreen?style=flat-square&logo=pre-commit&logoColor=yellow
    :target: https://github.com/pre-commit/pre-commit
    :alt: pre-commit


.. Github

.. |gh_release| image:: https://img.shields.io/github/v/release/Cielquan/azure-pipelines-template.svg?style=flat-square&logo=github
    :alt: Github - Latest Release
    :target: https://github.com/Cielquan/azure-pipelines-template/releases/latest

.. |gh_commits_since| image:: https://img.shields.io/github/commits-since/Cielquan/azure-pipelines-template/latest.svg?style=flat-square&logo=github
    :alt: Github - Commits since latest release
    :target: https://github.com/Cielquan/azure-pipelines-template/commits/master

.. |gh_last_commit| image:: https://img.shields.io/github/last-commit/Cielquan/azure-pipelines-template.svg?style=flat-square&logo=github
    :alt: Github - Last Commit
    :target: https://github.com/Cielquan/azure-pipelines-template/commits/master

.. |gh_stars| image:: https://img.shields.io/github/stars/Cielquan/azure-pipelines-template.svg?style=flat-square&logo=github
    :alt: Github - Stars
    :target: https://github.com/Cielquan/azure-pipelines-template/stargazers

.. |gh_forks| image:: https://img.shields.io/github/forks/Cielquan/azure-pipelines-template.svg?style=flat-square&logo=github
    :alt: Github - Forks
    :target: https://github.com/Cielquan/azure-pipelines-template/network/members

.. |gh_contributors| image:: https://img.shields.io/github/contributors/Cielquan/azure-pipelines-template.svg?style=flat-square&logo=github
    :alt: Github - Contributors
    :target: https://github.com/Cielquan/azure-pipelines-template/graphs/contributors

.. |gh_watchers| image:: https://img.shields.io/github/watchers/Cielquan/azure-pipelines-template.svg?style=flat-square&logo=github
    :alt: Github - Watchers
    :target: https://github.com/Cielquan/azure-pipelines-template/watchers
