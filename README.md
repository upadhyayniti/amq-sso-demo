# amq-sso-demo

Deploying Red Hat AMQ 7.9 with Red Hat Single Sign-On 7.5 (Keycloak) as an authentication provider.

## Deploy Red Hat Single Sign-On 7.5

Install the image stream and templates:

```
oc apply -f https://raw.githubusercontent.com/jboss-container-images/redhat-sso-7-openshift-image/sso75-cpaas-dev/templates/sso75-image-stream.json

oc apply -f https://raw.githubusercontent.com/jboss-container-images/redhat-sso-7-openshift-image/sso75-cpaas-dev/templates/sso75-x509-https.json
```

Create a new app, giving an explicit username and password, and telling OpenShift to look in the _current_ project for the SSO image:

```
oc new-app sso75-x509-https \
    -p SSO_ADMIN_USERNAME=rhssotest -p SSO_ADMIN_PASSWORD=L00kAr0undY0u \
    -p IMAGE_STREAM_NAMESPACE=$(oc project -q)
```

If your default requests and limits in your shared OpenShift cluster are pitifully small, you might need to set these too, to make the health check a little less severe, and succeed:

```
oc set resources dc/sso --requests=cpu=500m,memory=1Gi --limits=cpu=1,memory=1Gi

oc set probe dc/sso --liveness --initial-delay-seconds=180
```

Add the realm JSON definition to a ConfigMap, mount it in the SSO pod, and tell SSO to import it on startup:

```
oc create configmap keycloak-broker-realm --from-file=keycloak-broker-realm.json

oc set volume dc/sso --add --overwrite --name=realm-config --mount-path=/etc/realm/broker --type=configmap --configmap-name=keycloak-broker-realm

oc set env dc/sso SSO_IMPORT_FILE=/etc/realm/broker/keycloak-broker-realm.json
```

**Now** log on to SSO Admin console using _rhssotest_ user, and add a user into the realm. Set a password, and ensure the user is granted an appropriate role from the list.

## Deploy AMQ 7.9 from Template

Pull the AMQ image from a private registry, if required:

```
REGISTRY_HOST=registry.mycompany.com

oc create secret docker-registry my-private-registry \
    --docker-username USER --docker-password PASSWORD --docker-server ${REGISTRY_HOST}

oc secrets link default my-private-registry --for=pull
```

Deploy AMQ using a slightly fudged AMQ 7.8 template (note that this method is **deprecated** and will be unsupported in future!):

```
oc create configmap artemis-keycloak-config --from-file=artemis-keycloak-config

oc process -f amq-broker-78-custom-modified.yaml \
  -p AMQ_REQUIRE_LOGIN="true" \
  -p AMQ_USER=admin -p AMQ_PASSWORD=cheesecake \
  -p IMAGE=${REGISTRY_HOST}/amq-broker-7/amq-broker-79-openshift-rhel8:7.9-10 \
  | oc apply -f -

oc set env dc/broker-amq JAVA_ARGS="-Dhawtio.rolePrincipalClasses=org.apache.activemq.artemis.spi.core.security.jaas.RolePrincipal -Dhawtio.keycloakEnabled=true -Dhawtio.keycloakClientConfig=/home/jboss/broker/etc/rhsso-js-client.json -Dhawtio.authenticationEnabled=true -Dhawtio.realm=console -Dhawtio.roles=myapp_admin"
```

## Testing it with fmtn/a

[fmtn/a][a] is a nice little command-line ActiveMQ testing utility.

Set up a Docker Hub secret if necessary:

```
oc create secret docker-registry docker-hub \
    --docker-server docker.io --docker-username USER --docker-password PASS

oc secrets link default docker-hub --for=pull
```

Then run the _a_ image and try to put a message onto a queue:

```
oc run -i -t a --image=fmtn/a-util:1.5.0 --restart=Never -- sh

java -jar /a/a.jar --artemis-core --broker tcp://broker-amq-tcp:61616 --user bobby --pass password0 --put "my message" sandwiches.queue
```

## Debugging

Not sure what's going on at all. Need help? Use the included `util/logging.properties` file to see what's going on in the broker:

```
oc create configmap artemis-logging-properties --from-file=LOGGING_PROPERTIES=util/logging.properties
oc set env --from=configmap/artemis-logging-properties dc/broker-amq
```

## Troubleshooting/Issues

Some previous issues:

_"Error: Realm does not exist"_ in the Artemis logs:

- The realm wasn't imported into SSO properly. Make sure that the JSON file exists and that the `SSO_IMPORT_FILE` env var is set.

_"invalid_grant"_ in the Artemis logs, and _"User_not_found"_ in the Keycloak logs:

- This happens when the user does not exist in SSO/Keycloak.
- Create the user in the Realm using the Keycloak web UI.



[a]: https://github.com/fmtn/a
