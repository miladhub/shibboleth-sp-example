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
        -e SIMPLESAMLPHP_SP_ASSERTION_CONSUMER_SERVICE=https://docker.local/Shibboleth.sso/SAML2/POST \
        -e SIMPLESAMLPHP_SP_SINGLE_LOGOUT_SERVICE=https://docker.local/Shibboleth.sso/Logout \
        -d kristophjunge/test-saml-idp

The IdP is available at <http://docker.local:28080/simplesaml>.

# Installing the Shibboleth SP

    # Workaround for https://github.com/kubernetes/minikube/issues/8439
    MYIP=`minikube ssh grep host.minikube.internal /etc/hosts | cut -f1`
    cp ssl.conf my.ssl.conf
    sed -i '' "s/host\.docker\.internal/$MYIP/g" my.ssl.conf

    docker run -dit --name shib-sp -p 80:80 -p 443:443 unicon/shibboleth-sp:3.0.4
    curl http://docker.local:28080/simplesaml/saml2/idp/metadata.php -o idp-metadata.xml
    docker cp idp-metadata.xml shib-sp:/etc/shibboleth/
    docker cp shibboleth2.xml shib-sp:/etc/shibboleth/
    docker cp attribute-map.xml shib-sp:/etc/shibboleth/
    docker cp my.ssl.conf shib-sp:/etc/httpd/conf.d/ssl.conf
    docker restart shib-sp
    rm my.ssl.conf

To check the SP status, issue the following command:

    docker exec -it shib-sp /bin/sh -c "wget https://localhost/Shibboleth.sso/Status --no-check-certificate -qO-"

# Installing the dummy app

    nc -l 9176
    
If you navigate to the app at <https://docker.local/app>, the IdP will first prompt for credentials - just enter `user1` / `user1pass`; after that, the page will stay there hanging, but on the console you can see all SAML headers coming through, which means that SAML is working:

    GET / HTTP/1.1
    Host: 192.168.64.1:9176
    Cache-Control: max-age=0
    sec-ch-ua: " Not;A Brand";v="99", "Google Chrome";v="97", "Chromium";v="97"
    sec-ch-ua-mobile: ?0
    sec-ch-ua-platform: "macOS"
    Upgrade-Insecure-Requests: 1
    User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/97.0.4692.71 Safari/537.36
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
    Sec-Fetch-Site: cross-site
    Sec-Fetch-Mode: navigate
    Sec-Fetch-User: ?1
    Sec-Fetch-Dest: document
    Referer: http://docker.local:28080/
    Accept-Encoding: gzip, deflate, br
    Accept-Language: en-US,en;q=0.9
    Cookie: PHPSESSIDIDP=ba1023a832e4bb3677ddcf5b4ee9a589; SimpleSAMLAuthTokenIdp=_6958c3b06681672f22c03903af4b381cd9fcf01436; _shibsession_64656661756c7468747470733a2f2f73702e6578616d706c652e6f72672f73686962626f6c657468=_c77ea9fcd41b57f5c238e47da520cafe
    Shib-Cookie-Name: 
    Shib-Session-ID: _c77ea9fcd41b57f5c238e47da520cafe
    Shib-Session-Index: _233c95bc7ea13c40d4f19fdc99fddadb4123f09bd2
    Shib-Session-Expires: 1641848926
    Shib-Session-Inactivity: 1641824175
    Shib-Identity-Provider: http://docker.local:28080/simplesaml/saml2/idp/metadata.php
    Shib-Authentication-Method: urn:oasis:names:tc:SAML:2.0:ac:classes:Password
    Shib-Authentication-Instant: 2022-01-10T13:08:46Z
    Shib-AuthnContext-Class: urn:oasis:names:tc:SAML:2.0:ac:classes:Password
    Shib-AuthnContext-Decl: 
    Shib-Assertion-Count: 
    Shib-Handler: https://docker.local/Shibboleth.sso
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
    X-Forwarded-For: 192.168.64.1
    X-Forwarded-Host: docker.local
    X-Forwarded-Server: 172.17.0.4
    Connection: Keep-Alive

More Shibboleth info can be found at <https://localhost/Shibboleth.sso/Session> and <https://localhost/Shibboleth.sso/Status>.

These headers also include the URL to download the SAML assertion from:

    Shib-Assertion-01: https://localhost/Shibboleth.sso/GetAssertion?key=_102bdf3b015a9470794e647ec986e22e&ID=_23242d09d4b2cdc9c5e37fd32d2ed80e317cf657f7
