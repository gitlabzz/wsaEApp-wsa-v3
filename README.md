# wsaEApp-wsa-v3

## When running Maven from commandline:

- export MAVEN_OPTS='-Xms64m -Xmx256m -Dlog4j.configurationFile=src/main/resources/log4j/log4j2.xml'
- export AXWAY_APIM_CLI_HOME=src/main/environments/dev
- export APIM_HOST=dev.apim.demo.bct
- export APIM_ADMIN_PASSWORD=changeme
- export APIM_ADMIN_USER=apiadmin

## Test connectivity for the apim-cli:

mvn clean install exec:java@list-api

## Publish to 'DEV' Environment:

### used in api-config.json file to refer to environment specific downstream service:

- export BACKEND_PROTOCOL=http
- export BACKEND_HOST=soapui
- export BACKEND_PORT=8080

### to use env.properties file for 'dev' environment:

export ENV=dev

### Publish to 'DEV'

mvn clean install exec:java@import-api


