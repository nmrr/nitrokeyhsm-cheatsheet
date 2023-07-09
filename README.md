# nitrokeyhsm-cheatsheet

This document bas been made for **Nitrokey HSM 2** but these commands will work for every PKCS11 HSM.

 
List detected smartcard (HSM is considered as a smartcard) and associated object of smartcard: 

```
pkcs15-tool -D
```

Reset the HSM:

```
sc-hsm-tool --initialize --so-pin *XXXX* --pin *YYYY*
```
The HSM will be reinitialized. Evey keys will be erase.

Change the so-pin:

```
pkcs11-tool --module /usr/lib/[...]/pkcs11/opensc-pkcs11.so --login --login-type so --so-pin *XXXX* --change-pin --new-pin *YYYY*
```
**If the so-pin is lost, you must reflash your NitroKey and all keys will be lost !**

Generate an RSA keypair of 2048 bits with 1 as ID and "mynewkew" as label:

```
pkcs11-tool --module /usr/lib/[...]/pkcs11/opensc-pkcs11.so --login --keypairgen --key-type rsa:2048 --id 1 --label mynewkey
```
**ID** must be unique for each object. You can check with **pkcs15-tool -D** command if the chosen ID is not already used:


Read the public key of the just created RSA key (ID of the key must be passed):

```
pkcs15-tool --read-public-key 1
```
Output key is encored in **PEM**. Private key cannot be extracted.

Read the public key and decode the key with **openssl**:

```
pkcs15-tool --read-public-key 1 | openssl rsa -noout -text -inform PEM -pubin
```

List **URI** of smartcard :

```
p11tool --list-all
```
All **URI** start with **pkcs11:model=...**

List **URI** of objects that are stored in the **HSM**:

```
p11tool --login --list-all *URI*
```
**URI** is needed to perform action with the associate object (encrypt data with a stored key for example)


To encrypt a file with a RSA public key:

```
openssl rsautl -encrypt -engine pkcs11 -keyform engine -inkey *URI* -in data.txt -out data_encrypted.txt
```
**URI** must end with **type=public**

Note: This action can be done directly on the computer without using the HSM. A public key is public and can be shared with everyone

To decrypt the previous encrypted file with the private key:
```
openssl rsautl -decrypt -engine pkcs11 -keyform engine -inkey *URI* -in data_encrypted.txt -out data_clear.txt
```
**URI** must end with **type=private**

Length of data that can be encrypted by using RSA is limited by the size of the RSA key (256 byte for RSA-2048). Furthermore, padding will reduce the maximum length of data by 8 byte.


## PKI

Generate an RSA keypair:

```
pkcs11-tool --module /usr/lib/[...]/pkcs11/opensc-pkcs11.so --login --keypairgen --key-type rsa:2048 --id *ID* --label CA
```

Generate the **certification authority**:

```
openssl req -new -x509 -days 3650 -subj '/CN=CA-2023/' -sha256 -engine pkcs11 -keyform engine -key *URI* -out CA_2023.pem
```

**URI** must be the just generated private key.

Sign a certificate request (CSR file):

```
openssl x509 -req -CAkeyform engine -engine pkcs11 -in mycsr.csr -days 365 -CA CA-2023.pem -CAkey *URI* -set_serial *serial_number* -sha256 -extensions req_ext -extfile extension.cnf -out mycert.pem
```

Example of extension file:
```
[req]
distinguished_name = req_distinguished_name
req_extensions = req_ext
prompt = no

[req_distinguished_name]
CN  = abcd.local

[req_ext]
subjectAltName = @alt_names

[alt_names]
IP.1 = 127.0.0.1
DNS.1 = abcd.local
```
