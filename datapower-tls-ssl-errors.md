# DEBUGGING DATAPOWER TLS / SSL ERRORS

## EXECUTIVE SUMMARY
SSL / TLS is a core requirement for a secure infrastructure. Often, the first time that two systems need to communicate will result in the handshake failing. In older firmware versions there was little information logged as to the specific cause of the problem. This caused both sides of the connection to extensively walk their respective configurations, usually after a round of blaming each other. In the newer firmware, this has gotten much better by allowing the SSL Library to emit errors into the DataPower log.

This article goes into depth about common handshaking errors through examples. It should help to reduce the amount of time spent debugging so that the project can move on to higher value business testing.

## THE RED HERRING ERROR
Before we begin, we need to point out this error message that will almost always occur in the DataPower logs when SSL goes wrong:

```log
[0x81200025][ssl][error] ... : SSL peer did not send a certificate during the handshake
```

A junior DataPower administrator will hone in on this message and proudly declare that the client was misconfigured because it did not send a certificate. This sends the client team scurrying to their server and staring at their certificate. While the DataPower admin is technically correct that a certificate never arrived, it is more likely that the handshake failed long before the point where certificate presentation occurs.

It would be nice if the DataPower development team stopped logging this message, or at least changed the text for accuracy. It’s written too authoritatively. It’s almost never the root cause and it wastes critical debugging time. This waste gets much worse when the client is an external vendor and inexperienced in SSL troubleshooting.

In all of our examples, you will see this message written even though it had nothing to do with the certificate itself. Be aware.

## CLIENT SENDS HTTP REQUEST

```log
[0x8120002f][ssl][error] ssl-server ... : SSL library error: error:1407609C:SSL routines:SSL23_GET_CLIENT_HELLO:http request
[0x81200025][ssl][error] ssl-server ... : SSL peer did not send a certificate during the handshake
[0x80e00130][http][error] xmlfirewall ... : could not establish SSL for incoming connection
[0x80e00531][xmlfirewall][debug] xmlfirewall ... : Starting termination of connection 192.168.1.46:52959 192.168.1.10:9195
```

In this scenario, the client forgot that we’re an HTTPS endpoint and hit it with plain ol’ HTTP. In the old firmware, you would see an error about writing a descriptor(8). Today, you clearly see that the SSL library identified the client as HTTP. This error can be common when network health checkers are misconfigured and don’t use half-open to determine service availability. Effort should be made to fix the extraneous errors from occurring in the log.

## TLS 1.0 WHEN ONLY 1.1/1.2  ALLOWED

```log
[0x8120002f][ssl][error] ssl-server ... : SSL library error: error:140760FC:SSL routines:SSL23_GET_CLIENT_HELLO:unknown protocol
[0x81200025][ssl][error] ssl-server ... : SSL peer did not send a certificate during the handshake
[0x80e00130][http][error] xmlfirewall ... : could not establish SSL for incoming connection
[0x80e00531][xmlfirewall][debug] xmlfirewall ... : Starting termination of connection 192.168.1.46:49240 192.168.1.10:9195
```

In this scenario, the client sent a TLS 1.0 connection but DataPower is configured to only accept TLS1.1 or TLS1.2 connections. This is common when integrating with legacy environments that may not support the latest TLS protocols. DataPower writes that an unknown protocol was encountered.

## TLS 1.2 WHEN ONLY 1.1 ALLOWED

```log
[0x8120002f][ssl][error] ssl-server ... : SSL library error: error:1409442E:SSL routines:ssl3_read_bytes:tlsv1 alert protocol version
[0x81200025][ssl][error] ssl-server ... : SSL peer did not send a certificate during the handshake
[0x80e00130][http][error] xmlfirewall ... : could not establish SSL for incoming connection
[0x80e00531][xmlfirewall][debug] xmlfirewall ... : Starting termination of connection 192.168.1.46:57804 192.168.1.10:9195
```
In this scenario, the client send a TLS 1.2 connection but DataPower is configured to only accept TLS 1.1. This error message occurs when a higher TLS version arrives than what the server supports. For completeness, the same error occurred when the client sent TLS 1.1 but DataPower only accepted 1.0.

## SSLV3 WHEN ONLY TLS 1.0+ ALLOWED

```log
[0x8120002f][ssl][error] ssl-server ... : SSL library error: error:1408A10B:SSL routines:ssl3_get_client_hello:wrong version number
[0x81200025][ssl][error] ssl-server ... : SSL peer did not send a certificate during the handshake
[0x80e00130][http][error] xmlfirewall ... : could not establish SSL for incoming connection
[0x80e00531][xmlfirewall][debug] ... : Starting termination of connection 192.168.1.46:49316 192.168.1.10:9195
```
In this scenario, the client sends an SSLv3 connection but DataPower is configured to only accept TLS 1.0, 1.1 and 1.2. The same error message occurs if the client sends SSLv2. SSLv3 and SSLv2 are ancient versions and any occurrence of this error in the DataPower environment should be forwarded to the enterprise security team to ensure that its still suitable for the client to be so backleveled.

## CIPHER MISMATCH

```log
[0x8120002f][ssl][error] ssl-server ... : SSL library error: error:1408A0C1:SSL routines:ssl3_get_client_hello:no shared cipher
[0x81200025][ssl][error] ssl-server ... : SSL peer did not send a certificate during the handshake
[0x80e00130][http][error] xmlfirewall ... : could not establish SSL for incoming connection
[0x80e00531][xmlfirewall][debug] xmlfirewall ... : Starting termination of connection 192.168.1.46:49737 192.168.1.10:9195
```
In this scenario, the client allowable cipher list and the DataPower allowable cipher list have no matching entry. This should be an extremely rare scenario and the resolution will require the two administrators to talk, as it’s difficult to know the list of ciphers that the client supports.

## NO CLIENT CERTIFICATE

```log
0x8120002f][ssl][error] ssl-server ... : SSL library error: error:140890C7:SSL routines:ssl3_get_client_certificate:peer did not return a certificate
[0x80e0052e][xmlfirewall][debug] xmlfirewall ... : Starting SSL termination of connection 192.168.1.46:49218 192.168.1.10:9195
[0x81200025][ssl][error] ssl-server ... : SSL peer did not send a certificate during the handshake
```

In the scenario, the client legitimately did not present a certificate to the DataPower endpoint. We can tell that this is the root cause error as the SSL library error confirmed a lacking certificate.

## EXPIRED CERTIFICATE

```log
[0x8060010a][crypto][warn] ... : certificate validation failed for '/C=CA/ST=Ontario/L=Toronto/O=Orange Specs Incorporated/OU=Consulting/CN=client-5.ssltest.orangespecs.com' against 'SSLError_ValCred': certificate has expired
[0x8120002f][ssl][error] ssl-server ... : SSL library error: error:14089086:SSL routines:ssl3_get_client_certificate:certificate verify failed
[0x80e0052e][xmlfirewall][debug] xmlfirewall ... : Starting SSL termination of connection 192.168.1.46:64385 192.168.1.10:9195
[0x81200026][ssl][error] ssl-server ... : SSL handshake certificate validation error with validation credentials SSLError_ValCred: certificate has expired
```
In this scenario, the client has presented an expired certificate to DataPower. This will happen more often than it should, as certificate management is a critical component of a secure infrastructure but not enough attention is given to it in the enterprise.

## DATAPOWER UNABLE TO VERIFY CLIENT CERTIFICATE
```log
[0x8060010a][crypto][warn] valcred ... : certificate validation failed for '/C=CA/ST=Ontario/L=Toronto/O=Orange Specs Incorporated/OU=Consulting/CN=client-1.ssltest.orangespecs.com' against 'SSLError_ValCred': certificate not trusted
[0x8120002f][ssl][error] ssl-server ... : SSL library error: error:14089086:SSL routines:ssl3_get_client_certificate:certificate verify failed
[0x80e0052e][xmlfirewall][debug] xmlfirewall ... : Starting SSL termination of connection 192.168.1.46:49229 192.168.1.10:9195
[0x81200026][ssl][error] ssl-server ... : SSL handshake certificate validation error with validation credentials SSLError_ValCred: certificate not trusted
```
In this scenario, DataPower was unable to validate the certificate presented by the client. This configuration is stored in the Validation Credentials associated with the SSL Server Profile. Unfortunately, this error will occur for a broad range of verification errors, such as:

+ The client certificate is not in the certificate list when the certificate validation mode is ‘Exact Certificate’.
+ An intermediate signer chain certificate is missing when the certificate validation mode is  ‘Full Certificate Chain Checking (PKIX)’ or ‘Match Exact Certificate or Immediate Issuer’
+ The client certificate is self-signed and is not in the certificate list

Debugging this error will require a copy of the client certificate, either to inspect the signer chain or to add the certificate to the validation credentials.

## CLIENT CERTIFICATE VALIDATED

```log
[0x8060010b][crypto][info] valcred ... : certificate validation succeeded for '/C=CA/ST=Ontario/L=Toronto/O=Orange Specs Incorporated/OU=Consulting/CN=client-1.ssltest.orangespecs.com' against 'SSLError_ValCred'
```
In this scenario, the client certificate was successfully validated by DataPower. The log message includes the DN and the validation credential used. This message will only be seen in the logs when the log target is at the ‘info’ level for the ‘crypto’ object.

## CLIENT CANNOT VERIFY DATAPOWER SERVER CERTIFICATE
```log
[0x8120002f][ssl][error] ssl-server ... : SSL library error: error:14094418:SSL routines:ssl3_read_bytes:tlsv1 alert unknown ca
[0x8120002f][ssl][error] ssl-server ... : SSL library error: error:140940E5:SSL routines:ssl3_read_bytes:ssl handshake failure
[0x80e0052e][xmlfirewall][debug] xmlfirewall ... : Starting SSL termination of connection 192.168.1.46:51967 192.168.1.10:9195
[0x81200025][ssl][error] ssl-server ... : SSL peer did not send a certificate during the handshake
```
In this scenario, it’s the client who was unable to verify the certificate presented by DataPower via the identification credentials. Certificate verification is a two way street and if the client doesn’t verify the server cert then they will close the connection and this error will occur. In previous firmware levels, you would just see the connection close. The rule of thumb was that the side of the connection that closed the connection will also be the one that writes the error in the local logs. Luckily in the new firmware, we get a hint that the client didn’t like our cert.