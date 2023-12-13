# tomcat环境下部署verto客户端
## 1. generate a self-signed certificate for Tomcat using OpenSSL
Step 1: Generate a private key
```
openssl genpkey -algorithm RSA -out private.key
```
Step 2: Create a certificate signing request (CSR
```
penssl req -new -key private.key -out certificate.csr
```
Step 3: Generate the self-signed certificate
```
openssl x509 -req -days 365 -in certificate.csr -signkey private.key -out certificate.crt
```
Step 4: Prepare the certificate and private key for Tomcat

Combine the private key and the certificate into a PKCS12 keystore that Tomcat can use:
```
openssl pkcs12 -export -in certificate.crt -inkey private.key -out certificate.p12 -name your_alias
```
Replace your_alias with any alias you want to use for this certificate in the keystore.

Step 5: Copy the keystore to the Tomcat configuration folder

Copy the generated keystore (certificate.p12) to the Tomcat configuration folder. The default location for Tomcat's keystore is the <Tomcat_home>/conf directory.

Step 6: Configure Tomcat's server.xml

Open the server.xml file located in the <Tomcat_home>/conf directory.
Find the <Connector> element that you want to enable SSL for (usually the one with port 8443 or 443). Add the following attributes to the <Connector> element:
```
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
           maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
           keystoreFile="/path/to/your/certificate.p12"
           keystoreType="PKCS12" keystorePass="your_keystore_password"
           keyAlias="your_alias" />
```
Replace /path/to/your/certificate.p12 with the actual path to the keystore you copied to the Tomcat configuration folder. Set your_keystore_password to the password you used when generating the keystore. Use the your_alias you specified during the keystore creation.

Step 7: Restart Tomcat

Restart Tomcat to apply the changes. The SSL-enabled connector will now use the self-signed certificate for secure communication.

## 2. generate a certificate for FreeSWITCH by using certificate.p12 which generated above
Step 1: Convert the Certificate

If your certificate is not already in PEM format, you may need to convert it using tools like openssl. For example, to convert from PFX/P12 to PEM:
```
openssl pkcs12 -in certificate.p12 -out certificate.pem -nodes
```
Step 2: Configure FreeSWITCH

copy certificate.pem to your FreeSWITCH path/certs (e.g.,/usr/local/freeswitch/certs/)

Step 3: Restart FreeSWITCH

After making the necessary changes, restart  FreeSWITCH to apply the new configurations.

## 3. Testing
copy the verto applications to webapps/ROOT directory of tomcat.
```
cp -r verto/video_demo /opt/apache-tomcat-8.5.89/webapps/ROOT/
```
open your browser (e.g. Google Chrome) ,typing https://ip:8443/video_demo to check it works or not.
