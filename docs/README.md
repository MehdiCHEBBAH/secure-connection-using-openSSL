<div style="text-align:center;font-size:30px">﷽</div>







<div style="text-align:center; font-size: 48px">Implementing a secure exchange protocol with OpenSSL</div>









Work done by  **CHEBBAH Mehdi**

---



# Table of Contents

[TOC]

---

## Introduction

In this tutorial we will implement a hybrid secure exchange protocol that guarantees the following security services:

+ Confidentiality of the exchange .
+ Integrity of the exchanged data.
+ Non-repudiation of the sender.

For this we will try to implement the system whose design has been previously established: 

![](assets/DeepinScreenshot_select-area_20191023073446.png)



## Implementation

Concerning this part, we will divide it into 3 phases:

#### Phase 1: Creation of key pairs

In this phase each of the two protagonists will create their own key pair a public key and a private key (`PKi` and `SKi`)

##### 1. Script for creating key pairs

```bash
#!/bin/bash

# Ask the user to type the password
read -sp 'Entrez le mot de pass: ' mypass

# Generate a private key for RSA (skA.pem) then encrypt it using the DES algorithm and the password 'mypasswd'
openssl genrsa -out skA.pem -passout pass:$mypass -des 512

# Generation of the public key (pkA.pem) associated with the private key (skA.pem)
openssl rsa -in skA.pem -passin pass:$mypass -out pkA.pem -pubout
```

###### Obtained results:

1. Creation of two files `skA.pem` and `pkA.pem` in the current directory (of Alice)

Re-execute the script in Bob's directory changing `skA.pem` and `pkA.pem` to `skB.pem` and `pkB.pem` respectively.

###### Note:

+ Each one can publish his public key in the unsecured channel. (copy the file `pkX.pem`)

------------------

#### Phase 2: Session key exchange

In the opening of the session Alice will randomly generate a session key that will be communicated to Bob, this key will be encrypted using Bob's public key, the signature will be guaranteed in this transfer (this is the role of asymmetric encryption in this hybrid system)

##### 1. Session key generation and encryption script

```bash
#!/bin/bash

# Generation of a random key of 64 bytes then save it in the kAB file
openssl rand 16 > kAB

# Encrypting the kAB file with Bob's public key 
openssl rsautl -in kAB -out kAB.crypt -inkey pkB.pem -pubin -encrypt

# Hash the kAB key with MD5 and write the result in binary format.
openssl dgst -md5 -binary -out kAB.md5 kAB

# Signature of the kAB hash by Alice's private key
openssl rsautl -in kAB.md5 -out kAB.md5.sign -sign -inkey skA.pem

# Delete kAB.md5 (temporary file)
rm -f kAB.md5
```

###### Obtained results:

1. Create a `kAB` file containing the session key.
2. Creation of a `kAB.crypt` file containing the `kAB` cryptogram.
3. Creation of a `kAB.md5.sign` file containing the session key's hash signature.

###### Note:

The two files (`kAB.crypt` and `kAB.md5.sign`) will be sent to Bob and deleted from Alice's directory



##### 2. Decryption script and verification of non-repudiation and integrity of the session key

```bash
#!/bin/bash

# Decryption of the received file kAB.crypt by the private key (skB.pem) 
openssl rsautl -decrypt -in kAB.crypt -out kAB -inkey skB.pem

# Hash of the kAB with MD5.
openssl dgst -md5 -binary -out kAB1.md5 kAB

# Verification of the signature of the received files using the public key of Alice 
openssl rsautl -in kAB.md5.sign -out kAB2.md5 -pubin -inkey pkA.pem

# Comparison of the result of the received kAB Hash and the sent kAB Hash
var1=$(cat kAB1.md5)
var2=$(cat kAB2.md5)
if [ "$var1" == "$var2" ]
	then
		# Deleting temporary files
		rm -f kAB.crypt kAB1.md5 kAB.md5.sign kAB2.md5
		echo "La cle a ete correctement transmis."
		# Display the message 'Press any key to exit' and wait for a character
		read -n 1 -s -r -p "Press any key to exit"
		exit 0 # Exit with success code
	else 
		# Deleting temporary files
		rm -f kAB.crypt kAB1.md5 kAB.md5.sign kAB2.md5
		echo "Attention, la cle a ete modifie."
		# Display the message 'Press any key to exit' and wait for a character
		read -n 1 -s -r -p "Press any key to exit"
		exit 1 # Exit with failure code
fi
```

###### Obtained results:

1. Creation of a `kAB` file containing the session key sent by Alice 
2. display a success message ('The key has been correctly transmitted.') in case the transmission is correct (the received session key is the same as the one sent). Or the display of a failure message ('Attention, the key has been modified.') in case the sent file and the received file are different.

------------------

#### Phase 3: Exchange of messages

After the establishment of the two previous phases it is now possible to send messages between Alice and Bob using symmetric encryption for confidentiality and private key encryption for signature

##### 1. Messages encryption script

```bash
#!/bin/bash

# Encryption of the message (Message.txt) with the symmetric key (kAB)
openssl enc -des-cbc -in Message.txt -out Message.crypt -pass file:kAB

# Hash of the message (Message.txt) by the MD5 algorithm
openssl dgst -md5 -binary -out Message.md5 Message.txt

# Signature of the message hash with the private key of Alice
openssl rsautl -in Message.md5 -out Message.md5.sign -sign -inkey skA.pem

# Deleting intermediate files
rm -f Message.md5
```

###### Obtained results:

1. Creation of the file `Message.crypte` containing the message encrypted with the session key kAB.
2. Creation of the file `Message.md5.sign` containing the signature of the message hash

###### Note:

The two files (`Message.crypt` and `Message.md5.sign`) will be sent to Bob and then deleted from Alice's directory



##### 2. Messages decryption script 

```bash
#!/bin/bash

# Decryption of the received file Message.crypt by the kAB session key using the DES algorithm and the CBC strategy
openssl enc -in Message.crypt -out Message.txt -pass file:kAB -d -des-cbc

# Verification of the signature of the files received using Alice's public key
openssl rsautl -in Message.md5.sign -out Msg1.md5 -pubin -inkey pkA.pem

# Hash the message 
openssl dgst -md5 -binary -out Msg2.md5 Message.txt

# Comparison of the result of the received kAB Hash and the sent kAB Hash
var1=$(cat Msg1.md5)
var2=$(cat Msg2.md5)
if [ "$var1" == "$var2" ]
	then
		# Deleting temporary files
		rm -f Message.crypt Msg1.md5 Message.md5.sign Msg2.md5
		echo "Le message est transmi avec succe. Vous pouvez le consuler dans Message.txt"
		# Display the message 'Press any key to exit' and wait for a character
		read -n 1 -s -r -p "Press any key to exit"
		exit 0 # Exit with success code
	else 
		# Deleting temporary files
		rm -f Message.crypt Msg1.md5 Message.md5.sign Msg2.md5
		echo "Echec du transmission du message (Le message a ete modiffier)."
		# Display the message 'Press any key to exit' and wait for a character
		read -n 1 -s -r -p "Press any key to exit"
		exit 1 # Exit with failure code
fi
```

###### Obtained results:

1. Creation of the file `Message.txt` containing the message received from Alice.
2. Display a success message ('The message is transmitted with success. You can read it in `Message.txt`') in case the transmission is correct (the received message is the same as the sent message). Or the display of a failure message ('Message transmission failed (The message has been modified).') in case the sent message and the received message are different.

###### Note:

Bob can now read the message sent by Alice.
