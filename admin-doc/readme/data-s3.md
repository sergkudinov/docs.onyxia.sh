---
description: Enable S3 storage via MinIO S3
layout:
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: false
---

# 🗃 Data (S3)

### S3 Storage

Onyxia uses [AWS Security Token Service API](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html) to optain S3 tokens on behaf of your users. We support any S3 storage compatible with this API. In this context, we are using [MinIO](https://min.io/), which is compatible with the Amazon S3 storage service and we demonstrate how to integrate it with Keycloak.

#### Create a Keycloak client for Accessing Keycloak

Before configuring MinIO, let's create a new client for Keycloak (from the previous existing "datalab" realm).

Create a client called "minio".

1. _Client ID_: **minio**
2. _Client Protocol_: **openid-connect**
3. _Root URL_: **https://minio.lab.my-domain.net/**

Complete the content of client "minio" with the following values.

1. _Access Type_: **confidential**
2. _Valid Redirect URIs_ (two values are required): **https://minio.lab.my-domain.net/\*** and **https://minio-console.lab.my-domain.net/\***
3. _Web origins_: **\***

Save the content, a new tab called Credentials must be appear. Navigate to Credentials tab and **copy the secret value for the next section (**`KEYCLOAK_MINIO_CLIENT_SECRET`**)**.

Navigate to Mappers tab and create a protocol Mapper.

1. _Name_: **policy**
2. _Mapper Type_: **Hardcoded claim**

Complete the content of Mapper "policy" with the following values.

1. _Token Claim Name_: **policy**
2. _Claim value_: **stsonly**
3. _Add to ID token_: **on**
4. _Add to access token_: **on**
5. _Add to userinfo_: **on**

#### Install MinIO

We recommand you to follow [MinIO documentation](https://min.io/docs/minio/linux/administration/identity-access-management/oidc-access-management.html#minio-external-identity-management-openid) for this installation and you must activate OIDC authentification. We will use the official Helm in this tutorial. All Helm configuration values can be found within this [link](https://github.com/minio/minio/blob/master/helm/minio/values.yaml).

```bash
helm repo add minio https://charts.min.io/
 
DOMAIN=my-domain.net
# Replace xxxxx by the secret value defined 
# into the "minio" Keycloak client (see previous section)
KEYCLOAK_MINIO_CLIENT_SECRET=xxxxxxxx
# Make sure this match with what you have specified as oidc.username-claim
# In your onyxia-api configuration.  
# See: https://github.com/InseeFrLab/onyxia-api?tab=readme-ov-file#authentication-configuration
# If you haven't specified this parameter, leave preferred_username, it's
# the Keycloak's default.  
# You have also to make sure that the onyxia-minio client that you will
# create later on has a preffered_username claim in it's OIDC Access Tocken JWT
# (and that the value of this claim matches the Onyxia username of course).  
# As long as your onyxia and onyxia-minio client shares the same Keycloak realm
# it will be the case.  
OIDC_USERNAME_CLAIM=preferred_username

cat << EOF > ./minio-values.yaml
## replicas: 16
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
  path: /
  hosts:
    - minio.lab.$DOMAIN
  tls:
    - hosts:
        - minio.lab.$DOMAIN
consoleIngress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
  paths: /
  hosts:
    - minio-console.lab.$DOMAIN
  tls:
    - hosts:
        - minio-console.lab.$DOMAIN
environment:
  MINIO_BROWSER_REDIRECT_URL: https://minio-console.lab.$DOMAIN
oidc:
  enabled: true
  configUrl: "https://auth.lab.$DOMAIN/auth/realms/datalab/.well-known/openid-configuration"
  clientId: "minio"
  claimName: "policy"
  scopes: "openid,profile,email"
  redirectUri: "https://minio-console.lab.$DOMAIN/oauth_callback"
  claimPrefix: ""
  comment: ""
  clientSecret: $KEYCLOAK_MINIO_CLIENT_SECRET
policies:
  - name: stsonly
    statements:
      - resources:
          - 'arn:aws:s3:::oidc-${jwt:$OIDC_USERNAME_CLAIM}'
          - 'arn:aws:s3:::oidc-${jwt:$OIDC_USERNAME_CLAIM}/*'
        actions:
          - "s3:*"
EOF

helm install minio minio/minio -f minio-values.yaml
```

MinIO is now deployed and is accessible on the console url.

> By default, there are 16 MinIO containers running. If this number is too large for your Kubernetes cluster, you can limit it by configuring the 'replicas' key.

#### Create a Keycloak Client for Onyxia/MinIO

Before configuring the onyxia region to create tokens we should go back to Keycloak and create a new client to enable onyxia-web to request token for MinIO. This client is a little bit more complexe than other if you want to manage durations (here 7 days) and this client should have a claim name policy and with a value of stsonly according to our last deployment of MinIO.

From "datalab" realm, create a client called "onyxia-minio"

1. _Client ID_: **onyxia-minio**
2. _Client Protocol_: **openid-connect**
3. _Root URL_: **https://onyxia.my-domain.net/**

Complete the content of client "onyxia-minio" with the following values.

1. _Access Type_: **public**
2. _Valid Redirect URIs_: **https://onyxia.my-domain.net/\***
3. _Web origins_: **\***
4. In the **Advanced** tab:\
   1\. Access Token Lifespan : **7 days**\
   2\. Client Session Idle : **7 days**\
   3\. Client Session Max: **7 days**

Save the content and navigate to Mappers tab and create two protocol Mappers.

Create the first Mapper called "policy".

1. _Token Name_: **policy**
2. _Mapper Type_: **Hardcoded claim**
3. _Token Claim Name_: **policy**
4. _Claim value_: **stsonly**
5. _Add to ID token_: **on**
6. _Add to access token_: **on**
7. _Add to userinfo_: **on**

Create the second Mapper called "audience-minio".

1. _Token Name_: **audience-minio**
2. _Mapper Type_: **Audience**
3. \_Included Custom Audience \_: **minio**
4. _Add to ID token_: **on**
5. _Add to access token_: **on**

#### Update Onyxia

S3 storage is configured inside a region in Onyxia api. You have some options to configure this storage and  inform Onyxia web all needed informations. \
\
This configuration have two differents section ; one for configuring the storage directory of the users and one for configuring if Onyxia can dynamically create tokens with an STS endpoint.

When configuring the workingDirectory, there are two modes : 'multi' where each user has their own buckets, and 'shared' where users just have a subpath inside a specific bucket.

You should look all options for the version of your need on [github](https://github.com/InseeFrLab/onyxia-api/blob/master/docs/region-configuration.md#s3)

{% code title="onyxia-values.yaml" %}
```diff
serviceAccount:
  clusterAdmin: true
 ingress:
   enabled: true
   annotations:
     kubernetes.io/ingress.class: nginx
   hosts:
     - host: onyxia.my-domain.net
 web:
  env:
    KEYCLOAK_REALM: datalab
    KEYCLOAK_URL: https://auth.lab.my-domain.net/auth
    TERMS_OF_SERVICES: |
      { "en": "https://www.sspcloud.fr/tos_en.md", "fr": "https://www.sspcloud.fr/tos_fr.md" }
 api:
   env:
    authentication.mode: openidconnect
    keycloak.realm: datalab
    keycloak.auth-server-url: https://auth.lab.my-domain.net/auth
   regions:
     [
        {
           "id":"demo",
           "name":"Demo",
           "description":"This is a demo region, feel free to try Onyxia !",
           "services":{
              "type":"KUBERNETES",
              "singleNamespace": false,
              "namespacePrefix":"user-",
              "usernamePrefix":"oidc-",
              "groupNamespacePrefix":"projet-",
              "groupPrefix":"oidc-",
              "authenticationMode":"admin",
              "expose":{
                 "domain":"lab.my-domain.net"
              },
              "monitoring":{
                 "URLPattern":"todo"
              },
              "cloudshell":{
                 "catalogId":"inseefrlab-helm-charts-datascience",
                 "packageName":"cloudshell"
              },
              "initScript":"https://inseefrlab.github.io/onyxia/onyxia-init.sh"
           },
+          "data": {
+                "S3" : {
+                  "URL": "https://minio.lab.my-domain.net",
+                  "pathStyleAccess": true,
+                  "sts": {
+                      "durationSeconds": 86400,
+                      "oidcConfiguration":
+                      {
+                        "issuerURI": "https://auth.my-domain.net/auth/realms/datalab",
+                        "clientID": "onyxia-minio",
+                      },
+                  },
+                  "workingDirectory": {
+                      "bucketMode": "multi",
+                      "bucketNamePrefix": "user-",
+                      "bucketNamePrefixGroup": "projet-",
+                  },
+                },
+            },
           "auth":{
              "type":"openidconnect"
           },
           "location":{
              "lat":48.8164,
              "long":2.3174,
              "name":"Montrouge (France)"
           }
        }
     ]
```
{% endcode %}

```bash
helm upgrade onyxia inseefrlab/onyxia -f onyxia-values.yaml
```

Congratulation, all the S3 related features of Onyxia are now enabled in your instance! &#x20;

Next step in the installation process is to setup Vault to provide a way to your user so store secret and also to provide something that Onyxia can use as a persistance layer for user configurations. &#x20;

{% content-ref url="...rest.md" %}
[...rest.md](...rest.md)
{% endcontent-ref %}
