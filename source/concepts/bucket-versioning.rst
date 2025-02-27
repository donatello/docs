.. _minio-bucket-versioning:

=================
Bucket Versioning
=================

.. default-domain:: minio

.. contents:: Table of Contents
   :local:
   :depth: 2

Overview
--------

MinIO supports keeping multiple "versions" of an object in a single bucket.
Write operations which would normally overwrite an existing object instead
result in the creation of a new versioned object. MinIO versioning protects from
unintended overwrites and deletions while providing support for "undoing" a
write operation. Bucket versioning is a prerequisite for configuring
:ref:`object locking and retention rules <minio-object-locking>`.

For versioned buckets, any write operation that mutates an object results in a
new version of that object with a unique version ID. MinIO marks the "latest"
version of the object that clients retrieve by default. Clients can then
explicitly choose to list, retrieve, or remove a specific object version. 

.. card-carousel:: 1

   .. card:: Object with Single Version

      .. image:: /images/retention/minio-versioning-single-version.svg
         :alt: Object with single version
         :align: center

      MinIO adds a unique version ID to each object as part of write operations.

   .. card:: Object with Multiple Versions

      .. image:: /images/retention/minio-versioning-multiple-versions.svg
         :alt: Object with Multiple Versions
         :align: center

      MinIO retains all versions of an object and marks the most recent
      version as the "latest".

   .. card:: Retrieving the Latest Object Version

      .. image:: /images/retention/minio-versioning-retrieve-latest-version.svg
         :alt: Object with Multiple Versions
         :align: center

   .. card:: Retrieving a Specific Object Version

      .. image:: /images/retention/minio-versioning-retrieve-single-version.svg
         :alt: Object with Multiple Versions
         :align: center

:ref:`Deleting <minio-bucket-versioning-delete>` an object results in a special
``DeleteMarker`` tombstone that marks an object as deleted while retaining
all previous versions of that object.

Versioning is Per-Namespace
~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO uses the full namespace (the bucket and path to an object) for each object
as part of determining object uniqueness. For example, all of the following
namespaces are "unique" objects, where mutations of each object result in
the creation of new object versions *at that namespace*:

.. code-block:: shell

   databucket/object.blob
   databucket/blobs/object.blob
   blobbucket/object.blob
   blobbucket/blobs/object.blob

While ``object.blob`` might be the same binary across all namespaces, 
MinIO only enforces versioning with a specific namespace and therefore
considers each ``object.blob`` above as distinct and unique.

Versioning and Storage Capacity
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO does not perform incremental or differential-type versioning. For
mutation-heavy workloads, this may result in substantial disk usage by
older or aged object versions.

For example, consider a 1GB object containing log data. An application
appends 100MB of data to the log and uploads to MinIO. MinIO would then contain
both the 1GB and 1.1GB versions of the object. If the application repeated
this process every day for 10 days, the bucket would eventually contain more
than 14GB of data associated to a single object.

MinIO supports configuring configuring :ref:`object lifecycle management rules 
<minio-lifecycle-management>` to automatically expire or transition aged
object versions and free up storage capacity. For example, you can configure
a rule to automatically expire object versions 90 days after they become
non-current (i.e. no longer the "latest" version of that object). See 
:ref:`MinIO Object Expiration <minio-lifecycle-management-expiration>` for 
more information.

You can alternatively perform manual removal of object versions using the 
following commands:

- :mc-cmd-option:`mc rm versions` - Removes all versions of an object.
- :mc-cmd-option:`mc rm --versions --older-than <mc rm older-than>` -
   Removes all versions of an object older than the specified calendar date.

.. _minio-bucket-versioning-id:

Version ID Generation
~~~~~~~~~~~~~~~~~~~~~

MinIO generates a unique and immutable identifier for each versioned object as
part of write operations. Each object version ID consists of a 128-bit
fixed-size :rfc:`UUIDv4 <4122#section-4.4>`. UUID generation is sufficiently
random to ensure high likelihood of uniqueness for any environment, are
computationally difficult to guess, and do not require centralized registration
process and authority to guarantee uniqueness.

.. image:: /images/retention/minio-versioning-multiple-versions.svg
   :alt: Object with Multiple Versions
   :width: 600px
   :align: center

MinIO does not support client-managed version ID allocation. All version ID
generation is handled by the MinIO server process.

For objects created while versioning is disabled or suspended, MinIO 
uses a ``null`` version ID. You can access or remove these objects by specifying
``null`` as the version ID as part of S3 operations.

.. _minio-bucket-versioning-delete:

Versioned Delete Operations
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Performing a ``DELETE`` operation on a versioned object creates a 
0-byte ``DeleteMarker`` as the latest version of that object. Clients performing
``GET`` operations on that object do not return any results, as MinIO does not
return the ``DeleteMarker`` back as part of the response. Similarly, performing
a ``LIST`` operation by default returns only objects which are *not* a
``DeleteMarker``.

To permanently delete an object version, perform the ``DELETE`` operation and 
specify the version ID of the object to delete. Versioned delete operations 
are **irreversible**.

.. card-carousel:: 1

   .. card:: Deleting an Object

      .. image:: /images/retention/minio-versioning-delete-object.svg
         :alt: Deleting an Object
         :align: center

      Performing a ``DELETE`` operation on a versioned object produces a 
      ``DeleteMarker`` for that object.

   .. card:: Reading a Deleted Object

      .. image:: /images/retention/minio-versioning-retrieve-deleted-object.svg
         :alt: Object with Multiple Versions
         :align: center

      Clients by default retrieve the "latest" object version. MinIO returns
      a ``404``-like response if the latest version is a ``DeleteMarker``.

   .. card:: Retrieve Previous Version of Deleted Object

      .. image:: /images/retention/minio-versioning-retrieve-version-before-delete.svg
         :alt: Retrieve Version of Deleted Object
         :align: center

      Clients can retrieve any previous version of the object by specifying the
      version ID, even if the "Latest" version is a ``DeleteMarker``.

   .. card:: Delete a Specific Object Version

      .. image:: /images/retention/minio-versioning-delete-specific-version.svg
         :alt: Retrieve Version of Deleted Object
         :align: center

      Clients can delete a specific object version by specifying the version ID
      as part of the ``DELETE`` operation. Deleting a specific version is 
      **permanent** and does not result in the creation of a ``DeleteMarker``.

The following :mc:`mc` commands operate on ``DeleteMarkers`` or versioned 
objects:

- Use :mc-cmd-option:`mc ls versions` to view all versions of an object,
  including delete markers.

- Use :mc-cmd-option:`mc cp --version-id=UUID ... <mc cp version-id>` to 
  retrieve the version of the "deleted" object with matching ``UUID``.

- Use :mc-cmd-option:`mc rm --version-id=UUID ... <mc rm version-id>` to delete
  the version of the object with matching ``UUID``.

- Use :mc-cmd-option:`mc rm versions` to delete *all* versions of an object.

Tutorials
---------

Enable Bucket Versioning
~~~~~~~~~~~~~~~~~~~~~~~~

.. note::

   Versioning requires :ref:`erasure coding <minio-erasure-coding>` and is only
   available on :ref:`distributed MinIO deployments <deploy-minio-distributed>`.

You can enable versioning using the MinIO Console, the MinIO :mc:`mc` CLI, or
using an S3-compatible SDK. Versioning is a bucket-scoped feature. You cannot
enable versioning on only a prefix or subset of objects in a bucket.

.. tab-set::

   .. tab-item:: MinIO Console

      Select the :guilabel:`Buckets` section of the MinIO Console to access
      bucket creation and management functions. Select the bucket row from the
      list of buckets. You can use the :octicon:`search` :guilabel:`Search` bar
      to filter the list. 
      
      .. image:: /images/minio-console/console-bucket.png
         :width: 600px
         :alt: MinIO Console Bucket Management
         :align: center
      
      From the Bucket view, click the :guilabel:`Versioning`
      field to enable versioning on the bucket.

      .. image:: /images/minio-console/console-bucket-overview.png
         :width: 600px
         :alt: MinIO Console Bucket minio-lifecycle-management
         :align: center

      The MinIO Console also supports enabling versioning as part of bucket
      creation. See :ref:`minio-console-admin-buckets` for more information on
      bucket management using the MinIO Console.

   .. tab-item:: MinIO CLI

      Use the :mc-cmd:`mc version enable` command to enable versioning on an 
      existing bucket:

      .. code-block:: shell
         :class: copyable

         mc version enable ALIAS/BUCKET

      - Replace ``ALIAS`` with the :mc:`alias <mc alias>` of a configured 
        MinIO deployment.

      - Replace ``BUCKET`` with the 
        :mc:`target bucket <mc version enable TARGET>` on which to enable
        versioning.

Objects created prior to enabling versioning have a 
``null`` :ref:`version ID <minio-bucket-versioning-id>`.

Suspend Bucket Versioning
~~~~~~~~~~~~~~~~~~~~~~~~~

You can suspend bucket versioning at any time using the MinIO Console, the
MinIO :mc:`mc` CLI, or using an S3-compatible SDK.

.. tab-set::

   .. tab-item:: MinIO Console

      Select the :guilabel:`Buckets` section of the MinIO Console to access 
      bucket creation and management functions. 
      
      Select the bucket row from the list of buckets. You can use the
      :octicon:`search` :guilabel:`Search` bar to filter the list. 
      
      .. image:: /images/minio-console/console-bucket.png
         :width: 600px
         :alt: MinIO Console Bucket Management
         :align: center
      
      From the :guilabel:`Bucket` view, click the :guilabel:`Versioning` field 
      to disable versioning on the bucket.

      .. image:: /images/minio-console/console-bucket-overview.png
         :width: 600px
         :alt: MinIO Console Bucket minio-lifecycle-management
         :align: center

      See :ref:`minio-console-admin-buckets` for more information on bucket
      management using the MinIO Console.

   .. tab-item:: MinIO CLI

      Use the :mc-cmd:`mc version disable` command to enable versioning on an 
      existing bucket:

      .. code-block:: shell
         :class: copyable

         mc version disable ALIAS/BUCKET

      - Replace ``ALIAS`` with the :mc:`alias <mc alias>` of a configured 
        MinIO deployment.

      - Replace ``BUCKET`` with the 
        :mc:`target bucket <mc version enable TARGET>` on which to disable
        versioning.

Objects created while versioning is suspended are assigned a 
``null`` :ref:`version ID <minio-bucket-versioning-id>`. Any mutations to an
object while versioning is suspended results in overwriting that 
``null`` versioned object. MinIO does not remove or otherwise alter 
existing versioned objects as part of suspending versioning. Clients can 
continue interacting with any existing object versions in the bucket.

