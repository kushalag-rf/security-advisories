# RapidFort Security Advisories

RapidFort provides **validated, research-backed security advisory data** for all RapidFort Curated Images. This dataset enables partner scanners and security platforms to:

- Report vulnerabilities **with full parity to the RapidFort Analyzer**
- Integrate **per-package advisory data** seamlessly into their scanning workflow
- **Eliminate false positives** by excluding vulnerabilities that do not apply to RapidFort Curated Images

The repository contains a structured JSON database of per-package CVE advisories covering **Alpine Linux**, **Ubuntu**, and **Red Hat** distributions.

---

## Repository Structure

```
OS/
├── alpine/    # Alpine Linux advisory files
├── ubuntu/    # Ubuntu advisory files
└── redhat/    # Red Hat advisory files
```

Each advisory file covers a single source package and follows the naming convention:

```
OS/{os_name}/{package_name}_advisory.json
```

**Examples:**

```
OS/ubuntu/openssl_advisory.json
OS/alpine/busybox_advisory.json
OS/redhat/zlib_advisory.json
```

---

## Supported Operating Systems

> This list is updated as new OS releases are added to RapidFort Curated Images.

| Distribution | Supported Releases | Package Format |
|---|---|---|
| **Alpine Linux** | 3.20, 3.21, 3.22 | apk |
| **Ubuntu** | focal (20.04), jammy (22.04), noble (24.04) | dpkg |
| **Red Hat** | 5, 6, 7, 8, 9, 10 | rpm |

### Red Hat Stream Identifiers

Red Hat advisory events include an `identifier` field to disambiguate between distribution streams under the same major release:

| Prefix | Stream | Examples |
|---|---|---|
| `el` | RHEL / CentOS | `el6`, `el7`, `el8`, `el9` |
| `fc` | Fedora | `fc39`, `fc40`, `fc41`, `fc42`, `fc43` |

---

## Advisory JSON Schema

Each advisory file is a JSON object with the following structure:

```json
{
  "package_name": "zlib",
  "advisory": {
    "<release>": {
      "<CVE-ID>": {
        "cve_id": "CVE-2026-27171",
        "title": "Short vulnerability summary ...",
        "description": "Full vulnerability description.",
        "severity": "MEDIUM",
        "status": "fixed",
        "events": [
          {
            "introduced": "1.3.1-r1",
            "fixed": "1.3.2-r0",
            "identifier": "el9"
          }
        ]
      }
    }
  }
}
```

### Top-Level Fields

| Field | Type | Description |
|---|---|---|
| `package_name` | string | Distribution source package name |
| `advisory` | object | Keyed by OS release identifier (e.g. `"3.21"`, `"22.04"`, `"9"`) |

### CVE Entry Fields

| Field | Type | Description |
|---|---|---|
| `cve_id` | string | CVE identifier (matches the parent object key) |
| `title` | string | Short vulnerability summary |
| `description` | string | Full vulnerability description |
| `severity` | string | One of `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`, `UNKNOWN` |
| `status` | string | `"open"` (no fix available) or `"fixed"` (patch exists) |
| `events` | array | Version range entries describing affected and fixed versions |

### Event Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `introduced` | string | Yes | Version where the vulnerability was introduced. `"0"` means all versions are affected. |
| `fixed` | string | No | Version that resolves the vulnerability. Absent when no fix is available. |
| `identifier` | string | No | **Red Hat only.** Distribution stream tag (e.g. `el9`, `fc41`) used to disambiguate when a release key maps to multiple package streams. |

### Release Key Format by OS

| OS | Release Key Examples | Description |
|---|---|---|
| Alpine | `"3.20"`, `"3.21"`, `"3.22"` | Alpine minor version |
| Ubuntu | `"20.04"`, `"22.04"`, `"24.04"` | Ubuntu version number |
| Red Hat | `"5"`, `"6"`, `"7"`, `"8"`, `"9"`, `"10"` | Major release number |

---

## Usage Guide

### Step 1: Identify the OS and Release

Determine the operating system and release version of the target system being scanned (e.g. Alpine 3.21, Ubuntu 22.04, Red Hat 9).

### Step 2: Locate the Advisory File

Look up the advisory file for the installed package:

```
OS/{os_name}/{package_name}_advisory.json
```

**Examples:**

```
OS/ubuntu/openssl_advisory.json
OS/alpine/busybox_advisory.json
OS/redhat/yelp_advisory.json
```

### Step 3: Load and Navigate the Advisory

Parse the JSON file and navigate to the release key matching your target system:

```
advisory["{release}"] -> dictionary of CVE entries
```

For example, to get all CVEs affecting `zlib` on Ubuntu 22.04:

```
advisory["22.04"] -> { "CVE-2026-27171": { ... } }
```

### Step 4: Evaluate CVEs

For each CVE entry, determine whether it should be reported based on the `status` and version information:

#### Open CVEs

If `status = "open"`, **always report** the vulnerability. No fix is available.

```json
{
  "status": "open",
  "events": [{ "introduced": "0" }]
}
```

#### Fixed CVEs

If `status = "fixed"`, report **only if** the installed version is older than the fixed version:

```
installed_version < fixed_version  -->  report as vulnerable
installed_version >= fixed_version -->  not affected
```

#### Red Hat Identifier Matching

For Red Hat advisories, events may include an `identifier` field. When present, **only evaluate events whose `identifier` matches the target system's stream**:

- On RHEL 9, evaluate only events with `identifier = "el9"`
- On Fedora 41, evaluate only events with `identifier = "fc41"`

```json
{
  "events": [
    { "introduced": "0", "identifier": "el7" },
    { "introduced": "2:40.3-2.el9", "fixed": "2:40.3-2.el9_6.1", "identifier": "el9" },
    { "introduced": "2:42.2-6.fc41", "fixed": "42.2-9.fc41", "identifier": "fc41" }
  ]
}
```

---

## Severity Levels

| Severity | Description |
|---|---|
| `CRITICAL` | Exploitation is straightforward and typically results in system-level compromise |
| `HIGH` | Exploitation could result in significant data loss or service disruption |
| `MEDIUM` | Exploitation requires specific conditions but could impact confidentiality or integrity |
| `LOW` | Limited impact; exploitation is difficult or consequences are minimal |
| `UNKNOWN` | Severity has not been assessed by the upstream source |

---

## Version Formats

Version strings are OS-specific. Consumers must use the appropriate version comparison logic for each distribution.

| OS | Format | Example |
|---|---|---|
| Alpine | `{version}-r{revision}` | `1.3.1-r1` |
| Ubuntu | `{epoch}:{upstream}-{debian}{ubuntu}` | `1:1.2.11.dfsg-2ubuntu9.2` |
| Red Hat | `{epoch}:{version}-{release}.{dist}` | `2:42.2-5.fc40` |

**Notes:**
- The epoch prefix (e.g. `1:`, `2:`) is significant for version ordering and must not be ignored.
- An `introduced` value of `"0"` is a sentinel meaning "all versions from the beginning," not a literal version string.

---

## Schema Variations

A small number of packages (typically third-party or vendor-provided) have minor schema differences:

- Some Ubuntu packages (e.g. `kong`, `mongodb-org`) use a `summary` field instead of `description`.
- The `title` field may be an empty string for packages where the upstream source does not provide a short summary.
- The `severity` field may be `"UNKNOWN"` when no CVSS score is available from the upstream source.
