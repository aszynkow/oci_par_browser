# OCI Object Storage Browser

A lightweight, zero-install browser tool for sharing files with customers via Oracle Cloud Infrastructure (OCI) Object Storage **Pre-Authenticated Requests (PARs)**.

No backend. No login. No OCI account required for the end user. Just a single HTML file.

---

## What is it?

When you need to share files with a customer — or receive files from them — you typically have to deal with email attachments, OneDrive links, SFTP servers, or giving someone temporary OCI access. This tool removes all of that friction.

You create a **PAR** (Pre-Authenticated Request) on an OCI bucket, share the HTML file + PAR URL, and the customer gets a clean file browser right in their browser — no software to install, no account to create.

---

## Features

- 📂 Browse folders and files in an OCI bucket
- ⬇ Download individual files or select multiple for bulk download
- ⬆ Upload files via drag-and-drop or file picker (if PAR permits)
- 🗑 Delete files (if PAR permits)
- 📊 Real-time upload and download progress panel
- 🔐 Auto-detects PAR permissions — only shows actions the PAR allows
- 🗂 Sortable columns — Name, Size, Last Modified
- 🍞 Folder navigation with breadcrumb trail
- 🎨 Oracle Cloud console-style UI

---

## Screenshots

> Connect with a PAR URL → browse → transfer files

```
┌─────────────────────────────────────────────────────────────┐
│ ⬛ Oracle Cloud Infrastructure │ Object Storage Browser      │
├─────────────────────────────────────────────────────────────┤
│ Pre-Authenticated Request URL                               │
│ [ https://objectstorage.ap-sydney-1... ] [ Connect ]        │
├─────────────────────────────────────────────────────────────┤
│ Access: ✅ LIST  ✅ READ  ✅ WRITE  ❌ DELETE                │
├─────────────────────────────────────────────────────────────┤
│ ☁ Drop files here or click to upload                        │
├──────┬──────────────────────┬─────────┬────────────────┬────┤
│  ☐   │ NAME              ↑  │    SIZE │ LAST MODIFIED  │    │
├──────┼──────────────────────┼─────────┼────────────────┼────┤
│  📁  │ designs/             │      —  │      —         │  — │
│  ☐📄 │ proposal.pdf         │  2.4 MB │ 12 Jan 2025    │ DL │
│  ☐🖼 │ architecture.png     │  1.1 MB │ 10 Jan 2025    │ DL │
│  ☐📋 │ config.json          │   8 KB  │ 09 Jan 2025    │ DL │
└──────┴──────────────────────┴─────────┴────────────────┴────┘
```

---

## Quick Start

### 1. Download the tool

Download [`oci-par-browser.html`](./oci-par-browser.html) — that's the entire app.

### 2. Create a PAR on your OCI bucket

In the OCI Console:

1. Navigate to **Object Storage** → your bucket
2. Click **Pre-Authenticated Requests** → **Create Pre-Authenticated Request**
3. Set:
   - **Access Type:** `Permit object reads and writes on this bucket` (or read-only)
   - **Scope:** Bucket (not Object)
   - **Expiry:** Set an appropriate date
4. Copy the generated URL — you won't see it again

Or via OCI CLI:

```bash
# Read-only PAR (list + download)
oci os preauth-request create \
  --namespace <namespace> \
  --bucket-name <bucket> \
  --name "customer-share-$(date +%Y%m%d)" \
  --access-type AnyObjectRead \
  --time-expires $(date -u -v+7d +%Y-%m-%dT%H:%M:%SZ)

# Read + Write PAR (list + download + upload)
oci os preauth-request create \
  --namespace <namespace> \
  --bucket-name <bucket> \
  --name "customer-share-$(date +%Y%m%d)" \
  --access-type AnyObjectReadWrite \
  --time-expires $(date -u -v+7d +%Y-%m-%dT%H:%M:%SZ)
```

### 3. Set CORS on the bucket

Required for the browser to make cross-origin requests to OCI:

```bash
oci os bucket update \
  --namespace <namespace> \
  --bucket-name <bucket> \
  --cors-rules '[{
    "allowedOrigins": ["*"],
    "allowedMethods": ["GET","HEAD","PUT","DELETE"],
    "allowedHeaders": ["*"],
    "maxAgeSeconds": 3600
  }]'
```

### 4. Open the tool

Double-click `oci-par-browser.html` in your browser, paste the PAR URL, click **Connect**.

> **Tip:** If you hit CORS errors opening from disk, serve it locally:
> ```bash
> python3 -m http.server 8080
> # then open http://localhost:8080/oci-par-browser.html
> ```

---

## Use Cases

### Sharing deliverables with a customer
Create a read-only PAR, upload your files to the bucket, share the HTML file + PAR URL. The customer can browse and download without an OCI account.

### Collecting files from a customer
Create a write-only or read-write PAR. The customer can upload files directly into your bucket via the upload zone. You review them in the OCI Console or download via the tool.

### Internal file exchange between teams
Use a read-write PAR scoped to a specific prefix (folder) so teams can share large files without hitting email attachment limits or needing shared drives.

### Distributing software / release artefacts
Host build artefacts in an OCI bucket and distribute a read-only PAR to customers for self-service download. The tool shows file sizes and modification dates so customers know what they're getting.

### Workshop or event content delivery
Pre-load lab guides, datasets, or VM images into a bucket. Share the PAR with attendees so they can download what they need during the session.

---

## PAR Access Types

| Access Type | List | Download | Upload | Delete |
|---|:---:|:---:|:---:|:---:|
| `AnyObjectRead` | ✅ | ✅ | ❌ | ❌ |
| `AnyObjectWrite` | ❌ | ❌ | ✅ | ❌ |
| `AnyObjectReadWrite` | ✅ | ✅ | ✅ | ✅ |

> **Important:** Always use **Bucket** scope — object-scoped PARs point to a single file and cannot be used for browsing.

---

## Security Considerations

- **PAR URLs grant access to anyone who has them** — treat them like passwords
- Set the shortest expiry that makes sense for your use case
- Use `AnyObjectRead` unless you specifically need the customer to upload
- Consider creating a dedicated bucket per customer engagement rather than sharing a bucket
- CORS rules with `allowedOrigins: ["*"]` are fine for PAR-based access since the PAR token itself is the auth mechanism

---

## Architecture

```
Customer Browser
      │
      │  HTTPS (PAR URL as auth token)
      ▼
OCI Object Storage REST API
      │
      ├── GET  ?delimiter=/   → list objects/prefixes
      ├── GET  /object/key    → download file
      ├── PUT  /object/key    → upload file
      └── DELETE /object/key  → delete file
```

No backend required. The HTML file makes direct REST calls to OCI using the PAR token embedded in the URL.

---

## Requirements

- Any modern browser (Chrome, Firefox, Edge, Safari)
- An OCI Object Storage bucket with a valid bucket-scoped PAR
- A CORS policy on the bucket (see setup above)

---

## Support

Questions or issues — reach out to [adam.szynkowski@oracle.com](mailto:adam.szynkowski@oracle.com)
