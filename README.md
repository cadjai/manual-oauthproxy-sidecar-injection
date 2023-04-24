# manual-oauth2-proxy-sidecar-injection

This repository provides quick instructions on how to manually inject oauth-proxy sidecar into a pod or a deployment. There are plenty of ways to do this but this is to show a tenant on a shared multitenant OpenShift (or kubernetes) cluster who needs to enable authentication for their deployed application how they can leverage oauthproxy to make things easier and faster for themselves.  

For instance if the requirements to authenticate (authorization can be done in addition to this) is similar to what the platform requires (e.g. using the enterprise identity management solution laready integrated into the cluster), the tenant does not need to do an extensive and cumbersome integration with enterprise LDAP or other enterprise identity management to get this accomplished. The quickest way here would be to leverage the OpenShift oauthproxy, which already has the information and could be used and also make the process easier and simpler.

For more complex use cases where the application needs either to use a different enterprise identity provider or more attributes from the configurated provider than what the oauthproxy server in OpenShift is willing or able to share, the applicaiton can be configured to use the platform configured keycloak (or RHSSO) to achieve this. Again here the application could use a keycloak oauthproxy to get this information once the proper provider is made available by the platform. 

This configuration simplifies maintenance and enables sharing of cross cutting identity management concerns via the platform administrators instead of having each tenant standup their own integration to the entreprise identity solution. In addition this approach saves costs since instead of having several instances of keycloak deployed we are now using one central that is shared and leveraging sidecars, which are far less resource intensive .

The oauthproxy sidecar solution is the best, efficient, faster and secure way to provide enterprise level identity to the applications. To properly do this at scale, shared multitenant kubernetes platforms like OpenShift use various automated ways to inject the sidecars into the various applications that need it. One of the ways to do this is to use a mutatingwebhook. So through the use of a policy engine one can easily manage this oauthproxy isidecar injection into the various applications that needs it. 

When a policy engine is not readily available and tenants are not able to deploy mutatingwebhooks, they can still leverage the solution via self service approach that allows them to meet their enterprise authentication integration needs.  

To achieve this, we will use Ansible and Jinja2 templating to enable provisioning and injecting of the sidecar configuration into the designated application. Each tenant should then be able to clone the provided playbook and run it to generate and patch their application(s) to augment its(their) deployment with the oautproxy sidecar.

We will cover two oauth-proxy providers: keycloak and openshift.

For more information about oauth2proxy refer to the [oauth2 proxy github page](https://github.com/oauth2-proxy/oauth2-proxy) and the [provider configuration page](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider/).

For more information about the keycloak oauth-proxy available configurations, refer to [keycloak provider page](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider/#keycloak-auth-provider).

For more information about the openshift oauth2-proxy provider refer to the [openshift oauth proxy github page](https://github.com/openshift/oauth-proxy) and the [openshift oauth proxy configuration page](https://github.com/openshift/oauth-proxy/tree/aad1b28fcf4b32d9ad592eee33e439bd575565a2#command-line-options).



## Injecting oauth2-proxy container inside the  pod
To inject the oauth proxy sidecar into a pod or a deployment specification, run the provided playbook with the appropriate variables set for the type of oauthprxy you want injected. 



Play Variables
--------------

For the keycloak oauth2-proxy the following variables are used:
Note the folowing prefixes being used for the variables.
Any variable prefixed with `keycloak_` is related to the keycloak server we are using as our oauth proxy provider.
Any variable prefixed with `app_` is related to the application that we are injecting the sidecar into.
Any variable prefixed with `koa2p_` is related to the keycloak oauth2-proxy provider configuration. You can read more about all the [configuration options](https://github.com/oauth2-proxy/oauth2-proxy) 
- keycloak_realm_name: The realm within keycloak where the client was created. 
- keycloak_secret: The secret generated for the client that is being used for this oauth2-proxy setup.  
- keycloak_openid_issuer_url: The base URL for the keycloak server we are connecting to. If the keycloak is running on OpenShift, the URL will look like https://<keycloak-app-name>-<keycloak-app-namespace>.apps.<CLUSTERNAME>.<BASEDOMAIN>.
- koa2p_login_url: The URI for the keycloak openid connect for the client. It is composed of the keycloak_openid_issuer_url and the following suffix /auth/realms/<keycloak_realm_name>/protocol/openid-connect/auth.
- koa2p_redeem_url: The URI for the keycloak openid token for the client. It is composed of the keycloak_openid_issuer_url and th following suffix /auth/realms/<keycloak_realm_name>/protocol/openid-connect/token.
- koa2p_redirect_url: The URI for the keycloak openid provider redirect for the client. It is composed of the keycloak_openid_issuer_url and th following suffix /auth/realms/<keycloak_realm_name>/protocol/openid-connect/userinfo.
- koa2p_validate_url: The URI for the keycloak openid userinfo validation for the client. It is composed of the keycloak_openid_issuer_url and th following suffix /auth/realms/<keycloak_realm_name>/protocol/openid-connect/userinfo.
- koa2p_profile_url: The URI for the keycloak openid profile validation for the client. It is composed of the keycloak_openid_issuer_url and th following suffix /auth/realms/<keycloak_realm_name>/protocol/openid-connect/userinfo.
- keycloak_email_domain: The Email domain managed by the keycloak server. If not set the default is * .
- koa2p_cookie_secret: The base64 encoded random cookie secret to use for the client . Can be generated using the following python command `python2.7 -c 'import os,base64; print base64.urlsafe_b64encode(os.urandom(16))'` . Defaults to 0dSBdCNa2k1oOPl3eviZ0g== when not set
- koa2p_provider_button: Denotes whether to turn on the provider button or not. Default to true when not set .
- koa2p_http_address: The http port and address the oauth2-proxy sidecar listen on. Default to 0.0.0.0:8080 when not set . 
- koa2p_http_port: The http port the oauth2-proxy sidecar listen on. Default to 8080 when not set . 
- koa2p_upstream_address:  The http port the application the oauth2-proxy sidecar proxies to listen on. The http port the oauth2-proxy sidecar listen on. Default to 8080 when not set . This is the main port your application listen on.
- koa2p_image: The keycloak oauth2-proxy image used by the sidecar. Defaults to quay.io/oauth2-proxy/oauth2-proxy:latest when not set. 
- app_name: The name of the application the sidecar is being injected into.
- app_namespace: The name of the namespace or project the application is deployed in OpenShift.
- app_route: The name of route of the application the sidecar is being injected into.
- app_deploy_name: The name of the deployment the oauth2-proxy sidecar is being injected into.


For the openshift oauth2-proxy the following variables are used:
Note the folowing prefixes being used for the variables.
Any variable prefixed with `ocp_` is related to the OpenShift Oauth server we are using as our oauth proxy provider.
Any variable prefixed with `app_` is related to the application that we are injecting the sidecar into.
Any variable prefixed with `ocpoa2p_` is related to the openshift oauth2-proxy provider configuration. You can read more about all the [configuration options](https://github.com/openshift/oauth-proxy/tree/aad1b28fcf4b32d9ad592eee33e439bd575565a2#command-line-options).
- ocpoa2p_redeem_url: The URL for the openshift oauthproxy token redemption for the client. 
- ocpoa2p_review_url: The URL for the openshift subject access review.
- ocpoa2p_delegate_urls: The json definition for the openshift resources the authenticated user is allowed to access. Default to `get` `namespaces` defined as a json object like below
```
{"/":{ "resource":"namespaces",
   "verb":"get"
}}
```
- ocpoa2p_email_domain: The Email domain the openshift oauth-proxy is allowed to authenticate with . If not set the default is * .
- ocpoa2p_cookie_secret_name: The name of the cookie secret to use for the openshift oauth2proxy sidecar. The secret is automatically generated by the playbook. 
- ocpoa2p_provider_button: Denotes whether to turn on the provider button or not. Default to true when not set .
- ocpoa2p_http_address: The http port and address the oauth2-proxy sidecar listen on. Default to 0.0.0.0:4000 when not set . 
- ocpoa2p_http_port: The http port the oauth2-proxy sidecar listen on. Default to 4000 when not set . 
- ocpoa2p_upstream_address:  The http port the application the oauth2-proxy sidecar proxies to listen on. The http port the oauth2-proxy sidecar listen on. Default to 127.0.0.1:8080 when not set . This is the main port your application listen on.
- ocpoa2p_sar_url: The json definition for the openshift ressouces the authenticated user is allowed to review .
- ocpoa2p_redirect_url: The oauth redirect URL 
- ocpoa2p_validate_url: The access token validation URL.
- ocpoa2p_profile_url: The profile access endpoint URL. 
- ocpoa2p_scopes: The space separated list of scopes defined as resource:access. Defaults to  "user:info user:check-access user:list-projects" when not set. 
- ocpoa2p_service_account: The name of the service account used to run the sidecar. Defaults to <app_name>-sa when not set. 
- ocpoa2p_ca: The mount path for the pem tls cert within the sidecar. Defaults to /etc/pki/tls/cert.pem when not set . This value is used to set the -openshift-ca option within the sidecar and can be used to point to several files like done with the following.
- ocpoa2p_ca: The mount path for the pem tls ca cert within the sidecar. Defaults to /var/run/secrets/kubernetes.io/serviceaccount/ca.crt when not set. This value is used to set the -openshift-ca option within the sidecar and can be used to point to several files like done with the following.
- ocpoa2p_ca: The mount path for the pem tls cert within the sidecar. Defaults to /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem when not set . This value is used to set the -openshift-ca option within the sidecar and can be used to point to several files like done above.
- ocpoa2p_client_secret_file: The mount path for the client secret file within the sidecar. Defaults to /var/run/secrets/kubernetes.io/serviceaccount/token when not set . 
- ocpoa2p_image: The openshift oauth2-proxy image used by the sidecar. Defaults to registry.redhat.io/openshift4/ose-oauth-proxy@sha256:cb915bd0111e1cc1417a214fa6e013e3af9e1f6d9c4bcaeb71d8c84775a7f071 when not set. 
- app_namespace: The name of the namespace or project the application is deployed in OpenShift.
- app_name: The name of the application the oauth2-proxy is proxying traffic to in OpenShift.
- app_route: The name of route of the application the sidecar is being injected into.
- app_deploy_name: The name of the deployment the oauth2-proxy sidecar is being injected into.


The following variables are used for the cluster in either case:
- ocp_cluster_user: The username of the cluster-admin user to use to run this playbook.
- ocp_cluster_user_password: The password associated with the username above.
- ocp_cluster_console_url: The URL to the API for the cluster this will be deployed to.
- ocp_cluster_console_port: The API port for the cluster this will be deployed to. Default to 6443 when not set.
- ocp_cluster_user_token: The API token to use instead of the the user password if provided.
- openshift_cli: The path to the OpenShift client binary used to run the commands use to intereact with the cluster. It could be the path to kubectl or oc. Default to oc when not set assuming that the client on on your path.
- image_pullsecret_json: The encrypted value of the docker config json for the Red Hat registry where the images used are downloaded from. This is basically the base64 encrypted of runninga command like `podman login -u username -p password registry.redhat.io --authfile ~/.docker/json.

There are few other optional variables that you can change to your liking but don't have to change. 


Dependencies
------------

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.


Installation and Usage
-----------------------
Clone the repository to where you want to run this from and make sure you can connect to your cluster using the CLI . 
You will also need to have cluster-admin run in order to run the playbook since the cron job need to run privileged.
Finally before running the playbook make sure to set and update the variables as appropriate to your use case.

Playbooks
---------
To run the main playbook use the ansible-playbook command as follows   
`ansible-playbook inject-oauth2-proxy-into-deployment.yml  --vault-id @prompt -vvv`  
The above playbook creates the oauth2-proxy config and inject it into a deployment or pod based on the information provided. It can handle oauth proxy for a keycloak provider or for an openshift provider.  



License
-------

BSD

Author Information
------------------

