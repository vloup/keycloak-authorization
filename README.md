# Keycloak authorization module

The purpose of this module is to add authorization capabilities to Keycloak for a given client, whether the client
itself has the capability to handle authorization or not. This means that:

* Any type of Keycloak client can have an authorization layered on top of authentication, not just OIDC clients.
* The authorization only works as long as Keycloak is in the path of requests to a resource.

While the best practice is to have the client handle authorisation tasks, if necessary with the aid of an external service,
many clients do not have this capability. This is why IDPs such as Microsoft's Azure AD offer this service, and why we
provide this module for Keycloak.

It should be noted that the authorization step happens after authentication, so a user which is connected in SSO will
not need re-input his login details to when switching between clients he has access to, and clients which he doesn't
have access to.

Currently working on 4.8.3.Final (check tags for compatibility with previous Keycloak versions)

## How to install
This is an example with Keycloak available at /opt/keycloak

```Bash
#Create layer in Keycloak setup
install -d -v -m755 /opt/keycloak/modules/system/layers/authorization -o keycloak -g keycloak

#Setup the module directory
install -d -v -m755 /opt/keycloak/modules/system/layers/authorization/io/cloudtrust/keycloak-authorization/main/ -o keycloak -g keycloak

#Install jar
install -v -m0755 -o keycloak -g keycloak -D target/keycloak-authorization-4.8.3.Final.jar /opt/keycloak/modules/system/layers/authorization/io/cloudtrust/keycloak-authorization/main/

#Install module file
install -v -m0755 -o keycloak -g keycloak -D module.xml /opt/keycloak/modules/system/layers/authorization/io/cloudtrust/keycloak-authorization/main/

```

### Enable module and load theme

__layers.conf__

```Bash
layers=keycloak,wsfed,authorization
```

__standalone.xml__

```xml
<web-context>auth</web-context>
<providers>
    <provider>module:io.cloudtrust.keycloak-authorization</provider>
    ...
</providers>
...
<theme>
    <modules>
            <module>
                    io.cloudtrust.keycloak-authorization
            </module>
    </modules>
    ...
</theme>
...
```

In Keycloak, the admin theme must then be set to `authorization` in the master realm and in the realm where the local
authorization is used, and Keycloak must be restarted (This step is optional if you are only going to use OIDC clients).

## How to use
1) Open Keycloak's administration console
1) Go the client's settings page you want to configure
1) Enable Authorization (and save)
1) In the Authorization tab (visible only when authorization is enabled):
    1) In the `Resources` sub-tab, create a resource with name `Keycloak Client Resource` and all the other fields empty
    1) In the `Policies` sub-tab, create a policy of you liking (ex. Role Policy requiring a certain realm-role)
    1) In the `Permissions` sub-tab, create a `resource-based` permission using the `Keycloak Client Resource` created before, and the Policy

Your client is now protected with the specified policy.


## How this module works

This module is unfortunately very strongly linked to Keycloak's code. This is due to the fact that we are rewriting the
behaviour of Keyclock's protocol classes without actually modifying the Keycloak code. The way this works is as follows:

1) We declare in the LoginProtocolFactory services' file that we are declaring the factories for Keycloak's protocols.
As we have the base class as a dependency, this means that our module's declaration comes after Keycloak's own. Since
Keycloak uses a map to store this information, we overwrite the dependency
1) We use a local copy the relevant Keycloak classes that we need to work with for each protocol:
    * **The login protocol factory** This ensures that when the factory is created, it will call local code first, due to
    wildfly's classloader priority order (local classes are called before dependency classes). In this case, it includes
    the creation of the endpoints and of the login protocol.
    * **The login protocol** This is only class we actually modify. In the authenticated method we invoke the code to
    verify that the client is authorised to access the client it wishes to connect to.
    * For OIDC we need some extra classes to account for the different manners of obtaining a token (**TokenEndpoint**
    and **OIDCLoginProtocolService**)
1) We use a theme to add to the administrator pages the option to enable authorisation for all clients.

We call Keycloak's own existing authorisation methods and framework for a user's authorisation. This is done in the
methods of the class io.cloudtrust.keycloak.protocol.LocalAuthorizationService.

## How to update the code when Keycloak changes version number

When Keycloak changes version number, we must replace all classes (login protocol factory, login
protocol, extra OIDC classes) of the module with their new version from Keycloak and the corresponding Keycloak WS-FED
module. A call to the LocalAuthorisationService's methods must be then added in the login protocol's `authenticated` method.

The LocalAuthorisationService must only be updated if the functions called change signature. The same is true for the
tests. For these steps it is necessary to understand how the Keycloak classes and authorization functions work.

## Removing the dependency on WS-FED

This module is designed to work with our WS-FED protocol module. However, to remove this dependency simply:
1) Remove the package `com.quest.keycloak.protocol.wsfed` in the main and test directories
1) Remove the dependency to `keycloak-wsfed` in the `pom.xml`
1) Remove the line `<module name="com.quest.keycloak-wsfed"/>` in the module.xml
1) Build the module and deploy it as you would otherwise, the only difference is that there will not be the `wsfed`
entry in the `layers.conf` file.

## The class loader

This part explains how we override existing classes in Keycloak (to add the client check inside the existing flows)
- Keycloak uses JBoss/Wildfly Module class loader (a class loader per module)
- To load each class (ex: `org.keycloak.protocol.oidc.endpoints.TokenEndpoint`) The method `Module.loadClassModule` is called.
- It retrieves all the loaders where the path(ex: `org/keycloak/protocol/oidc/endpoints`) of the class can be found.
- The paths and their respective loaders are stored in a map: `Map<String, List<LocalLoader>>`.
- This map is populated when loading the module (`ModuleLoader.loadModule`), when calling `module.relinkIfNecessary()`
- a `Linkage` is then created with all the paths by iterating over all the dependencies defined in the unlinked `Linkage`
- These dependencies are instanciated when first defining the module in `ModuleLoader.defineModule` by calling `module.setDependencies` with the `moduleSpec.dependencies`
- Finally, moduleSpec.dependencies is instanciated using the ModuleXmlParser which add a local dependency on the module before adding its dependencies

On the other hand when hot deploying modules (ex: arquillian) in the `ModuleSpecProcessor.createModuleService`,
`this.createDependencies(specBuilder, dependencies, false);` is called before `specBuilder.addDependency(DependencySpec.createLocalDependencySpec());`

## Tests
For now (Until the class Loader is fixed), to test the module we need to deploy it manually on a local Keycloak, and run the tests without using Arquillian.
The integration tests do the following:
We import a realm with:
- user1 having the role test-role and user2 without it.
- a resource without URI called "Keycloak Client Resource" (this is important)
- a Role policy
- a permission linking the resource with policy

For each protocol: OIDC, SAML, WSFED
- we check that user1 can retrieve an access token
- we check that user2 can't
