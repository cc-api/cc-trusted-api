[![Python Code Scan](https://github.com/cc-api/cc-trusted-api/actions/workflows/pylint.yaml/badge.svg)](https://github.com/cc-api/cc-trusted-api/actions/workflows/pylint.yaml)
[![Document Scan](https://github.com/cc-api/cc-trusted-api/actions/workflows/doclint.yaml/badge.svg)](https://github.com/cc-api/cc-trusted-api/actions/workflows/doclint.yaml)
[![Python License Check](https://github.com/cc-api/cc-trusted-api/actions/workflows/pylicense.yaml/badge.svg)](https://github.com/cc-api/cc-trusted-api/actions/workflows/pylicense.yaml)
[![VMSDK Python Test](https://github.com/cc-api/cc-trusted-api/actions/workflows/vmsdk-test-python.yaml/badge.svg)](https://github.com/cc-api/cc-trusted-api/actions/workflows/vmsdk-test-python.yaml)

# CC Trusted API

CC Trusted API helps the diverse applications to access and process the trust states
which was represented by integrity measurement, event record, report/quote in the confidential
computing environment.

![](docs/cc-trusted-api-overview.png)

## 1. TCB Measurement

The diverse application in confidential computing could be firmware or monolithic application
in Confidential VM(CVM), micro service or macro service on Kubernetes. Although
different type application might get the trust states measured in different Trusted
Computing Base (TCB), the definition and structure of integrity measurement register and
event log follows the below specifications.

![](docs/cc-trusted-api-usage.png)
| TCB | Measured By | Specification |
| --- | -------- | ------------- |
| Initial TEE | Trusted Security Manager (TSM), such as Intel TDX module, SEV secure processor | Vendor Specification such as [Intel TDX Module 1.5 ABI Specification](https://cdrdv2.intel.com/v1/dl/getContent/733579) |
| Firmware | EFI_CC_MEASUREMENT_PROTOCOL </br> CCEL ACPI Table </br> EFI_TCG2_PROTOCOL </br> TCG ACPI Table | [UEFI Specification 2.10](https://uefi.org/specs/UEFI/2.10/38_Confidential_Computing.html#virtual-platform-cc-event-log) </br> [ACPI Specification 6.5](https://uefi.org/specs/ACPI/6.5/05_ACPI_Software_Programming_Model.html#cc-event-log-acpi-table) </br> [TCG EFI Protocol Specification](https://trustedcomputinggroup.org/resource/tcg-efi-protocol-specification/) </br> [TCG ACPI Specification](https://trustedcomputinggroup.org/resource/tcg-acpi-specification/) |
| Boot Loader | EFI_CC_MEASUREMENT_PROTOCOL </br> EFI_TCG2_PROTOCOL | Grub2/Shim |
| OS | Integrity Measurement Architecture (IMA) | [Specification](https://sourceforge.net/p/linux-ima/wiki/Home/) |
| Cloud Native | Confidential Cloud Native Primitives (CCNP) | [Repository](https://github.com/intel/confidential-cloud-native-primitives) |

## 2. Trusted Foundation

Normally Trusted Platform Module(TPM) provides root of trust for PC client platform.
In confidential computing environment, vTPM (virtual TPM) might be provided different
vendor or CSP, which root of trust should be hardened by vendor secure module. Some
vendor also provided simplified solution:

|           | Measurement Register | Event Log      | Specification |
| --------- | -------------------- | ---------      | ------------- |
| vTPM      | TPM PCR              | TCG2 Event Log | [TPM2 Specification](https://trustedcomputinggroup.org/resource/tpm-library-specification/) </br> [TCG PC Client Platform TPM Profile Specification](https://trustedcomputinggroup.org/resource/pc-client-platform-tpm-profile-ptp-specification/) </br> [TCG PC Client Platform Firmware Profile Specification](https://trustedcomputinggroup.org/resource/pc-client-specific-platform-firmware-profile-specification/) |
| Intel TDX | TDX MRTD/RTMR        | CC Event Log   | [Intel® TDX Module 1.5 Base Architecture Specification](https://cdrdv2.intel.com/v1/dl/getContent/733575) </br> [Intel® TDX Virtual Firmware Design Guide](https://cdrdv2.intel.com/v1/dl/getContent/733585) </br> [td-shim specification](https://github.com/confidential-containers/td-shim/blob/main/doc/tdshim_spec.md) |

![](docs/cc-trusted-foundation.png)

## 3. APIs

CC Trusted APIs aims to collect confidential primitives (i.e., measurement, event log, quote) for zero-trust design, supporting multiple deploy environments (firmware/VM/cloud native cluster).
The [APIs](common/python/cctrusted_base/api.py) are designed to be vendor agnostic and TCG compliant APIs. The APIs will keep evolving on demand. 

| API | Description  | Parameters  | Response  |
| --- | ------------- |----- |----- |
| get_default_algorithms | Get the default Digest algorithms supported by trusted foundation. | | The default algorithms |
| get_measurement_count | Get the count of measurement register. | | The count of measurement registers |
| get_measurement | Get measurement register according to given selected index and algorithms. | imr_select ([int, int]): The first is index of measurement register, the second is the algorithms ID | The count of measurement registers |
| get_quote | Get the quote for given nonce and data. | | The `Quote` object |
| get_eventlog | Get eventlog for given index and count. | | `TcgEventLog` object |

## 4. SDKs

It provides different SDKs for producing the confidential primitives in different deployment environments.
Choose correct SDK according to your environment.

| SDK | Deployment Scenarios |
| --- | --------------- |
| Firmware SDK | Firmware Application |
| [VM SDK](https://github.com/cc-api/cc-trusted-api/tree/main/vmsdk) | Confidential Virtual Machine |
| [Confidential Cloud Native Primitives (CCNP)](https://github.com/intel/confidential-cloud-native-primitives) | Confidential Cluster/Container |

## 5. How to use the APIs

Below example code collects measurement register count of the platform using API `get_measurement_count` using `VMSDK` in python. You can find more examples at [API usage example](docs/API-usage-example.md).

```
from cctrusted import CCTrustedVmSdk

# Get total count of measurement registers, Intel® TDX is 4, vTPM is 24
count = CCTrustedVmSdk.inst().get_measurement_count()
for index in range(CCTrustedVmSdk.inst().get_measurement_count()):
    # Get default digest algorithms, Intel® TDX is SHA384, vTPM is SHA256
    alg = CCTrustedVmSdk.inst().get_default_algorithms()
    # Get digest object for given index and given algorithms
    digest_obj = CCTrustedVmSdk.inst().get_measurement([index, alg.alg_id])

    hash_str = ""
    for hash_item in digest_obj.hash:
        hash_str += "".join([f"{hash_item:02x}", " "])

    LOG.info("Algorithms: %s", str(alg))
    LOG.info("HASH: %s", hash_str)
```

Run [cc_imr_cli.py](vmsdk/python/cc_imr_cli.py).

```
$ sudo su
# source setupenv.sh
# cd vmsdk/python
# python3 cc_imr_cli.py
```

Below is the example output on Intel® TDX via VM SDK:
```
cctrusted.cvm DEBUG    Successful open device node /dev/tdx_guest
cctrusted.cvm DEBUG    Successful read TDREPORT from /dev/tdx_guest.
cctrusted.cvm DEBUG    Successful parse TDREPORT.
cctrusted.cvm INFO     ======================================
cctrusted.cvm INFO     CVM type = TDX
cctrusted.cvm INFO     CVM version = 1.5
cctrusted.cvm INFO     ======================================
__main__ INFO     Algorithms: TPM_ALG_SHA384
__main__ INFO     HASH: c1 57 27 ca c1 f5 7d 0e 91 10 6d a1 80 b3 ea ba 72 11 66 61 e1 7b a0 55 37 73 84 3a 9b 07 2e cf a3 8c c8 03 df b5 5e 0f 87 ec 23 67 80 ad b3 a6
cctrusted.cvm INFO     ======================================
cctrusted.cvm INFO     CVM type = TDX
cctrusted.cvm INFO     CVM version = 1.5
cctrusted.cvm INFO     ======================================
__main__ INFO     Algorithms: TPM_ALG_SHA384
__main__ INFO     HASH: ee 35 46 2b 47 53 58 1b 4c 5a 53 8d c1 92 51 89 ba 9d 21 f5 19 7b 6b 15 ce 10 a6 00 fb d3 12 e0 e3 5c 2b 87 01 fc b2 17 51 82 43 3c 9b 12 b9 dc
cctrusted.cvm INFO     ======================================
cctrusted.cvm INFO     CVM type = TDX
cctrusted.cvm INFO     CVM version = 1.5
cctrusted.cvm INFO     ======================================
__main__ INFO     Algorithms: TPM_ALG_SHA384
__main__ INFO     HASH: 9a c0 ba 4e db 45 03 08 9a a4 a9 2a fe 97 cb 15 94 18 2f 44 aa e0 e5 8d 6f 90 a2 22 9c f9 a4 22 86 5d 87 35 d6 0b 87 3d 6b ec 36 41 d8 96 68 00
cctrusted.cvm INFO     ======================================
cctrusted.cvm INFO     CVM type = TDX
cctrusted.cvm INFO     CVM version = 1.5
cctrusted.cvm INFO     ======================================
__main__ INFO     Algorithms: TPM_ALG_SHA384
__main__ INFO     HASH: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00

```

## 6. Contributors

<!-- spell-checker: disable -->

<!-- readme: contributors -start -->
<table>
<tr>
    <td align="center">
        <a href="https://github.com/kenplusplus">
            <img src="https://avatars.githubusercontent.com/u/31843217?v=4" width="100;" alt="kenplusplus"/>
            <br />
            <sub><b>Lu Ken</b></sub>
        </a>
    </td>
    <td align="center">
        <a href="https://github.com/Ruoyu-y">
            <img src="https://avatars.githubusercontent.com/u/70305231?v=4" width="100;" alt="Ruoyu-y"/>
            <br />
            <sub><b>Ying Ruoyu</b></sub>
        </a>
    </td>
    <td align="center">
        <a href="https://github.com/intelzhongjie">
            <img src="https://avatars.githubusercontent.com/u/56340883?v=4" width="100;" alt="intelzhongjie"/>
            <br />
            <sub><b>Shi Zhongjie</b></sub>
        </a>
    </td>
    <td align="center">
        <a href="https://github.com/wenhuizhang">
            <img src="https://avatars.githubusercontent.com/u/2313277?v=4" width="100;" alt="wenhuizhang"/>
            <br />
            <sub><b>Wenhui Zhang</b></sub>
        </a>
    </td>
    <td align="center">
        <a href="https://github.com/jyao1">
            <img src="https://avatars.githubusercontent.com/u/12147155?v=4" width="100;" alt="jyao1"/>
            <br />
            <sub><b>Jiewen Yao</b></sub>
        </a>
    </td>
    <td align="center">
        <a href="https://github.com/leyao-daily">
            <img src="https://avatars.githubusercontent.com/u/54387247?v=4" width="100;" alt="leyao-daily"/>
            <br />
            <sub><b>Le Yao</b></sub>
        </a>
    </td></tr>
<tr>
    <td align="center">
        <a href="https://github.com/dongx1x">
            <img src="https://avatars.githubusercontent.com/u/34326010?v=4" width="100;" alt="dongx1x"/>
            <br />
            <sub><b>Xiaocheng Dong</b></sub>
        </a>
    </td>
    <td align="center">
        <a href="https://github.com/hairongchen">
            <img src="https://avatars.githubusercontent.com/u/105473940?v=4" width="100;" alt="hairongchen"/>
            <br />
            <sub><b>Hairongchen</b></sub>
        </a>
    </td></tr>
</table>
<!-- readme: contributors -end -->

<!-- spell-checker: enable -->
