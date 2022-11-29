## Adding managed cluster
The yamls in this directory are one method to connect a managed cluster to a hub cluster.  (The import process from the hub provides these yamls as a base64 encoded string to decode and apply on the managed cluster)
**NOTE:** The kubeconfig file example here has the certs removed, no secrets are contained here.
This is just provided as an example, It doesn't seem feasible to add managed clusters via kustomize, as the hub cluster creates objects (a namespace of the same name as the importing cluster) with service accounts, secrets, etc... (more to come, taking inventory of all these things being added).
