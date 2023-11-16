..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   A proposal for how an RSP user would make data available to EPO for use in their Zooniverse citizen project.

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

Context
=======

Banner usecase
---------------

An RSP user who is also PI for a Zooniverse project wishes to make data available to that project.

Constraints
-----------

* Zooniverse expects data to be hosted at a public URL it can access dynamically.
* Data rights protections and constraints need to be applied.

Relevant documents
-------------------

* LSE-131 Interface Requirements between Data Management and EPO

Architecture
============

  .. figure:: /_static/epo_zooniverse.png
     :name: Illustrative architecture

User workflow
-------------

The RSP user henceforth referred to as the (Zooniverse) PI is manipulating data either interactively or though some kind of user batch process to create a Butler run collection that contains the files they wish to make available to their Zooniverse project:

.. code-block:: python

   butler = Butler(REPO, run="u/alien/zooniverse-1")
   butler.put(myimage, "zooniverse_cutout", dataId=mydataId)

Note that in the case that the user is using a batch processing pipeline for this, the butler put could be done for them.

In the final system, the user will have to grant permission to an EPO service account to retrieve data from their collection.

EPO service workflow
--------------------

A notional EPO service would have to perform the following tasks

* Retrieve the user's files from the Butler, eg.

.. code-block:: bash

   butler retrieve-artifacts REPO TARGET_BUCKET --collections=u/alien/zooniverse-1

* Create any manifests. metadata etc required by zooniverse

* Apply any data rights controls or quarantine outgoing data for approval

* Maintain an index of PI projects and their run collections that it can use to batch retrieve or poll user collections for data


Image conversion
----------------

On the assumption that Zooniverse wants png or jpeg or some other kind of non-native representation of the pixel, a Butler dataset type can be defined with the appropriate formatter. The butler ``Formatter`` would have to be written to convert the Python object to the appropriate on-disk format (e.g. PNG).

Separation of Concerns
======================

* The RSP system ensures the PI is authenticated and is able to query and retrieve their data of interest, manipulate it (if desired) and store it into the RSP/DF butler registry. It is also responsible for supplying the authorisation model that allows the user to permit an EPO service account to read their data.

* The EPO system is responsible for the code that retrieves the user's data, (optionally but recommended) validate it for exfiltration according to applicable policies, and publish it in an http-accessible location from where it can be retrieved without authentication.

* Science Pipelines is responsible for providing any butler-specific configuration (such as a PNG formatter) required to meet the DM-EPO interface specifications.

* If there was a python package or other client that would allow the PI to manage aspects of their zooniverse project(s) - eg project creation, deletion, it can be installed at the RSP so that it is available from the notebook aspect environment.

Notes
=====

* I think we'd all feel better if zooniverse could access an authenticated web server

* There is lack of clarity on whether u/user/ collections are permanent, attached to DR-specific registries, or unguaranteed

* A tutorial notebook or helper class could be made available to walk PIs through the process. The notebook could be added to the mobu harness to alert to any interface drifts.

* When RSP's semaphore service is extended to deal with per-user notification, we could provide an API that allows EPO to send per-user notifications informing them of relevant status, such as that their files have been retrieved and can be safely removed.

* If there is any specific metadata that zooniverse needs that is dropped by the butler retrieve-artifacts, DM can work with EPO to advice on how to obtain it so it can be included with the manifest.

Potential Use Cases
===================

Catalog of Objects
------------------

A scientist has a catalog of objects for which they would like to obtain cutouts as multi-band PNGs from deep coadds to send to Zooniverse.
This catalog may have a non-trivial number of rows.
It is assumed that the scientist has permission for these cutouts to be sent to Zooniverse.

Possible Workflow
^^^^^^^^^^^^^^^^^

Assume the number of objects is sufficient that the cutout retrieval must be parallelized and can not be run in a user notebook environment.
Therefore assume this can run in a user-batch environment using standard Rubin pipeline infrastructure (``PipelineTask``).

1. The catalog is "all sky" and so does not have an obvious Butler "data coordinate".
   In the future it may be possible to upload it using an opaque dataId but in the nearer term a relatively simple script could be used to shard the catalog (similar to a refcat for example) and upload the shards to the Butler repostory.
2. Submit a batch job using a pipeline that will take relevant images and the sharded catalogs as input and extract the cutouts.
3. The cutouts per band/tract/patch would be stored using ``lsst.meas.algorthms.Stamps`` (or derived class).

The above is a fairly general use case that does not involve any specific EPO functionality.
The butler repository will have a collection of multi-extension FITS files grouped by band, tract and patch (it is possible that the system could decide that patches are not needed and per-tract is enough).

The Zooniverse step would then be to retrieve these multi-extension FITS files for each tract/patch and combine the different bands into images.
A Python object would need to be defined that represents a collection of RGB images.
For example, this could be achieved using ``PIL.Image`` objects in a special container class.
The steps would then be:

1. For each tract/patch read in the ``Stamps`` object for each band.
2. Construct RGB image data for each cutout and store in the container class.
3. Store the container object in a Butler collection.

If constructed as a ``PipelineTask`` this step could be executed at scale using BPS and the user would specify how the images are converted to RGB via configuration.

The Butler storage step requires that a special "formatter" class be written that knows how to convert the RGB image container class to a file on disk.
A default proposal for this would be that the resulant output is a ZIP file containing PNG images.


Once these ZIP files have been stored EPO would then retrieve them and unpack them as discussed in an earlier section.

Unanswered questions are:

* The ZIP file would automatically be named after run collection and the tract and patch.
  The naming of the PNG files within that ZIP archive is at the discretion of the formatter.
  Is the dataId plus a counter acceptable?
* How much metadata should be included in the PNG files?
  With `Pillow <https://pillow.readthedocs.io/en/stable/handbook/tutorial.html>`__ it is possible to store all available FITS metadata in the EXIF segment of the PNG including a WCS (for example using the `AVM standard <https://en.wikipedia.org/wiki/Astronomy_Visualization_Metadata>`__).


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
