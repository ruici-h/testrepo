
Stephen Davies
Red Hat
23 March 2022

Several methods have been used for OCP authorization configuration.

The first is found under basic. This simply hard codes a Group and
CRB for cluster-admins. This is no longer in use but has been left
here for documentation.

The ldap configuration was tried but due to the complexity of the
Computershare LDAP tree it presented challenges and the decision
was made to try using the group-sync-operator. Resources in the
ldap directory are of historical interest but are not currently
in use.

The reources under group-sync-operator are currently in use.
