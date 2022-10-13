# Sumo Alerts Notes

Stephen Davies<br/>
Red Hat<br/>
1 June 2022

## Overview

Sumo alerts are configured in two places. Firstly as an ACM
Application (this directory). The second part is found
in managed-subscriptions/policies.
See policy-configure-alertmanager.yaml.

The reason for this is that it was easier to use template
functions in a ConfigurationPolicy to customise the URL
settings required by alert-manager config.

The Bitnami Sealed Secrets operator is used to conceal the
sensitive portions of the Sumo Alerts API.
