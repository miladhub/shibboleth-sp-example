Shibboleth SAML Service Provider (SP) example
===

This project installs a testing SAML Identity Provider (IdP) and a Shibboleth Service Provider (SP) using Docker.

The app-specific configuration is contained in file `ssl.conf`:

    <Location /helk>
        AuthType shibboleth
        ShibRequestSetting requireSession 1
        require shib-session
        ShibUseHeaders On
        ProxyPass http://host.docker.internal:9176
        ProxyPassReverse http://host.docker.internal:9176
        ShibRequestSetting exportAssertion true
    </Location>

The sample app will be available at <https://localhost/helk>.

Shibboleth session information: <https://localhost/Shibboleth.sso/Session>.

Shibboleth status: <https://localhost/Shibboleth.sso/Status>.

# Installing the SAML IdP

    docker run --name=test-idp \
        -p 8080:8080 \
        -p 8443:8443 \
        -e SIMPLESAMLPHP_SP_ENTITY_ID=https://sp.example.org/shibboleth \
        -e SIMPLESAMLPHP_SP_ASSERTION_CONSUMER_SERVICE=https://localhost/Shibboleth.sso/SAML2/POST \
        -e SIMPLESAMLPHP_SP_SINGLE_LOGOUT_SERVICE=https://localhost/Shibboleth.sso/Logout \
        -d kristophjunge/test-saml-idp

The IdP is available at <http://localhost:8080/simplesaml>.

# Installing the Shibboleth SP

    curl http://localhost:8080/simplesaml/saml2/idp/metadata.php -o idp-metadata.xml
    docker run -dit --name shib-sp -p 80:80 -p 443:443 unicon/shibboleth-sp:3.0.4
    docker cp idp-metadata.xml shib-sp:/etc/shibboleth/
    docker cp shibboleth2.xml shib-sp:/etc/shibboleth/
    docker cp attribute-map.xml shib-sp:/etc/shibboleth/
    docker cp ssl.conf shib-sp:/etc/httpd/conf.d/
    docker restart shib-sp

To check the SP status, issue the following command:

    docker exec -it shib-sp /bin/sh -c "wget https://localhost/Shibboleth.sso/Status --no-check-certificate -qO-"

