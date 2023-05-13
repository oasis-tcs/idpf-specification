# IDPF (Infrastructure Data-Plane Function)

|Version|Date|Comments|
| ----- | -- | ------ |
|0.9|2/2/2023|1st Draft for Oasis IDPF TC|
|0.91|2/7/2023|Added OEM descriptor format range to mailbox descriptors.|
|0.91|2/14/2023|Added place holders for multiple Default vport capability and Container Dedicated Queues.|
|0.91|2/14/2023|vport Attributes flow to be described|
|0.91|2/28/2023|Updated virtchnl2 descriptor structure. <br /> Updated TX descriptor IDs in virtchnl2_lan_desc.h|
|0.91|4/19/2023|Name updated to match the Oasis TC|

# Introduction

This document describes the host Interface, device behavior, setup, and configuration flows of a PCIE network function device. This device can be presented to an OS as a PCIe Physical function (PF) device or a virtual function (VF) device from a NIC or as a completely emulated device in Host SW. An IDPF device is OS and Vendor Agnostic. 

# Context and Motivation

There is a need for standardized Network function devices that can support high throughput over Ethernet and RDMA. The Goal is to define a multi-vendor interface that takes away most of the CPU cost of an emulated Interface and is inherently extensible and scalable. A useful device must:

* Support easy maintenance to bring down Datacenter operating cost using standardized interfaces.
* Support zero downtime for Tenants while Datacenter operator brings down servers for maintenance
* Support getting more value out of a server by reducing CPU overhead costs on the server 
* Support providing very high  throughput and multiple offload features to the Tenant to keep up with higher demands 
* Provide Flexibility and extensibility to support multi-vendor and per-need negotiable feature sets 
* Be O/S agnostic 
* Be usable in both Virtualized and Bare-metal scenarios

None of the standard interfaces available in the market provide for all these needs. This Spec is an effort towards achieving all the above goals.

# Key Concepts

## IDPF Network Function Device (“Device”)

This is an abstraction of a physical or emulated Network PCIE Device as seen by the driver. It is modeled as having internally a **data path** component that carries out Packet handling and a **control plane** component that is used to configure and monitor the operations of the data-path.  An IDPF Device has the same interface whether used as a Physical Function (PF) PCIe device, a Virtual Function (VF) device (when virtualization is used) , or a composed PCIe entity. It is also Operating-system agnostic, and does not impose assumptions on the implementation of conformant Devices.

![Figure 1 : Network Function Device Overview](Diagrams/network_function_device_overview.svg)

## Abstraction

Although this Interface is designed to be high  performance, there is a big effort to abstract the resources from their actual device representations. This is to facilitate Live Migration.

Abstracted SW Elements:
* **Adapter or a Device**: This is an abstraction of a physical or emulated Network PCIE Device as seen by the driver. 
* **vSwitch**: Typically the Device will support a vSwitch implementation in order to classify traffic to multiple Network end-points exposed by the Device. This Spec does not concern itself with the programmability or the features and capabilities of the vSwitch but depending on the underlying vSwitch the Device capabilities will vary and will be negotiated as such.
* **vPort**: A vPort is used by the Device to represent a L2 network end-point. This endpoint is a source/destination for network traffic, and can be associated with a Host, VM, a Container, an application on a host, etc.  A single application or VM can have 1 or many logical network endpoints (e.g. a VM can have several virtual NICs), and hence can have multiple vPorts associated with it. Internally it is common for a network Device to implement vPorts as sets of DMA targets with a queue hierarchy to determine priority of service of these DMA targets. It is also common for network devices to support a load-balancing mechanism to distribute traffic among these queues.

  For example, in a NIC Device, vPort can be a group of DMA (Direct Memory Access) queues or just one (that deposits the packet into Host or VM memory). When a vPort is a group of DMA queues, there is a host side load balancer or classifier implemented in the Device/SW Data path past the vSwitch to pick one of the queues of the vPort for final delivery of the packet.
  
  Note: There is no replication of packets within a vPort.
* **Buffers, Descriptors, and descriptor queues**: The main communication mechanism between the Device and host-side software (including the driver) is by placing relevant data (received or to be transmitted) and requests/responses in Memory buffers and creating data-structures called Descriptors that point to these buffers (and hold some metadata describing various attributes of the data in the buffers). These Descriptors are placed into Circular Ring-queues (See Figure 1 ) called Descriptor Queues.

  These Descriptor queues are often used in pairs, with one queue used in the Host-to-Device direction, and the other in Device-To-Host direction.
* **Descriptor Queue**: A descriptor Queue is a Memory region whose length is some multiple of the size of the relevant Descriptors it is meant to hold. (As noted above, Descriptors are data-structures containing pointers to Data-buffers + additional flags and meta-data used in processing that data).

  A Descriptor queue is an in-memory ring-queue shared between the SW and the Device’s HW. It starts at a memory location pointed at by the relevant BASE pointer, and uses two additional pointers, called HEAD and TAIL to coordinate usage (see Figure 2).<br/>HW “owns” the Queue entries from HEAD to TAIL-1. It will next process the descriptor pointed to by “head” and will process items from the queue going forward until it processes the item at location TAIL-1; When HW is done with the 1st descriptor in the range it “owns”, it advances HEAD thus moving that Descriptor into the “for SW processing” part of the Queue.
  
  SW “owns” the descriptors outside the range the HW owns. When SW adds a Descriptor to the Queue it will place it where TAIL points, and then advance TAIL to the next location, thus making the just-added descriptor part of the range available for HW processing. When TAIL reaches the maximum length of the queue it wraps around. 

![Figure 2: Descriptor Queue Overview](Diagrams/descriptor_queue_overview.svg)

* **Mailbox**:This is an Out-of-band communication mechanism that may be implemented in HW or SW. A typical implementation uses a specific agreed-upon pair of Descriptor queues to pass back and forth Commands from the driver to the Control-plane of the Device and responses and event notifications from the Device control-plane to the driver (see drawing 1). This control/monitoring side-band is called a Mailbox.
The Mailbox is used for:

  * To learn about the Device capabilities.
  * To configure, enable or disable device capabilities
  * To receive Events from the Device

  Typically, this out-of-band communication is assumed to be not very chatty and can be implemented as a low throughput mechanism. 

* **virtchannel**: The device driver communicates with the Control Plane Agent over a mailbox in order to  request configuration and learn about capabilities etc. Virtchannel is a negotiation protocol with the Control plane and also for both sides to send asynchronous events as mentioned in the Mailbox description.

  Virtchannel as it is a sideband communication protocol with CP, the driver negotiates a protocol version first with CP and then follows the rules of the protocol for learning about capabilities and setting up Configs and offloads etc.
Each virtchannel message and the capabilities for which they are used and the flow with CP is described in detail as part of the Negotiation, Mailbox flows and offload Capabilities Sections.

  A driver using a feature without negotiation will be marked as a malicious activity by the driver on the CP side and it can result in the driver going through reset and failing to load.
* **Control Plane Agent (CP)**:This is a SW/FW/HW entity that is part of the device which communicates with the SW driver over the Mailbox using virtchannel in order for the SW Driver to learn about the capabilities, configure the device etc.
* **Tx & RX Process, and Completion Descriptors**: The general pattern of communication between host-side Software and the device is that Software posts request Descriptors in the Host-to-Device Descriptor queue, and the Device posts responses and events in the associated Device-to-Host Descriptor queue. In particular for the reception (RX) and transmission (TX) of packets the process is as follows (note this is the general case, some nuances and optimizations are described in the relevant section)

  1. **RX Process Overview**: Software prepares empty buffers in Host Memory (perhaps in VM’s Memory space if a VM is involved) to hold the data and headers of the frames when they are received.

      Software Builds “RX Request” descriptors that point to these buffers and places them on a Host-to-device Descriptor queue called “RX Queue”.
    
      When a packet is received by the Device’s data path, it reads the next Descriptor from the RX queue, and using DMA places the data received into the buffers pointed to by the Descriptor.
    
      The device then indicates to the SW that data has been received and placed in host memory that the SW should process. It does that by writing a “RX Completion Descriptor” that points to the data received, and holds additional meta-data describing relevant attributes for the data received (e.g. that its CRC was verified by the device, and the SW does not need to repeat this check).
    
      This completion Descriptor is typically placed in the Device-to-Host descriptor queue paired with the RX descriptor used by the relevant vPort, this queue is called the “RX Completion Queue". 

  2. **TX Process Overview**: Software prepares buffers in host memory containing the data to be transmitted.

      Software builds “TX request Descriptors” that point to these buffers, and places them on a Host-to-Device descriptor queue called “TX Descriptor queue”.
    
      The device’s Data-path reads the next TX request Descriptor from this queue and uses its content to read the data to be transmitted from the buffers, and processes them into one or more outgoing packets sent to the destination.
    
      The device then indicates to the SW that the data has been transmitted by building “TX Completion Descriptors” and placing them in a “TX Completion queue” which is the Device-to-Host  part of the queue-pair used for this vPort.

## Modularization

When configuring device resources, care was taken that data structures group the resources into categories such as

* DMA Resources: Example: queues
* PCIE Resources: Example: Interrupts, Virtual functions
* Network resource: Example vPort 

## Composition

The IDPF interface does not make any assumption about how the device is composed, whether it is a complete passthrough Device like an SRIOV interface or a PF interface or a composed device such as an SIOV device. Some parts of the device memory space can be passthrough and parts of the device memory space can be emulated in the backend SW or a completely emulated device.

## Live Migration

The Host interface Specification is designed to support Live migration assisted by the device, although there is no assumption that the driver needs to do anything special for Live Migration of the VM; the design concepts such as abstraction of device resources etc.  keep live migratability in mind.

## Adaptability and Elasticity

IDPF allows for adding or removing any of the offloads and capabilities based on what the device can support or not. Also, the Capability Negotiation Protocol allows for extending capabilities in the future.

IDPF allows for supporting more than one vPort through the same PCIE device interface to provide dedicated resources and control for a Software network interface that can be tied to individual applications, namespaces, containers etc.

## Negotiation

All device capabilities and offloads are negotiable; this allows for running the driver on a huge set of physical, virtual, or emulated devices.

## Profiling

Since the capabilities are negotiated with the device, this allows the Device Control Plane to enforce SLAs by creating templates of capabilities and offloads to expose to the Interface.

# PCIE Host Interface

## Device identification 

The IDPF will use the following device IDs (Vendor ID = 0x8086), Device ID = 0x1452 as a PF device and 0x145c as a VF Device). The RevID  can be used by OEMs to specify different revisions of the Device. The subvendor and subsystem IDs are TBD.

## BARs

IDPF functions shall expose two BARs:

* BAR0 (32 bits) or BAR0-1 (64 bits) that includes the registers described in the Registers section. This BAR size depends on the number of resources exposed by the function

* BAR2 (32 bits) or BAR2-3 (64 bits) that includes the MSI-X vectors. The location of the vectors and the PBA bits shall be pointed by the standard MSI-X capability registers. The IMS vectors may be mapped in the BAR also.

## PCIe capabilities

The following capabilities shall be part of an IDPF function configuration space:

* Power Management
* MSI-X. MSI support is optional.
* PCI Express

The following capabilities shall be part of an IDPF function that exposes SR-IOV functions:

* ARI
* SR-IOV

The following capabilities shall be part of an IDPF function that  exposes ADIs through SIOV:

* SIOV DVSEC

Other capabilities like ATS, VPD, serial number and others are optional and may be exposed based on the specific hardware capabilities. 

## PCIE MMIO (Memory Mapped IO) map  

### Registers

Fixed: Mailbox registers. See register Register Section

Flexible: Queue and Interrupt registers. See Register section.

Note : When the driver writes a device register and then reads that same register the device might return the old register value (value before the register write) unless the driver keeps 2 ms latency between the write access and the read accesses.
 
# Versioning (Device, Driver, virtchannel version)

## Device versioning

As mentioned in the Device identification, a Device ID and Subdev ID defines the Device version. A base Device ID for PF and VF Device is defined and a subdev ID of zero as a default for the device. The Device ID and subdev ID can be used to have separate register offsets for the fixed portions of register space such as the mailbox. 

## Virtchannel: Mailbox protocol versioning

The Driver shall start the Mailbox communication with the Control plane by first negotiating the mailbox protocol version that both can support. The information is passed in the mailbox buffer pointed to by the mailbox command descriptor.

VF/PF posts its highest supported version number to the CP (Control Plane Agent). CP responds with its virtchannel version number in the same format, along with a return code. Reply from CP has CP’s virtchannel major/minor versions.

If there is a major version mismatch, then the VF/PF goes to the lowest version of virtchannel as the CP is assumed to support at least version 2.0 of virtchannel. If there is a minor version mismatch, then the PF/VF can operate but should add a warning to the system log.
```C
    struct virtchnl_version_info {
	    u32 major;
	    u32 minor;
    };
```
## Driver versioning

The spec does not define a driver version, and this is maintained strictly to identify the revisions of driver codes and features.

# Compatibility Requirements

For multiple generations of devices to support this interface, there is a small base set of interface definitions that needs to be fixed.

# Virtchannel and Mailbox Queue

Virtchannel is a SW protocol defined on top of a Device supported mailbox in case of a passthrough sideband going directly to Device. Device as mentioned previously can be a HW or a SW Device.

## Virtchannel Descriptor Structure
```C
    struct virtchnl2_descriptor {
    __le16 flags;
    __le16 opcode;
    __le16 datalen;
    union {
	    __le16 retval;
	    __le16 pfid_vfid;
    };
    __le32 v_opcode:28;
    __le32 v_dtype:4
    __le32 v_retval;
    __le32 reserved;
    __le16 sw_cookie;
    __le16 v_flags;
    __le32 addr_high;
    __le32 addr_low;
    };
```
## Mailbox queues (Device Backed or SW emulated)

The driver holds a transmit and receive mailbox queue pair for communication with the CP.

For both RX and TX queues the interface is like the “In order, single queue model” (see RX In order single queue model and TX In order single queue model) LAN (Local Area Network) queue with the following exceptions:

* TX mailbox:
  * The Maximum packet size is 4KB.
  * Packet is pointed by a single 32B descriptor. Packet data resides in a single buffer.
  * Packet can be a zero-byte packet (in that case, the descriptor does not point to a data buffer).
  * Device reports completion (descriptor write back) for every mailbox packet/descriptor, which will be received into the RX Mailbox queue.
* RX mailbox:
  * SW fills the RX queue with 32B descriptors that point to 4KB buffers.
  * The Maximum packet size is 4KB. RX packet data resides in a single buffer.
  * Packet can be a zero-byte packet (in that case, data buffer remains empty).
  * RX mailbox is set by device as fast drop in absence of sufficient RX descriptors/buffers posted by the driver, in order to avoid DDoS and blocking of the pipe.

The mailbox queues CSRs (Control Status Registers) are in fixed addresses and are described in Appendix Section [- PF/VF- Mailbox Queues]().

## Mailbox descriptor formats

Both queues use the same descriptor structure for command posting (TX descriptor format) and command completion (RX descriptor format). 
All descriptors and commands are defined using little endian notation with 32-bit words. Drivers using other conventions should take care to do the proper conversions.

Mailbox commands are
* Indirect – commands, which in addition to Descriptor info, can carry an additional payload buffer of up to 4KB (pointed by “Data address high/low”).

### Generic Descriptor fields description

| Name | Bytes.Bits | Description | 
| ---- | ---------- | ----------- |
| Flags.DD | 0.0 | This flag indicates that  the current descriptor is done and processed by hardware or IDPF.|
| Flags.CMP| 0.1 | This flag indicates the current descriptor is Completed and processed by hardware or IDPF. This is part of the legacy format and needs to be set together with Flag.DD in cases defined below.|
| Flags.Reserved | 0.2-1.1 | Reserved for hardware uses.|
| Flags.RD | 1.2 |This flag can be coupled with Flag.BUF from below to indicate Mailbox TX Hardware that IDPF provides data in an additional buffer (pointed by Data Address low/high below) to be sent to CP in Mailbox command.<br />Not relevant for the RX queue. |
| Flags.VFC | 1.3 |This flag indicates that command is sent by IDPF running on VF.|
| Flags.BUF | 1.4 |This flag indicates that a buffer exists for this descriptor, used for RX descriptor advertisement. For TX, it needs to be set together with Flag.RD.|
| Flags.Reserved | 1.5 – 1.7| Reserved for hardware uses.|
| Message Infrastructure Type | 2-3| TX Side:<br />0x0801 – “Send to CP” opcode.<br />The driver may use this command to send a mailbox message to its CP.<br />Other opcodes will be rejected and not sent to the destination.<br /><br />RX side: 0x0804– “Send to Peer IDPF Driver” opcode.<br />The driver will receive this opcode inside descriptors coming from CP.|
| Message Data Length | 4-5 | Attached buffer length in bytes (this field is set to 0 when buffer is not attached). Up to 4KB. Used only when Flag.BUFF and optionally Flag.RD(TX case) are set.|
| Hardware Return Value | 6-7 | Hardware Return value, which indicates that Descriptor was processed by Hardware successfully. Relevant when Flag.DD and CMP are set for descriptor.|
| VirtChnl Message Opcode | 8-11.3 | Includes VirtChnl opcode value,. i.e. VIRTCHNL2_OP_GET_CAPS = 500.<br />Opcodes 0x0000 - 0x7FFF are in use.<br />Codes 0x8000-0xFFFF - are reserved.<br />Note: The upper 4 bits in byte 11 are reserved for the VirtChnl descriptor Format.
| VirtChnl Descriptor Type | 11.4-11.7 | v_dtype, default value is 0 but can be used in future to enhance the VirtChnl descriptor format.<br />Values 1-7 are reserved for future expansion. Values 8-15 are reserved for OEM descriptor formats.|
| VirtChnl Message Return Value | 12-15 | VirtChnl Command return value sent by CP will be received here. Not relevant for TX.|
| Message Parameter 0 | 16-19 | Can be used to include message parameter|
| SW Cookie | 20-21 | Used for cookies to be delivered to the receiver.|
| VirtChnl Flags | 22-23 | When a Flags capability is negotiated, it can be used to give more direction to CP whether a response is required etc.|
| Data Address high / Message Parameter 2 | 24-27 | When RD and BUF are set, describe the attached buffer address.<br />Else, used as a third general use as additional parameter.|
| Data Address low / Message Parameter 3 | 28-31 | When RD and BUF are set, describe the attached buffer address.<br />Else, used as a fourth general use additional parameter.|

### TX descriptor Command Submit format

The table below describes the descriptor structure fields, which are expected to be filled by the driver before incrementing Mailbox TX Queue Tail.
All fields inside the TX descriptor, which are not provided by the sender, but are expected to be filled by the responder (output fields) should be set to 0 by the sender.

| Name | Bytes.Bits | Description | 
| ---- | ---------- | ----------- |
| Flags.DD | 0.0 | Should be set to 0.|
| Flags.CMP| 0.1 | Should be set to 0.|
| Flags.Reserved | 0.2-1.1 |Should be set to 0.|
| Flags.RD | 1.2 | Set by the IDPF when a buffer is attached to this Command. |
| Flags.VFC | 1.3 |Should be set to 0.|
| Flags.BUF | 1.4 |Set by the IDPF when a buffer is attached to this Command.|
| Flags.Reserved | 1.5 – 1.7| Should be set to 0.|
| Message Infrastructure Type | 2-3| 0x0801 – “Send to CP” opcode.<br />The driver uses this command opcode only to send a mailbox message to its CP.<br />Other opcodes will be rejected and not sent to the destination.|
| Message Data Length | 4-5 | Attached buffer length in bytes (this field is set to 0 when buffer is not attached). Up to 4KB.|
| Hardware Return Value | 6-7 |Should be set to 0. |
| VirtChnl Message Opcode | 8-11.3 | Includes VirtChnl opcode value,. i.e. VIRTCHNL2_OP_GET_CAPS = 500.<br />Opcodes 0x0000 - 0x7FFF are in use.<br />Codes 0x8000-0xFFFF - are reserved.<br />Note: The upper 4 bits in byte 11 are reserved for the VirtChnl descriptor Format.|
| VirtChnl Descriptor Type | 11.4-11.7 | v_dtype, default value is 0 but can be used in future to enhance the VirtChnl descriptor format.<br />Values 1-7 are reserved for future expansion. Values 8-15 are reserved for OEM descriptor formats.|
| VirtChnl Message Return Value | 12-15 | Should be set to 0.|
| Message Parameter 0 | 16-19 | Can be used to include message parameter|
| SW Cookie | 20-21 | Used for cookies to be delivered to the receiver.|
| VirtChnl Flags | 22-23 | When a Flags capability is negotiated, it can be used to give more direction to CP whether a response is required etc.|
| Data Address high / Message Parameter 2 | 24-27 | When RD and BUF are set, describe the attached buffer address.<br />Else, used as a third general use parameter. Copied to the RX packet descriptor of the receiver.|
| Data Address low / Message Parameter 3 | 28-31 | When RD and BUF are set, describe the attached buffer address.<br />Else, used as a fourth general use parameter Copied to the RX packet descriptor of the receiver.|

### TX descriptor command Completed format

This is the TX descriptor view, after it was processed and sent by the Device, while Flag.DD = 1 indicated this. This is an indication for the sender driver that the current TX descriptor and its buffer are completed by hardware and can be reused for a new command. Command’s response will be then received asynchronously into the RX Mailbox queue.

| Name | Bytes.Bits | Description | 
| ---- | ---------- | ----------- |
| Flags.DD | 0.0 | Descriptor done.<br />Should be set to 1 by Device.|
| Flags.CMP| 0.1 | Complete.<br />Should be set to 1 by Device.|
| Flags.Reserved | 0.2-1.1 | Should be ignored by Driver.|
| Flags.RD | 1.2 | Should be ignored by Driver.|
| Flags.VFC | 1.3 |Should be ignored by Driver.|
| Flags.BUF | 1.4 |Should be ignored by Driver.|
| Flags.Reserved | 1.5 – 1.7| Should be ignored by Driver.|
| Message Infrastructure Type | 2-3 | Preserve the field value in the descriptor.|
| Message Data Length | 4-5 | Preserve the field value in the descriptor.|
| Hardware Return Value | 6-7 | Expected to be set to 0 (Success) by Device.|
| VirtChnl Message Opcode | 8-11.3 | Includes VirtChnl opcode value,. i.e. VIRTCHNL2_OP_GET_CAPS = 500.<br />Opcodes 0x0000 - 0x7FFF are in use.<br />Codes 0x8000-0xFFFF - are reserved.<br />Note: The upper 4 bits in byte 11 are reserved for the VirtChnl descriptor Format.|
| VirtChnl Descriptor Type | 11.4-11.7 | v_dtype, default value is 0 but can be used in future to enhance the VirtChnl descriptor format.<br />Values 1-7 are reserved for future expansion. Values 8-15 are reserved for OEM descriptor formats.|
| VirtChnl Message Return Value | 12-15 | Should be set to 0.|
| Message Parameter 0 | 16-19 | Can be used to include message parameter|
| SW Cookie | 20-21 | Used for cookies to be delivered to the receiver.|
| VirtChnl Flags | 22-23 | When a Flags capability is negotiated, it can be used to give more direction to CP whether a response is required etc.|
| Data Address high / Message Parameter 2 | 24-27 | Preserve the field value in the descriptor.|
| Data Address low / Message Parameter 3 | 28-31 | Preserve the field value in the descriptor.|

### RX descriptor command Submit format

The driver must advertise empty RX descriptors into Mailbox RX queues,potentially including free 4KB buffers. During free descriptor preparation, the driver will clear any unused fields (including unused Flags) and set data pointers and data length to a mapped DMA pointer, see table below.

| Name | Bytes.Bits | Description | 
| ---- | ---------- | ----------- |
| Flags.DD | 0.0 | Should be set to 0.|
| Flags.CMP| 0.1 | Should be set to 0.|
| Flags.Reserved | 0.2-1.1 | Should be set to 0.|
| Flags.RD | 1.2 | Should be set to 0.|
| Flags.VFC | 1.3 |Should be set to 0.|
| Flags.BUF | 1.4 |Receive queue elements should always have an additional buffer. Should be set to 1.|
| Flags.Reserved | 1.5 – 1.7| Should be set to 0.|
| Message Infrastructure Type | 2-3 | Should be set to 0.|
| Message Data Length | 4-5 | Attached buffer length in bytes (this field is set to 4KB to advertise maximum buffer length)|
| Hardware Return Value | 6-7 | Should be set to 0.|
| VirtChnl Message Opcode | 8-11.3 | Should be set to 0.<br />Note: The upper 4 bits in byte 11 are reserved for the VirtChnl descriptor Format.|
| VirtChnl Descriptor Type | 11.4-11.7 |v_dtype, default value is 0 but can be used in future to enhance the VirtChnl descriptor format.<br />Values 1-7 are reserved for future expansion. Values 8-15 are reserved for OEM descriptor formats.|
| VirtChnl Message Return Value | 12-15 | Should be set to 0.|
| Message Parameter 0 | 16-19 | Should be set to 0.|
| SW Cookie | 20-21 | Used for cookies to be delivered to the receiver.|
| VirtChnl Flags | 22-23 | 
When a Flags capability is negotiated, it can be used to give more direction to CP whether a response is required etc.|
| Data Address high / Message Parameter 2 | 24-27 | As BUF is always set - 4KB buffer address to be pointed.|
| Data Address low / Message Parameter 3 | 28-31 | As BUF is always set - 4KB buffer address to be pointed.|

### RX descriptor command Completed format

When a mailbox command arrives from the device to Driver RX Mailbox Queue, it will use previously advertised RX descriptors and additional 4KB buffers  and fill it with mailbox command data. This structure defines what fields will be seen by the driver.

| Name | Bytes.Bits | Description | 
| ---- | ---------- | ----------- |
| Flags.DD | 0.0 | Descriptor done.<br />Should be set to 1 by the device.|
| Flags.CMP| 0.1 | Complete.<br />Should be set to 1 by the device.|
| Flags.Reserved | 0.2-1.1 | Should be ignored by Driver.|
| Flags.RD | 1.2 | Should be ignored by Driver.|
| Flags.VFC | 1.3 |Should be ignored by Driver.|
| Flags.BUF | 1.4 |0 - no buffer data is attached to the incoming command.<br />1 - buffer data is attached to the incoming command.|
| Flags.Reserved | 1.5 – 1.7| Should be ignored by Driver.|
| Opcode | 2-3 | Should be ignored by Driver.|
| Message Data Length | 4-5 | Attached buffer length in bytes (this field is set to 0 when buffer is not attached). Up to 4KB|
| Hardware Return Value | 6-7 | Should be ignored by Driver.|
| VirtChnl Message Opcode | 8-11.3 | Includes Internal virtChnl opcode value, all 32 bits. i.e. VIRTCHNL_OP_GET_CAPS = 100.<br />Opcodes 0x0000 - 0x7FFF are in use.<br />Codes 0x8000-0xFFFF - are reserved.<br />Note: The upper 4 bits in byte 11 are reserved for the VirtChnl descriptor Format.|
| VirtChnl Descriptor Type | 11.4-11.7 | v_dtype, default value is 0 but can be used in future to enhance the VirtChnl descriptor format.<br />Values 1-7 are reserved for future expansion.<br />Values 8-15 are reserved for OEM descriptor formats.|
| VirtChnl Message Return Value | 12-15 | Should be set to 0.|
| Message Parameter 0 | 16-19 | Can be used to include message parameter|
| SW Cookie | 20-21 | Used for cookies to be delivered to the receiver.|
| VirtChnl Flags | 22-23 | When a Flags capability is negotiated, it can be used to give more direction to CP whether a response is required etc.|
| Data Address high / Message Parameter 2 | 24-27 | When BUF is set, it specifies the attached buffer address (the one was advertised by the driver previously).<br />Else, used as a third general use parameter copied from the TX packet descriptor.|
| Data Address low / Message Parameter 3 | 28-31 | When BUF is set, it describes the attached buffer address (the one was advertised by the driver previously).<br />Else, used as a fourth general use parameter copied from the TX packet descriptor.|

## Mailbox Flows

### Mailbox Initialization flow

Most of the Mailbox queues initialization is done by default by CP, while the IDPF driver needs to provide rings’ description and enable the queues using MMIO registers access. Here are the configuration to be performed by driver:

1. The driver clears the head and tail register for its TX/RX queues - PF/VF_MBX_ATQT, PF/VF_MBX_ARQT, PF/VF_MBX_ATQH, PF/VF_MBX_ARQH.
2. The driver programs the base address PF/VF_MBX_ATQBAL/H and PF/VF_MBX_ARQBAL/H (must be 128B aligned)
3. The driver set the PF/VF_MBX_ARQLEN/PF/VF_MBX_ATQLEN.Length and “Enable” field to 1 to signal that its mailbox is now enabled.

Driver must follow this exact sequence of operations in order to initialize the mailbox queue context.
After this Driver writes the first message to TX mailbox and waits for 20 msec for response, if no response it tries at least 10 times with a gap of 20 msec before it fails to load the driver.

### Mailbox Queues Disable

The mailbox queues are disabled by the CP ONLY while the driver cannot directly disable its own mailbox queues. 

Any queue disable request (clearing of the PF/VF_MBX_ARQLEN/PF/VF_MBX_ATQLEN.Enable flag ) from the driver can be ignored by Device. 

### Mailbox Queues Operation

As stated above Mailbox queues in both directions operate as “In order Single Queue Model”.

On the TX mailbox queue the IDPF driver moves the Tail pointer of the ring to notify device to transmit new mailbox packet while device informs the driver that packet was transmitted by overriding the descriptor with DD bit set.

On the RX queue IDPF driver moves the Tail pointer of the ring to notify the device that new buffers are available to use in the RX queue while the device notifies the driver that the packet was received into the queue by overriding the descriptor with DD bit set.

Note that the driver must not use the device head pointer to understand the device progress.

There several Mailbox specific operations to be done by IDPF for Mailbox queues, here they are:

TX direction:

* IDPF reads the PF/VF_MBX_ATQLEN.Enable field to check if value is 1 before posting packets/buffers to the TX/RX queue. This in order to verify that the queue is enabled and commands can be posted. If the queue is found disabled, this means that the queue was disabled by CP and IDPF is expected to be under FLR and need to perform Function Level Reset flow as described in chapter below Function Level reset.
* IDPF fills buffer with Command data, IDPF fills descriptor structure with proper information.
* IDPF increases Tail (PF/VF_MBX_ATQT) by 1 to send commands to the device.
* The device transmits the TX packet and overrides the TX descriptor  of the transmitted packet in the descriptor queue.
* IDPF polls the TX descriptor queue for completed descriptors (or waits for interrupt to be launched).
* The descriptor that belongs to completed packets are ones with the DD flag set.

RX direction:

* IDPF adds descriptors to the RX queue pointing to empty buffers and then sends the Tail point to notify the device that buffers are available.
* The device receives the RX packet, writes it to the host memory and then overrides the RX descriptor of the received packet in the descriptor queue.
* IDPF polls the RX descriptor queue for completed descriptors (or waits for interrupt to be launched).
* The descriptor that belongs to completed packets are ones with the DD flag set.
IDPF reads the additional buffer (“Data Address low/high”), which arrives together with the mailbox command.
* After all command information is fetched from the ring, IDPF advertise new free descriptor to the RX ring, as described in “RX descriptor command submit format” above and move Tail (PF/VF_MBX_ATQT) by 1 to make it available for device.  

# Data Queues

TX and RX queue types are valid in single as well as split queue models. With Split Queue model, 2 additional types are introduced - TX_COMPLETION and RX_BUFFER. In split queue model, RX corresponds to the queue where Device posts completions.

## LAN RX queues

A receive descriptor queue is made of a list of descriptors that point to memory buffers in host memory for storing received packet data of a particular queue.

Each queue is a cyclic ring made of a sequence of receive descriptors in contiguous memory. 
A packet may be stored in a single receive descriptor or spread across multiple receive descriptors, depending on the size of the buffers associated with the receive descriptor and the size of the received packet.

This section describes the different RX queue modes supported by this specification, describes how SW posts new RX descriptors for Device, and describes how Device indicates to SW that it can reuse the buffers and descriptors for new packets.
The mode of operation is a per queue configuration. Each queue can be configured to work in a different model:

* In order single queue model. 
  This option is supported only when VIRTCHNL2_CAP_RXQ_MODEL_SPLIT feature is negotiated. 
* Out of order split queue model.
  This option is supported only when VIRTCHNL2_CAP_RXQ_MODEL_SINGLE feature is negotiated. 

The device must negotiate at least one of the features above.

The minimal and maximal buffers sizes that are supported by HW are negotiable HW capabilities.
The possible values and default values (in the absence of negotiation) of those capabilities are described in the RX packet/buffer sub-section under the HW capabilities section.

Note that the device is not obligated to check the maximal packet length of a packet delivered to driver.

### In order single queue model

In the single queue model, the same descriptor queue is used by SW to post descriptors to Device and used by Device to report completed descriptors to SW.

SW writes the packet descriptors to the queue and updates the QRX_TAIL register to point to the descriptor after the last descriptor written by SW (points to the first descriptor owned by SW).
Tail pointer should be updated in 8 descriptors granularity. 

After Device writes the packet to the data buffers (pointed by the fetched descriptors) it reports a completion of a received packet by overriding the packet descriptors in ring with the write format descriptor (as described in LAN Receive Descriptor Write Back Formats).

In this model, completions are reported “in order” - according to the packet placement order in the queue.

The diagram below illustrates the descriptors’ Device/SW ownership based on the Tail pointer:

* TAIL pointer represents the following descriptor after last descriptor written to the ring by SW.
* HEAD pointer represents the following descriptor after last descriptor written by Device.
* White descriptors (owned by Device) are waiting to be processed by Device or currently processed by Device.
* Blue descriptors (owned by SW) were already processed by Device and can be reused again by SW.
* Zero descriptor are owned by Device when TAIL == HEAD
* The SW should never set the TAIL to a value above the HEAD minus 1.

![Descriptor](Diagrams/descriptor.PNG)

![Device Receive Descriptor](Diagrams/device_receive_desc.png)




