# MONGODB-CONNECTION-USING-TLS

We will be using below REPLCIASET to enable TLS connection. 

![alt text](https://github.com/nitin-pandey-27/mongodb-connection-tls/blob/main/MONGODB-USING-TLS.png)

<!-- Reference Links
https://mydbops.wordpress.com/2020/05/02/securing-mongodb-cluster-with-tls-ssl/
https://www.getfilecloud.com/supportdocs/display/cloud/Configure+MongoDB+Cluster+to+Use+TLS-SSL+with+Cluster+Authentication+and+Mongodb+Authentication+on+Linux
https://www.getfilecloud.com/supportdocs/display/cloud/Configure+MongoDB+Cluster+to+Use+TLS-SSL+with+Cluster+Authentication+and+Mongodb+Authentication+on+Linux
-->


## Create Private Certifcate 

 `openssl genrsa -out example-ca.key -aes256 8192`
   
  We need to enter a strong password and make sure to take backup of the password or remember the password. 
   
   
## Sign CA Public Certificate 
  
 `openssl req -x509 -new -extensions v3_ca -key example-ca.key -days 365 -out example-ca-pub.crt`
 
  This CA public certificate needs to be shared with all the members. 
  Kindly do provide all the details asked while generating the certiicate 
  
  
## Generate the CSR and Private key for all member
 
  `openssl req -nodes -newkey rsa:4096 -sha256 -keyout mongod1.key -out mongod1.csr` 
  
  `openssl req -nodes -newkey rsa:4096 -sha256 -keyout mongod2.key -out mongod2.csr`
  
  `openssl req -nodes -newkey rsa:4096 -sha256 -keyout mongod3.key -out mongod3.csr`
   
   For each replica member, we need to generate a CSR and Private Key. 
   Make sure to provide correct CN name. Use either FQDN or HOST IP Address in CN name. 
   Please provide the similar information as mentioned for CA Certicate. 
   Kindly do provide all the details asked while generating the certiicate 
    
## Signing CSR using CA Private & Public Key
 
 `openssl x509 -req -in mongod1.csr -CA example-ca-pub.crt -CAkey example-ca.key -CAcreateserial -out mongod1.crt`
 
 `openssl x509 -req -in mongod2.csr -CA example-ca-pub.crt -CAkey example-ca.key -CAcreateserial -out mongod2.crt`
 
 `openssl x509 -req -in mongod3.csr -CA example-ca-pub.crt -CAkey example-ca.key -CAcreateserial -out mongod3.crt`
  
  Use example-ca.key password.
  
  
## Generate PEM file for each host 

`cat mongod1.key mongod1.crt > mongod1.pem`

`cat mongod2.key mongod2.crt > mongod2.pem`

`cat mongod3.key mongod3.crt > mongod3.pem`
 
 Now, copy <node#>.pem & example-ca-pub.crt file to respective replica members. 
 
 
## Create a configuration file for generating client CSR 


 Client certificates need to be signed by the same CA who signed all replica member certificate.
 Common name filed can enter wildcard name (*.example.com) or we can use the SAN field which allows multiple name entries on a single certificate.
 we need to specify a different value for O or OU filed compare to the replica set member certificate.

 Create example.conf file 

 `vi cat example.conf`
 
  
 ```
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
default_keyfile = example-client.key
prompt = no

[req_distinguished_name]
C = IN
ST = UP
L = NOIDA
O = CLIENT
OU = DSD-CLIENT
CN = 10.132.214.224

[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = XX.XX.XX.XX
DNS.2 = XX.XX.XX.XX
DNS.3 = XX.XX.XX.XX
```

 Provide all the member HOSTNAME / IP ADDRESS in `[alt_names]`
  
## Generete CSR file and private key from above configuration file. 

`openssl req -new -nodes -out example-client.csr -config example.conf`
  
  
## Sign the client CSR using CA public and private key

 `openssl x509 -req -in example-client.csr -CA example-ca-pub.crt -CAkey example-ca.key -out example-client.crt` 
 
 concatenate the key and the signed certificate
  
  `cat example-client.key example-client.crt > example-client.pem`
  

## Configure REPLICASET to use TSL 

 Change the configuration file /etc/mongod.conf and perform below changes. Perform these changes on all the files. 
 
 ```
 ##### network interfaces
 net:
  port: 27017
  bindIp: 127.0.0.1,10.132.214.224  # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.
  tls:
   mode: requireTLS
   certificateKeyFile: /opt/mongokeys/mongodb-tls/mongod1.pem
   CAFile: /opt/mongokeys/mongodb-tls/example-ca-pub.crt
 ```
   
 
 ## Change the permission and restart the daemon
 
  `chown -R mongod:mongod /opt/mongokeys/mongodb-tls/`
  
   `systemctl restart mongod`
   
 ## Access the system using Certificate 
 
  `mongo --tls --tlsCAFile example-ca-pub.crt -u "eauth" -p "eauth123" 10.132.214.224:27017 --authenticationDatabase "idxadmin" --tlsCertificateKeyFile example-client.pem`
