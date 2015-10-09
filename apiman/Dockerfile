FROM jboss/wildfly:8.2.0.Final
MAINTAINER Tim Dudgeon <tdudgeon@informaticsmatters.com>

ENV APIMAN_VERSION 1.1.8.Final

RUN cd $JBOSS_HOME \
 && curl http://downloads.jboss.org/overlord/apiman/$APIMAN_VERSION/apiman-distro-wildfly8-$APIMAN_VERSION-overlay.zip -o apiman-distro-wildfly8-$APIMAN_VERSION-overlay.zip \
 && bsdtar -xf apiman-distro-wildfly8-$APIMAN_VERSION-overlay.zip \
 && rm apiman-distro-wildfly8-$APIMAN_VERSION-overlay.zip

COPY postgresql-9.4-1203-jdbc41.jar /opt/jboss/wildfly/standalone/deployments/
COPY apiman-ds.xml /opt/jboss/wildfly/standalone/deployments/
COPY apiman.properties /opt/jboss/wildfly/standalone/configuration/
COPY standalone-apiman.xml /opt/jboss/wildfly/standalone/configuration/
COPY keystore.jks /opt/jboss/wildfly/standalone/configuration/

RUN /opt/jboss/wildfly/bin/add-user.sh admin admin --silent
 


CMD ["/opt/jboss/wildfly/bin/standalone.sh", "-b", "0.0.0.0", "-bmanagement", "0.0.0.0", "-c", "standalone-apiman.xml"]
