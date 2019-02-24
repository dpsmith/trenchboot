Security Engine
===============

* [Purpose](#purpose)
* [Background](#background)
* [Approach](#approach)
   * [Core Engine](#core-engine)
      * [Policy File](#policy-file)
	 * [Format](#format)
	 * [Main Object](#main-object)
	 * [Collector Object](#collector-object)
	 * [Attestor Object](#attestor-object)
	 * [Launcher Object](#launcher-object)
	 * [Locating](#locating)
   * [Evidence Collectors](#evidence-collectors)
      * [Main Library Interface](#main-library-interface)
      * [Evidence Collector Interface](#evidence-collector-interface)
	 * [Combined Data Event](#combined-data-event)
      * [Exemplar Modules](#exemplar-modules)
   * [Attestor](#attestor)
      * [Main Library Interface](#main-library-interface-1)
      * [Attestor Interface](#attestor-interface)
      * [Exemplar Modules](#exemplar-modules-1)
   * [Target Launcher](#target-launcher)
      * [Exemplar Modules](#exemplar-modules-2)
   * [tpmtool Enhancement](#tpmtool-enhancement)
      * [TPM State](#tpm-state)
      * [Intel TXT Log Format](#intel-txt-log-format)


## Purpose

The Security Engine consist of the logical components that will consume a
launch policy and take one or more actions to evaluate the state of the system
necessary to enforce the policy. This may include but not limited to collecting
additional evidence, making attestation assertions, encryption key retrieval,
and file/block/drive decryption. The enforcement of the security policy will
result in a full, partial, or failed boot of the system.

## Background


## Approach

Logical components
 - Evidence Store
   - A runtime data store that can hold TPM log events from launch as well as
     any measurements taken by uroot system
 - Evidence Collector
   - Collects the evidence about system properties, parsing TPM log, hashing
     rootfs, hashing the target kernel and its configuration, manifest of
     hardware, firmware version(s), etc
 - Attestor
   - Conducts an attestation, including but not limited to local/key unseal,
     KMIP, and other attestation protocols using the collected evidence and
     captures the response
 - Target Launcher
   - Consume result from Attestor to prepare, such as unlock LUKS file system
     or obtain NFS/iSCSI mount, and launch target kernel
 - Core Engine
   - Reads policy and tasks the Evidence Collector to collect the evidence
     mandated, then request Attestator to make the configured attestation using
     the evidence collected, and pass the response to the Launcher

The sequencing of these components can be seen in the following diagram,
![Secure Engine Sequence Diagram](images/SE%20sequence.png)


### Core Engine

Implementation will be a u-root uinit main file with the following
responsibilities,
 - Responsible for requesting TPM locality 2
 - Responsible for loading policy/config file
   - The policy file must be measured and extended into PCR21
 - Responsible for parsing the policy into JSON configs for Collector(s),
   Attestor(s), and Launcher to be passed to the respective module frameworks

#### Policy File

The policy file will be a JSON file containing configuration and policy actions
for the u-root runtime that is the Core Engine.

##### Format

##### Main Object

| Field | Value | Description |
|:-----:|:-----:|:-----------:|
| default_action | halt, reboot, recovery | Default action for the Core Engine to take when an unrecoverable error occurs |
| collectors | [Collector Object] | Array of Collector JSON Objects |
| attestor | Attestor Object | Attestor JSON Object |
| launcher | Launcher Object | Launcher JSON Object |

##### Collector Object

| Field | Value | Description |
|:-----:|:-----:|:-----------:|
| type | string | Collector type |
| {key} | {value} | Collector relevant key/value pairs |

##### Attestor Object

| Field | Value | Description |
|:-----:|:-----:|:-----------:|
| type | string | Attestor type |
| {key} | {value} | Attestor relevant key/value pairs |

##### Launcher Object

| Field | Value | Description |
|:-----:|:-----:|:-----------:|
| type | string | Launcher type |
| {key} | {value} | Launcher relevant key/value pairs |

##### Locating

By default the Core Engine will iterate through each local block device, mount
the block device, and look for the file ```securelaunch.policy``` under one of
the standard directories (```/```, ```/efi```, ```/boot```). This search can be
overridden with the kernel command line parameter ```sl_policy```. The
```sl_policy``` parameter is of the format ```<type>:<option1>,..,<optionN>```. 


### Evidence Collectors

All Evidence Collector's core logic will be found under ```pkg/measurement```
and will implement the standard read, hash, and extend sequence to measure. The
Evidence Collector should assume that if it was invoked then the Core Engine
successfully requested locality 2 from the TPM and can use PCRs 20 and 21. The
PCR scheme to be used follows the TCG practice of the first PCR (20) for static
data and the second PCR (21) for dynamic data. In this situation static data is
considered data that is static across multiple instances with the same release
versioning and dynamic data is that data that may vary between different
instances as well as different instantiation of the same instance. For example
the target kernel would be static data but the kernel parameters is dynamic
data. Every measurement event should be recorded in an EvidenceLog.

#### Main Library Interface

The Evidence Collector main library should expose a GetCollector method that
accepts a JSON config for an individual collector. The GetCollector call will
consults an array of generator functions for all known collector modules. It
will then pass the config to each generator until the first one successfully
returns an instance of the Collector interface. The definition for the call
will look like the following,

```go
GetCollector(config []byte) (Collector, error)
```

#### Evidence Collector Interface

Each module will implement a generator function that returns a Collector
instance if provided a valid JSON config. The generator function will be
responsible for parsing the JSON config and creating an instance of the
Collector configured with the parsed configuration. The expected pattern for
the generator function will be as follows,

```go
NewTpmCollector(config []byte) (Collector, error)
```

A collector must implement the Collector interface that exposes a Collect
function. The Core Engine will call the Collect method for every collector JSON
config that resulted in a Collector instance. The Collect method will use the
TPM interface to send measurements to the TPM and record the event into a log.
The Collector interface is defined as follows,

```go
import "github.com/TrenchBoot/tpmtool/pkg/tpm"

type Collector interface {
	Collect(t *TPM) error
}
```

##### Combined Data Event

A Collector must balance the need for a fine grained event log overhead with
resource consumption. As a result a Collector may opt to conduct a
```COMBINED_EVENT``` to measure a collection of related data. The Collector
should strive to capature the related data into a CBOR object stored in the
event data field.

#### Exemplar Modules

The target exemplar Collectes are as follows,
 - DmiCollector - Able to hash DMI platform information
 - TpmCollector - Parses TPM logs from UEFI, Intel TXT, and AMD Secure Launch
 - StorageCollector - Able to hash block devices, i.e. local and remote
 - FileCollector - Able to hash file paths, e.g. physical files, virtual
   filesystem nodes (procfs, sysfs), etc.

### Attestor

All Attestor's core logic will be found under ```pkg/attestation``` and will
implement a series of local and remote attestation protocols.

#### Main Library Interface

The Attestor main library should expose a GetAttestor method that accepts a
JSON config for an individual attestor. The GetAttestor call will consults an
array of generator functions for all known attestor modules. It will then pass
the config to each generator until the first one successfully returns an
instance of the Attestor interface. The definition for the call will look like
the following,

```go
GetAttestor(config []byte) (Attestor, error)
```

#### Attestor Interface

Each module will implement a generator function that returns a Attestor
instance if provided a valid JSON config. The generator function will be
responsible for parsing the JSON config and creating an instance of the
Attestor configured with the parsed configuration. The expected pattern for the
generator function will be as follows,

```go
NewUnsealAttestor(config []byte) (Attestor, error)
```

An attestor must implement the Attestor interface that exposes an Attest
function. The Core Engine will call the Attest function of the Attestor that
resulted from the policy file. The Attest function will return an AttestResult
container type that contains any data necessary for the Target Launcher to
Launch the target kernel. The Attestor interface is defined as follows,

```go
import "github.com/TrenchBoot/tpmtool/pkg/tpm"

ResultType uint32

const (
	KeyResult Resulttype = iota
)

type AttestResult struct {
	Type	ResultType
	Value	[]byte
}

type Attestor interface {
	Attest(t *TPM) (AttestResult, error)
}
```

#### Exemplar Modules

The target exemplar Attestors are as follows,
 - UnsealAttestor - An attestor that acheives attestation by being able to
   Unseal a TPM wrapped blob, typically an encrypted boot key.
 - KmipAttestor - An attestor that sends a KMIP Attestation Credential to a
   KMIP server to obtain an encrypted boot key

### Target Launcher

All Target Launchers will be standalone commands implemented under ```cmd```.
The launcher configuration from the policy file will provide mapping on how to
pass the AttestResult to the launcher command.

#### Exemplar Modules

The target exemplar Launchers are as follows,
 - LuksBoot - will accept a key passed via stdin along with block device path
   on the command line and will proceed to unlock the device to search for a
   boot configuration

### tpmtool Enhancement

A majority of TrenchBoot components will require interacting with the TPM and
recording those interactions as events in an TPM eventlog. The tpmtool from the
SystemBoot u-root distro project provides a fairly complete TPM interaction
package. TrenchBoot will enhance tpmtool to meet its needs, in particular for
Late Launch.

#### TPM State

For TrenchBoot to support advanced features such as forward sealing and fine
grained attestation, it is necessary to have a reconstructable representation
of the TPM's current PCR state. Specifically it must be able possible to
conduct actions such as replacing measurements at random places within the
event log or identify what measurement(s) in the history do not match the
expected history. These two actions are the primary motivating actions but
should not be thought to be the exclusive set of actions that need to be
supported.

#### Intel TXT Log Format

The tpmtool has a fairly extensive support for known Static Launch event log
formats. This needs to be extended to suppor the additional event log formats
used by Intel TXT.
