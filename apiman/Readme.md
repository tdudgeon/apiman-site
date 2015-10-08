docker run -it --link apimansite_postgres_1:postgres -e POSTGRES_DATABASE=keycloak -e POSTGRES_USER=keycloak -e POSTGRES_PASSWORD=keycloak --rm -v $PWD:/tmp/json apimansite_keycloak /opt/jboss/keycloak/bin/standalone.sh -b 0.0.0.0 -Dkeycloak.migration.action=import -Dkeycloak.migration.provider=singleFile -Dkeycloak.migration.file=/tmp/json/apiman-realm.json -Dkeycloak.migration.strategy=OVERWRITE_EXISTING


/opt/jboss/wildfly/standalone/configuration/standalone-apiman.xml


{
  "realm": "apiman",
  "realm-public-key": "MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCrVrCuTtArbgaZzL1hvh0xtL5mc7o0NqPVnYXkLvgcwiC3BjLGw1tGEGoJaXDuSaRllobm53JBhjx33UNv+5z/UMG4kytBWxheNVKnL6GgqlNabMaFfPLPCF8kAgKnsi79NMo+n6KnSY8YeUmec/p2vjO2NjsSAVcWEQMVhJ31LwIDAQAB",
  "auth-server-url": "http://192.168.59.103:9080/auth",
  "ssl-required": "external",
  "resource": "apimanui",
  "credentials": {
    "secret": "password"
  }
}


https://192.168.59.103:8443/apimanui/*