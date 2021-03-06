README

ref: http://pages.cs.wisc.edu/~zmiller/ca-howto/
ref: https://www.openssl.org/docs/apps/x509v3_config.html
ref: http://stackoverflow.com/questions/18233835/creating-an-x509-v3-user-certificate-by-signing-csr

1. Files list

 cakey.pem  Root CA private key
 cacert.pem     Root CA for cakey.pem
 ca2key.pem RSA private key
 ca2cert.pem    Second-level RSA cert for ca2key.pem
 dsakey.pem DSA private key
 dsacert.pem    Third level DSA cert for dsakey.pem
 rsakey.pem RSA private key
 rsacert.pem    Third level RSA cert for rsacert.pem
 hmackey.bin    HMAC key ('secret')
 expired.key    key for expired cert 
 expired.crt    expired certificate 

2. How certificates were generated:

 A. Create new CA 
    > CA.pl -newca
    > cp ./demoCA/cacert.pem .
    > cp ./demoCA/private/cakey.pem .
    > openssl x509 -text -in cacert.pem

 B. Generate RSA key and second level CA
    > openssl genrsa -out ca2key.pem
    > openssl req -new -key ca2key.pem -out ca2req.pem
    > openssl ca -cert cacert.pem -keyfile cakey.pem \
        -out ca2cert.pem -infiles ca2req.pem
    > openssl verify -CAfile cacert.pem ca2cert.pem

 C. Generate and sign DSA key with second level CA
    > penssl dsaparam -out dsakey.pem -genkey 512
    > openssl req -new -key dsakey.pem -out dsareq.pem
    > openssl ca -cert ca2cert.pem -keyfile ca2key.pem \
        -out dsacert.pem -infiles dsareq.pem
    > openssl verify -CAfile cacert.pem -untrusted ca2cert.pem dsacert.pem

 D. Generate and sign RSA key with second level CA
    > openssl genrsa -out rsakey.pem
    > openssl req -new -key rsakey.pem -out rsareq.pem
    > openssl ca -cert ca2cert.pem -keyfile ca2key.pem \
        -out rsacert.pem -infiles rsareq.pem
    > openssl verify -CAfile cacert.pem -untrusted ca2cert.pem rsacert.pem

3. Converting key and certs between PEM and DER formats

  Convert PEM private key file to DER file
  RSA key:
    > openssl rsa -inform PEM -outform DER -in rsapriv.pem -out rsapriv.der
  DSA key:
    > openssl dsa -inform PEM -outform DER -in dsakey.pem -out dsakey.der
  
  Convert PEM public key file to DER file
  RSA key:
    > openssl rsa -inform PEM -outform DER -pubin -pubout -in lugh.key -out lugh.der
  DSA key:
    > openssl dsa -inform PEM -outform DER -pubin -pubout -in lugh.key -out lugh.der
   
  If you aren't sure if the public key is RSA or DSA, just run one of
  the above commands, and the error messaging will make it clear :)
   
  Convert PEM cert file to DER file
    > openssl x509 -outform DER -in ca2cert.pem -out ca2cert.der 

4. Converting an unencrypted PEM or DER file containing a private key
   to an encrypted PEM or DER file containing the same private key but
   encrypted
     > openssl pkcs8 -in unencryptedfile.<der|pem> -inform <der|pem> -out encryptedfile.<der|pem> -outform <der|pem> -topk8

5. NSS is unfriendly towards standalone private keys.
   This procedure helps convert raw private keys into PKCS12 form that is 
   suitable for not only NSS but all crypto engines.

   5a.
       Input: DSA/RSA private key in PEM or DER format
       Output: A PKCS12 file containing the private key, and a self-signed 
               certificate with the corresponding public key
    
       # first convert key file to PEM format, if not already in that format
       > openssl <dsa|rsa> -inform der -outform pem -in key.der -out key.pem
    
       # answer questions at the prompt
       # Note: use a unique subject (=issuer) for each self-signed cert you 
       # create (since there is no way to specify serial # using the command 
       # below)
       > openssl req -new -keyform <der|pem> -key key.<der|pem> -x509 -sha1 -days 999999 -outform pem -out cert.pem
    
       # now using the cert and key in PEM format, conver them to a PKCS12 file
       # enter some password on prompt
       > openssl pkcs12 -export -in cert.pem -inkey key.pem -name <nickname> -out keycert.p12
    
       # This pkcs12 file can be used directly on the xmlsec command line, or
       # can be pre-loaded into the crypto engine database (if any).
    
       # In the case of NSS, you can pre-load the key using pk12util.
       # The key and cert will have the nickname "nickname" (used in above step)
       > pk12util -d <nss_config_dir> -i keycert.p12
    
   5b.
       Input: DSA/RSA private key in PEM or DER format
              KeyCert containing corresponding public key
              Other certs in the chain leading from KeyCert to the root
       Output: A PKCS12 file containing the private key, the KeyCert and the
               certs in the chain
    
       # first convert key file to PEM format, if not already in that format
       > openssl <dsa|rsa> -inform der -outform pem -in key.der -out key.pem

       # convert all cert files to PEM format, if not already in that format    
       > openssl x509 -inform der -outform pem -in cert.der -out cert.pem

       # concatenate all cert.pem files created above to 1 file - allcerts.pem
       > cat keycert.pem cert1.pem cert2.pem  .... > allcerts.pem

       # now using the certs and key in PEM format, conver them to a PKCS12 file
       # enter some password on prompt
       > openssl pkcs12 -export -in allcerts.pem -inkey key.pem -name <nickname of key & keycert> [-caname <nickname of cert1> -caname <nickname of cert2>.... ] -out keycert.p12
    
       # This pkcs12 file can be used directly on the xmlsec command line, or
       # can be pre-loaded into the crypto engine database (if any).
    
       # In the case of NSS, you can pre-load the key using pk12util.
       # The key and certs will have the nickname "nickname" 
       # (used in above step)
       > pk12util -d <nss_config_dir> -i keycert.p12
    

