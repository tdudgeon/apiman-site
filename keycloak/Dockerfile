FROM jboss/keycloak-postgres
MAINTAINER Tim Dudgeon <tdudgeon@informaticsmatters.com>

#COPY standalone-nossl.xml keycloak/standalone/configuration/

COPY standalone-ssl.xml keycloak/standalone/configuration/standalone.xml
COPY keystore.jks keycloak/standalone/configuration/keystore.jks
#COPY keycloak-server.json /opt/jboss/keycloak/standalone/configuration/keycloak-server.json

EXPOSE 8443
