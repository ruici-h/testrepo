## Observations
- Importing a cluster creates a namespace $CLUSTERNAME on both the hub and the managed cluster.  i.e., if you name the cluster "sb06", then a namespace "sb06" is created on both clusters with all kinds of goodies that are documented here in INVENTORY-HUB.md and INVENTORY-MANAGED.md



## MISC stuff
### Get inventory of all objects
```bash
kubectl api-resources --verbs=list -o name \
  | xargs -n 1 kubectl get \
  --show-kind \
  --ignore-not-found \
  --all-namespaces \
  -o=custom-columns=NAMESPACE:.metadata.namespace,KIND:.kind,NAME:.metadata.name,VERISION:.metadata.resourceVersion,CREATED:.metadata.creationTimestamp \
  --no-headers >$TMPDIR/state1.txt
```
### Comparing objects before and after
```bash
cat $TMPDIR/state1.txt|sort |egrep -v 'openshift-operator-lifecycle-manager|ldap-group-sync| Event '|awk '{print $1,$2,$3}' >$TMPDIR/tmp1
cat $TMPDIR/state2.txt|sort |egrep -v 'openshift-operator-lifecycle-manager|ldap-group-sync| Event '|awk '{print $1,$2,$3}' >$TMPDIR/tmp2
comm -13 $TMPDIR/tmp1 $TMPDIR/tmp2
```
