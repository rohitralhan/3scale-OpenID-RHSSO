
# Enhancing API Security with Red Hat 3scale & Red Hat SSO (Keycloak)

Many companies have adopted APIs to ensure seamless interoperability between their internal systems and to extend their business and integrations with partner ecosystems. A critical factor in the successful adoption and evolution of an API platform is selecting the right tools to meet specific business needs, particularly when choosing an API management platform. Another key consideration is how to expose these APIs securely.

This article will look at how to securely expose APIs using the Red Hat 3scale API Management Platform and Red Hat Single Sign On (keycloak).

## Introduction
Red Hat [3scale](https://developers.redhat.com/products/3scale/overview) (3Scale) API Management simplifies API management for both internal and external users. It enables you to share, secure, distribute, control, and monetize your APIs on a high-performance infrastructure platform designed for customer control and future scalability.

Red Hat [Single Sign-On](https://access.redhat.com/products/red-hat-single-sign-on) ([RH-SSO](https://access.redhat.com/products/red-hat-single-sign-on)/[Keycloak](https://www.keycloak.org/)) is an open-source identity and access management solution built on Keycloak, offering enterprise-level security and scalability for modern applications.


## In this article
 - Configuring Red Hat SSO (Keycloak)
 - Configure Red Hat 3Scale
 - Configure API as a Product
	 - Setting up an API Backend in 3Scale
	 - Setting up an API Product in 3Scale 

## Prerequisite
 - Access to Red Hat OpenShift Container Platform
 - Red Hat SSO operator deployed
 - Red Hat 3Scale operator deployed
 - **oc** CLI Access

## References
 - [Red Hat 3scale API Management 2.14](https://docs.redhat.com/en/documentation/red_hat_3scale_api_management/2.14/)
 - [Red Hat Single Sign-On 7.6](https://docs.redhat.com/en/documentation/red_hat_single_sign-on/7.6)

## Configure Red Hat SSO (Keycloak)
In this section, we will configure Red Hat SSO
 - Setup a Security Realm
 - Setup a Client for 3Scale to sync with Red Hat SSO

1. Use the below resource definition to **Create a Keycloak instance**

   ```yml
   kind: Keycloak
   apiVersion: keycloak.org/v1alpha1
   metadata:
    name: keycloak-sso
    labels:
      app: sso
   spec:
    instances: 1
    externalAccess:
      enabled: true
   ```

2. Get the Red Hat SSO **URL** and **login user** and **password**

   ```
    oc get route keycloak -n <<keycloak namespace>> -o jsonpath={.spec.host}

    oc get secret credential-example-keycloak -n <<keycloak namespace>> -o json | jq -r .data.ADMIN_USERNAME | base64 -d

    oc get secret credential-example-keycloak -n <<keycloak namespace>> -o json | jq -r .data.ADMIN_PASSWORD | base64 -d
   ```
   
3. Login into Red Hat SSO using the above URL and login details, on the top left, as shown in the screenshot hover over the realm name to bring up the **Add Realm** menu.

[![Add Realm](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/add-realm.png?raw=true)](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/add-realm.png?raw=true)

[![Add Realm](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/realm.png?raw=true)](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/realm.png?raw=true)


4. Next, we will configure the client used by 3Scale to sync with Red Hat SSO as shown in the screenshot below

[![Add Realm](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/zync-client.png?raw=true)](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/zync-client.png?raw=true)

[![Add Realm](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/zync-client-sa.png?raw=true)](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/zync-client-sa.png?raw=true)

[![Add Realm](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/zync-client-creds.png?raw=true)](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/zync-client-creds.png?raw=true)

That's all that needs to happen in Red Hat SSO. Let's configure 3Scale now.

## Configure Red Hat 3Scale
In this section, we will configure Red Hat 3Scale. Below is the sequence of steps for setting up API as a Product. We will us 
 1. Create a Backend
 2. Create a Product
 3. Create a Developer Account and Developer User
 4. Create an Application

 - Use the below resource definition to create the API Backend

```yml
apiVersion: capabilities.3scale.net/v1beta1
kind: Backend
metadata:
  name: backend-sb-app
spec:
  name: "backend app"
  systemName: backend-sb-app
  privateBaseURL: "http://<<host>>:<<port>>/"
```
- Use the below resource definition to create the API Product
```yaml
apiVersion: capabilities.3scale.net/v1beta1
kind: Product
metadata:
  name: product-sb-app
spec:
  name: "product-sb-app"
  systemName: "product-sb-app"
  applicationPlans:
    plan01:
      name: "Plan1"
      setupFee: "10"
      appsRequireApproval: false
      published: true
    plan02:
      name: "Plan2"
      trialPeriod: 3
      costMonth: "5"
      appsRequireApproval: false
      published: true
  mappingRules:
    - httpMethod: GET
      pattern: "/"
      increment: 1
      metricMethodRef: hits      
  backendUsages:
    backend-sb-app:
      path: /
  deployment:
    apicastHosted:
      authentication:
        oidc:
          issuerType: "keycloak"
          issuerEndpoint: "https://<<client id>:<<client secret>>@<<keycloak URL>>/auth/realms/<<realm name>>"
          authenticationFlow:
            standardFlowEnabled: true
            implicitFlowEnabled: false
            serviceAccountsEnabled: true
            directAccessGrantsEnabled: false
          jwtClaimWithClientID: "azp"
          jwtClaimWithClientIDType: "plain"
          credentials: "headers"
```

- Use the below resource definition to create the API Developer Account, Developer User and Password Secret
```yml
apiVersion: capabilities.3scale.net/v1beta1
kind: DeveloperAccount
metadata:
  name: api-consumer
spec:
  orgName: Devcorp
```
```yml
apiVersion: v1
kind: Secret
metadata:
  name: user1
stringData:
  password: "User123"
```
```yml
apiVersion: capabilities.3scale.net/v1beta1
kind: DeveloperUser
metadata:
  name: developer-user
spec:
  username: user1
  email: user1@example.com
  passwordCredentialsRef:
    name: user1
  role: admin
  developerAccountRef:
    name: api-consumer
```

- Use the below resource definition to create the Application.
```yml
kind: Application
apiVersion: capabilities.3scale.net/v1beta1
metadata:
  name: application-sb-app
spec:
  accountCR:
    name: api-consumer
  productCR:
    name: product-sb-app
  applicationPlanName: plan01
  description: 'SB Demo App'
  name: sb-demo-app
  suspend: false
```

Once the application is created, it will create a corresponding SSO Client in Red Hat SSO. **Make a note** of the **Client ID** and **Client Secret** from  **Dashboard --> Audience --> Applications --> Listings --> API Credentials**

[![Add Realm](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/audience.png?raw=true)](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/audience.png?raw=true)

The above resource definitions will create the various objects in 3Scale these can be validated from the 3Scale Dashboard page as well as from the Red Hat OpenShift Console.

[![Add Realm](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/3scale-dash.png?raw=true)](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/3scale-dash.png?raw=true)

[![Add Realm](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/ocp-all-insts.png?raw=true)](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/ocp-all-insts.png?raw=true)


## Testing the API

Finally we will use Postman (you could also use curl) to test out API(s) for this we will need the **Client Id** and **Client Secret** along with the **Red Hat SSO (Keycloak) URL** from the above step after creating the Application and update them in the Postman screen as shown below.

[![Add Realm](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/postman-get-token.png?raw=true)](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/postman-get-token.png?raw=true)


[![Add Realm](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/postman-send-request.png?raw=true)](https://github.com/rohitralhan/3scale-OpenID-RHSSO/blob/main/images/postman-send-request.png?raw=true)
