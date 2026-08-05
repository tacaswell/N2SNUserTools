===============
N2SN User Tools
===============

NSLS-II N2SN User Manipulation Tools for Python


Configuration
*************

The tools read a YAML configuration file from either
``~/.config/n2sn_tools.yml`` or ``/etc/n2sn_tools.yml``. Each instrument
defines a ``roles`` mapping of role name to the Active Directory group that
implements that role::

    instruments:
      xpd:
        name: XPD
        roles:
          staff: n2sn-inststaff-xpd
          user: n2sn-instusers-xpd

These Active Directory groups contain users directly and are *role* groups in
the NSLS-II RBAC model.

.. note::

   The ``roles`` key was previously named ``rights``. The ``rights`` key is
   deprecated and still accepted for backward compatibility (with a
   ``DeprecationWarning``), but support will be removed in a future release.
   Update configuration files to use ``roles``.


Reporting issues
****************

Please report issues to the NSLS-II internal issue tracker at
https://jira.nsls2.bnl.gov/projects/N2SNUT.
