# How-to-configure-a-secure-MQ-Explorer-connection

IBM WebSphere: How to create a jdbc connection for a Snowflake Database <BR> 
Version 1.0 <BR>

- Richard Niewolik


Content
-------

[1 General](#1-general) <BR>
[2 Sample scenario](#2)

1 General
=========

This asset describes an example scenario for a secure IBM MQ Explorer connection. 
It is **only** an example and needs to be tested to verify that it meets the existing requirements.

For better cut and paste all commands mentioned below here in a [text file](https://github.com/ricniew/How-to-configure-a-secure-MQ-Explorer-connection/blob/main/ContentAsText)

2 Sample scenario
==================

**A.** On a **ServerHost** (here Linux) 

1. create the test MQ Queue Manager (here called **ssltestqm**)
    ``` 
    endmqm ssltestqm ; sleep 2; dltmqm ssltestqm ; sleep 2;  crtmqm ssltestqm ; sleep 2;  strmqm ssltestqm
    ``` 
1. Create a MQ user if required (for example `mqm`)
    ```
    adduser myuser
    usermod -a -G mqm  userid
    passwd myuser
    ```

**B.** On **ClientHost** (where MQ Explorer is going to be used. Usually on Windows)

1. Create temporary folder
    ```
    cd c:\
    mkdir mqexplorer_keystore
    cd mqexplorer_keystore
    ```
2.  Create SSL client key database 
    ```
    runmqckm -keydb -create -db mqexplorer.jks  -pw clientpazzword -type jks
    ```
3. Create the certificate
    ```
    runmqckm -cert -create -db mqexplorer.jks -pw clientpazzword -label ibmwebspheremqmqexplorer -dn "CN=mqexplorer,O=IBM,C=DE" -expire 365
    ```
4. Extract the cert
    ```
    runmqckm -cert -extract -db mqexplorer.jks -pw clientpazzword -label ibmwebspheremqmqexplorer -target mqexplorer.crt -format ascii
    ```
**C.** Transfer the created `mqexplorer.crt` (extracted cleint certificate) to the queue mangager on **ServerHost**. Into the folder `/var/mqm/qmgrs/ssltestqm/ssl`


**D.** on Queue Manager **ssltestqm** on **ServerHost** (here Linux)

1. Create server SSL server key database
    ```
    cd /var/mqm/qmgrs/ssltestqm/ssl
    runmqckm -keydb -create -db ssltestqm.kdb  -pw serverpazzword -type cms -expire 365 -stash
    ```
2. Create server certificate
    ```
    runmqckm -cert -create -db  ssltestqm.kdb -pw serverpazzword -label ibmwebspheremqssltestqm -dn "CN=ssltestqm,O=HDI,C=DE" -expire 365
    ```
3. List the server certificate
    ```
    runmqckm -cert -list -db ssltestqm.kdb  -pw serverpazzword
    ```
4. Add the client certificate to server db (if you did not transfer the mqexplorer.crt from MQ Explorer host yet, do it now)
    ```
    chown mqm:mqm mqexplorer.crt
    runmqckm -cert -add -db ssltestqm.kdb -pw serverpazzword -label ibmwebspheremqmqexplorer -file mqexplorer.crt -format ascii
    ```
5. List the server certificates
    ```
    runmqckm -cert -list -db ssltestqm.kdb  -pw serverpazzword
    ```
6. Extract the public SSL server certificate and copy it to the SSL client side
    ```
    runmqckm -cert -extract -db ssltestqm.kdb -pw serverpazzword -label ibmwebspheremqssltestqm -target ssltestqm.crt -format ascii
    ```
8. Transver `ssltestqm.crt` to **ClientHost** (MQ Explorer) 

**E.** On **ClientHost**
1. Copy the public SSL **server** certificate to the SSL **client** side (if you did not transfer the `ssltestqm.crt` from qmgr host yet, do it now)
    ```
    cd c:\mqexplorer_keystore
    runmqckm -cert -add -db mqexplorer.jks -pw clientpazzword -label ibmwebspheremqssltestqm -file ssltestqm.crt -format ascii
    ```
2. List Client certs
    ```
    runmqckm -cert -list -db mqexplorer.jks  -pw clientpazzword
    ```

**F.** on Queue manager **ssltestqm** on **ServerHost**

Here it is Linux. All names, PORT numbers or SSLCIPH values can be changed. They are specific for this example.

1. Configure your Queue Manager MGR using for example `runmqsc ssltestqm`
    ```
    DEFINE LISTENER(ssltestqm.LISTENER) TRPTYPE(TCP) PORT(8414) CONTROL(QMGR)
    START LISTENER(ssltestqm.LISTENER) 
    DEFINE CHANNEL(SSL.ADMIN) CHLTYPE(SVRCONN) 
    DEFINE QLOCAL(MYQ) 
    ALTER QMGR SSLKEYR('/var/mqm/qmgrs/ssltestqm/ssl/ssltestqm')
    ALTER QMGR SSLFIPS(NO)
    DEFINE CHANNEL('SSL.ADMIN') CHLTYPE(SVRCONN) TRPTYPE(TCP) SSLCIPH(ECDHE_RSA_AES_128_CBC_SHA256) SSLCAUTH(REQUIRED) REPLACE
    SET CHLAUTH(SSL.ADMIN) TYPE(BLOCKUSER) USERLIST('nobody')
    REFRESH SECURITY TYPE(SSL)
    end
    ```
2. Set required permissions (an example)
    ```
    cd  /var/mqm/qmgrs/ssltestqm/ssl
    chmod 770 *
    # Grant  "myuser" or any other you access to qmgr
    setmqaut -m ssltestqm -t qmgr -p myuser +connect +inq +dsp
    ```
    
**E.** On **MQ Explorer**

To connect MQ Explorer to new server "ssltestqm" using SSL. All names, PORT numbers or SSLCIPH values can be changed. They are specific for this example.

1. add new remote queue manager connection
1. Select the Queue Manager Name (in this example "ssltestqm"), and connection Method, in this example: Connect DIrectly
1. Specify New Connection Details: Set hostname and port (in this example 8414) and Server Connection Channel (in this example "SSL.ADMIN")
1. Specify User authentication Details and set your user
1. Specify SSL Certificate key repository Details and pointed to "c:\mqexplorer_keystore\mqexplorer.jks" keystore for both truststore and keystore 
1. Specify SSL Options Details Cipherspec, in this example set it to ECDHE_RSA_AES_128_CBC_SHA256  

DONE



