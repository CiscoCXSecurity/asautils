# asautils

These tools automate some common Cisco ASA Tasks

## sendconf.py

**No internal product information was used to develop this tool. Windows tool Fiddler was used to identify the protocol used by ASDM.

This tools automates common tasks associated with configuring Cisco ASA physical or virtual appliances.

The tool simulates HTTP protocol to interact with the ASA mimicking ASDM.

It supports the following functions:
 - Install CA certificates from specified pem files.
 - Install identity certificates from specified pkcs12 files.
 - Upload files to flash.
 - Deploy configs from supplied text files.

The tasks do not need to be all specified at once. You can execute one or multiple tasks.

The tasks are executed in the order of the list above: CA certificates, Identity certificates, file uploads and finally configs. If one of the tasks is ommited it is simply skipped.

Since the tool simulates ASDM, the ASA is expected to be configured with accessible HTTP server and authentication enabled.

-u and -p options specify credentials to the ASA. If either or both of these arguments are omitted, they will be collected interactively.

-i and -t options specify both the trustpoint name as well as the file to be installed in those trustpoints. See examples below.

There is no validation of the contents of the certificates. CA certificate files are expected to be in base64 PEM format. Binary DER format will fail. PKCS12 files are expected to be in the standard binary form. The data from the file is converted to base64 when sent to the ASA.

Before certificates are installed, the tools makes the attempt to remove the match private key and the trustpoint. You may see errors if the items to be removed are not configured.

File upload feature runs *verify* command on the ASA to calculate MD5 hash of the file if it already exists on the device. If that hash matches that of the local file, the upload is skipped. When specifying the file name on the ASA, omit *disk0:/*. For example, to upload *disk0:/vpnprofiles/vpn1.xml*, specify *vpnprofiles/vpn1.xml*

You can use file upload feature to upload Dynamic Access Policy (DAP) as well as hostscan conditions (process, registry, etc). The DAP configuration is stored in disk0:/dap.xml and hostscan checks in disk0:/sdesktop/data.xml

When uploading configs, the ASA will accept exec level commands such as *write mem*. There are some commands on the ASA that takes a few seconds to commit, such as setting anyconnect image. if *write mem* is executed immediately after *anyconnect image* command, it will fail. The tool can be run again a few seconds later with just *write mem* command in the supplied config file.

The only non-default module that needs to be installed is **requests**.

Here's full usage help of the tool:

    usage: sendconf.py [-h] -a <ASAIP>[:<PORT>] [-u <username>] [-p <password>] [-t <trustpoint>=<pemfile> [<trustpoint>=<pemfile> ...]]
                    [-i <trustpoint>=<pfxfile>,<password> [<trustpoint>=<pfxfile>,<password> ...]] [-f <devicefile>=<localfile> [<devicefile>=<localfile> ...]]
                    [-c <configfile> [<configfile> ...]] [-d <level>]

    ASA Configuration Tool

    optional arguments:
    -h, --help            show this help message and exit
    -a <ASAIP>[:<PORT>]   ASA FQDN or IP address. HTTPS Port can be specified optionally
    -u <username>         ASDM Username. If ommited, an interactive prompt will be displayed.
    -p <password>         ASDM Password. If ommited, an interactive prompt will be displayed.
    -t <trustpoint>=<pemfile> [<trustpoint>=<pemfile> ...]
                            Root CA Trustpoints and PEM files
    -i <trustpoint>=<pfxfile>,<password> [<trustpoint>=<pfxfile>,<password> ...]
                            Identity Trustpoints and PKCS12 files
    -f <devicefile>=<localfile> [<devicefile>=<localfile> ...]
                            Upload files. Device file relative to disk0:/. Eg. sdesktop/data.xml=/tmp/data.xml
    -c <configfile> [<configfile> ...]
                            Path one or more config files. Configs will be applied in order.
    -d <level>            Debug level. 1-Warning, 2-Verbose (default), 3-Debug

### Example 1
Installing CA Certificates

    $ ./sendconf.py -a 1.2.3.4 -u cisco -p cisco -t x1=LetsEncryptX1.cer x3=LetsEncryptX3.cer 
    2021-08-06 22:38:47,565 - INFO - Attempting to import CA certificate x3
    2021-08-06 22:38:47,667 - INFO - Received response:
    ERROR: CA trustpoint 'x3' is not known.
    Enter the certificate in base64 representation....
    End with the word "quit" on a line by itself.
    .
    INFO: Certificate has the following attributes:
    Fingerprint:     4887d3a7 5ce51770 de002f32 1f5bc5ce 

    Trustpoint 'x3' is a subordinate CA and holds a non self-signed certificate.

    Trustpoint CA certificate accepted.

    Cryptochecksum (changed): 84db7dfe c06c042c cf974b51 ef74a419 
    Config OK

    2021-08-06 22:38:47,667 - INFO - Attempting to import CA certificate x3
    2021-08-06 22:38:47,954 - INFO - Received response:
    WARNING: Removing an enrolled trustpoint will destroy all 
    certificates received from the related Certificate Authority.
    INFO: Be sure to ask the CA administrator to revoke your certificates.
    Enter the certificate in base64 representation....
    End with the word "quit" on a line by itself.
    .
    INFO: Certificate has the following attributes:
    Fingerprint:     4887d3a7 5ce51770 de002f32 1f5bc5ce 

    Trustpoint 'x3' is a subordinate CA and holds a non self-signed certificate.

    Trustpoint CA certificate accepted.

    Cryptochecksum (changed): 84db7dfe c06c042c cf974b51 ef74a419 
    Config OK

### Example 2
Installing Identity Certificates. Note that a password has to be supplied for each pkcs12. In the example below, **cisco** is the password.

    $ ./sendconf.py -a 1.2.3.4 -u cisco -p cisco -i tp1=cisco1.pfx,cisco tp2=cisco2.pfx,cisco
    2021-08-06 22:48:08,218 - INFO - Attempting to import PKCS12 identity certificate tp1
    2021-08-06 22:48:08,332 - INFO - Received response:
    ERROR: CA trustpoint 'tp1' is not known.
    ERROR: The specified RSA keypair does not exist (tp1).
    Enter the PKCS12 data in base64 representation....
    ..WARNING: Identical public key already exists as ciscodemo
    WARNING: CA certificates can be used to validate VPN connections,
    by default.  Please adjust the validation-usage of this
    trustpoint to limit the validation scope, if necessary.
    INFO: Import PKCS12 operation completed successfully.

    Cryptochecksum (changed): ec6892c1 a323794c 63234df9 e6d93d9b 
    Config OK

    2021-08-06 22:48:08,333 - INFO - Attempting to import PKCS12 identity certificate tp2
    2021-08-06 22:48:08,622 - INFO - Received response:
    ERROR: CA trustpoint 'tp2' is not known.
    ERROR: The specified RSA keypair does not exist (tp2).
    Enter the PKCS12 data in base64 representation....
    .WARNING: Identical public key already exists as vblan
    WARNING: CA certificates can be used to validate VPN connections,
    by default.  Please adjust the validation-usage of this
    trustpoint to limit the validation scope, if necessary.
    INFO: Import PKCS12 operation completed successfully.

    Cryptochecksum (changed): d4eee601 cc534495 a792d79b 96e49f5c 
    Config OK

### Example 3
Uploading files.

    $ ./sendconf.py -a 1.2.3.4 -u cisco -p cisco -f sdesktop/data.xml=data.xml anyconnect-win-4.10.01075-webdeploy-k9.pkg=anyconnect-win-4.10.01075-webdeploy-k9.pkg dap.xml=dap.xml
    2021-08-06 23:10:29,875 - INFO - Attempting to upload sdesktop/data.xml
    2021-08-06 23:10:30,098 - INFO - Received Response:
    2173 bytes uploaded

    2021-08-06 23:10:30,787 - INFO - anyconnect-win-4.10.01075-webdeploy-k9.pkg is already on the device. Upload skipped.
    2021-08-06 23:10:31,008 - INFO - Attempting to upload dap.xml
    2021-08-06 23:10:31,235 - INFO - Received Response:
    6028 bytes uploaded

### Example 4
Performing all functions at once

    $ ./sendconf.py -a 1.2.3.4 -u cisco -p cisco -f sdesktop/data.xml=data.xml anyconnect-win-4.10.01075-webdeploy-k9.pkg=anyconnect-win-4.10.01075-webdeploy-k9.pkg dap.xml=dap.xml -i tp1=cisco1.pfx,cisco tp2=cisco2.pfx,cisco -t x1=LetsEncryptX1.cer x3=LetsEncryptX3.cer -c config1.txt saveconfig.txt
    2021-08-06 23:14:38,982 - INFO - Attempting to import CA certificate x3
    2021-08-06 23:14:39,086 - INFO - Received response:
    WARNING: Removing an enrolled trustpoint will destroy all 
    certificates received from the related Certificate Authority.
    INFO: Be sure to ask the CA administrator to revoke your certificates.
    Enter the certificate in base64 representation....
    End with the word "quit" on a line by itself.
    .
    INFO: Certificate has the following attributes:
    Fingerprint:     4887d3a7 5ce51770 de002f32 1f5bc5ce 

    Trustpoint 'x3' is a subordinate CA and holds a non self-signed certificate.

    Trustpoint CA certificate accepted.

    Cryptochecksum (changed): 84db7dfe c06c042c cf974b51 ef74a419 
    Config OK

    2021-08-06 23:14:39,087 - INFO - Attempting to import CA certificate x3
    2021-08-06 23:14:39,369 - INFO - Received response:
    WARNING: Removing an enrolled trustpoint will destroy all 
    certificates received from the related Certificate Authority.
    INFO: Be sure to ask the CA administrator to revoke your certificates.
    Enter the certificate in base64 representation....
    End with the word "quit" on a line by itself.
    .
    INFO: Certificate has the following attributes:
    Fingerprint:     4887d3a7 5ce51770 de002f32 1f5bc5ce 

    Trustpoint 'x3' is a subordinate CA and holds a non self-signed certificate.

    Trustpoint CA certificate accepted.

    Cryptochecksum (changed): 84db7dfe c06c042c cf974b51 ef74a419 
    Config OK

    2021-08-06 23:14:39,370 - INFO - Attempting to import PKCS12 identity certificate tp1
    2021-08-06 23:14:39,683 - INFO - Received response:
    WARNING: Removing an enrolled trustpoint will destroy all 
    certificates received from the related Certificate Authority.
    INFO: Be sure to ask the CA administrator to revoke your certificates.
    ERROR: The specified RSA keypair does not exist (tp1).
    Enter the PKCS12 data in base64 representation....
    ..WARNING: Identical public key already exists as ciscodemo
    WARNING: CA certificates can be used to validate VPN connections,
    by default.  Please adjust the validation-usage of this
    trustpoint to limit the validation scope, if necessary.
    INFO: Import PKCS12 operation completed successfully.

    Cryptochecksum (changed): ec6892c1 a323794c 63234df9 e6d93d9b 
    Config OK

    2021-08-06 23:14:39,683 - INFO - Attempting to import PKCS12 identity certificate tp2
    2021-08-06 23:14:39,973 - INFO - Received response:
    WARNING: Removing an enrolled trustpoint will destroy all 
    certificates received from the related Certificate Authority.
    INFO: Be sure to ask the CA administrator to revoke your certificates.
    ERROR: The specified RSA keypair does not exist (tp2).
    Enter the PKCS12 data in base64 representation....
    .WARNING: Identical public key already exists as vblan
    WARNING: CA certificates can be used to validate VPN connections,
    by default.  Please adjust the validation-usage of this
    trustpoint to limit the validation scope, if necessary.
    INFO: Import PKCS12 operation completed successfully.

    Cryptochecksum (changed): d4eee601 cc534495 a792d79b 96e49f5c 
    Config OK

    2021-08-06 23:14:40,201 - INFO - Attempting to upload sdesktop/data.xml
    2021-08-06 23:14:40,426 - INFO - Received Response:
    2173 bytes uploaded

    2021-08-06 23:14:41,127 - INFO - anyconnect-win-4.10.01075-webdeploy-k9.pkg is already on the device. Upload skipped.
    2021-08-06 23:14:41,343 - INFO - Attempting to upload dap.xml
    2021-08-06 23:14:41,571 - INFO - Received Response:
    6028 bytes uploaded

    2021-08-06 23:14:41,572 - INFO - Attempting to post configuration from config1.txt
    2021-08-06 23:14:47,031 - INFO - Received response:

    INFO: Webvpn Cache is disabled by default on this release.
        Please refer to the documentation to enable WebVPN Cache using CLI or ASDM.

    Cryptochecksum (changed): 380140fd 0cfff750 ce486d4e 9bea18d0 
    Config OK

    2021-08-06 23:14:47,031 - INFO - Attempting to post configuration from saveconfig.txt
    2021-08-06 23:14:47,323 - INFO - Received response:

    Cryptochecksum (changed): d41d8cd9 8f00b204 e9800998 ecf8427e 
    Config OK