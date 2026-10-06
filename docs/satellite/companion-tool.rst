.. _hubble_satellite_companion_tool:

Dual-Stack Companion Tool
#########################

``tools/dual-stack-companion.py`` is a host-side Python script that provisions
a dual-stack device over Bluetooth LE.

The ``sat-dual-stack`` samples all wait for this tool on boot. These samples
won't beacon to either terrestrial or satellite networks until this tool has
provisioned the device.

.. important::

   The samples do not persist the provisioning data across power cycles. The
   data is stored in RAM only, so the device must be re-provisioned after
   every power cycle.

Purpose
*******

A satellite is only overhead for a few minutes at a time. To save power, a
dual-stack device predicts the next pass with :c:func:`hubble_sat_next_pass_get`
and keeps its satellite radio off until then. That prediction uses Unix time, the
device's location, and the satellite's orbital parameters to calculate visible
passes.

In a product, a phone app, a gateway, or a factory fixture would complete
the provisioning step. This tool is a reference for that flow and the fastest
way to get a sample running. See :ref:`hubble_satellite_orbital_params` for
other ways to supply orbital parameters, including compiling them into the
firmware, and :ref:`hubble_pass_prediction_best_practices` for how to use pass
prediction in your own application.

Function
********

The script does the following in order:

#. Fetches orbital parameters from the Hubble API, authenticated with
   ``HUBBLE_API_TOKEN``.
#. Uses the device location from ``--location``, or the host's location looked
   up from its public IP address.
#. Scans for up to 30 seconds for a device advertising the Hubble service UUID
   ``0xFCA7``.
#. Connects and writes, over GATT: one record per satellite, the location, and
   the host's current time.
#. Disconnects and exits with status ``0`` on success or ``1`` on any failure.

After the script disconnects, the device starts beaconing on the Hubble
Terrestrial Network and turns on its satellite radio for each predicted pass.

Requirements
************

#. A host computer with Bluetooth LE and internet access.
#. Python 3.10 or newer.
#. A device running one of the ``sat-dual-stack`` samples.
#. A Hubble API token.

Usage Steps
***********

The commands below are run from the SDK repository root.

.. _hubble_companion_api_token:

1. Get an API token
===================

Create a token by following the
`Hubble Platform API documentation <https://hubble.com/docs/api-specification/hubble-platform-api#generate-an-api-key>`_. The tool uses
it only to fetch orbital parameters; it is never written to the device.


2. Install the dependencies
===========================

Install the tool's dependencies into a virtual environment:

Linux & macOS:

.. code-block:: bash

   python3 -m venv .venv
   source .venv/bin/activate
   pip install -r tools/requirements-companion.txt

Windows (PowerShell):

.. code-block:: powershell

   python -m venv .venv
   .venv\Scripts\Activate.ps1
   pip install -r tools\requirements-companion.txt

3. Build, flash and boot the device
===================================

Build and flash one of the ``sat-dual-stack`` samples by following its README.

4. Run the tool
===============

The tool connects to the first unprovisioned device it finds. We recommend only
powering one device at a time while you provision it, if provisioning multiple devices.

Linux & macOS:

.. code-block:: bash

   export HUBBLE_API_TOKEN=<your-hubble-api-token>
   python tools/dual-stack-companion.py

Windows (PowerShell):

.. code-block:: powershell

   $env:HUBBLE_API_TOKEN = "<your-hubble-api-token>"
   python tools\dual-stack-companion.py

A successful run ends with ``Provisioned device successfully``, and the device
console then logs ``Next pass at`` with the predicted pass time.

Options
*******

.. list-table::
   :widths: 30 70
   :header-rows: 1

   * - Option
     - Description
   * - ``--location LAT LON``
     - Device latitude and longitude, in decimal degrees. Without it, the
       location is looked up from the host's public IP address.
   * - ``--label NAME``
     - Only connect to a device advertising exactly this name, for example
       ``Hubble-ESP``.
   * - ``-v``, ``--verbose``
     - Debug logging.
   * - ``HUBBLE_API_TOKEN`` (environment variable)
     - Required. Bearer token for the Hubble API.
   * - ``HUBBLE_TARGET_SATELLITE_IDS`` (environment variable)
     - Optional. Comma-separated satellite IDs to fetch, for example
       ``101,102``. Without it, all satellites are fetched. The samples store
       at most **6**; extra satellites are dropped with a warning.

Accuracy
********

Passes are predicted for the provisioned location and time, so errors in
either make the device transmit when no satellite is overhead.

* **Location.** If your device will be somewhere far from the host computer,
  pass the device's planned location with ``--location``.
* **Time.** The tool sends the host's clock, so make sure it is accurate.

Troubleshooting
***************

.. list-table::
   :widths: 40 60
   :header-rows: 1

   * - Message or symptom
     - What to do
   * - ``HUBBLE_API_TOKEN env var must be set``
     - Set it in the same shell you run the tool from.
   * - ``IP geolocation failed``
     - Pass ``--location LAT LON`` instead of relying on IP geolocation.
   * - ``Device not found``
     - Check that the device is powered, in range, and waiting for
       provisioning. A provisioned device stops advertising for it, so reset
       it first. If you passed ``--label``, check that it matches exactly.
