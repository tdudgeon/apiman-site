FROM jboss/wildfly:8.2.0.Final
MAINTAINER Tim Dudgeon <tdudgeon@informaticsmatters.com>

ENV APIMAN_VERSION 1.1.8.Final

RUN cd $JBOSS_HOME \
 && curl http://downloads.jboss.org/overlord/apiman/$APIMAN_VERSION/apiman-distro-wildfly8-$APIMAN_VERSION-overlay.zip -o apiman-distro-wildfly8-$APIMAN_VERSION-overlay.zip \
 && bsdtar -xf apiman-distro-wildfly8-$APIMAN_VERSION-overlay.zip \
 && rm apiman-distro-wildfly8-$APIMAN_VERSION-overlay.zip

RUN /opt/jboss/wildfly/bin/add-user.sh admin admin --silent

COPY postgresql-9.4-1203-jdbc41.jar /opt/jboss/wildfly/standalone/deployments/
COPY apiman-ds.xml /opt/jboss/wildfly/standalone/deployments/
COPY apiman.properties /opt/jboss/wildfly/standalone/configuration/
COPY standalone-apiman.xml /opt/jboss/wildfly/standalone/configuration/
COPY keystore.jks /opt/jboss/wildfly/standalone/configuration/

# remove elasticsearch as we're using separate elasticsearch container
RUN rm /opt/jboss/wildfly/standalone/deployments/apiman-es.war

# optional - add the demo echo service. remove if you don't want this
# see instructions if you need to build a different version
COPY apiman-quickstarts-echo-service-$APIMAN_VERSION.war /opt/jboss/wildfly/standalone/deployments/

VOLUME /opt/jboss/wildfly/standalone/data

CMD ["/opt/jboss/wildfly/bin/standalone.sh", "-b", "0.0.0.0", "-bmanagement", "0.0.0.0", "-c", "standalone-apiman.xml"]
