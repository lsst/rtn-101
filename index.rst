###############################################################
Preparations for a distributed data release processing campaign
###############################################################

.. abstract::

   This document presents an overview of the preparations required at the Rubin data facilities for a Data Release Processing (DRP) campaign. It presents, in particular, how the input data is organized before the campaign can start.

Introduction
============

Rubin's Data Release Processing (DRP) campaigns are performed using computing and storage resources located at several data processing facilities collectively known as the Rubin Data Facilities, hosted and operated by several organizations.

Before a campaign can start, specific input data relevant for the campaign (e.g. raw exposures, calibration data, reference catalogs) must be present at each facility and ingested into a local Data Butler repository. Once the input data set is ready, the Campaign Management team can conduct the execution of the campaign proper, which is typically composed of several stages or varying complexity and length. The image processing algorithms executed at each stage generate either intermediate data products intended to be consumed by another stage downstream or final products intended to be included in the data release. The final products generated in the context of the campaign are transferred to the US Data Facility (USDF) where they are combined into a consistent data release which is eventually made available to the scientific community.

In the sections below we present in some detail how the input data gets distributed to the processing facilities and how it is ingested into the campaign's butler repository for a typical campaign.

Preparation of Input Data
=========================

Input data is distributed from the :abbr:`USDF (US Data Facility)` to the European facilities, namely, :abbr:`FrDF (France Data Facility)` and :abbr:`UKDF (UK Data Facility)`. Although not always the case, at :abbr:`USDF (US Data Facility)` that data is typically already part of a Butler repository. Rubin's data distribution infrastructure uses CERN's `Rucio`_ for cataloguing purposes and `FTS`_ for data movement (see `DMTN-306`_ for details). `DMTN-213`_ describes the architecture of Rubin's data distribution system and how Rubin uses Rucio data management capabilities for its specifics needs.

For the purposes of moving data among the facilities, each data facility exposes two storage endpoints: one for storing immutable input data (e.g. raw exposures, reference catalogs, calibration data) and a separate one for storing data products resulting from processing inputs at the facility.

The input data endpoint is configured as a *Rucio Storage Element* (RSE) with a name of the form ``{SITE}_RAW_DISK`` whilst the name of the data products storage endpoint is like ``{SITE}_BUTLER_DISK``, as shown in :ref:`this table <table-rses>`.


.. _table-rses:

.. table:: Names of the Rucio Storage Elements for moving data among the Rubin Data Facilities.

    +-------------------+------------+--------------------------------------------+
    | **Data Facility** | **Site**   | **Rucio Storage Elements**                 |
    +===================+============+============================================+
    | FrDF              | IN2P3      | ``IN2P3_RAW_DISK``, ``IN2P3_BUTLER_DISK``  |
    +-------------------+------------+--------------------------------------------+
    | UKDF              | LANCS      | ``LANCS_RAW_DISK``, ``LANCS_BUTLER_DISK``  |
    +-------------------+------------+--------------------------------------------+
    | UKDF              | RAL        | ``RAL_RAW_DISK``, ``RAL_BUTLER_DISK``      |
    +-------------------+------------+--------------------------------------------+
    | USDF              | SLAC       | ``SLAC_RAW_DISK``, ``SLAC_BUTLER_DISK``    |
    +-------------------+------------+--------------------------------------------+


Among the information associated to each :abbr:`RSE (Rucio Storage Element)` there is the base URL of the storage specific to each site, the protocol that endpoint exposes and a measure of the network distance among them. Importantly, Rucio is configured for using the identity algorithm for translating from a logical to a physical file name so that the path of each file, relative to the root of the :abbr:`RSE (Rucio Storage Element)`, is preserved and is identical at all the facilities.

 
Distribution of Input Data
--------------------------

Raw exposures
^^^^^^^^^^^^^

Under the root path of the ``{SITE}_RAW_DISK`` endpoint a directory named after the instrument holds the raw exposures for that instrument (e.g. ``raw/LSSTCam``, ``raw/LSSTComCam``).

In order to reduce the possibilities of accidental deletion or modification of precious data, write permission to this endpoint is limited to a few service accounts or individuals.

Raw exposures are replicated from the :abbr:`USDF (US Data Facility)` to the European facilities as they come out of embargo. Off-sky exposures are replicated without delay, typically by the end of the observation day, while replication of on-sky exposures starts when their embargo period expires. Rucio replication rules are configured to drive the replication of those new exposures from the ``SLAC_RAW_DISK`` to the European facilities' endpoint for raw data. A copy of all the exposures is sent to ``FR_RAW_DISK`` whilst a copy of selected exposures is sent to the storage endpoints of the sites composing the :abbr:`UKDF (UK Data Facility)`.

For human convenience, the file name of raw exposures includes an identifier of the observation day as well as the sequence number of the exposure within that day. A exposure is stored as a multi-gigabyte ``ZIP`` file and a small metadata file in ``YAML`` format. For instance, raw exposure number 123 recorded on 2025-06-14 captured by instrument LSSTCam is stored in files:

- ``raw/LSSTCam/MC_O_20250614_000123.zip``
- ``raw/LSSTCam/MC_O_20250614_000123_dimensions.yaml``

Rubin developed the tool `transfer_embargo <https://github.com/lsst-dm/transfer_embargo/>`_ to register into Rucio new unembargoed raw exposures and to trigger replication. ``transfer_embargo`` runs periodically at SLAC, selects the appropriate raw exposures according to their embargo expiration time, registers them into Rucio and creates the replication rules, delegating replication proper to Rucio.

Ancillary data
^^^^^^^^^^^^^^

In addition to raw exposures, other input data sets are required to populate a campaign's Butler repository. Examples of those data sets include calibration exposures, the details of the spatial partitioning of the sky and reference catalogs.

Ancillary data sets deemed suitable for a given :abbr:`DRP (Data Release Processing)` campaign are usually prepared at the :abbr:`USDF (US Data Facility)` and registered into Rucio typically using the ``ancillary`` Rucio scope and then replicated to the data facilities under the ``{SITE}_RAW_DISK`` :abbr:`RSE (Rucio Storage Element)`.

Creation of Local Butler Repositories
=====================================

The process of creation of the Butler repository for the campaign at each processing facility can start once all the required input data is locally available. For a given campaign, a directory is created under the root path of each ``{SITE}_BUTLER_DISK`` endpoint to host the data store of the facility's local repository. The name of that directory is identical at all the facilities (e.g. ``dp2``, ``dr1``). This is important to ensure that the data products can be moved via Rucio to other facilities.

The steps for creating the Butler repository are conceptually the same for all the campaigns, namely:

- creation of an empty repository
- registration of the instrument
- ingestion of raw exposures
- ingestion of calibration data
- definition of the sky map
- ingestion of reference catalogs
- definition of visits
- creation and arrangement of Butler collections

The specifics for each of those steps depend on the campaign and are the subject of dedicated documents (e.g. `RTN-100`_).


Consolidation of Data Products in the Archive Center
====================================================

Final data products, intended to be included in the data release, are consolidated at the Archive Center at the :abbr:`USDF (US Data Facility)`. For this purpose, Rubin uses the tool `rucio_register <https://github.com/lsst/rucio_register/>`_ which extracts the relevant data from the execution facility's local Butler repository and registers those files into Rucio. Relevant metadata is attached to those files which is used for ingesting them into the receiving Butler repository.

Ingestion at reception is gradually performed by the daemon `ingestd <https://github.com/lsst-dm/ctrl_ingestd>`_ when the files are successfuly replicated by Rucio according to the configured replication rules for those files. Description of the control plane underlying this mechanism can be found in `DMTN-306`_.

The same mechanism is used to consolidate some intermediate products, which although not included in the data release, are useful for other purposes (e.g. monitoring).



.. _Rucio: https://rucio.cern.ch
.. _FTS: https://fts.web.cern.ch/fts/
.. _DMTN-306: https://dmtn-306.lsst.io
.. _DMTN-213: https://dmtn-213.lsst.io
.. _RTN-100: https://rtn-100.lsst.io