<img src="./doodad.png" width="100%"/>

# amq-sso-demo

Demo of using _Red Hat Single Sign-On 7.5 (Keycloak)_ as an authentication provider for _Red Hat AMQ 7.9_ on OpenShift.

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

Update the Keycloak adapter JSON configuration files, setting the correct URL to your SSO deployment in the field `auth-server-url`. Then, add the files into a ConfigMap, which will be mounted into the AMQ pod:

```
# Edit the 'auth-server-url' attribute in the JSON files first!
oc create configmap artemis-keycloak-config --from-file=artemis-keycloak-config
```

Pull the AMQ image from a private registry, if required:

```
REGISTRY_HOST=registry.mycompany.com

oc create secret docker-registry my-private-registry \
    --docker-username USER --docker-password PASSWORD --docker-server ${REGISTRY_HOST}

oc secrets link default my-private-registry --for=pull
```

Deploy AMQ using a slightly fudged AMQ 7.8 template (note that this method is **deprecated** and will be unsupported in future!):

```
oc process -f amq-broker-78-custom-modified.yaml \
  -p AMQ_REQUIRE_LOGIN="true" \
  -p AMQ_USER=admin -p AMQ_PASSWORD=cheesecake \
  -p IMAGE=${REGISTRY_HOST}/amq7/amq-broker:7.8-31 \
  | oc apply -f -

oc set env dc/broker-amq JAVA_ARGS="-Dhawtio.rolePrincipalClasses=org.apache.activemq.artemis.spi.core.security.jaas.RolePrincipal -Dhawtio.keycloakEnabled=true -Dhawtio.keycloakClientConfig=/home/jboss/broker/etc/rhsso-js-client.json -Dhawtio.authenticationEnabled=true -Dhawtio.realm=console"

# Remove this so that the readiness probe succeeds (otherwise it will fail due to the redirect to Keycloak)
oc set probe dc/broker-amq --remove --readiness
```

Note that the role(s) required to access Hawtio is set using the `HAWTIO_ROLE` property in `artemis.profile`. This is then translated into the `hawtio.role` property in the Artemis startup script (`/home/jboss/broker/bin/artemis`). [See docs][hawtio].

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

If all is well, the test client should output something like this:

```
Message sent
Operation completed in 306ms (excluding connect)
```

...and you might also see a line like this in the Artemis logs, which is logged when the Keycloak adapter in Artemis makes a connection to Keycloak:

```
2021-09-29 07:39:31,095 INFO  [org.keycloak.adapters.KeycloakDeployment] Loaded URLs from https://sso-toms-sso-demo.apps.shared.openshift.example.com/auth/realms/mycorp-amq-sso/.well-known/openid-configuration
```

## Debugging

Not sure what's going on at all. Need help? Use the included `util/logging.properties` file to see what's going on in the broker:

```
oc create configmap artemis-logging-properties --from-file=LOGGING_PROPERTIES=util/logging.properties
oc set env --from=configmap/artemis-logging-properties dc/broker-amq
```

## Troubleshooting/Issues

Some previous issues and solutions, in case it's useful for you.

Can't authenticate to Artemis, nothing in the Keycloak logs, and nothing in the Artemis logs except _"Unable to validate user from"_:

- Have you set the `auth-server-url` correctly in _rhsso-direct-access.json_?
- If the URL is incorrect, you won't see a log in Artemis.

_"Error: Realm does not exist"_ in the Artemis logs:

- The realm wasn't imported into SSO properly. Make sure that the JSON file exists and that the `SSO_IMPORT_FILE` env var is set.

_"invalid_grant"_ in the Artemis logs, and _"User_not_found"_ in the Keycloak logs:

- This happens when the user does not exist in SSO/Keycloak.
- Create the user in the Realm using the Keycloak web UI.

How do I get broker to pick up updated ConfigMap values?

- Scale the broker deployment down and then back up - e.g. `oc scale dc/broker-amq --replicas=0 && oc scale dc/broker-amq --replicas=1`

---

_Banner created with [Pattern Generator][pattern]_

[a]: https://github.com/fmtn/a
[hawtio]: https://hawt.io/docs/configuration/
[pattern]: https://doodad.dev/pattern-generator/
