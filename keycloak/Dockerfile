FROM jboss/keycloak-postgres
MAINTAINER Tim Dudgeon <tdudgeon@informaticsmatters.com>

#COPY standalone-nossl.xml keycloak/standalone/configuration/

COPY standalone-ssl.xml keycloak/standalone/configuration/standalone.xml
COPY keystore.jks keycloak/standalone/configuration/keystore.jks
EXPOSE 8443
