========================
Temporary URL middleware
========================

To discover whether your Object Storage system supports this feature,
check with your service provider or send a **GET** request using the ``/info``
path.

A temporary URL gives users temporary access to objects. For example, a
website might want to provide a link to download a large object in
Object Storage, but the Object Storage account has no public access. The
website can generate a URL that provides time-limited **GET** access to
the object. When the web browser user clicks on the link, the browser
downloads the object directly from Object Storage, eliminating the need
for the website to act as a proxy for the request.

Ask your cloud administrator to enable the temporary URL feature. For
information, see `Temporary
URL <http://docs.openstack.org/havana/config-reference/content/object-storage-tempurl.html>`__
in the *OpenStack Configuration Reference*.

Note
~~~~

To use **POST** requests to upload objects to specific Object Storage
locations, use :doc:`form_post_middleware` instead of temporary URL middleware.

Temporary URL format
~~~~~~~~~~~~~~~~~~~~

A temporary URL is comprised of the URL for an object with added query
parameters:

**Example Temporary URL format**

.. code::

    https://swift-cluster.example.com/v1/my_account/container/object
    ?temp_url_sig=da39a3ee5e6b4b0d3255bfef95601890afd80709
    &temp_url_expires=1323479485
    &filename=My+Test+File.pdf

The example shows these elements:


**Object URL**: Required. The full path URL to the object.

**temp\_url\_sig**: Required. An HMAC-SHA1 cryptographic signature that defines
the allowed HTTP method, expiration date, full path to the object, and the
secret key for the temporary URL.

**temp\_url\_expires**: Required. An expiration date as a UNIX Epoch timestamp,
which is an integer value. For example, ``1390852007`` represents
``Mon, 27 Jan 2014 19:46:47 GMT``.

For more information, see `Epoch & Unix Timestamp Conversion
Tools <http://www.epochconverter.com/>`__.

**filename**: Optional. Overrides the default file name. Object Storage
generates a default file name for **GET** temporary URLs that is based on the
object name. Object Storage returns this value in the ``Content-Disposition``
response header. Browsers can interpret this file name value as a file
attachment to be saved.

.. _secret_keys:

Secret Keys
~~~~~~~~~~~

The cryptographic signature used in Temporary URLs and also in
:doc:`form_post_middleware` uses a secret key. Object Storage allows you to
store two secret key values per account, and two per container. When validating
a request, Object Storage checks signatures against all keys. Using two keys at
each level enables key rotation without invalidating existing temporary URLs.

To set the keys at the account level, set one or both of the following
request headers to arbitrary values on a **POST** request to the account:

-  ``X-Account-Meta-Temp-URL-Key``

-  ``X-Account-Meta-Temp-URL-Key-2``

To set the keys at the container level, set one or both of the following
request headers to arbitrary values on a **POST** or **PUT** request to the
container:

-  ``X-Container-Meta-Temp-URL-Key``

-  ``X-Container-Meta-Temp-URL-Key-2``

The arbitrary values serve as the secret keys.

For example, use the **swift post** command to set the secret key to
*``MYKEY``*:

.. code::

    $ swift post -m "Temp-URL-Key:MYKEY"

Note
~~~~

Changing these headers invalidates any previously generated temporary
URLs within 60 seconds, which is the memcache time for the key.

HMAC-SHA1 signature for temporary URLs
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Temporary URL middleware uses an HMAC-SHA1 cryptographic signature. This
signature includes these elements:

-  The allowed method. Typically, **GET** or **PUT**.

-  Expiry time. In the example for the HMAC-SHA1 signature for temporary
   URLs below, the expiry time is set to ``86400`` seconds (or 1 day)
   into the future.

-  The path. Starting with ``/v1/`` onwards and including a container
   name and object. In the example below, the path is
   ``/v1/my_account/container/object``. Do not URL-encode the path at
   this stage.

-  The secret key. Use one of the key values as described
   in :ref:`secret_keys`.

This sample Python code shows how to compute a signature for use with
temporary URLs:

**Example HMAC-SHA1 signature for temporary URLs**

.. code::

    import hmac
    from hashlib import sha1
    from time import time
    method = 'GET'
    duration_in_seconds = 60*60*24
    expires = int(time() + duration_in_seconds)
    path = '/v1/my_account/container/object'
    key = 'MYKEY'
    hmac_body = '%s\n%s\n%s' % (method, expires, path)
    signature = hmac.new(key, hmac_body, sha1).hexdigest()


Do not URL-encode the path when you generate the HMAC-SHA1 signature.
However, when you make the actual HTTP request, you should properly
URL-encode the URL.

The *``MYKEY``* value is one of the key values as described
in :ref:`secret_keys`.

For more information, see `RFC 2104: HMAC: Keyed-Hashing for Message
Authentication <http://www.ietf.org/rfc/rfc2104.txt>`__.

swift-temp-url script
~~~~~~~~~~~~~~~~~~~~~

Object Storage provides the **swift-temp-url** script that
auto-generates the *``temp_url_sig``* and *``temp_url_expires``* query
parameters. For example, you might run this command:

.. code::

    $ bin/swift-temp-url GET 3600 /v1/my_account/container/object MYKEY

This command returns the path:

.. code::

    /v1/my_account/container/object
    ?temp_url_sig=5c4cc8886f36a9d0919d708ade98bf0cc71c9e91
    &temp_url_expires=1374497657

To create the temporary URL, prefix this path with the Object Storage
storage host name. For example, prefix the path with
``https://swift-cluster.example.com``, as follows:

.. code::

    https://swift-cluster.example.com/v1/my_account/container/object
    ?temp_url_sig=5c4cc8886f36a9d0919d708ade98bf0cc71c9e91
    &temp_url_expires=1374497657
