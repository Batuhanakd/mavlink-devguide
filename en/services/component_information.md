# Component Information Protocol

> **Note** This service is still marked as "work in progress", and should not be relied upon in production setups.

The *Component Information Protocol* is a MAVLink service for requesting information from (and about) MAVLink components. 
It is intended to provide autopilot- and version- independent feature discovery and configuration, allowing a GCS to configure its UI and/or a device without knowing anything about the connected system.

Information shared using this service may include:
- What types of component information are supported (by this component).
- What MAVLink commands are supported (both in missions and in other modes).
- Parameter metadata for parameters supported by the vehicle.
- Metadata about events emitted by the system
- Self-describing configuration UIs (i.e. similar to MAVLink camera configuration files).
- Translations of other metadata.

The service interface uses the [COMPONENT_INFORMATION](#COMPONENT_INFORMATION) message to deliver information about the location of JSON-format [component information files](#schema_files) (and associated translation files).

Component information is requested using [MAV_CMD_REQUEST_MESSAGE](#MAV_CMD_REQUEST_MESSAGE), specifying the [COMPONENT_INFORMATION](#COMPONENT_INFORMATION) message and [type](#COMP_METADATA_TYPE) of component information required.

> **Note**  The `COMP_METADATA_TYPE_VERSION` type provides information about the data types the component supports, and should therefore be requested first.

The `COMPONENT_INFORMATION` message provides a URL to where a file containing the requested information can be downloaded, along with a uid that can be used to determine if the file has changed since last downloaded (effectively a hash for the file). 
The URL is either a HTTP or MAVLink FTP address; the second is preferred because the file supplied by a vehicle can always be assumed to be up to date.

In addition, the `COMPONENT_INFORMATION` may include a URL and uid for a translation file.
If supplied, this contains translation strings for the information file specified in the message, in JSON translation file format (superseding any translations in the information file).

Information supplied by the service is assumed to be invariant after boot.
There is no mechanism, for example, to provide an update if the set of supported parameters was to change after boot.

> **Note** Component information files may be **.gz** compressed.
  Systems that request component information must support extraction of .gz compressed JSON files.

## Message/Enum Summary

Message | Description
-- | --
<span id="COMPONENT_INFORMATION"></span>[COMPONENT_INFORMATION](../messages/common.md#COMPONENT_INFORMATION) | Message providing a download url for a component information file of a specific [type](../messages/common.md#COMP_METADATA_TYPE) of (and associated translations). The message is requested using [MAV_CMD_REQUEST_MESSAGE](#MAV_CMD_REQUEST_MESSAGE)
<span id="MAV_CMD_REQUEST_MESSAGE"></span>[MAV_CMD_REQUEST_MESSAGE](../messages/common.md#MAV_CMD_REQUEST_MESSAGE) | Use this command to request that a component emit [COMPONENT_INFORMATION](#COMPONENT_INFORMATION). Use `param1=395` (the message id of `COMPONENT_INFORMATION`) and `param2` is the requested information type (a [COMP_METADATA_TYPE](#COMP_METADATA_TYPE)).


Enum | Description
-- | --
<span id="COMP_METADATA_TYPE"></span>[COMP_METADATA_TYPE](../messages/common.md#COMP_METADATA_TYPE) | A specific type of metadata about a component - e.g. version information, parameter metadata, supported commands, events etc. This is used to indicate the type of metadata requested with/referenced in a [COMPONENT_INFORMATION](#COMPONENT_INFORMATION) message.

## Component Information Schema/Format {#schema_files}

Component information files are written in JSON and must conform to the schema definitions found in the folder [/component_information](https://github.com/mavlink/mavlink/tree/master/component_information).
The schema are (at time of writing):

Type | Schema | Description
--- | --- | ---
`COMP_METADATA_TYPE_VERSION` | [version.schema.json](https://github.com/mavlink/mavlink/blob/master/component_information/version.schema.json) | Hardware/firmware vendor and version information. Information about what (other) metadata types are supported. Must be supported if this protocol is supported. Must be present on vehicle and delivered by MAVLink FTP.
`COMP_METADATA_TYPE_PARAMETER` | [parameter.schema.json](https://github.com/mavlink/mavlink/blob/master/component_information/parameter.schema.json) | Parameter metadata - information about parameters supported by the vehicle (on boot). 
`COMP_METADATA_TYPE_COMMANDS` | - | Schema not yet defined. Intended to specify which commands are supported in missions.

All schema files are *versioned* using a `version` integer.

Schema versions are backwards compatible - i.e. a ground station that was able to populate its UI from a file based on an older schema should be able to do so from a newer version (albeit losing information provided by the newer format).

Generally this means that new versions may add fields but should not delete them, and also that a recipient can ignore fields that it does not understand.

The schema are *managed* in the same way as the standard MAVLink definitions (i.e. breaking changes are avoided and modifications require consensus of the MAVLink stakeholders (as provided by the MAVLink dev call).

## Sequences 

### Get Component Information

The "high level" sequence for getting component information of any type is given below.

The sequence should be run first with `COMP_METADATA_TYPE_VERSION` to get the set of metadata types supported by the component (see `supportedCompMetadataTypes` key in the version metadata).
It can then be run for the other types once they are known.

> **Note** A component that supports this service must support the `COMP_METADATA_TYPE_VERSION` type and return a component file URI that is hosted on the vehicle (accessed using MAVLink FTP).


[![](https://mermaid.ink/img/eyJjb2RlIjoic2VxdWVuY2VEaWFncmFtO1xuICAgIHBhcnRpY2lwYW50IEdDU1xuICAgIHBhcnRpY2lwYW50IENvbXBvbmVudFxuICAgIE5vdGUgb3ZlciBDb21wb25lbnQsIEdDUzogR0NTOiBSZXF1ZXN0IGluZm9ybWF0aW9uIG9mIHBhcnRpY3VsYXIgdHlwZSAoZS5nLiBDT01QX01FVEFEQVRBX1RZUEVfVkVSU0lPTikuXG4gICAgR0NTLT4-Q29tcG9uZW50OiBNQVZfQ01EX1JFUVVFU1RfTUVTU0FHRSg8YnI-IHBhcmFtMT0zOTUsPGJyPiBwYXJhbTI9Q09NUF9NRVRBREFUQV9UWVBFX1hYWFgpXG4gICAgR0NTLS0-PkdDUzogU3RhcnQgQUNLIHJlY2VpdmUgdGltZW91dFxuICAgICAgQ29tcG9uZW50LT4-R0NTOiBDTURfQUNLXG4gICAgICBDb21wb25lbnQtPj5HQ1M6IENPTVBPTkVOVF9JTkZPUk1BVElPTihtZXRhZGF0YV91cmksbWV0YWRhdGFfdWlkKVxuICAgIE5vdGUgb3ZlciBDb21wb25lbnQsIEdDUzogR0NTIGdldHMgZmlsZSBmcm9tICdtZXRhZGF0YV91cmknPGJyPihvciB1c2VzIHN0b3JlZCBpbmZvcm1hdGlvbiBpZiAnbWV0YWRhdGFfdWlkJyBtYXRjaGVzIGNhY2hlKS4gXG5cbiAgICIsIm1lcm1haWQiOnsidGhlbWUiOiJkZWZhdWx0In0sInVwZGF0ZUVkaXRvciI6ZmFsc2V9)](https://mermaid-js.github.io/mermaid-live-editor/#/edit/eyJjb2RlIjoic2VxdWVuY2VEaWFncmFtO1xuICAgIHBhcnRpY2lwYW50IEdDU1xuICAgIHBhcnRpY2lwYW50IENvbXBvbmVudFxuICAgIE5vdGUgb3ZlciBDb21wb25lbnQsIEdDUzogR0NTOiBSZXF1ZXN0IGluZm9ybWF0aW9uIG9mIHBhcnRpY3VsYXIgdHlwZSAoZS5nLiBDT01QX01FVEFEQVRBX1RZUEVfVkVSU0lPTikuXG4gICAgR0NTLT4-Q29tcG9uZW50OiBNQVZfQ01EX1JFUVVFU1RfTUVTU0FHRSg8YnI-IHBhcmFtMT0zOTUsPGJyPiBwYXJhbTI9Q09NUF9NRVRBREFUQV9UWVBFX1hYWFgpXG4gICAgR0NTLS0-PkdDUzogU3RhcnQgQUNLIHJlY2VpdmUgdGltZW91dFxuICAgICAgQ29tcG9uZW50LT4-R0NTOiBDTURfQUNLXG4gICAgICBDb21wb25lbnQtPj5HQ1M6IENPTVBPTkVOVF9JTkZPUk1BVElPTihtZXRhZGF0YV91cmksbWV0YWRhdGFfdWlkKVxuICAgIE5vdGUgb3ZlciBDb21wb25lbnQsIEdDUzogR0NTIGdldHMgZmlsZSBmcm9tICdtZXRhZGF0YV91cmknPGJyPihvciB1c2VzIHN0b3JlZCBpbmZvcm1hdGlvbiBpZiAnbWV0YWRhdGFfdWlkJyBtYXRjaGVzIGNhY2hlKS4gXG5cbiAgICIsIm1lcm1haWQiOnsidGhlbWUiOiJkZWZhdWx0In0sInVwZGF0ZUVkaXRvciI6ZmFsc2V9)

In summary:
1. Send `MAV_CMD_REQUEST_MESSAGE` specifying `param1=395` and `param2=COMP_METADATA_TYPE_VERSION`.
   - This is a normal [command protocol](../services/command.md) request with timeouts and resends based on the ACK.
1. The component will ACK the command and immediately send the requested `COMPONENT_INFORMATION` message (populated with appropriate information and translation uri and uids).
1. GCS waits for the `COMPONENT_INFORMATION` message
   - If not recieved the GCS should resend the request (typically in the application level).
   - Once information is received the GCS checks if `COMPONENT_INFORMATION.metadata_uid` matches cached component information.
     If so, the sequence is **complete**.
     Otherwise ...
1. GCS downloads the file specified in the `metadata_uri` (also the translation files, if any).
   For `COMP_METADATA_TYPE_VERSION` the file must be hosted on the vehicle and downloaded using [MAVLink FTP](../services/ftp.md).
1. GCS parses version metadata for `supportedCompMetadataTypes` and caches the results.
1. The sequence above is repeated to get the other supported data types (either immediately, or as needed).
