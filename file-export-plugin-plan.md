# File Export Plugin — Implementation Plan

## Overview

A generic `file_export` plugin for Sharly Chess that uploads tournament files (TRF, PGN, etc.)
to an external storage target. The backend (FTP, S3, SFTP, …) is a pluggable strategy, so
adding a new upload destination requires only a new backend class — no changes to the plugin
core.

The plugin integrates into the existing `apluggy`-based plugin system (see `src/plugins/`) and
reuses the same patterns found in `chess_results` and `ffe`.

---

## Goals

- Upload tournament files automatically when tournament data changes.
- Support multiple storage backends: **FTP** and **S3-compatible** (AWS S3, MinIO, Cloudflare R2) on day one.
- Make adding further backends (SFTP, WebDAV, local disk, …) trivial — no plugin core changes.
- Store backend credentials securely in the existing plugin config database.
- Provide a settings UI consistent with the existing tournament/event admin forms.

---

## Directory Structure

```
src/plugins/
└── file_export/
    ├── __init__.py                       # PLUGIN_NAME, PLUGIN_DIR, TMP_DIR constants
    ├── file_export.py                    # Main Plugin class + hookimpl implementations
    ├── utils.py                          # PluginData classes (config / event / tournament)
    ├── backends/
    │   ├── __init__.py                   # BACKENDS registry + get_backend() factory
    │   ├── base.py                       # Abstract BackendConfig + StorageBackend
    │   ├── ftp_backend.py                # FTP implementation (stdlib ftplib)
    │   └── s3_backend.py                 # S3 implementation (boto3)
    ├── formatters/
    │   ├── __init__.py
    │   ├── base.py                       # Abstract FileFormatter
    │   └── trf_formatter.py              # TRF export (reuses existing TRF logic)
    ├── templates/
    │   ├── file_export_tournament_form_fields.html
    │   ├── file_export_ftp_fields.html
    │   └── file_export_s3_fields.html
    └── locale/                           # i18n strings
```

---

## Architecture

The design applies the **Strategy Pattern**: the plugin core is completely decoupled from
transport concerns. The selected backend is resolved at runtime from the stored `backend_id`.

```
Plugin[FileExportConfigPluginData]
        │
        └── on_tournament_data_updated()
                │
                ├── FileFormatter.generate()      → bytes
                │
                └── FileExportConfigPluginData.get_backend()
                            │
                    ┌───────┴────────┐
                    ▼                ▼
             FtpBackend          S3Backend        ← (SftpBackend, WebDavBackend …)
              upload()            upload()
                │                   │
              ftplib             boto3 / s3transfer
```

---

## Implementation Steps

### Step 1 — `__init__.py`

Define module-level constants following the pattern of existing plugins.

```python
# src/plugins/file_export/__init__.py
from pathlib import Path
import common
from plugins import PLUGINS_DIR

PLUGIN_NAME: str = 'file_export'
PLUGIN_DIR: Path = PLUGINS_DIR / PLUGIN_NAME
TMP_DIR: Path = common.TMP_DIR / PLUGIN_NAME
TMP_DIR.mkdir(parents=True, exist_ok=True)
```

---

### Step 2 — Abstract Backend Layer (`backends/base.py`)

```python
# src/plugins/file_export/backends/base.py
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Any, Self


@dataclass
class BackendConfig(ABC):
    """Credentials and settings for one specific storage backend.
    Each concrete subclass owns its own fields and (de)serialisation."""

    @classmethod
    @abstractmethod
    def backend_id(cls) -> str:
        """Unique slug, e.g. 'ftp', 's3'. Used as the discriminator in stored config."""

    @classmethod
    @abstractmethod
    def from_stored_value(cls, data: dict[str, Any]) -> Self: ...

    @abstractmethod
    def to_stored_value(self) -> dict[str, Any]: ...

    @classmethod
    @abstractmethod
    def from_form_data(cls, data: dict[str, str]) -> Self: ...

    @abstractmethod
    def to_form_data(self) -> dict[str, str]: ...


class StorageBackend(ABC):
    """Strategy interface. All upload targets implement this single method."""

    config_class: type[BackendConfig]

    def __init__(self, config: BackendConfig):
        self.config = config

    @abstractmethod
    def upload(self, file_bytes: bytes, remote_filename: str) -> None:
        """Upload *file_bytes* to the configured remote destination."""
```

---

### Step 3 — FTP Backend (`backends/ftp_backend.py`)

Uses **stdlib `ftplib`** — no extra dependency.

```python
# src/plugins/file_export/backends/ftp_backend.py
from dataclasses import dataclass, field
from ftplib import FTP
from io import BytesIO
from typing import Any, Self

from plugins.file_export.backends.base import BackendConfig, StorageBackend
from web.controllers.base_controller import WebContext


@dataclass
class FtpBackendConfig(BackendConfig):
    host: str = ''
    user: str = ''
    password: str = ''
    remote_dir: str = '/'
    port: int = 21
    passive: bool = True

    @classmethod
    def backend_id(cls) -> str:
        return 'ftp'

    @classmethod
    def from_stored_value(cls, data: dict[str, Any]) -> Self:
        return cls(
            host=data.get('host', ''),
            user=data.get('user', ''),
            password=data.get('password', ''),
            remote_dir=data.get('remote_dir', '/'),
            port=data.get('port', 21),
            passive=data.get('passive', True),
        )

    def to_stored_value(self) -> dict[str, Any]:
        return {
            'host': self.host, 'user': self.user, 'password': self.password,
            'remote_dir': self.remote_dir, 'port': self.port, 'passive': self.passive,
        }

    @classmethod
    def from_form_data(cls, data: dict[str, str]) -> Self:
        return cls(
            host=WebContext.form_data_to_str(data, 'ftp_host') or '',
            user=WebContext.form_data_to_str(data, 'ftp_user') or '',
            password=WebContext.form_data_to_str(data, 'ftp_password') or '',
            remote_dir=WebContext.form_data_to_str(data, 'ftp_remote_dir') or '/',
            port=WebContext.form_data_to_int(data, 'ftp_port') or 21,
            passive=WebContext.form_data_to_bool(data, 'ftp_passive'),
        )

    def to_form_data(self) -> dict[str, str]:
        return WebContext.values_dict_to_form_data({
            'ftp_host': self.host, 'ftp_user': self.user, 'ftp_password': self.password,
            'ftp_remote_dir': self.remote_dir, 'ftp_port': self.port,
            'ftp_passive': self.passive,
        })


class FtpBackend(StorageBackend):
    config_class = FtpBackendConfig

    def upload(self, file_bytes: bytes, remote_filename: str) -> None:
        cfg: FtpBackendConfig = self.config  # type: ignore[assignment]
        with FTP() as ftp:
            ftp.connect(cfg.host, cfg.port)
            ftp.login(cfg.user, cfg.password)
            ftp.set_pasv(cfg.passive)
            ftp.cwd(cfg.remote_dir)
            ftp.storbinary(f'STOR {remote_filename}', BytesIO(file_bytes))
```

---

### Step 4 — S3 Backend (`backends/s3_backend.py`)

Uses **`boto3`** (add to `requirements.txt`). Supports AWS S3, MinIO, Cloudflare R2 and any
S3-compatible service via the optional `endpoint_url`.

```python
# src/plugins/file_export/backends/s3_backend.py
from dataclasses import dataclass
from typing import Any, Self

from plugins.file_export.backends.base import BackendConfig, StorageBackend
from web.controllers.base_controller import WebContext


@dataclass
class S3BackendConfig(BackendConfig):
    bucket: str = ''
    region: str = ''
    access_key_id: str = ''
    secret_access_key: str = ''
    prefix: str = ''           # optional key prefix / sub-folder
    endpoint_url: str = ''     # empty = AWS; set for MinIO / Cloudflare R2 / etc.

    @classmethod
    def backend_id(cls) -> str:
        return 's3'

    @classmethod
    def from_stored_value(cls, data: dict[str, Any]) -> Self:
        return cls(
            bucket=data.get('bucket', ''),
            region=data.get('region', ''),
            access_key_id=data.get('access_key_id', ''),
            secret_access_key=data.get('secret_access_key', ''),
            prefix=data.get('prefix', ''),
            endpoint_url=data.get('endpoint_url', ''),
        )

    def to_stored_value(self) -> dict[str, Any]:
        return {
            'bucket': self.bucket, 'region': self.region,
            'access_key_id': self.access_key_id,
            'secret_access_key': self.secret_access_key,
            'prefix': self.prefix, 'endpoint_url': self.endpoint_url,
        }

    @classmethod
    def from_form_data(cls, data: dict[str, str]) -> Self:
        return cls(
            bucket=WebContext.form_data_to_str(data, 's3_bucket') or '',
            region=WebContext.form_data_to_str(data, 's3_region') or '',
            access_key_id=WebContext.form_data_to_str(data, 's3_access_key_id') or '',
            secret_access_key=WebContext.form_data_to_str(data, 's3_secret_access_key') or '',
            prefix=WebContext.form_data_to_str(data, 's3_prefix') or '',
            endpoint_url=WebContext.form_data_to_str(data, 's3_endpoint_url') or '',
        )

    def to_form_data(self) -> dict[str, str]:
        return WebContext.values_dict_to_form_data({
            's3_bucket': self.bucket, 's3_region': self.region,
            's3_access_key_id': self.access_key_id,
            's3_secret_access_key': self.secret_access_key,
            's3_prefix': self.prefix, 's3_endpoint_url': self.endpoint_url,
        })


class S3Backend(StorageBackend):
    config_class = S3BackendConfig

    def upload(self, file_bytes: bytes, remote_filename: str) -> None:
        import boto3  # lazy import — optional dependency
        cfg: S3BackendConfig = self.config  # type: ignore[assignment]

        client_kwargs: dict[str, Any] = {
            'aws_access_key_id': cfg.access_key_id,
            'aws_secret_access_key': cfg.secret_access_key,
            'region_name': cfg.region or None,
        }
        if cfg.endpoint_url:
            client_kwargs['endpoint_url'] = cfg.endpoint_url  # MinIO / R2 / etc.

        s3 = boto3.client('s3', **client_kwargs)
        key = f'{cfg.prefix}/{remote_filename}'.lstrip('/')
        s3.put_object(Bucket=cfg.bucket, Key=key, Body=file_bytes)
```

---

### Step 5 — Backend Registry (`backends/__init__.py`)

```python
# src/plugins/file_export/backends/__init__.py
from plugins.file_export.backends.ftp_backend import FtpBackend, FtpBackendConfig
from plugins.file_export.backends.s3_backend import S3Backend, S3BackendConfig
from plugins.file_export.backends.base import StorageBackend

BACKENDS: dict[str, type[StorageBackend]] = {
    FtpBackendConfig.backend_id(): FtpBackend,
    S3BackendConfig.backend_id(): S3Backend,
    # Register new backends here — no other change needed.
}


def get_backend(backend_id: str, config_data: dict) -> StorageBackend:
    backend_cls = BACKENDS[backend_id]
    config = backend_cls.config_class.from_stored_value(config_data)
    return backend_cls(config)
```

---

### Step 6 — PluginData (`utils.py`)

Three data classes following the same pattern as `chess_results/utils.py`:

| Class | Scope | Key fields |
|---|---|---|
| `FileExportConfigPluginData` | Global (app-level) | `backend_id`, `backend_config` |
| `FileExportEventPluginData` | Per event | `auto_upload`, file format (`trf`, `pgn`) |
| `FileExportTournamentPluginData` | Per tournament | `auto_upload` override, `last_upload` timestamp |

Key method on the config class:

```python
def get_backend(self) -> StorageBackend:
    from plugins.file_export.backends import get_backend
    return get_backend(self.backend_id, self.backend_config)
```

The `backend_config` is stored as a plain `dict[str, Any]` — the backend class owns the
schema of that dict. This allows backend-specific fields to grow independently.

---

### Step 7 — Main Plugin Class (`file_export.py`)

Key hooks to implement:

| Hook | Purpose |
|---|---|
| `get_event_plugin_data_class` | Register `FileExportEventPluginData` |
| `get_tournament_plugin_data_class` | Register `FileExportTournamentPluginData` |
| `get_tournament_form_fields_template_and_data` | Per-tournament auto-upload toggle |
| `validate_tournament_form_fields` | Validate form values |
| `on_tournament_data_updated` | **Trigger the upload** (mirroring `chess_results`) |
| `get_nav_upload_items` | Add "File Export" item to the upload nav |

Core upload logic (mirrors `ChessResultsBackgroundUploader`):

```python
@hookimpl
def on_tournament_data_updated(
    self, stored_event: 'StoredEvent', stored_tournament: 'StoredTournament'
) -> None:
    config = self.get_plugin_data()
    if not config.backend_config:
        return   # backend not configured yet

    # Use the same background upload pattern as chess_results
    FileExportBackgroundUploader.schedule_upload(
        stored_event, stored_tournament, config
    )
```

`FileExportBackgroundUploader` (a separate class in `file_export_background_uploader.py`)
handles threading and status tracking, copying the structure of
`ChessResultsBackgroundUploader`.

---

### Step 8 — Register in `manager.py`

```python
# src/plugins/manager.py
from plugins.file_export.file_export import FileExportPlugin

plugins = [
    PairingAccelerationPlugin(),
    ChessResultsPlugin(),
    FfePlugin(),
    ChessEventPlugin(),
    FRASchoolsPlugin(),
    HandicapGamesPlugin(),
    FileExportPlugin(),      # ← add here
]
```

---

### Step 9 — Dependencies

Add to `requirements.txt` (or make it optional with a `try/except ImportError`):

```
boto3>=1.34        # S3 backend — optional, only needed when S3 is selected
```

`ftplib` is part of the Python standard library — no extra dependency.

For SFTP (future): `paramiko>=3.0`.

---

## Adding a New Backend in the Future

To add e.g. **SFTP**:

1. Create `backends/sftp_backend.py` with `SftpBackendConfig(BackendConfig)` + `SftpBackend(StorageBackend)`.
2. Add one line to `backends/__init__.py`:
   ```python
   BACKENDS['sftp'] = SftpBackend
   ```
3. Add a Jinja2 partial template `file_export_sftp_fields.html`.
4. Add the new option to the backend selector in the settings form.

**Zero changes** to `file_export.py`, `utils.py`, or any other plugin file.

---

## UI Considerations

- The tournament settings form has a **backend selector** (`<select name="file_export_backend_id">`).
- On change, the relevant backend-specific sub-form is rendered (via HTMX or JS toggle).
- The global plugin config page (account or admin level) holds the credentials.
- Status messages reuse the `NavUploadItem` + WebSocket channel pattern from `chess_results`.

---

## Security Notes

- **Passwords/secrets are stored as plain text** in the plugin config SQLite DB by default.
  For production, consider encrypting them with the same AES approach used in
  `ChessResultsUtils.encrypt()` and storing the key in an environment variable.
- S3 `secret_access_key` should use **least-privilege IAM policies** (PutObject only on the
  target bucket).
- FTP passwords travel in plaintext — prefer SFTP or FTPS in production.

---

## Out of Scope (Phase 2 / Future)

- SFTP backend (`paramiko`)
- WebDAV backend
- Local-disk backend (useful for testing/NAS scenarios)
- PGN formatter (only TRF in phase 1)
- Per-round upload (currently triggers on any tournament data change)
- Retry logic / exponential back-off on upload failure
