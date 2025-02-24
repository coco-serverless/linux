.. SPDX-License-Identifier: GPL-2.0

===================================================================
The Definitive SEV Guest API Documentation
===================================================================

1. General description
======================

The SEV API is a set of ioctls that are used by the guest or hypervisor
to get or set a certain aspect of the SEV virtual machine. The ioctls belong
to the following classes:

 - Hypervisor ioctls: These query and set global attributes which affect the
   whole SEV firmware.  These ioctl are used by platform provisioning tools.

 - Guest ioctls: These query and set attributes of the SEV virtual machine.

2. API description
==================

This section describes ioctls that is used for querying the SEV guest report
from the SEV firmware. For each ioctl, the following information is provided
along with a description:

  Technology:
      which SEV technology provides this ioctl. SEV, SEV-ES, SEV-SNP or all.

  Type:
      hypervisor or guest. The ioctl can be used inside the guest or the
      hypervisor.

  Parameters:
      what parameters are accepted by the ioctl.

  Returns:
      the return value.  General error numbers (-ENOMEM, -EINVAL)
      are not detailed, but errors with specific meanings are.

The guest ioctl should be issued on a file descriptor of the /dev/sev-guest
device.  The ioctl accepts struct snp_user_guest_request. The input and
output structure is specified through the req_data and resp_data field
respectively. If the ioctl fails to execute due to a firmware error, then
the fw_error code will be set, otherwise fw_error will be set to -1.

The firmware checks that the message sequence counter is one greater than
the guests message sequence counter. If guest driver fails to increment message
counter (e.g. counter overflow), then -EIO will be returned.

::

        struct snp_guest_request_ioctl {
                /* Message version number */
                __u32 msg_version;

                /* Request and response structure address */
                __u64 req_data;
                __u64 resp_data;

                /* bits[63:32]: VMM error code, bits[31:0] firmware error code (see psp-sev.h) */
                union {
                        __u64 exitinfo2;
                        struct {
                                __u32 fw_error;
                                __u32 vmm_error;
                        };
                };
        };

The host ioctls are issued to a file descriptor of the /dev/sev device.
The ioctl accepts the command ID/input structure documented below.

::
        struct sev_issue_cmd {
                /* Command ID */
                __u32 cmd;

                /* Command request structure */
                __u64 data;

                /* Firmware error code on failure (see psp-sev.h) */
                __u32 error;
        };


2.1 SNP_GET_REPORT
------------------

:Technology: sev-snp
:Type: guest ioctl
:Parameters (in): struct snp_report_req
:Returns (out): struct snp_report_resp on success, -negative on error

The SNP_GET_REPORT ioctl can be used to query the attestation report from the
SEV-SNP firmware. The ioctl uses the SNP_GUEST_REQUEST (MSG_REPORT_REQ) command
provided by the SEV-SNP firmware to query the attestation report.

On success, the snp_report_resp.data will contains the report. The report
contain the format described in the SEV-SNP specification. See the SEV-SNP
specification for further details.

2.2 SNP_GET_DERIVED_KEY
-----------------------
:Technology: sev-snp
:Type: guest ioctl
:Parameters (in): struct snp_derived_key_req
:Returns (out): struct snp_derived_key_resp on success, -negative on error

The SNP_GET_DERIVED_KEY ioctl can be used to get a key derive from a root key.
The derived key can be used by the guest for any purpose, such as sealing keys
or communicating with external entities.

The ioctl uses the SNP_GUEST_REQUEST (MSG_KEY_REQ) command provided by the
SEV-SNP firmware to derive the key. See SEV-SNP specification for further details
on the various fields passed in the key derivation request.

On success, the snp_derived_key_resp.data contains the derived key value. See
the SEV-SNP specification for further details.


2.3 SNP_GET_EXT_REPORT
----------------------
:Technology: sev-snp
:Type: guest ioctl
:Parameters (in/out): struct snp_ext_report_req
:Returns (out): struct snp_report_resp on success, -negative on error

The SNP_GET_EXT_REPORT ioctl is similar to the SNP_GET_REPORT. The difference is
related to the additional certificate data that is returned with the report.

The ioctl uses the SNP_GUEST_REQUEST (MSG_REPORT_REQ) command provided by the SEV-SNP
firmware to get the attestation report.

On success, the snp_ext_report_resp.data will contain the attestation report
and snp_ext_report_req.certs_address will contain the certificate blob. If the
length of the blob is smaller than expected then snp_ext_report_req.certs_len will
be updated with the expected value.

See GHCB specification for further detail on how to parse the certificate blob.

2.4 SNP_PLATFORM_STATUS
-----------------------
:Technology: sev-snp
:Type: hypervisor ioctl cmd
:Parameters (out): struct sev_user_data_snp_status
:Returns (out): 0 on success, -negative on error

The SNP_PLATFORM_STATUS command is used to query the SNP platform status. The
status includes API major, minor version and more. See the SEV-SNP
specification for further details.

2.5 SNP_COMMIT
--------------
:Technology: sev-snp
:Type: hypervisor ioctl cmd
:Returns (out): 0 on success, -negative on error

SNP_COMMIT is used to commit the currently installed firmware using the
SEV-SNP firmware SNP_COMMIT command. This prevents roll-back to a previously
committed firmware version. This will also update the reported TCB to match
that of the currently installed firmware.

2.6 SNP_SET_CONFIG
------------------
:Technology: sev-snp
:Type: hypervisor ioctl cmd
:Parameters (in): struct sev_user_data_snp_config
:Returns (out): 0 on success, -negative on error

The SNP_SET_CONFIG is used to set the system-wide configuration such as
reported TCB version in the attestation report. The command is similar to
SNP_CONFIG command defined in the SEV-SNP spec. The current values of the
firmware parameters affected by this command can be queried via
SNP_PLATFORM_STATUS.

2.7 SNP_SET_CONFIG_START / SNP_SET_CONFIG_END
---------------------------------------------
:Technology: sev-snp
:Type: hypervisor ioctl cmd
:Parameters (out): struct sev_user_data_snp_config_transaction
:Returns (out): 0 on success, -negative on error

When requesting attestation reports, SNP guests have the option of issuing
an extended guest request which allows host userspace to supply additional
certificate data that can be used to validate the signature used to sign
the attestation report. This signature is generated using a key that is
derived from the reported TCB that can be set via the SNP_SET_CONFIG and
SNP_COMMIT ioctls, so the accompanying certificate data needs to be kept in
sync with the changes made to the reported TCB via these ioctls.

To allow for this, SNP_SET_CONFIG_START can be issued prior to performing
any updates to the reported TCB or certificate data that will be fetched
from userspace. Any attestation report requests via extended guest requests
that are in-progress, or received after SNP_SET_CONFIG_START is issued, will
result in the guest receiving a GHCB-defined error message instructing it to
retry the request. Once the updates are completed on the host,
SNP_SET_CONFIG_END must be issued to resume normal servicing of extended
guest requests.

In general, hosts should avoid having more than 1 outstanding
SNP_SET_CONFIG_{START,END} transaction in flight at any point in time, but
each ioctl will return a transaction ID in the response so the caller can
monitor whether the start/end ID both match. If they don't, the caller
should assume the transaction has been invalidated and retry the full update
sequence.

3. SEV-SNP CPUID Enforcement
============================

SEV-SNP guests can access a special page that contains a table of CPUID values
that have been validated by the PSP as part of the SNP_LAUNCH_UPDATE firmware
command. It provides the following assurances regarding the validity of CPUID
values:

 - Its address is obtained via bootloader/firmware (via CC blob), and those
   binaries will be measured as part of the SEV-SNP attestation report.
 - Its initial state will be encrypted/pvalidated, so attempts to modify
   it during run-time will result in garbage being written, or #VC exceptions
   being generated due to changes in validation state if the hypervisor tries
   to swap the backing page.
 - Attempts to bypass PSP checks by the hypervisor by using a normal page, or
   a non-CPUID encrypted page will change the measurement provided by the
   SEV-SNP attestation report.
 - The CPUID page contents are *not* measured, but attempts to modify the
   expected contents of a CPUID page as part of guest initialization will be
   gated by the PSP CPUID enforcement policy checks performed on the page
   during SNP_LAUNCH_UPDATE, and noticeable later if the guest owner
   implements their own checks of the CPUID values.

It is important to note that this last assurance is only useful if the kernel
has taken care to make use of the SEV-SNP CPUID throughout all stages of boot.
Otherwise, guest owner attestation provides no assurance that the kernel wasn't
fed incorrect values at some point during boot.


Reference
---------

SEV-SNP and GHCB specification: developer.amd.com/sev

The driver is based on SEV-SNP firmware spec 0.9 and GHCB spec version 2.0.
