# Alfresco Solr Communication Modes

This project includes sample Docker compose deployments for Alfresco Repository and Alfresco Search Services (SOLR). Note that *Shared Secret in HTTP Header* `secret` mode is only available from ACS 7.1.

* `http` communication between Repository and SOLR happens using plain HTTP protocol with no authentication. Repository REST API Endpoints and SOLR Endpoints must be protected.
* `mtls` communication between Repository and SOLR happens using Mutual TLS protocol with digital certificate authentication. Repository and SOLR must be configured in order to use coordinated `truststores` and `keystores`.
* `secret` communication between Repository and SOLR happens using plain HTTP protocol with a Shared Secret in HTTP Header.

## Communication flow between Repository and SOLR

Alfresco Repository is running inside *Apache Tomcat* server, while SOLR is running inside *Jetty* server. Both servers are managing incoming HTTP requests.

Additionally, Alfresco Repository and Search Services are using *Java HttpClient* library to perform outgoing HTTP requests to Apache Tomcat (Repository) and Jetty (SOLR).

**Searching**

Alfresco Repository is sending an HTTP(s) Requests to SOLR with the search query.

* Alfresco Repository must include communication settings in `alfresco-global.properties` or Java Environment variables
* SOLR must include communication settings in Jetty (`solr.in.sh`) or Java Environment variables

**Indexing**

Search Services is sending HTTP(s) Requests to Alfresco Repository with the tracking/indexing request.

* Alfresco Repository must include communication settings in `server.xml` configuration for Tomcat
* Search Services must include communication settings in `solrcore.properties` (for each core: `alfresco` and `archive`)

**Secret encryption**

Note that *Repository Metadata Encryption* is not related with Alfresco-SOLR communication. This is why every Docker Compose template (http, mtls and secret) include the same settings for this feature in `alfresco` service:

```
encryption.keystore.type=JCEKS
encryption.cipherAlgorithm=DESede/CBC/PKCS5Padding
encryption.keyAlgorithm=DESede
encryption.keystore.location=/usr/local/tomcat/shared/classes/alfresco/extension/keystore/keystore
metadata-keystore.password=mp6yc0UD9e
metadata-keystore.aliases=metadata
metadata-keystore.metadata.password=oKIWzVdEdA
metadata-keystore.metadata.algorithm=DESede
```

>> When using `mtls` for Alfresco-SOLR communication, related properties are prefixed with           `alfresco.encryption.ssl.*` in Repository


## HTTP

Sample Docker Compose template is available in [http](http) folder.

**Alfresco Repository**

Recommended settings for `alfresco-global.properties` (or Java Environment variables for Docker Containers)

```
solr.host=solr6
solr.port=8983
solr.secureComms=none
```

>> Default `server.xml` Tomcat configuration works as expected, since there is a Connector listening to port 8080

**Search Services**

Recommended settings for `solrcore.properties` (remember to add this settings to both cores `alfresco` and `archive`)

```
alfresco.host=alfresco
alfresco.port=8080
alfresco.secureComms=none
```

>> Default Jetty configuration works as expected, since there is a Listener in port 8983

**Unauthorized accesses**

Since this protocol doesn't include authentication by default, NGINX Web Proxy is configured to prevent external accesses to Alfresco Repository and Search Services.

Sample configuration is available in [http/config](http/config) folder.

```
# No access to Alfresco Repository SOLR API is allowed when using the Web Proxy
# Communication will happen inside the Docker Compose network (private)
location ~ ^(/.*/service/api/solr/.*)$ {return 403;}
location ~ ^(/.*/s/api/solr/.*)$ {return 403;}
location ~ ^(/.*/wcservice/api/solr/.*)$ {return 403;}
location ~ ^(/.*/wcs/api/solr/.*)$ {return 403;}
location ~ ^(/.*/proxy/alfresco/api/solr/.*)$ {return 403 ;}
location ~ ^(/.*/-default-/proxy/alfresco/api/.*)$ {return 403;}

# Access to SOLR is protected by user/password when using the Web Proxy
# Communication will happen inside the Docker Compose network (private)
location /solr/ {
  proxy_pass http://solr6:8983;
  auth_basic "Solr web console";
  auth_basic_user_file /etc/nginx/conf.d/nginx.htpasswd;
}
```


## MTLS

Sample Docker Compose template is available in [mtls](mtls) folder.

Sample *Keystore* and *Truststore* files are available in [mtls/keystores/alfresco](mtls/keystores/alfresco) folder for Alfresco Repository and in [mtls/keystores/solr](mtls/keystores/solr) for Search Services.

>> You may use [alfresco-ssl-generator](https://github.com/Alfresco/alfresco-ssl-generator) project to create custom keystore files.

**Alfresco Repository**

Recommended settings for `alfresco-global.properties` (or Java Environment variables for Docker Containers)

```
solr.host=solr6
solr.port.ssl=8983
solr.secureComms=https

dir.keystore=/usr/local/tomcat/alf_data/keystore
alfresco.encryption.ssl.keystore.type=JCEKS
alfresco.encryption.ssl.truststore.type=JCEKS
```

Credentials for *Keystore* and *Truststore* configuration is recommended to be used by passing Java Environment variables.

```
-Dssl-keystore.password=kT9X6oe68t
-Dssl-keystore.aliases=ssl-alfresco-ca,ssl-repo
-Dssl-keystore.ssl-alfresco-ca.password=kT9X6oe68t
-Dssl-keystore.ssl-repo.password=kT9X6oe68t
-Dssl-truststore.password=kT9X6oe68t
-Dssl-truststore.aliases=alfresco-ca,ssl-repo-client
-Dssl-truststore.alfresco-ca.password=kT9X6oe68t
-Dssl-truststore.ssl-repo-client.password=kT9X6oe68t
```

Additionally, in order to manage incoming HTTPs requests, additional Tomcat Connector in port 8443 should be declared in `server.xml` configuration file. Default Tomcat Connector is listening to plain HTTP request in port 8080.

```
<Connector port="8443" protocol="HTTP/1.1"
    SSLEnabled="true" scheme="https" secure="true" clientAuth="want" sslProtocol="TLS"
    keystoreFile="/usr/local/tomcat/alf_data/keystore/ssl.keystore"
    keystorePass="kT9X6oe68t" keystoreType="JCEKS"
    truststoreFile="/usr/local/tomcat/alf_data/keystore/ssl.truststore"
    truststorePass="kT9X6oe68t" truststoreType="JCEKS">
<\/Connector>
```

**Search Services**

Recommended settings for `solrcore.properties` (remember to add this settings to both cores `alfresco` and `archive`)

```
alfresco.host=alfresco
alfresco.port.ssl=8443
alfresco.secureComms=https

alfresco.encryption.ssl.keystore.location=/opt/alfresco-search-services/keystore/ssl-repo-client.keystore
alfresco.encryption.ssl.keystore.passwordFileLocation=
alfresco.encryption.ssl.keystore.type=JCEKS
alfresco.encryption.ssl.truststore.location=/opt/alfresco-search-services/keystore/ssl-repo-client.truststore
alfresco.encryption.ssl.truststore.passwordFileLocation=
alfresco.encryption.ssl.truststore.type=JCEKS
```

Credentials for *Keystore* and *Truststore* configuration is recommended to be used by passing Java Environment variables.

```
-Dsolr.jetty.truststore.password=kT9X6oe68t
-Dsolr.jetty.keystore.password=kT9X6oe68t
-Dssl-keystore.password=kT9X6oe68t
-Dssl-keystore.aliases=ssl-alfresco-ca,ssl-repo-client
-Dssl-keystore.ssl-alfresco-ca.password=kT9X6oe68t
-Dssl-keystore.ssl-repo-client.password=kT9X6oe68t
-Dssl-truststore.password=kT9X6oe68t
-Dssl-truststore.aliases=ssl-alfresco-ca,ssl-repo,ssl-repo-client
-Dssl-truststore.ssl-alfresco-ca.password=kT9X6oe68t
-Dssl-truststore.ssl-repo.password=kT9X6oe68t
-Dssl-truststore.ssl-repo-client.password=kT9X6oe68t
-Dsolr.ssl.checkPeerName=false
-Dsolr.allow.unsafe.resourceloading=true
```

Additionally, in order to manage incoming HTTPs requests, Jetty must be configured for TLS using `solr.in.sh` file.

```
SOLR_SSL_TRUST_STORE=/opt/alfresco-search-services/keystore/ssl-repo-client.truststore
SOLR_SSL_TRUST_STORE_PASSWORD=kT9X6oe68t
SOLR_SSL_TRUST_STORE_TYPE=JCEKS
SOLR_SSL_KEY_STORE=/opt/alfresco-search-services/keystore/ssl-repo-client.keystore
SOLR_SSL_KEY_STORE_PASSWORD=kT9X6oe68t
SOLR_SSL_KEY_STORE_TYPE=JCEKS
SOLR_SSL_NEED_CLIENT_AUTH=true
```

**Unauthorized accesses**

When using this configuration, HTTPs requests are protected by using digital certificates for authentication.

Accessing SOLR Web Console (available by default in https://localhost:8983/solr) requires the installation on the client computer of the digital certificate available in [mtls/keystores/client](mtls/keystores/client) folder.

## SECRET

Sample Docker Compose template is available in [secret](secret) folder.

**Alfresco Repository**

Recommended settings for `alfresco-global.properties` (or Java Environment variables for Docker Containers)

```
solr.host=solr6
solr.port=8983
solr.secureComms=secret
solr.sharedSecret=secret
```

>> Default `server.xml` Tomcat configuration works as expected, since there is a Connector listening to port 8080

**Search Services**

Recommended settings for `solrcore.properties` (remember to add this settings to both cores `alfresco` and `archive`)

```
alfresco.host=alfresco
alfresco.port=8080
alfresco.secureComms=secret
alfresco.secureComms.secret=secret
```

>> Default Jetty configuration works as expected, since there is a Listener in port 8983

**Unauthorized accesses**

This protocol doesn't include authentication by default, but it will require to add the shared secret word (`secret`) in HTTP Header using by default `X-Alfresco-Search-Secret` property.

Accessing SOLR Web Console (available by default in http://localhost:8983/solr) requires using a Browser plugin to add this header to the HTTP Requests.


## Running the Docker Compose templates

Select the configuration to test (`http`, `mtls` or `secret`) and run following command from that folder:

```
$ docker-compose up --build --force-recreate
```

>> If you are experimenting some problem, check some issues related to Docker Volumes that are described in https://github.com/alfresco/alfresco-docker-installer#docker-volumes
