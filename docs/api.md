Capture Agent API (Version 2)
=============================
*working draft*

This document specifies the communication between Opencast enabled capture
devices (capture agents) and Opencast itself. It defines an extension or
modification of the already existing API for Opencast 2.2 and prior versions.

A basic idea behind this API update is to have a simple base or core API and
several, optional extensions which capture agents may implement to extend their
functionality.


Primary Changes Compared to CA-APIv1
------------------------------------

- Registering capture agents
    - The URL is optional
- Getting calendar/schedule
    - The format can be specified
	 - JSON is available additional to iCal
- Setting capture agent configuration is available as extension


Base API
--------

This part of the API is mandatory for CA vendors.

*Note: This is a slightly modified version of the current API with some
additional clarifications and without the need to send configuration data to
the core.*


### General Capture Agent Behavior

The following rules apply to the capture agent base API as well as to all
extensions if not explicitly defined otherwise.


- The core must not attempt to connect to the agent. The communication is
  unidirectional from agent to core. Extensions may ignore this rule, but even
  then, agents have to work without a bidirectional connection.
- The agent must try to send its state to the core on a semi regular basis.
  This should happen at least once the state changes (e.g. a recording is
  started, stopped, …)
- The agent must try to send the recording state to the core on a semi regular
  basis. This should happen at least once the recording state changes (e.g. a
  recording is started, stopped or failed)
- The agent should try to update its calendaring data on a regular bases
- The agent is required to request the locations of the required REST endpoints
  from the service registry on a semi regular basis
- If multiple endpoints for one task are available (e.g. multiple ingest
  endpoints) the endpoint to talk to has to be picked randomly (random function
  with uniform distribution) to ensure a balanced load on the system
- If there is an error during the communication with the core (e.g. network
  failure), the agent must assume the old data to be still valid (schedule and
  endpoints) and may ignore failures on status requests


### Quick Overview

The following pseudo code defines what you are required to do as a capture
agent. Note, that these are only the minimal requirements and you can do better.
A more specific explanation of all the steps can be found below.


    request rest endpoints              (1)
    register ca and set status          (2)
    repeat:
       while no recording
          get schedule/calendar         (3)
       set agent and recording state    (4)
       start recording                  (5)
       set agent and recording state    (6)
       update ingest endpoints          (7)
       ingest recording                 (8)
       set agent and recording state    (9)


#### Get Endpoint Locations From Service Registry (1)

First, you need to get the locations of the REST endpoints to talk to.  These
information can be retrieved from the central service registry. It is likely
that Matterhorn is running as a distributed system which means you cannot
expect all endpoints on a single host.

Three endpoint locations need to be requested:

- The capture-admin endpoint to register the agent and set status and
  configuration: org.opencastproject.capture.admin
- The scheduler endpoint to get the current schedule for the agent from:
  org.opencastproject.scheduler
- The ingest endpoint to upload the recordings to once they are done:
  org.opencastproject.ingest


To ask for an endpoint you would send an HTTP GET request to

    ${HOST}/services/available.json?serviceType=<SERVICE>

A result would look like

    {
      "services" : {
        "service" : {
          "active" : true,
          "host" : "http://example.opencast.org:8080",
          "path" : "/capture-admin",
          ...
        }
      }
    }


These endpoints should be requested once when starting up the capture agent
(1). While the capture-admin and scheduler endpoints may then be assumed to
never change during the runtime of the capture agent, the ingest endpoint may
change and the data should be re-requested every time before uploading
(ingesting) data to the core.


#### Register the Capture Agent and set Current Status (2)

Once the endpoints to talk to are knows, it is time to register the capture
agent at the core so that scheduled events can be added. This can be done by
sending an HTTP POST request to:

    ${CAPTURE-ADMIN-ENDPOINT}/agents/<name>


…including the following data fields:

    state=idle
    address=http(s)://<ca-web-ui> (OPTIONAL)


The name has to be a unique identifier of a single capture agent. Using the
same name for several agents would mean sharing the same schedule and status
which in general should be avoided.

Sending this request will register the capture agent. After doing this, the
capture agent should appear in the admin interface and schedules can be added
for this agent.


#### Getting the Calendar/Schedule (3)

The calendar can be retrieved by sending an HTTP GET request to

    ${SCHEDULER-ENDPOINT}/calendars<.format>?agentid=<name>

The format can either be `.ical` or `.json`. If the format is omitted, iCal is
being returned.  The file contains all scheduled upcoming recordings the
capture agent should handle.

Depending on the amount of recordings scheduled for the particular capture
agent, this file may become very large. That is why there are two way of
limiting the amount of necessary data to transfer and process:


1. Send along the optional parameter cutoff to limit the schedule to a
	particular time span in the future.

       ${SCHEDULER-ENDPOINT}/calendars?agentid=<name>&cutoff=<time>

   The value for cutoff represents milliseconds from now.

2. Use the HTTP ETag and If-Not-Modified header the have Matterhorn only sent
	schedules when they have actually changed.


#### Set Agent and Recording State (4)

Setting the agent state is identical to the registration of the capture agent
and done by sending an HTTP POST request to:

    ${CAPTURE-ADMIN-ENDPOINT}/agents/<name>

…including the following data fields:

    state=capturing
    address=http(s)://<ca-web-ui> (OPTIONAL)

Additionally, set the recording state with an HTTP POST request to

    ${CAPTURE-ADMIN-ENDPOINT}/recordings/<recording_id>

…including the data field:

    state=capturing


#### Recording (5)

This task is device specific. You may execute any code necessary to get the
recording on your device.


#### Set Agent and Recording State (6)

This step is identical to step 4 except for the state.

If the recording has failed, the recording state is updated with
`capture_error` while the agents state is set back to `idle` if the error is
non-permanent.

If the recording was successful, both states are set to `uploading`.


#### Get Ingest Endpoint Locations From Service Registry (7)

This step is identical to step 1 expect that it is sufficient to request the
location for the service `org.opencastproject.ingest` only. If this request
fails, assume the old data to be valid.


#### Ingest (Upload) Recording (8)

Use the ingest endpoint to upload the recording.

- Recording files may be deleted if the ingest was successful.
- Recordings should be stored in case of a failure.

*TODO: Descripbe ingest requests*

#### Set Agent and Recording State (6)

Again, this step is identical to step 4 except for the state.

If the upload has failed, the recording state is updated with
`upload_error` while the agents state is set back to `idle` if the error is
non-permanent.

If the upload was successful, the recording status is set to `upload_finished`
while the agents state is set back to `idle`.


Extensions
----------

Extensions may be specified as addition to the base API. Extensions are always
required to be optional and parts of them may not be required by Opencast
itself or capture agents to function.


### Extension: Capture Agent Configuration (id: CFG)

*TODO: Old configuration but accepts JSON as well*


### Extension: Recording Preview (id: PREVIEW)

*TODO: Provide locazion of preview image or stream for Opencast to use as
preview on a capture agent dashboard*


### Extension: Bi-directional CA Communication (id: TALKBACK)

*TODO: Enable use of WebSockets or longpolling for faster communication and
less stress on the admin node.*
