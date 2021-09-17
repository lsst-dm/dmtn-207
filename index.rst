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

The RSP user henceforth referred to as the (Zooniverse) PI is
manipulating data either interactively or though some kind of user batch
process to create a Butler run collection that contains the files they
wish to make available to their Zooniverse project:

.. code-block:: python

   butler = Butler(REPO, run="u/alien/zooniverse-1")
   butler.put(myimage, "zooniverse_cutout", dataId=mydataId)

Note that in the case that the user is using a batch processing pipeline
for this, the butler put could be done for them.

In the final system, the user will have to grant permission to an EPO
service account to retrieve data from their collection.

EPO service workflow
--------------------

A notional EPO service would have to perform the following tasks

* Retrieve the user's files from the Butler, eg.

.. code-block:: bash

   butler retrieve-artifacts REPO TARGET_BUCKET --collections=u/alien/zooniverse-1

* Create any manifests. metadata etc required by zooniverse

* Apply any data rights controls or quaratine outgoing data for approval

* Maintain an index of PI projects and their run collections that it can use to batch retrieve or
poll user collections for data


Image conversion
----------------

On the assumption that Zooniverse wants png or jpeg or some other kind
of non-native representation of the pixel, a Butler dataset type can be
defined with the appropriate formatter. The butler ``Formatter`` would have to
be written to convert the Python object to the appropriate on-disk
format (e.g. PNG).

Separation of Concerns
======================

* The RSP system ensures the PI is authenticated and is able to query
and retrieve their data of interest, manipulate it (if desired) and store it into the
RSP/DF butler registry. It is also responsible for supplying the
auhorisation model that allows the user to permit an EPO service account
to read their data.

* The EPO system is responsible for the code that retrieves the user's
data, (optionally but recommended) validate it for exfiltration
according to applicable policies, and publish it in an http-accessible
location from where it can be retrieved without authentication.

* Science Pipelines is responsible for providing any butler-specific configuration
(such as a PNG formatter) required to meet the DM-EPO interface
specifications.

* If there was a python package or other client that would allow the PI
to manage apsects of their zooniverse project(s) - eg project creation,
deletion, it can be installed at the RSP so that it is available from
the notebook aspect enviroment.

Notes
=====

* I think we'd all feel better if zooniverse could access an
authenticated web server

* There is lack of clarity on whether u/user/ collections are permanent,
atatched to DR-specific registries, or unguaranteed

* A tutorial notebook or helper class could be made available to walk
PIs through the process. The notebook could be added to the mobu harness
to alert to any interface drifts.

* When RSP's semaphore service is extended to deal with per-user
notification, we could provide an API that allows EPO to send per-user
notifications informing them of relevant status, such as that their
files have been retrieved and can be safely removed.


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
