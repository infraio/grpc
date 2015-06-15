gRPC Connectivity Semantics and API
===================================

This document describes the connectivity semantics for gRPC channels and the
corresponding impact on RPCs. We then discuss an API.

States of Connectivity
----------------------

gRPC Channels provide the abstraction over which clients can communicate with
servers.The client-side channel object can be constructed using little more
than a DNS name. Channels encapsulate a range of functionality including name
resolution, establishing a TCP connection (with retries and backoff) and TLS
handshakes. Channels can also handle errors on established connections and
reconnect, or in the case of HTTP/2 GO_AWAY, re-resolve the name and reconnect.

To hide the details of all this activity from the user of the gRPC API (i.e.,
application code) while exposing meaningful information about the state of a
channel, we use a state machine with four states, defined below:

CONNECTING: The channel is trying to establish a connection and is waiting to
make progress on one of the steps involved in name resolution, TCP connection
establishment or TLS handshake. This is the initial state for all channels upon
creation.

READY: The channel has successfully established a connection all the way
through TLS handshake (or equivalent) and all subsequent attempt to communicate
have succeeded (or are pending without any known failure ).

TRANSIENT_FAILURE: There has been some transient failure (such as a TCP 3-way
handshake timing out or a socket error). Channels in this state will eventually
switch to the CONNECTING state and try to establish a connection again. Since
retries are done with exponential backoff, channels that fail to connect will
start out spending very little time in this state but as the attempts fail
repeatedly, the channel will spend increasingly large amounts of time in this
state. For many non-fatal failures (e.g., TCP connection attempts timing out
because the server is not yet available), the channel may be stuck in this
state for an indefinitely large amount of time.

FATAL_FAILURE: There has been a fatal failure and the channel will never
attempt to establish a connection again.  (e.g., a server presenting an invalid
TLS certificate)

Channels that enter this state never leave this state.

The following table lists the legal transitions from one state to another and
corresponding reasons. Empty cells denote disallowed transitions.

<table style='border: 1px solid black'>
  <tr>
    <th>From/To</th>
    <th>CONNECTING</th>
    <th>READY</th>
    <th>TRANSIENT_FAILURE</th>
    <th>FATAL_FAILURE</th>
  </tr>
  <tr>
    <th>CONNECTING</th>
    <td>Incremental progress during connection establishment</td>
    <td>All steps needed to establish a connection succeeded</td>
    <td>Any failure in any of the steps needed to establish connection</td>
    <td>Fatal failure encountered while attempting a connection.</td>
  </tr>
  <tr>
    <th>READY</th>
    <td></td>
    <td>Incremental successful communication on established channel.</td>
    <td>Any failure encountered while expecting successful communication on
        established channel.</td>
    <td></td>
  </tr>
  <tr>
    <th>TRANSIENT_FAILURE</th>
    <td>Wait time required to implement (exponential) backoff is over.</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <th>FATAL_FAILURE</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>


Channel State API
-----------------

All gRPC libraries will expose a channel-level API method to poll the current
state of a channel. In C++, this method is called GetCurrentState and returns
an enum for one of the four legal states.

All libraries should also expose an API that enables the application (user of
the gRPC API) to be notified when the channel state changes. Since state
changes can be rapid and race with any such notification, the notification
should just inform the user that some state change has happened, leaving it to
the user to poll the channel for the current state.

The synchronous version of this API is:

```cpp
bool WaitForStateChange(gpr_timespec deadline, ChannelState source_state);
```

which returns true when the state changes to something other than the
source_state and false if the deadline expires. Asynchronous and futures based
APIs should have a corresponding method that allows the application to be
notified when the state of a channel changes.

Note that a notification is delivered every time there is a transition from any
state to any *other* state. On the other hand the rules for legal state
transition, require a transition from CONNECTING to TRANSIENT_FAILURE and back
to CONNECTING for every recoverable failure, even if the corresponding
exponential backoff requires no wait before retry. The combined effect is that
the application may receive state change notifications that appear spurious.
e.g., an application waiting for state changes on a channel that is CONNECTING
may receive a state change notification but find the channel in the same
CONNECTING state on polling for current state because the channel may have
spent infinitesimally small amount of time in the TRANSIENT_FAILURE state.