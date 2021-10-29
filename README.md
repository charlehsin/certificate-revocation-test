# Steps to use openssl to test certificate revocation

1. Download Opeenssl executable from https://wiki.openssl.org/index.php/Binaries and https://bintray.com/vszakats/generic/openssl/1.1.1d .

2. Follow https://docs.microsoft.com/en-us/iis/manage/creating-websites/scenario-build-a-static-website-on-iis to use IIS to serve a file.
   - Also set the site's MIME type maping for .pem file with text/plain mimeType.
   - The web site folder is at website.
   - Test that we can serve http://localhost/intermediatecrl.pem

3. Check and follow https://jamielinux.com/docs/openssl-certificate-authority/introduction.html
   - Create key
      - cd ca
      - openssl\openssl.exe genrsa -aes256 -out private\cakey.pem 4096
      - (passphrase : secretpassword)
   - Create root cert
      - openssl\openssl.exe req -config openssl.cnf -key private\cakey.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs\cacert.pem
   - Verify root cert
      - openssl\openssl.exe x509 -noout -text -in certs\cacert.pem
   - Create intermediate key
      - openssl\openssl.exe genrsa -aes256 -out intermediate\private\intermediatekey.pem 4096
      - (passphrase : secretpassword)
   - Create intermediate cert
      - openssl\openssl.exe req -config intermediate\openssl.cnf -new -sha256 -key intermediate\private\intermediatekey.pem -out intermediate\csr\intermediatecsr.pem
      - openssl\openssl.exe ca -config openssl.cnf -extensions v3_intermediate_ca -days 3650 -notext -md sha256 -in intermediate\csr\intermediatecsr.pem -out intermediate\certs\intermediatecert.pem
   - Verify intermediate cert
      - openssl\openssl.exe x509 -noout -text -in intermediate\certs\intermediatecert.pem
      - openssl\openssl.exe verify -CAfile certs\cacert.pem intermediate\certs\intermediatecert.pem
   - Create chain file
      - type intermediate\certs\intermediatecert.pem certs\cacert.pem > intermediate\certs\cachaincert.pem
      - type intermediate\certs\cachaincert.pem
   - Create leaf key
      - openssl\openssl.exe genrsa -aes256 -out intermediate\private\leafkey.pem 2048
      - (passphrase : secretpassword)
   - Create a leaf server cert 
      - openssl\openssl.exe req -config intermediate\openssl.cnf -key intermediate\private\leafkey.pem -new -sha256 -out intermediate\csr\leafcsr.pem
      - (values use the web site tutorial's values)
      - openssl\openssl.exe ca -config intermediate\openssl.cnf -extensions server_cert -days 375 -notext -md sha256 -in intermediate\csr\leafcsr.pem -out intermediate\certs\leafcert.pem
   - Verify the leaf server cert
      - openssl\openssl.exe x509 -noout -text -in intermediate\certs\leafcert.pem
   - Verify the chain of trust
      - openssl\openssl.exe verify -CAfile intermediate\certs\cachaincert.pem intermediate\certs\leafcert.pem

4. Check and follow https://jamielinux.com/docs/openssl-certificate-authority/certificate-revocation-lists.html
   - Update the intermediate cert's config file to add CRL for server cert
      - Follow https://jamielinux.com/docs/openssl-certificate-authority/certificate-revocation-lists.html#prepare-the-configuration-file to update the intermediate\openssl.cnf   
   - Create the CRL file
      - openssl\openssl.exe ca -config intermediate\openssl.cnf -gencrl -out intermediate\crl\intermediatecrl.pem
   - Check the CRL file
      - openssl\openssl.exe crl -in intermediate\crl\intermediatecrl.pem -noout -text
      - (no revoked cert yet)
   - create a leaf server cert 
      - openssl\openssl.exe req -config intermediate\openssl.cnf -key intermediate\private\leafkey.pem -new -sha256 -out intermediate\csr\leaf2csr.pem
      - (values use the web site tutorial's values)
      - openssl\openssl.exe ca -config intermediate\openssl.cnf -extensions server_cert -days 375 -notext -md sha256 -in intermediate\csr\leaf2csr.pem -out intermediate\certs\leaf2cert.pem
   - Verify the leaf server cert
      - openssl\openssl.exe x509 -noout -text -in intermediate\certs\leaf2cert.pem
      - (the CRL distribution point should be there)
   - Verify the chain of trust
      - openssl\openssl.exe verify -CAfile intermediate\certs\cachaincert.pem intermediate\certs\leaf2cert.pem
   - Copy the intermediate\crl\intermediatecrl.pem file to the web server

5. Check and follow similar things at https://raymii.org/s/articles/OpenSSL_manually_verify_a_certificate_against_a_CRL.html
   - This is basically just use crl's pem file to check revocation. This is not using CRL distribution point yet.
   - Combine crl and chain
      - type intermediate\certs\cachaincert.pem intermediate\crl\intermediatecrl.pem > intermediate\certs\crl_chain.pem 
   - Verify crl
      - openssl\openssl.exe verify -crl_check -CAfile intermediate\certs\crl_chain.pem intermediate\certs\leaf2cert.pem 
      - (this passes since there is no revocation yet.)

6. Use Windows certutil tool at https://sysadmins.lv/retired-msft-blogs/pki/basic-crl-checking-with-certutil.aspx
   - Check the CRL distribution point to serve the file
      - certutil -URL http://localhost/intermediatecrl.pem
      - (This is fine. no entry in revocation list.)
   - Verify the cert using the CRL distribution point
      - certutil -f –urlfetch -verify intermediate\certs\leaf2cert.pem
      - (This fails with "Incomplete cert chain - cannot find intermediate CA cert")
   - Install roo CA and intermediate CA to local machine cert store 
      - openssl\openssl.exe x509 -in intermediate\certs\intermediatecert.pem -inform PEM -out intermediate\certs\intermediatecert.crt
      - Then install intermediate\certs\intermediatecert.crt to local machine's intermediate CA store.
      - openssl\openssl.exe x509 -in certs\cacert.pem -inform PEM -out certs\cacert.crt
      - Then install certs\cacert.crt to local machine's trusted root CA store.
   - Verify the cert using the CRL distribution point
      - certutil -f –urlfetch -verify intermediate\certs\leaf2cert.pem
      - (This passes. "Leaf certificate revocation check passed.")

7. Revoke a cert. Follow https://jamielinux.com/docs/openssl-certificate-authority/certificate-revocation-lists.html#revoke-a-certificate .
   - Revoke the cert
      - openssl\openssl.exe ca -config intermediate\openssl.cnf -revoke intermediate\certs\leaf2cert.pem
   - Create the CRL file
      - openssl\openssl.exe ca -config intermediate\openssl.cnf -gencrl -out intermediate\crl\intermediatecrl.pem
   - Check the CRL file
      - openssl\openssl.exe crl -in intermediate\crl\intermediatecrl.pem -noout -text
      - (There is 1 revoked cert now.)
   - Copy the intermediate\crl\intermediatecrl.pem file to the web server
   - Check the CRL distribution point to serve the file
      - certutil -URL http://localhost/intermediatecrl.pem
      - (There is 1 entry in the revocation list now.)
   - Verify the cert using the CRL distribution point
      - certutil -f –urlfetch -verify intermediate\certs\leaf2cert.pem
      - (This works, and it says that the cert is revoked.)

