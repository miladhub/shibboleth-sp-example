Shibboleth SAML Service Provider (SP) example
===

This project installs a testing SAML Identity Provider (IdP) and a Shibboleth Service Provider (SP) using Docker.

These instructions were tested on MacOS. The app behind the SP is just a demo server that dumps all HTTP headers on the console. The app-specific configuration is contained in file `ssl.conf`:

    <Location /app>
        AuthType shibboleth
        ShibRequestSetting requireSession 1
        require shib-session
        ShibUseHeaders On
        ProxyPass http://host.docker.internal:9176
        ProxyPassReverse http://host.docker.internal:9176
        ShibRequestSetting exportAssertion true
    </Location>

You can replace it by pointing at your app.

# Installing the SAML IdP

    docker run --name=test-idp \
        -p 28080:8080 \
        -p 28443:8443 \
        -e SIMPLESAMLPHP_SP_ENTITY_ID=https://sp.example.org/shibboleth \
        -e SIMPLESAMLPHP_SP_ASSERTION_CONSUMER_SERVICE=https://localhost/Shibboleth.sso/SAML2/POST \
        -e SIMPLESAMLPHP_SP_SINGLE_LOGOUT_SERVICE=https://localhost/Shibboleth.sso/Logout \
        -d kristophjunge/test-saml-idp

The IdP is available at <http://localhost:8080/simplesaml>.

# Installing the Shibboleth SP

    docker run -dit --name shib-sp -p 80:80 -p 443:443 unicon/shibboleth-sp:3.0.4
    curl http://localhost:28080/simplesaml/saml2/idp/metadata.php -o idp-metadata.xml
    docker cp idp-metadata.xml shib-sp:/etc/shibboleth/
    docker cp shibboleth2.xml shib-sp:/etc/shibboleth/
    docker cp attribute-map.xml shib-sp:/etc/shibboleth/
    docker cp ssl.conf shib-sp:/etc/httpd/conf.d/
    docker restart shib-sp

To check the SP status, issue the following command:

    docker exec -it shib-sp /bin/sh -c "wget https://localhost/Shibboleth.sso/Status --no-check-certificate -qO-"

# Installing the dummy app

    nc -l 9176
    
If you navigate to the app at <https://localhost/app>, the IdP will first prompt for credentials - just enter `user1` / `user1pass`; after that, the page will stay there hanging, but on the console you can see all SAML headers coming through, which means that SAML is working:

    GET / HTTP/1.1
    Host: host.docker.internal:9176
    Cache-Control: max-age=0
    Upgrade-Insecure-Requests: 1
    User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.122 Safari/537.36
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
    Sec-Fetch-Site: cross-site
    Sec-Fetch-Mode: navigate
    Sec-Fetch-Dest: document
    Referer: http://localhost:28080/simplesaml/saml2/idp/SSOService.php?SAMLRequest=fVK7TsMwFP2VyHtj1w0lsZpKoR2oxKMihYEFOcmlseTYxtfh8fc0TUFlYfHi89ZdoOy0E0UfWvMAbz1giD47bVAcP3LSeyOsRIXCyA5QhFqUxe2N4DETzttga6tJVCCCD8qalTXYd%2BBL8O%2BqhseHm5y0ITgUlGpbS91aDLRsVVVZDaGNES0dBDnd3pc7Eq0PCZSRg9bIPCcKnrKUUVSd0zAEpMPDqWocLcv7k2nsWkeizTonLzLNUmCz6WuSgaxYxS5lkqQNv8jmkDRZcoAh9rAxGKQJOeGMswlLJpzvppngc5HMnkm0PfW8UqZRZv%2F%2FKNUIQnG9220nY6kn8HgsdACQ5WIILY7G%2Fmzs%2F2Xlz8Jk%2BbMnuhg%2B5TBFbP2e4u%2BoC3rmMNo5cXeQ3Ky3Vqv6Kyq0th8rDzJATqaELkfK30NYfgM%3D&RelayState=ss%3Amem%3A89b56d3926131e1df7fbff133de0639e8d27cfe04f76905709ce03dfb28699de
    Accept-Encoding: gzip, deflate, br
    Accept-Language: en-US,en;q=0.9,it;q=0.8,pl;q=0.7
    Cookie: PHPSESSIDIDP=2480f68fbde75255a8e9aa131acaea5f; SimpleSAMLAuthTokenIdp=_facbf4cf27abba77af57d947bbc45b2c9ff185bbac; _shibsession_64656661756c7468747470733a2f2f73702e6578616d706c652e6f72672f73686962626f6c657468=_102bdf3b015a9470794e647ec986e22e
    Shib-Cookie-Name: 
    Shib-Session-ID: _102bdf3b015a9470794e647ec986e22e
    Shib-Session-Index: _9ed042750b79e0843a989bbb56e31c352923e4d7f8
    Shib-Session-Expires: 1587611869
    Shib-Session-Inactivity: 1587587203
    Shib-Identity-Provider: http://localhost:28080/simplesaml/saml2/idp/metadata.php
    Shib-Authentication-Method: urn:oasis:names:tc:SAML:2.0:ac:classes:Password
    Shib-Authentication-Instant: 2020-04-22T19:14:49Z
    Shib-AuthnContext-Class: urn:oasis:names:tc:SAML:2.0:ac:classes:Password
    Shib-AuthnContext-Decl: 
    Shib-Assertion-Count: 01
    Shib-Handler: https://localhost/Shibboleth.sso
    subject-id: 
    pairwise-id: 
    eppn: 
    affiliation: 
    entitlement: 
    persistent-id: 
    uid: 1
    eduPersonAffiliation: group1
    email: user1@example.com
    Shib-Application-ID: default
    REMOTE_USER: 
    Shib-Assertion-01: https://localhost/Shibboleth.sso/GetAssertion?key=_102bdf3b015a9470794e647ec986e22e&ID=_23242d09d4b2cdc9c5e37fd32d2ed80e317cf657f7
    X-Forwarded-For: 172.17.0.1
    X-Forwarded-Host: localhost
    X-Forwarded-Server: 172.17.0.3
    Connection: Keep-Alive

More Shibboleth info can be found at <https://localhost/Shibboleth.sso/Session> and <https://localhost/Shibboleth.sso/Status>.

These headers also include the URL to download the SAML assertion from:

    Shib-Assertion-01: https://localhost/Shibboleth.sso/GetAssertion?key=_102bdf3b015a9470794e647ec986e22e&ID=_23242d09d4b2cdc9c5e37fd32d2ed80e317cf657f7
