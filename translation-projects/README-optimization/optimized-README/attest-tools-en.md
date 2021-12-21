# attest-tools

## INTRODUCTION

Managing the complete lifecycle of remote attestation can be very complicated. A TSS library performs only part of the operations necessary for the creation and verification of TPM objects. Other open-source solutions, such as IBM Attestation Client-Server (ACS), have fixed purposes and for this reason, they are difficult to customize and integrate into other solutions.

attest-tools is a set of libraries and tools that can be used for managing the complete lifecycle of remote attestation. Its advantages are:

- library API: main functions are exposed as a library: applications wanting to provide Trusted Computing services can simply link this library;
  
- modular design: variable parts, such as event log parsers or verifiers, are modularized; attest-tools functionality can be easily extended with third-party modules that support new event log format or new verifiers;
  
- simplicity: data to be processed and the status of the processing are stored in contexts created by the applications; library functions directly operate on these contexts;
  
- completeness: the library provides functions for all phases of the remote attestation lifecycle; it supports both implicit and explicit remote attestation.



## EXPLICIT vs IMPLICIT ATTESTATION

Explicit attestation means establishing a communication between a target system and a verifier to determine whether the former satisfies the requirements of the latter. It is called explicit because the goal of the communication is explicitly to attest to a system.

Implicit attestation means that attestation is not the primary purpose of the communication, but requirements can be verified as part of the establishment or execution of another protocol (e.g. TLS). This alternative should be preferable to the explicit one, as it is easier to integrate into legacy products. On the other end, implicit attestation introduces additional challenges such as having a fixed representation of the system state that can be translated into one or multiple Platform Configuration Register values.

attest-tool allows verifiers to perform both explicit and implicit attestation.



## SOFTWARE ARCHITECTURE

attest-tools are composed of different libraries, RA client/server, and TLS client/server.


                   +---------------------+ +---------------------+
                   | attest-tools client | | attest-tools server |
                   +---------------------+ +----------------------
      +----------+ +---------------------+ +---------------------+
      | SKAE lib | |  enroll client lib  | |  enroll server lib  |
      +----------+ +---------------------+ +---------------------+
    +-------------------------------------------------------------------------+
    |                     Application API (north-bound)                       |
    | +--------------------+ +---------------------+ +------+ +-----+ +-----+ |
    | |                    | |                     | | util | | pcr | | ctx | |
    | |     Event log      | |     Verifier        | +------+ +-----+ +-----+ |
    | |                    | |                     | +----------+ +-----+     |
    | |                    | |                     | | ctx_json | | tss |     |
    | +--------------------+ +---------------------+ +----------+ +-----+     |
    |                                                        base lib         |
    |                      Developer API (south-bound)                        |
    +-------------------------------------------------------------------------+
      +--------------------+ +---------------------+
      | event log parsers  | | event log verifiers |
      |     (plugins)      | |      (plugins)      |
      +--------------------+ +---------------------+


### Base lib - libattest.so

The base library is the main library of attest-tool and provides basic services for the other more complex libraries. Essentially, it is
responsible to manage the data and verifier contexts, and verifying the TPM structure TPMS_ATTEST (which might contain a quote or a certified info). The base library exposes a north-bound interface (Application API), intended to be used by applications, and a south-bound interface (Developer API), intended to be used by developers willing to extend the library functionality.


### SKAE lib - libskae.so

The SKAE library is the library that manages the Subject Key Attestation Evidence X.509 extension defined by TCG:

https://trustedcomputinggroup.org/wp-content/uploads/IWG_SKAE_Extension_1-00.pdf

It offers functions for the creation and the verification of the data inside the extension.


### Enroll client lib - libenroll_client.so

The enrollment library for clients is responsible for performing the preliminary steps necessary for remote attestation: obtaining an AK certificate from the Privacy CA (implemented by attest_tools_server), for obtaining a certificate for generated TPM keys (which can be used with openssl_tpm2_engine), and for verifying a quote.


### Enroll server lib - libenroll_server.so

The enrollment library for servers is responsible to verify the requests sent by clients and issuing certificates. The function to issue certificates can be replaced with functions from applications' developers.


### Event log parsers - libeventlog_*.so

These libraries implement parsers for event logs. Currently, the following parsers are available:

- BIOS event log (TCG v1.2 and v2.0);

- IMA event log (ima-ng and ima-sig templates, little endian).


### Event log verifiers - libverifier_*.so

These libraries implement verifiers for event logs. Each verifier might take requirements from remote attestation verifiers, that must be satisfied by the system being attested. Currently, the following verifiers are available:

- BIOS: no checks are done at the moment; verifiers must explicitly specify 'always-true' as a requirement to skip verification;
  
- IMA boot aggregate: calculates the digest of PCRs 0-7 obtained by simulating the PCR extend operation with digests from the provided event logs, and compares the result with the file digest in the first entry of the IMA measurement list;
  
- IMA cp: obtains the path of the files from the IMA measurements list that don't have a signature, so that they can be transferred from systems being attested to remote attestation verifiers;
  
- IMA sig: verifies the signature of files by using the content of /etc/keys/x509_ima.der (must be provided) as certificate; verifier must accept the public key by providing the certificate common name as a requirement (in the future, it must check the validity of the certificate);
  
- IMA policy: checks that the loaded IMA policy is one of the pre-defined types (currently, only the 'exec-policy' type is defined); the desired policy type must be specified by the verifier as a requirement;
  
- EVM key: checks that the EVM key is securely generated by the TPM (this probably will change, as it seems that the TCG specification doesn't allow the sensitive data origin bit to be set for sealed data blobs); it also checks the PolicyPCR policy by using the event logs supplied by the system being attested and the PCR selection provided by the verifier as a requirement.


### RA client - attest_ra_client

Contacts RA server for AK/TLS certificate and for verifying a quote. Iusesse TCP/IP for communication.


### RA server - attest_ra_server

Processes requests from RA client It uses TCP/IP for communication.


### TLS client - attest_tls_client

It establishes a TLS communication with the TLS server. Before establishing TLS, it exchanges attestation data with the TLS server, so that both client and server certificates (the SKAE extension) can be verified.


### TLS server - attest_tls_server

It receives requests from the TLS client. Before establishing TLS, it exchanges attestation data with the TLS clients, so that both client and server certificates (the SKAE extension) can be verified.



### DATA AND VERIFIER CONTEXTS

Passing data required to perform remote attestation might not be always possible. For example, to perform the verification of the SKAE extension in a certificate, an application might pass the skae_callback() function as an argument to SSL_CTX_set_verify(). OpenSSL defines the callback function, and new parameters cannot be introduced.

To overcome this issue, the data and verifier contexts concept has been defined. A data context is a data structure that contains a linked list of data pointers and length for each type of defined information (for example a Privacy CA certificate, or the public part of an AK). The reader can have a look to include/ctx.h for more details.

A verifier context contains the list of requirements provided by a remote attestation verifier (these requirements are passed to the verifier's plugins mentioned above). It also contains a log for each verification step executed and its status. If a verification step failed, the first log contains the reason for the failure, while previous logs (created by the callers of the failed function) have as reason '<called function> failed'.

For the data context, the base library provides functions to add binary data from buffers and files. It also provides functions to import/print JSON strings. Support for other data formats might be provided.

For the verifier context, the base library provides functions to set verifier requirements, to set the mask of PCRs to check, and to print the logs with the list and result of the executed verification steps.

An application might pass a NULL pointer as data or verifier context. In this case, a global context (defined in the library) is used. This is the only way to use the skae_callback() for OpenSSL, as it is not possible to add a data and verifier context as parameters.



## USE CASES

This section provides an overview of how to attest tools can accomplish the most common tasks. The reader can modify these examples to implement his scenario.


### Create an AK and request a certificate:

#### Preliminary Steps (on the server)
1) use existing CA or generate a new custom CA by executing:
```
$ generate_demoCA demoCA
```
Update /etc/ssl/openssl.cnf, with:
```
[ CA_default ]
dir = <absolute path of demoCA>
...
unique_subject = no
...
copy_extensions = copy
...
input_password = <password>
```
2) configure a TPM (a software TPM is sufficient) 
3) install openssl_tpm2_engine  

#### Preliminary Steps (on the client)
The following steps assume that a software TPM is provisioned by libvirt for the VM.

1) Copy swtpm certificates from /var/lib/swtpm-localca in the host to the VM:
```
cp issuercert.pem swtpm-localca-rootca-cert.pem /etc/attest-tools/ek_ca_certs
```

#### Steps (on the server)
1) execute:
```
$ attest_ra_server -r /etc/attest-tools/req_examples/req-dummy.json
```

#### Steps (on the client)
1) execute:
```
$ attest_ra_client -a -s <attest_server FQDN>
```

### Create a TPM key not bound to any PCR, save attestation data to attest.txt, and request a certificate:

#### Steps (on the client)
1) execute:
```
$ attest_ra_client -k -s <attest_server FQDN> -r attest.txt
```

### Perform implicit RA:

#### Steps (on the server)
1) execute:
```
$ attest_tls_server -e -k /etc/attest-tools/tls_key.pem \
  -c /etc/attest-tools/tls_key_cert.pem \
  -d /etc/attest-tools/tls_key_ca_cert.pem -a attest.txt
```

#### Steps (on the client)
1) execute:
```
$ attest_tls_client -S -V -s <attest_server FQDN> \
  -d /etc/attest-tools/tls_key_ca_cert.pem \
  -r /etc/attest-tools/req_examples/req-dummy.json
```

### Create a TPM key bound to PCRs 0-9,10 and request a certificate:

#### Preliminary Steps (on the client)
1) ensure that the client has a BIOS event log accessible from /sys/kernel/security/tpm0/binary_bios_measurements  
2) ensure that the client has an IMA event log accessible from  /sys/kernel/security/ima/binary_runtime_measurements and that no IMA policy is loaded  

#### Steps (on the server)
1) execute:
```
$ attest_ra_server -r /etc/attest-tools/req_examples/req-bios-ima.json \
  -p 0,1,2,3,4,5,6,7,8,9,10
```

#### Steps (on the client)
1) run:
```
$ attest_ra_client -k -s <attest_server FQDN> -r attest.txt -b -i \
  -p 0,1,2,3,4,5,6,7,8,9,10
```

### Perform implicit RA:

#### Steps (on the server)
1) execute:

```
$ attest_tls_server -e -k /etc/attest-tools/tls_key.pem \
  -c /etc/attest-tools/tls_key_cert.pem \
  -d /etc/attest-tools/tls_key_ca_cert.pem -a attest.txt
```

#### Steps (on the client)
1) execute:
```
$ attest_tls_client -S -V -s <attest_server FQDN>
  -d /etc/attest-tools/tls_key_ca_cert.pem \
  -r /etc/attest-tools/req_examples/req-bios-ima.json \
  -p 0,1,2,3,4,5,6,7,8,9,10
```

### Perform explicit RA:

#### Steps (on the client)
1) run:
```
$ attest_ra_client -q -s <attest_server FQDN> -b -i \
  -p 0,1,2,3,4,5,6,7,8,9,10
```

### Update PCR and perform again explicit RA:

#### Steps (on the client)
1) run:
```
$ tsspcrextend -halg sha1 -ha 10 -ic "test"
$ attest_ra_client -q -s <attest_server FQDN> -b -i \
  -p 0,1,2,3,4,5,6,7,8,9,10
```

This time RA should fail.
