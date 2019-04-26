Initialize and install software
===============================

Initialize your Spack environment with pre-defined environments.
"Environments" are stored as ``conf/xxx.env`` files with settings for ``SPACK_BIN_ROOT``, ``SPACK`` 

::

  $ export PLATFORM=sandybridge; ./install --env env/pi2-system-bootstrap.yaml

then edit ``env/pi2-system-sb.yaml``  and install system software

:: 

  $ export PLATFORM=sandybridge; ./install --env env/pi2-system-sb.yaml

Sync cached source packages to mirror
=====================================

 $ ./install -e system --sync

Reference
=========

* "Spack Configuration Files" https://spack.readthedocs.io/en/latest/configuration.html
