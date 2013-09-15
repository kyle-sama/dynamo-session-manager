Tomcat Dynamo Session Manager
============================

Deprecated Fork
---------------

I have left EnergyHub, so development of d-s-m is being continued by [Mark Whitney](https://github.com/kramimus/dynamo-session-manager)


Overview
--------

This is a tomcat session manager that saves sessions in Amazon Dynamo DB, based on the
[Mongo version by David Dawson.](https://github.com/dawsonsystems/Mongo-Tomcat-Sessions)
It is made up of DynamoManager, that provides the save/load functions, DynamoSessionTrackerValve that controls the
timing of the save, and DynamoTableRotator that manages session expiration by moving active sessions to a new table and
deleting the old table.

The manager can use a local implementation of Dynamo for QA / testing purposes, by specifying the 'dynamoEndpoint'
parameter. One implementation that works (albeit slowly) is [Fake Dynamo.](https://github.com/ananthakumaran/fake_dynamo)

Usage
-----

Add the following into your tomcat server.xml, or context.xml

    <Valve className="net.energyhub.session.DynamoSessionTrackerValve" />
    <Manager className="net.energyhub.session.DynamoManager"
			dynamoEndpoint="http://localhost:9090"
			maxInactiveInterval="3600"
			sessionSize="2"
			requestsPerSecond="20"
			ignoreUris=".*(/regex/to/ignore.html|/regex/to/other.html).*"
			ignoreHeaders="X-MOBILE-CLIENT-TOKEN"

	/>

The Valve must be before the Manager.


The following parameters are available on the Manager :-

<table>
<tr><td>awsAccessKey</td><td>For production / staging environments. Your AWS / Dynamo credentials</td></tr>
<tr><td>awsSecretKey</td><td>For production / staging environments. Your AWS / Dynamo credentials</td></tr>
<tr><td>dynamoEndpoint</td><td>Optional, for QA Environment: The endpoint of a mock Dynamo implementation such
as Fake Dynamo: e.g. http://localhost:9090/.</td></tr>
<tr><td>requestsPerSecond</td><td>Expected maximum requests per second (for Dynamo provisioning)</td></tr>
<tr><td>sessionSize</td><td>Expected average session size, in kB</td></tr>
<tr><td>eventualConsistency</td><td>Use eventual consistency reads or standard reads</td></tr>
<tr><td>maxInactiveInterval</td><td>Optional, the initial maximum time interval, in seconds, between client requests
before a session is invalidated. A negative value will result in sessions never timing out. If the attribute is not
provided, a default of 60 seconds is set by the base class, but due to hourly billing and table creation latency
we recommend at least an hour.</td></tr>
<tr><td>tableBaseName</td><td>Optional, the base Dynamo table name to use. The default is 'tomcat-sessions'</td></tr>
<tr><td>ignoreUris</td><td>Optional, if the request URI matches this regex, the session will not be saved to Dynamo.</td></tr>
<tr><td>ignoreHeaders</td><td>Optional, if the request has a header name matching this regex, the session will not be saved to Dynamo.</td></tr>
<tr><td>statsdHost</td><td>Optional, if set we will send statistics to <a href="https://github.com/etsy/statsd/">statsd</a> about session size and request timing</td></tr>
<tr><td>statsdPort</td><td>Optional, set the port for statsd (default is 8125)</td></tr>
<tr><td>logSessionContents</td><td>If true, log the session attributes on load and save (at FINE level)</td></tr>
</table>

Set backgroundProcessDelay to a small number (usually in server.xml)

    <Engine name="Catalina"
        backgroundProcessorDelay="1"
    >

Copy the dynamo-session-manager jar and the dependencies from target/lib/ into the tomcat lib directory
(e.g. /usr/share/tomcat6/lib) and you're good to go.


Expiration
----------

We take the approach that sessions that haven't been accessed for a certain time should be removed from the store.
Unlike the Mongo session manager, scanning for expired sessions in Dynamo is very expensive (it literally costs you
money!) and is hard to provision for. The approach taken here is to rotate the Dynamo table and drop whole tables when
they are expired. Active sessions will be read from the previous table and stored in the current table, inactive ones
will be left in the previous table and dropped when the table is deleted.

- Table A: current table, read/write
- Table B: previous table. If session not found in A, look in B. Save to A.
- Table C: expired table, will be deleted on next check.

The tables are named like
   tableName + startTimestamp

The current table timestamp is (currentTime - currentTime % maxInactiveInterval).

A background task creates the next table before it is required, and deletes the expired table after rotation. We have
found that table creation takes approximately 60s.

The table rotation should be synchronized as much as possible across different tomcat servers. In practice with Elastic
Load Balancers (even in non-sticky mode) this is mitigated by a general stickiness of a client to a server, but we can
reduce the gap between different servers rotating tables by checking more frequently. In server.xml, set the
Engine.backgroundProcessDelay to 1s, as shown above.

Provisioning
------------

You need to provision one read and one write unit per request, i.e. if your app has a peak throughput of 10 requests per
second, you need 10 reads per second and 10 writes per second during normal activity.

- 10 reads per second = 1x (block of 50 read units per second) = $0.01 / hour
- 10 writes per second = 1x (block of 10 write units per second) = $0.01 / hour

If your sessions are greater than 1kB, you need to multiply your provisioned units by the session size in kB.

Because we use multiple tables, you will actually pay roughly 2X this cost (current table + previous table). At times,
we will also have a future table or an expired table for a few seconds.

License: Apache 2.0
