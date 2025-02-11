.. _minio-sse:

=============================
Encryption and Key Management
=============================

.. default-domain:: minio

.. contents:: Table of Contents
   :local:
   :depth: 1

MinIO supports key security features around object-level and network-level
encryption:

Server-Side Object Encryption
-----------------------------

MinIO supports :ref:`Server-Side Object Encryption (SSE) <minio-sse>` of
objects, where MinIO uses a secret key to encrypt and store objects on disk
(encryption at-rest). MinIO SSE requires :ref:`minio-encryption-tls`.

MinIO supports the following SSE encryption types:

.. _minio-encryption-sse-s3:

SSE-S3
   The MinIO server en/decrypts an object using a secret key managed by
   an external Key Management System (KMS). MinIO SSE-S3 requires using
   :minio-git:`MinIO KES </kes>` for supporting scalable distributed
   cryptographic operations using the KMS.

   MinIO supports both automatic and client-driven encryption of objects 
   *before* storing the data to disk. MinIO automatically decrypts
   objects using the required keys on read for any client with
   :ref:`read permission <minio-policy>` for the bucket and object.
   
   MinIO can only decrypt an object if it can retrieve the 
   :ref:`Customer Master Key (CMK) <minio-encryption-sse-encryption-process>` 
   used to encrypt an object from the configured KES service.

   MinIO SSE-S3 provides similar functionality to
   Amazon :s3-docs:`Server-Side Encryption with Amazon-Managed Keys 
   <UsingServerSideEncryption.html>`. MinIO SSE-S3 supports multiple
   KMS providers in addition to AWS KMS, including Thales CipherTrust
   (formerly Gemalto KeySecure) and Hashicorp KeyVault.

.. _minio-encryption-sse-c:

SSE-C 
   The MinIO server en/decrypts an object using a secret encryption key
   provided by the application as part of the HTTP request. SSE-C
   requires the application to manage key creation and storage. MinIO
   never stores encryption keys specified as part of SSE-C.

   MinIO supports client-driven encryption of objects *before* storing
   the data to disk. MinIO automatically decrypts objects using a
   client-specified key on read. Only clients which specify the correct
   encryption key can successfully retrieve the protected object. 
   Clients must additionally have :ref:`read permission <minio-policy>`
   for the bucket and object.

   SSE-C encrypted objects are not compatible with MinIO 
   :ref:`bucket replication <minio-bucket-replication>`. For MinIO
   deployments configured for SSE-S3 automatic encryption,
   writing an object with SSE-C encryption headers prevents SSE-S3 
   encryption of that object.

   MinIO SSE-C provides similar functionality to 
   Amazon :s3-docs:`Server-Side Encryption with Customer Keys 
   <ServerSideEncryptionCustomerKeys.html>`.

.. _minio-encryption-sse-encryption-process:

Encryption Process Overview
~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO uses three distinct keys when performing server-side encryption or
decryption:

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Key
     - Description

   * - External Key (EK) 
     - An external secret key used to generate and encrypt additional encryption
       keys. 
      
       - For SSE-C, the EK is the client-provided encryption key. MinIO
         never stores this key to disk and requires the application to perform
         all key management operations.

       - For SSE-S3, the EK is generated by :minio-git:`MinIO KES </kes>` using
         a Customer Master Key (CMK) stored in the configured external KMS. 
         KES returns a plain-text representation of the EK for performing
         cryptographic operations, and a CMK-encrypted representation of the
         EK for storage alongside the object metadata.
         
         See the :minio-git:`KES Wiki </kes/wiki>` for more information on
         configuring KES.

       MinIO cannot decrypt objects encrypted using SSE-C or SSE-S3 without the
       associated EK or CMK respectively. Disabling or deleting these keys
       results in associated encrypted objects being rendered unreadable. See
       :ref:`minio-encryption-sse-secure-erasure-locking` for more information.

   * - Key Encryption Key (KEK)
     - A 256-bit unique random secret key deterministically generated by MinIO
       at the time of each cryptographic operation. MinIO never stores the KEK
       on disk.

       The key-derivation algorithm uses a pseudo-random function 
       (:ref:`PRF <minio-encryption-sse-primitives>`) that
       takes the EK, a randomly generated initialization vector, and
       a context consisting of values like the bucket and object name.

   * - Object Encryption Key (OEK) 
     - A 256-bit unique random secret key used to encrypt or decrypt an object.
       MinIO generates an OEK when first encrypting an object. MinIO encrypts
       the OEK using the KEK and stores the encrypted OEK as metadata with the
       object.

       MinIO decrypts the OEK into RAM as part of cryptographic operations. 
       MinIO never stores the OEK on disk in an unecrypted format.

All secret keys generated by MinIO are 256-bits long. MinIO uses a
cryptographically secure pseudo-random generator (CSPRNG) as part of key
generation. In the context of SSE, the MinIO CSPRNG only enforces that the
generated values are unique rather than truly random.

The following diagrams describe the key hierarchy built by MinIO
for each object. Each tab corresponds to the specific hierarchy for 
SSE-S3 and SSE-C respectively:

.. tab-set::
   
   .. tab-item:: SSE-S3

      .. image:: /images/minio-encryption-sse-s3-key-hierarchy.svg
         :width: 600px
         :alt: Key Hierarchy for MinIO SSE-S protected objects. Customer Master Key  to  Encryption Key  to Key Encryption Key  to Object Encryption Key. Only encrypted key metadata is kept on disk.
         :align: center

   .. tab-item:: SSE-C

      .. image:: /images/minio-encryption-sse-c-key-hierarchy.svg
         :width: 600px
         :alt: Key Hierarchy for MinIO SSE-S protected objects. Customer Master Key  to  Encryption Key  to Key Encryption Key  to Object Encryption Key. Only encrypted key metadata is kept on disk.
         :align: center

.. _minio-encryption-sse-content-encryption:

Content Encryption
~~~~~~~~~~~~~~~~~~

The MinIO server uses an authenticated encryption scheme 
(:ref:`AEAD <minio-encryption-sse-primitives>`) to en/decrypt and authenticate
the object content. The AEAD is combined with some state to build a 
**Secure Channel**. A Secure Channel is a cryptographic construction that
ensures confidentiality and integrity of the processed data. In particular, the
Secure Channel splits the plaintext content into fixed size chunks and
en/decrypts each chunk separately using an unique key-nonce combination.

The following text diagram illustrates Secure Channel Construction of an
encrypted object:

The Secure Channel splits the object content into chunks of a fixed size of
``65536`` bytes. The last chunk may be smaller to avoid adding additional
overhead and is treated specially to prevent truncation attacks. The nonce
value is ``96`` bits long and generated randomly per object / multi-part part.
The Secure Channel supports plaintexts up to ``65536 * 2^32 = 256 TiB``.

For S3 multi-part operations, each object part is en/decrypted with the Secure
Channel Construction scheme shown above. For each part, MinIO generates a secret
key derived from the Object Encryption Key (OEK) and the part number using a
pseudo-random function (:ref:`PRF <minio-encryption-sse-primitives>`), such that
``key = PRF(OEK, part_id)``.

.. _minio-encryption-sse-primitives:

Cryptographic Primitives
~~~~~~~~~~~~~~~~~~~~~~~~

The MinIO server uses the following cryptographic primitive implementations:

.. list-table::
   :stub-columns: 1
   :widths: 40 60
   :width: 100%

   * - Pseudo-Random Functions (PRF) 
     - HMAC-SHA-256 

   * - :ref:`AEAD <minio-encryption-sse-content-encryption>` 
     - ``ChaCha20Poly-1305`` by default. 
     
       ``AES-256-GCM`` for x86-64 CPUs with the AES-NI extension.

.. _minio-encryption-sse-secure-erasure-locking:

Secure Erasure and Locking
~~~~~~~~~~~~~~~~~~~~~~~~~~

For SSE-S3 encrypted objects, MinIO requires access to both the KMS *and* the
:ref:`Customer Master Key (CMK) <minio-encryption-sse-encryption-process>` used
to encrypt the object(s). You can leverage this functionality to erase or lock
some or all encrypted objects by disabling access to the EK or CMK used for
SSE-S3 encryption. For example:

- Seal the KMS such that it cannot be accessed by MinIO server anymore. This
  locks all SSE-S3 encrypted objects protected by any CMK stored on the KMS. The
  encrypted objects remain unreadable as long as the KMS remains sealed.

- Seal/Unmount one/some CMK(s). This locks all SSE-S3 encrypted objects
  protected by the CMK(s). The encrypted objects remain unreadable as long
  as the CMK(s) remains sealed.

- Delete one/some CMK(s). This renders all SSE-S3 encrypted objects protected
  by the CMK(s) as permanently unreadable. This is equivalent to securely
  erasing all SSE-S3 encrypted objects protected by the CMK(s). The combination
  of deleting CMK(s) and deleting the data may fulfill regulatory requirements
  around secure deletion of data.

  Deleting a master key is typically irreversible. Exercise extreme caution
  before intentionally deleting a master key.

For SSE-C encrypted objects, MinIO requires access to the encryption key
used to secure those objects. You can render an encrypted object as temporarily
or permanently unreadable by disabling or deleting the 
:ref:`secret key <minio-encryption-sse-encryption-process>` used to encrypt
that object. 

.. _minio-encryption-tls:

Transport Layer Security (TLS)
------------------------------

MinIO supports :ref:`Transport Layer Security (TLS) <minio-TLS>` encryption of
incoming and outgoing traffic. 

TLS is the successor to Secure Socket Layer (SSL) encryption. SSL is fully
`deprecated <https://tools.ietf.org/html/rfc7568>`__ as of June 30th, 2018. 
MinIO uses only supported (non-deprecated) TLS protocols (TLS 1.2 and later).

See :ref:`Transport Layer Security (TLS) <minio-TLS>`
for more complete instructions on configuring MinIO for TLS.

.. toctree::
   :titlesonly:
   :hidden:

   /security/encryption/server-side-encryption-sse-s3
   /security/encryption/server-side-encryption-sse-c
   /security/encryption/transport-layer-security
