# SC4S Configuration Variables

Other than device filter creation, SC4S is almost entirely controlled by environment variables.  Here are the categories
and variables needed to properly configure SC4S for your environment.

## Global Configuration

| Variable | Values        | Description |
|----------|---------------|-------------|
| SPLUNK_HEC_URL | url | URL(s) of the Splunk endpoint, can be a single URL space seperated list |
| SPLUNK_HEC_TOKEN | string | Splunk HTTP Event Collector Token |
| SC4S_USE_REVERSE_DNS | yes or no(default) | use reverse DNS to identify hosts when HOST is not valid in the syslog header |
| SC4S_CONTAINER_HOST | string | variable passed to the container to identify the actual log host for container implementations |

* NOTE:  Do _not_ configure HEC Acknowledgement when deploying the HEC token on the Splunk side; the underlying syslog-ng http
destination does not support this feature.  Moreover, HEC Ack would significantly degrade performance for streaming data such as
syslog.

* NOTE:  Use of the `SC4S_USE_REVERSE_DNS` variable can have a significant impact on performance if the reverse DNS facility
(typically a caching nameserver) is not performant.  If you notice events being indexed far later than their actual timestamp
in the event (latency between `_indextime` and `_time`), this is the first place to check.


## Splunk HEC Destination Configuration

| Variable | Values        | Description |
|----------|---------------|-------------|
| SC4S_DEST_SPLUNK_HEC_GLOBAL | yes | Send events to Splunk using HEC.  This applies _only_ to the primary HEC destination. |
| SC4S_DEST_SPLUNK_HEC_CIPHER_SUITE | comma separated list | Open SSL cipher suite list |
| SC4S_DEST_SPLUNK_HEC_SSL_VERSION |  comma separated list | Open SSL version list |
| SC4S_DEST_SPLUNK_HEC_TLS_CA_FILE | _container_ path `/etc/syslog-ng/tls/server.pem` | Custom trusted cert file, specified as a full path in the _container_ filesystem: `/etc/syslog-ng/tls/<ca-file>`<br>Ensure that the container TLS directory `/etc/syslog-ng/tls` is available locally via container mount in the `docker-compose.yml` or systemd unit file, and that you place the CA file in the locally-mounted directory. |
| SC4S_DEST_SPLUNK_HEC_TLS_VERIFY | yes(default) or no | verify HTTP(s) certificate |
| SC4S_DEST_SPLUNK_HEC_WORKERS | numeric | Number of destination workers (default: 10 threads).  This should rarely need to be changed; consult sc4s community for advice on appropriate setting in extreme high- or low-volume environments. |
| SC4S_DEST_SPLUNK_INDEXED_FIELDS | facility,<br>severity,<br>container,<br>loghost,<br>destport,<br>fromhostip,<br>proto<br><br>none | List of sc4s indexed fields that will be included with each event in Splunk (default is the entire list except "none").  Two other indexed fields, `sc4s_vendor_product` and `sc4s_syslog_format`, will also appear along with the fields selected via the list and cannot be turned on or off individually.  If no indexed fields are desired (including the two internal ones), set the value to the single value of "none".  When setting this variable, separate multiple entries with commas and do not include extra spaces.<br><br>This list maps to the following indexed fields that will appear in all Splunk events:<br>facility: sc4s_syslog_facility<br>severity: sc4s_syslog_severity<br>container: sc4s_container<br>loghost: sc4s_loghost<br>dport: sc4s_destport<br>fromhostip: sc4s_fromhostip<br>proto: sc4s_proto

* NOTE:  When using alternate HEC destinations, the destination operating paramaters outlined above (`CIPHER_SUITE`, `SSL_VERSION`, etc.) can be
individually controlled per `DESTID` (see "Configuration of Additional Splunk HEC Destinations" immediately below).  For example, to set the number of workers
for the alternate HEC destination `d_hec_FOO` to 24, set `SC4S_DEST_SPLUNK_HEC_FOO_WORKERS=24`.

## Configuration of Alternate Destinations

In addition to the standard HEC destination that is used to send events to Splunk, alternate distinations can be created and configured
in SC4S.  All alternate destinations (including alternate HEC destinations discussed below) are configured using the environment
variables below.  Global and/or source-specific forms of the variables below can be used to send data to additional and/or alternate
destinations.

* NOTE:  The administrator is responsible for ensuring that any non-HEC alternate destinations are configured in the
local mount tree, and that the underlying syslog-ng process in SC4S properly parses them.

* NOTE:  Do not include the primary HEC destination (`d_hec`) in any list of alternate destinations.  The configuration of the primary HEC
destination is configured separately from that of the alternates below.  However, _alternate_ HEC destinations (e.g. `d_hec_FOO`) should be
configured below, just like any other user-supplied destination.

| Variable | Values        | Description |
|----------|---------------|-------------|
| SC4S_DEST_GLOBAL_ALTERNATES | Comma or space-separated list of destinations | Send all sources to alternate destinations |
| SC4S_DEST_&lt;VENDOR_PRODUCT&gt;_ALTERNATES | Comma or space-separated list of syslog-ng destinations  | Send specific sources to alternate syslog-ng destinations using the VENDOR_PRODUCT syntax, e.g. `SC4S_DEST_CISCO_ASA_ALTERNATES`  |

## Configuration of Filtered Alternate Destinations (Advanced)

Though source-specific forms of the variables configured above will limit configured alternate destinations to a specifc data source, there
are cases where even more granularity is desired within a specific data source (e.g. to send all Cisco ASA "debug" traffic to Cisco Prime for
analysis).  This extra traffic may or may not be needed in Splunk.  To accomudate this use case, Filtered Alternate Destinations allow a
filter to be supplied to redirect a _portion_ of a given source's traffic to a list of altnerate destinations (and, optionally, to prevent
matching events from being sent to Splunk).  Again, these are configured through environment variables similar
to the ones above:

| Variable | Values        | Description |
|----------|---------------|-------------|
| SC4S_DEST_&lt;VENDOR_PRODUCT&gt;_ALT_FILTER | syslog-ng filter | Filter to determine which events are sent to alternate destination(s) |
| SC4S_DEST_&lt;VENDOR_PRODUCT&gt;_FILTERED_ALTERNATES | Comma or space-separated list of syslog-ng destinations  | Send filtered events to alternate syslog-ng destinations using the VENDOR_PRODUCT syntax, e.g. `SC4S_DEST_CISCO_ASA_FILTERED_ALTERNATES`  |


* NOTE:  This is an advanced capability, and filters and destinations using proper syslog-ng syntax must be constructed prior to utilizing
this feature.

* NOTE:  Unlike the standard alternate destinations configured above, the regular "mainline" destinations (including the primary HEC
destination or configured archive destination (`d_hec` or `d_archive`)) are _not_ included for events matching the configured alternate
destination filter.  If an event matches the filter, the list of filtered alternate destinations completely replaces any mainline destinations
including defaults and global or source-based standard alternate destinations.  Be sure to include them in the filtered destination list if
desired.

* HINT:  Since the filtered alternate destinations completely replace the mainline destinations (including HEC to Splunk), a filter that
matches all traffic can be used with a destination list that does _not_ include the standard HEC destination to effectively turn off HEC
for a given data source.

## Creation of Additional Splunk HEC Destinations

Additional Splunk HEC destinations can be dynamically created through environment variables. When set, the destinations will be
created with the `DESTID` appended, for example: `d_hec_FOO`.  These destinations can then be specified for use (along with any other
destinations created locally) either globally or per source.  See the "Alternate Destination Use" in the next section for details.

| Variable | Values        | Description |
|----------|---------------|-------------|
| SPLUNK_HEC_ALT_DESTS | Comma or space-separated UPPER case list of destination IDs | Destination IDs are UPPER case, single-word friendly strings used to identify the new destinations which will be named with the `DESTID` appended, for example `d_hec_FOO` |
| SPLUNK_HEC_&lt;DESTID&gt;_URL | url | Example: `SPLUNK_HEC_FOO_URL=https://splunk:8088`  `DESTID` must be a member of the list specified in `SPLUNK_HEC_ALT_DESTS` configured above |
| SPLUNK_HEC_&lt;DESTID&gt;_TOKEN | string | Example: `SPLUNK_HEC_BAR_TOKEN=<token>`  `DESTID` must be a member of the list specified in `SPLUNK_HEC_ALT_DESTS` configured above |

* NOTE:  The `DESTID` specified in the `URL` and `TOKEN` variables above _must_ match the `DESTID` entries enumerated in the
`SPLUNK_HEC_ALT_DESTS` list. For each `DESTID` value specified in `SPLUNK_HEC_ALT_DESTS` there must be a corresponding `URL` and `TOKEN`
variable set as well. Failure to do so will cause destinations to be created without proper HEC parameters which will result in connection
failure.

* NOTE:  Additional Splunk HEC destinations will _not_ be tested at startup.  It is the responsiblity of the admin to ensure that additional destinations
are provisioned with the correct URL(s) and tokens to ensure proper connectivity.

* NOTE: The disk and CPU requirements will increase proportionally depending on the number of additional HEC destinations in use (e.g. each HEC
destination will have its own disk buffer by default).

## Configuration of timezone for legacy sources

Legacy sources (those that remain non compliant with RFC5424) often leave the recipient to
guess at the actual time zone offset. SC4S uses an advanced feature of syslog-ng to "guess" the correct time zone for real time sources.
However, this feature requires the source (device) clock to be synchronized to within +/- 30s of the SC4S system clock.
Industry accepted best practice is to set such legacy systems to GMT (sometimes inaccurately called UTC).
However, this is not always possible and in such cases two additional methods are available. For a list of [time zones see](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones). Only the "TZ Database name" OR "offset" format may be used.

### Change Global default time zone

Set the `SC4S_DEFAULT_TIMEZONE` variable to a recognized "zone info" (Region/City) time zone format such as `America/New_York`.
Setting this value will force SC4S to use the specified timezone (and honor its associated Daylight Savings/Summer Time rules)
for all events without a timezone offset in the header or message payload.

### Change by host or subnet match

Using the following example "vendor_product_by_source" configuration as a guide, create a matching host wildcard pattern (glob)
to identify all devices in the "east" datacenter located in the Eastern US time zone.  Though not shown in the example, IP/CIDR
blocks and other more complex filters can also be used, but be aware of the performance implications of complex filtering.

```
#vendor_product_by_source.conf
#Note that all filter syntax options of syslog-ng are available here, but be aware that complex filtering
#can have a negative impact on performance.

filter f_tzfif_dc_us_eastxny {
    host("*-D001-*" type(glob))
};

#vendor_product_by_source.csv
#Add the following line

f_dc_us_east,sc4s_time_zone,"America/New_York"
```

## SC4S Disk Buffer Configuration

Disk buffers in SC4S are allocated _per destination_.  Keep this in mind when using additional destinations that have disk buffering configured.  By
default, when alternate HEC destinations are configured as outlined above disk buffering will be configured identically to that of the main HEC
destination (unless overriden individually).

### Important Notes Regarding Disk Buffering:

* "Reliable" disk buffering offers little advantage over "normal" disk buffering, at a significant performance penalty.
For this reason, normal disk buffering is recommended.

* If you add destinations locally in your configuration, pay attention to the _cumulative_ buffer requirements when allocating local
disk.

* Disk buffer storage is configured via container volumes and is persistent between restarts of the conatiner.
Be sure to account for disk space requirements on the local sc4s host when creating the container volumes in your respective
runtime environment (outlined in the "getting started" runtime docs). These volumes can grow significantly if there is
an extended outage to the SC4S destinations (HEC endpoints). See the "SC4S Disk Buffer Configuration" section on the Configruation
page for more info.

* The values for the variables below represent the _total_ sizes of the buffers for the destination.  These sizes are divded by the
number of workers (threads) when setting the actual syslog-ng buffer options, because the buffer options apply to each worker rather than the
entire destination.  Pay careful attention to this when using the "BYOE" version of SC4S, where direct access to the syslog-ng config files
may hide this nuance.  Lastly, be sure to factor in the syslog-ng data structure overhead (approx. 2x raw message size) when calculating the
total buffer size needed. To determine the proper size of the disk buffer, consult the "Data Resilience" section below.

* When changing the disk buffering directory, the new directory must exist.  If it doesn't, then syslog-ng will fail to start.

* When changing the disk buffering directory, if buffering has previously occurd on that instance, a persist file may exist which will prevent syslog-ng from changing the directory.

### Disk Buffer Variables

| Variable | Values/Default   | Description |
|----------|---------------|-------------|
| SC4S_DEST_SPLUNK_HEC_DISKBUFF_ENABLE | yes(default) or no | Enable local disk buffering  |
| SC4S_DEST_SPLUNK_HEC_DISKBUFF_RELIABLE | yes or no(default) | Enable reliable/normal disk buffering (normal recommended) |
| SC4S_DEST_SPLUNK_HEC_DISKBUFF_MEMBUFSIZE | bytes (10241024) | Memory buffer size in bytes (used with reliable disk buffering) |
| SC4S_DEST_SPLUNK_HEC_DISKBUFF_MEMBUFLENGTH |messages (15000) | Memory buffer size in message count (used with normal disk buffering) |
| SC4S_DEST_SPLUNK_HEC_DISKBUFF_DISKBUFSIZE | bytes (53687091200) | Size of local disk buffer in bytes (default 50 GB) |
| SC4S_DEST_SPLUNK_HEC_DEFAULT_DISKBUFF_DIR | path | Location to store the disk buffer files.  This variable should _only_ be set when using BYOE; this location is fixed when using the Container.  |

## Archive File Configuration

This feature is designed to support compliance or "diode mode" archival of all messages. Instructions for mounting the appropriate
local directory to use this feature are included in each "getting started" runtime document. The files will be stored in a folder
structure at the mount point using the pattern shown in the table below depending on the value of the `SC4S_GLOBAL_ARCHIVE_MODE` variable.
All events for both modes are formatted using syslog-ng's EWMM template.

| Variable | Value/Default    | Location/Pattern |
|----------|------------------|------------------|
| SC4S_GLOBAL_ARCHIVE_MODE | compliance(default) | ``<archive mount>/${YEAR}/${MONTH}/${DAY}/${fields.sc4s_vendor_product}_${YEAR}${MONTH}${DAY}${HOUR}${MIN}.log"`` |
| SC4S_GLOBAL_ARCHIVE_MODE | diode | ``<archive mount>/${.splunk.sourcetype}/${HOST}/$YEAR-$MONTH-$DAY-archive.log`` |

**WARNING POTENTIAL OUTAGE CAUSING CONSEQUENCE**

Use the following variables to select global archiving or per-source archiving.  C4S does not prune the files that are created;
therefore the administrator must provide a means of log rotation to prune files and/or move them to an archival system to avoid exhaustion of disk space.

| Variable | Values        | Description |
|----------|---------------|-------------|
| SC4S_ARCHIVE_GLOBAL | yes or undefined | Enable archive of all vendor_products |
| SC4S_ARCHIVE_&lt;VENDOR_PRODUCT&gt; | yes(default) or undefined | See sources section of documentation enables selective archival |


## Syslog Source Configuration

| Variable | Values/Default | Description |
|----------|----------------|-------------|
| SC4S_SOURCE_TLS_ENABLE | yes or no(default) | Enable TLS globally.  Be sure to configure the cert as shown immediately below. |
| SC4S_LISTEN_DEFAULT_TLS_PORT | undefined or 6514 | Enable a TLS listener on port 6514 |
| SC4S_LISTEN_DEFAULT_RFC6425_PORT | undefined or 5425 | Enable a TLS listener on port 5425 |
| SC4S_SOURCE_TLS_OPTIONS | `no-sslv2` | Comma-separated list of the following options: `no-sslv2, no-sslv3, no-tlsv1, no-tlsv11, no-tlsv12, none`.  See syslog-ng docs for the latest list and defaults |
| SC4S_SOURCE_TLS_CIPHER_SUITE | See openssl | Colon-delimited list of ciphers to support, e.g. `ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384`.  See openssl docs for the latest list and defaults |
| SC4S_SOURCE_TCP_MAX_CONNECTIONS | 2000 | Max number of TCP Connections |
| SC4S_SOURCE_TCP_IW_SIZE | 20000000 | Initial Window size |
| SC4S_SOURCE_TCP_FETCH_LIMIT | 2000 | Number of events to fetch from server buffer at once |
| SC4S_SOURCE_UDP_SO_RCVBUFF | 17039360 | UDP server buffer size in bytes. Make sure that the host OS kernel is configured [similarly](gettingstarted/index.md#prerequisites). |
| SC4S_SOURCE_LISTEN_UDP_SOCKETS | 1 | Number of kernel sockets per active UDP port, which configures multi-threading of the UDP input buffer in the kernel to prevent packet loss.  Total UDP input buffer is the multiple of SC4S_SOURCE_LISTEN_UDP_SOCKETS * SC4S_SOURCE_UDP_SO_RCVBUFF |
| SC4S_SOURCE_STORE_RAWMSG | undefined or "no" | Store unprocessed "on the wire" raw message in the RAWMSG macro for use with the "fallback" sourcetype.  Do _not_ set this in production; substantial memory and disk overhead will result. Use for log path/filter development only. |
| SC4S_IPV6_ENABLE | yes or no(default) | enable (dual-stack)IPv6 listeners and health checks |

## Syslog Source TLS Certificate Configuration

* Create a folder ``/opt/sc4s/tls`` if not already done as part of the "getting started" process.
* Uncomment the appropriate mount line in the unit or yaml file (again, documented in the "getting started" runtime documents).
* Save the server private key in PEM format with NO PASSWORD to ``/opt/sc4s/tls/server.key``
* Save the server certificate in PEM format to ``/opt/sc4s/tls/server.pem``
* Ensure the entry `SC4S_SOURCE_TLS_ENABLE=yes` exists in ``/opt/sc4s/env_file``

## SC4S metadata configuration

### Log Path overrides of index or metadata

A key aspect of SC4S is to properly set Splunk metadata prior to the data arriving in Splunk (and before any TA processing 
takes place).  The filters will apply the proper index, source, sourcetype, host, and timestamp metadata automatically by
individual data source.  Proper values for this metadata, including a recommended index and output format (template), are
included with all "out-of-the-box" log paths included with SC4S and are chosen to properly interface with the corresponding
TA in Splunk.  The administrator will need to ensure all recommneded indexes be created to accept this data if the defaults
are not changed.

It will be common to override default values in many installations. To accomodate this, each log path consults
an internal lookup file that maps Splunk metadata to the specific data source being processed.  This file contains the
defaults that are used by SC4S to set the appropriate Splunk metadata (`index`, `host`, `source`, and `sourcetype`) for each
data source.  This file is not directly available to the administrator, but a copy of the file is deposited in the local mouunted directory
(by default `/opt/sc4s/local/context/splunk_metadata.csv.example`) for reference.  It is important to note that this copy is _not_ used
directly, but is provided solely for reference.  To add to the list, or to override default entries, simply create an override file without
the `example` extension (e.g. `/opt/sc4s/local/context/splunk_metadata.csv`) and modify it according to the instructions below.

`splunk_metadata.csv` is a CSV
file containing a "key" that is referenced in the log path for each data source.  These keys are documented in the individual
source files in this section, and allow one to override Splunk metadata either in whole or part. The use of this file is best
shown by example.  Here is the Netscreen "Sourcetype and Index Configuration" table from the Juniper
[source documentation](sources/Juniper/index.md):

| key                    | sourcetype          | index          | notes         |
|------------------------|---------------------|----------------|---------------|
| juniper_netscreen      | netscreen:firewall  | netfw          | none          |

Here is a line from a typical `splunk_metadata.csv` override file:

```bash
juniper_netscreen,index,ns_index
```

The columns in this file are `key`, `metadata`, and `value`.  To make a change via the override file, consult the `example` file (or
the source documentation) for the proper key when overrdiing an existing source and modify and/or add rows in the table, specifying one or
more of the following `metadata/value` pairs for a given `key`:

   * `key` which refers to the vendor and product name of the data source, using the `vendor_product` convention.  For overrides, these keys
   will be listed in the `example` file.  For new (custom) sources, be sure to choose a key that accurately reflects the vendor and product
   being configured, and that matches what is specified in the log path.
   * `index` to specify an alternate `value` for index
   * `source` to specify an alternate `value` for source
   * `host` to specify an alternate `value` for host
   * `sourcetype` to specify an alternate `value` for sourcetype (be _very_ careful when changing this; only do so if an upstream
    TA is _not_ being used, or a custom TA (built by you) is being used.)
   * `sc4s_template` to specify an alternate `value` for the syslog-ng template that will be used to format the event that will be
   indexed by Splunk.  Changing this carries the same warning as the sourcetype above; this will affect the upstream TA.  The template
   choices are documented [elsewhere](configuration.md#splunk-connect-for-syslog-output-templates-syslog-ng-templates) in this Configuration section.

In our example above, the `juniper_netscreen` key references a new index used for that data source called `ns_index`.

In general, for most deployments the index should be the only change needed; other default metadata should almost
never be overridden (particularly for the "Out of the Box" data sources).  Even then, care should be taken when considering any alternates,
as the defaults for SC4S were chosen with best practices in mind.

* NOTE:  The `splunk_metadata.csv` file is a true override file and the entire `example` file should not be copied over to the
override.  In most cases, the override file is just one or two lines, unless an entire index category (e.g. `netfw`) needs to be overridden.
This is similar in concept to the "default" and "local" conf file precedence in Splunk Enterprise.

* NOTE The `splunk_metadata.csv` file should always be appended with an appropriate new key and default for the index when building a custom
SC4S log path, as the new key will not exist in the internal lookup (nor the `example` file).  Care should be taken during log path design to
choose appropriate index, sourctype and template defaults so that admins are not compelled to override them.  If the custom log path is later
added to the list of SC4S-supported sources, this addendum can be removed.

* NOTE:  As noted above, the `splunk_metadata.csv.example` file is provided for reference only and is not used directly by SC4S.  However,
it is an exact copy of the internal file, and can therefore change from release to release.  Be sure to check the example file first to make
sure the keys for any overrides map correctly to the ones in the example file.

### Override index or metadata based on host, ip, or subnet (compliance overrides)

In other cases it is appropriate to provide the same overrides but based on PCI scope, geography, or other criterion rather than globally.
This is accomplished by the use of a file that uniquely identifies these source exceptions via syslog-ng filters,
which maps to an associated lookup of alternate indexes, sources, or other metadata.  In addition, (indexed) fields can also be
added to futher classify the data.

* The `conf` and `csv` files referenced below will be populated into the `/opt/sc4s/local/context` directory when SC4S is run for the first
time after being set up according to the "getting started" runtime documents, in a similar fashion to `splunk_metadata.csv`.
After this first-time population of the files takes place, they can be edited (and SC4S restarted) for the changes to take effect.  To get started:

* Edit the file ``compliance_meta_by_source.conf`` to supply uniquely named filters to identify events subject to override.
* Edit the file ``compliance_meta_by_source.csv`` to supply appropriate field(s) and values.

The three columns in the `csv` file are `filter name`, `field name`, and `value`.  Filter names in the `conf` file must match one or more
corresonding `filter name` rows in the `csv` file.  The `field name` column obeys the following convention:

   * `.splunk.index` to specify an alternate `value` for index
   * `.splunk.source` to specify an alternate `value` for source
   * `.splunk.sourcetype` to specify an alternate `value` for sourcetype (be _very_ careful when changing this; only do so if a downstream
    TA is _not_ being used, or a custom TA (built by you) is being used.)
   * `fields.fieldname` where `fieldname` will become the name of an indexed field sent to Splunk with the supplied `value`    

This file construct is best shown by an example.  Here is a sample ``compliance_meta_by_source.conf`` file:

```
filter f_test_test {
   host("something-*" type(glob)) or
   netmask(192.168.100.1/24)
};
```
and the corresponding ``compliance_meta_by_source.csv`` file:

```
f_test_test,.splunk.index,"pciindex"
f_test_test,fields.compliance,"pci"
```

First off, ensure that the filter name(s) in the `conf` file match
one or more rows in the `csv` file. In this case, any incoming message with a hostname starting with `something-` or arriving from a netmask
of `192.168.100.1/24` will match the `f_test_test` filter, and the corresponding entries in the `csv` file will be checked for overrides.
In this case, the new index is `pciindex`, and an indexed field named `compliance` will be sent to Splunk, with it's value set to `pci`.
To add additional overrides, simply add another `filter foo_bar {};` stanza to the `conf` file, and add appropriate entries to the `csv` file
that match the filter name(s) to the overrides you deisre.

* IMPORTANT:  The files above are actual syslog-ng config file snippets that get parsed directly by the underlying syslog-ng
process.  Take care that your syntax is correct; for more information on proper syslog-ng syntax, see the syslog-ng
[documentation](https://www.syslog-ng.com/technical-documents/doc/syslog-ng-open-source-edition/3.24/administration-guide/57#TOPIC-1298086).
A syntax error will cause the runtime process to abort in the "preflight" phase at startup.

Finally, to update your changes for the systemd-based runtimes, restart SC4S using the commands:
```
sudo systemctl daemon-reload
sudo systemctl restart sc4s
```

For the Docker Swarm runtime, redeploy the updated service using the command:
```
docker stack deploy --compose-file docker-compose.yml sc4s
```

## Dropping all data by ip or subnet

In some cases rogue or port-probing data can be sent to SC4S from misconfigured devices or vulnerability scanners. Update
the `vendor_product_by_source.conf` filter `f_null_queue` with one or more ip/subnet masks to drop events without
logging. Note that drop metrics will be recorded.

## Fixing (overriding) the host field

In some cases the host value is not present in an event (or an IP address is in its place).  For administrators
who require a true hostname be attached to each event, SC4S provides an optional facilty to perform a reverse IP to
name lookup. If the variable `SC4S_USE_REVERSE_DNS` is set to "yes", SC4S
will first check `host.csv` and replace the value of `host` with the value specified that matches the incoming IP address.
If a value is not found in `host.csv` then a reverse DNS lookup will be attempted against the configured nameserver.
The IP address will only be used as the host value as a last resort.

* NOTE:  Use of this variable can have a significant impact on performance if the reverse DNS facility (typically a caching
nameserver) is not performant.  If you notice events being indexed far later than their actual timestamp in the event (latency
between `_indextime` and `_time`), this is the first place to check.

## Splunk Connect for Syslog output templates (syslog-ng templates)

Splunk Connect for Syslog utilizes the syslog-ng template mechanism to format the output payload (event) that will be sent to Splunk.
These templates can format the messages in a number of ways (straight text, JSON, etc.) as well as utilize the many syslog-ng
"macros" (fields) to specify what gets placed in the payload that is delivered to the destination.  Here is a list of the templates
used in SC4S, which can be used in the metadata override section immediately above.  New templates can also be added by the
administrator in the "local" section for local destinations; pay careful attention to the syntax as the templates are "live"
syslog-ng config code.

| Template name       | Template contents                        |  Notes                                                           |
|---------------------|------------------------------------------|------------------------------------------------------------------|
| t_standard          | ${DATE} ${HOST} ${MSGHDR}${MESSAGE}      |  Standard template for most RFC3164 (standard syslog) traffic    |
| t_msg_only          | ${MSGONLY}                               |  syslog-ng $MSG is sent, no headers (host, timestamp, etc.)      |
| t_msg_trim          | $(strip $MSGONLY)                        |  As above with whitespace stripped                               |
| t_everything        | ${ISODATE} ${HOST} ${MSGHDR}${MESSAGE}   |  Standard template with ISO date format                          |
| t_hdr_msg           | ${MSGHDR}${MESSAGE}                      |  Useful for non-compliant syslog messages                        |
| t_legacy_hdr_msg    | ${LEGACY_MSGHDR}${MESSAGE}               |  Useful for non-compliant syslog messages                        |
| t_hdr_sdata_msg     | ${MSGHDR}${MSGID} ${SDATA} ${MESSAGE}    |  Useful for non-compliant syslog messages                        |
| t_program_msg       | ${PROGRAM}[${PID}]: ${MESSAGE}           |  Useful for non-compliant syslog messages                        |
| t_program_nopid_msg | ${PROGRAM}: ${MESSAGE}                   |  Useful for non-compliant syslog messages                        |
| t_JSON_3164         | $(format-json --scope rfc3164<br>--pair PRI="<$PRI>"<br>--key LEGACY_MSGHDR<br>--exclude FACILITY<br>--exclude PRIORITY)   |  JSON output of all RFC3164-based syslog-ng macros.  Useful with the "fallback" sourcetype to aid in new filter development. |
| t_JSON_5424         | $(format-json --scope rfc5424<br>--pair PRI="<$PRI>"<br>--key ISODATE<br>--exclude DATE<br>--exclude FACILITY<br>--exclude PRIORITY)  |  JSON output of all RFC5424-based syslog-ng macros; for use with RFC5424-compliant traffic. |
| t_JSON_5424_SDATA   | $(format-json --scope rfc5424<br>--pair PRI="<$PRI>"<br>--key ISODATE<br>--exclude DATE<br>--exclude FACILITY<br>--exclude PRIORITY)<br>--exclude MESSAGE  |  JSON output of all RFC5424-based syslog-ng macros except for MESSAGE; for use with RFC5424-compliant traffic. |

## Data Resilience - Local Disk Buffer Configuration

SC4S provides capability to minimize the number of lost events if the connection to all the Splunk Indexers goes down. 
This capability utilizes the disk buffering feature of Syslog-ng. SC4S receives a response from the Splunk HTTP Event
Collector (HEC) when a message is received successfully. If a confirmation message from the HEC endpoint is not
received (or a “server busy” reply, such as a “503” is sent), the load balancer will try the next HEC endpoint in the pool.
If all pool members are exhausted (such as would occur if there were a full network outage to the HEC endpoints), events
will queue to the local disk buffer on the SC4S Linux host. SC4S will continue attempting to send the failed
events while it buffers all new incoming events to disk. If the disk space allocated to disk buffering fills up then SC4S
will stop accepting new events and subsequent events will be lost. Once SC4S gets confirmation that events are again being
received by one or more indexers, events will then stream from the buffer using FIFO queueing. The number of
events in the disk buffer will reduce as long as the incoming event volume is less than the maximum SC4S (with the disk
buffer in the path) can handle. When all events have been emptied from the disk buffer, SC4S will resume streaming events
directly to Splunk.

For more detail on the Syslog-ng behavior the documentation can be found here:
https://www.syslog-ng.com/technical-documents/doc/syslog-ng-open-source-edition/3.22/administration-guide/55#TOPIC-1209280

SC4S has disk buffering enabled by default and it is strongly recommended that you keep it on, however this feature does
have a performance cost.
Without disk buffering enabled SC4S can handle up to 345K EPS (800 bytes/event avg)
With “Normal” disk buffering enabled SC4S can handle up to 60K EPS (800 bytes/event avg) -- This is still a lot of data!

To guard against data loss it is important to configure the appropriate type and amount of storage for SC4S disk buffering.
To estimate the storage allocation, follow these steps:

* Start with your estimated maximum events per second that each SC4S server will experience. Based on the maximum
throughput of SC4S with disk buffering enabled, the conservative estimate for maximum events per second would be 60K
(however, you should use the maximum rate in your environment for this calculation, not the max rate SC4S can handle). 
* Next is your average estimated event size based on your data sources. It is common industry practice to estimate log
events as 800 bytes on average. 
* Then, factor in the maximum length of connectivity downtime you want disk buffering to be able to handle. This measure
is very much dependent on your risk tolerance.
* Lastly, syslog-ng imposes significant overhead to maintain its internal data structures (primarily macros) so that the
data can be properly "played back" upon network restoration.  This overhead currently runs at about 1.7x above the total
storage size for the raw messages themselves, and can be higher for "fallback" data sources due to the overlap of syslog-ng
macros (data fields) containing some or all of the original message.

For example, to protect against a full day of lost connectivity from SC4S to all your indexers at maximum throughput the
calculation would look like the following:

60,000 EPS * 86400 seconds * 800 bytes * 1.7 = 6.4 TB of storage

To configure storage allocation for the SC4S disk buffering, do the following:

* Edit the file /opt/sc4s/default/env_file
* Add the SC4S_DEST_SPLUNK_HEC_DISKBUFF_DISKBUFSIZE variable to the file and set the value to the number of bytes based
on your estimation (e.g. 7050240000000 in the example above)
* Splunk does not recommend reducing the disk allocation below 500 GB
* Restart SC4S

Given that in a connectivity outage to the Indexers events will be saved and read from disk until the buffer is emptied,
it is ideal to use the fastest type of storage available. For this reason, NVMe storage is recommended for SC4S disk buffering.

It is best to design your deployment so that the disk buffer will drain after connectivity is restored to the Splunk Indexers
(while incoming data continues at the same general rate).  Since "your mileage may vary" with different combinations of
data load, instance type, and disk subsystem performance, it is good practice to provision a box that performs twice as
well as is required for your max EPS. This headroom will allow for rapid recovery after a connectivity outage.
