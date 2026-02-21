# Plan: File Export Plugin for SharlyChess (#1422)

## Context

**Issue**: [#1422](https://github.com/Sharly-Chess/sharly-chess/issues/1422) — "Add a way to export HTML files to a custom location"

Arbiters want to publish tournament pages (pairings, standings, results) to their own website without relying on Chess-Results.com or FFE. Currently screens are only viewable through the live SharlyChess web server. This plugin will render screen pages as self-contained static HTML and push them to a configurable storage (local folder, FTP, S3, or any future storage type).

Architecture adopts the **Strategy Pattern** from [file-export-plugin-plan.md](file-export-plugin-plan.md) — the plugin core is decoupled from transport. Adding a new storage requires only a new storage class and one registry line. No changes to the plugin core.

---

## Architecture

```
FileExportPlugin
    │
    └── on_tournament_data_updated()
            │
            ├── StaticScreenRenderer.render_screens()  → dict[str, bytes]
            │     (boards.html, ranking.html, results.html, players.html, index.html)
            │     + static assets (CSS, JS, fonts)
            │
            └── FileExportEventPluginData.get_storage()
                        │
                ┌───────┴─────────┬──────────────────┐
                ▼                 ▼                   ▼
         LocalStorage       FtpStorage           S3Storage      ← future: SftpStorage, WebDavStorage
          upload()           upload()             upload()
            │                  │                    │
          pathlib            ftplib              boto3
```

**Key design**: `Storage.upload(file_bytes, remote_path)` is the single method all storages implement. The `remote_path` supports subdirectories (e.g., `open-tournament/boards.html`). Each storage handles directory creation internally.

---

## Directory Structure

```
src/plugins/file_export/
├── __init__.py                        # PLUGIN_NAME, PLUGIN_DIR, TMP_DIR
├── file_export.py                     # Plugin class + hookimpl implementations
├── file_export_background_exporter.py # Background Timer-based exporter
├── file_export_controller.py          # Web controller (status modal, manual export)
├── file_export_renderer.py            # StaticScreenRenderer + StaticExportClient
├── utils.py                           # PluginData classes (event / tournament)
├── storages/
│   ├── __init__.py                    # STORAGES registry + get_storage() factory
│   ├── base.py                        # Abstract StorageConfig + Storage base classes
│   ├── local_storage.py               # Local folder (pathlib)
│   ├── ftp_storage.py                 # FTP/FTPS (stdlib ftplib)
│   └── s3_storage.py                  # S3-compatible (boto3, lazy import)
├── templates/
│   ├── file_export_upload_modal.html
│   ├── file_export_upload_results.html
│   ├── file_export_event_form_fields.html
│   └── file_export/
│       └── static_export/
│           ├── base.html              # Self-contained HTML (no HTMX)
│           ├── screen.html            # Includes existing screen sub-templates
│           └── index.html             # Landing page with tournament links
└── locale/                            # i18n strings
```

---

## Phase 1: Storage Layer

### 1.1 Abstract Storage (`storages/base.py`)

Following [file-export-plugin-plan.md](file-export-plugin-plan.md):

```python
@dataclass
class StorageConfig(ABC):
    @classmethod
    @abstractmethod
    def storage_id(cls) -> str: ...        # 'local', 'ftp', 's3'

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


class Storage(ABC):
    config_class: type[StorageConfig]

    def __init__(self, config: StorageConfig):
        self.config = config

    @abstractmethod
    def upload(self, file_bytes: bytes, remote_path: str) -> None:
        """Upload file_bytes to remote_path (may include subdirectories)."""

    def close(self) -> None:
        """Clean up resources (FTP connection, etc.)."""

    def __enter__(self): return self
    def __exit__(self, *exc): self.close()
```

### 1.2 Storage Implementations

**`local_storage.py`**:
```python
class LocalStorageConfig(StorageConfig):
    path: str = ''
    storage_id() → 'local'

class LocalStorage(Storage):
    def upload(self, file_bytes, remote_path):
        full = Path(self.config.path) / remote_path
        full.parent.mkdir(parents=True, exist_ok=True)
        full.write_bytes(file_bytes)
```

**`ftp_storage.py`** — using `ftplib` (stdlib), per [file-export-plugin-plan.md](file-export-plugin-plan.md):
```python
class FtpStorageConfig(StorageConfig):
    host, user, password, remote_dir, port, use_tls, passive
    storage_id() → 'ftp'

class FtpStorage(Storage):
    # Lazy connect. Caches created dirs. FTP_TLS + prot_p() when use_tls=True.
    # mkdir_recursive for subdirs. 30s timeout.
    def upload(self, file_bytes, remote_path):
        self._ensure_connected()
        full = f'{self.config.remote_dir}/{remote_path}'
        self._mkdir_recursive(dirname(full))
        self._ftp.storbinary(f'STOR {full}', BytesIO(file_bytes))
```

**`s3_storage.py`** — using `boto3` (lazy import), per [file-export-plugin-plan.md](file-export-plugin-plan.md):
```python
class S3StorageConfig(StorageConfig):
    bucket, region, access_key_id, secret_access_key, prefix, endpoint_url
    storage_id() → 's3'

class S3Storage(Storage):
    def upload(self, file_bytes, remote_path):
        import boto3  # lazy — optional dependency
        key = f'{self.config.prefix}/{remote_path}'.lstrip('/')
        content_type = guess_content_type(remote_path)  # text/html, text/css, etc.
        self._client().put_object(
            Bucket=self.config.bucket, Key=key, Body=file_bytes,
            ContentType=content_type,
        )
```

### 1.3 Storage Registry (`storages/__init__.py`)

```python
STORAGES: dict[str, type[Storage]] = {
    'local': LocalStorage,
    'ftp': FtpStorage,
    's3': S3Storage,
}

def get_storage(storage_id: str, config_data: dict) -> Storage:
    storage_cls = STORAGES[storage_id]
    config = storage_cls.config_class.from_stored_value(config_data)
    return storage_cls(config)
```

**Adding a future storage** (SFTP, WebDAV) = one new file + one line here. Zero changes elsewhere.

---

## Phase 2: Plugin Skeleton + Data Classes

### 2.1 Plugin init (`__init__.py`)

```python
PLUGIN_NAME: str = 'file_export'
PLUGIN_DIR: Path = PLUGINS_DIR / PLUGIN_NAME
TMP_DIR: Path = common.TMP_DIR / PLUGIN_NAME
TMP_DIR.mkdir(parents=True, exist_ok=True)
```

### 2.2 Plugin data classes (`utils.py`)

Following [chess_results/utils.py](src/plugins/chess_results/utils.py):

```python
@dataclass
class FileExportEventPluginData(PluginData):
    auto_export: bool = False
    auto_export_delay: int = 3            # minutes
    storage_id: str = 'local'             # 'local' | 'ftp' | 's3'
    storage_config: dict[str, Any] = field(default_factory=dict)
    # storage_config is a plain dict — the StorageConfig subclass owns its schema

    def get_storage(self) -> Storage:
        return get_storage(self.storage_id, self.storage_config)

@dataclass
class FileExportTournamentPluginData(PluginData):
    auto_export: bool | None = None       # None = inherit event setting
    last_export: datetime | None = None
    export_boards: bool = True
    export_players: bool = True
    export_ranking: bool = True
    export_results: bool = True
```

### 2.3 Registration

**Modify**: [manager.py:37-45](src/plugins/manager.py#L37-L45) — add `FileExportPlugin()` to `plugins` list.

---

## Phase 3: Static Screen Renderer

### 3.1 Renderer (`file_export_renderer.py`)

Extracts column-building logic from [base_screen_user_controller.py:148-191](src/web/controllers/user/base_screen_user_controller.py#L148-L191) into a standalone method:

```python
class StaticExportClient:
    """Stub — all permissions False. Makes url_for guards in sub-templates unreachable."""
    def can_update_results(self, *a): return False
    def can_enter_results(self, *a): return False
    def can_set_special_results(self, *a): return False
    can_manage_screens = False
    can_view_private_screens = False

class StaticScreenRenderer:
    @classmethod
    def render_screen(cls, event: Event, screen: Screen) -> str:
        columns = cls._build_columns(event, screen)
        context = {
            'screen': screen, 'user_event': event,
            'columns_by_tournament_id': columns,
            'sharly_chess_config': SharlyChessConfig(),
            'static_export': True,
            'static_base_path': '..',
            'client': StaticExportClient(),
            'rotator': None, 'display_controller': None,
            'last_result_updated': None,
            'last_illegal_move_updated': None,
            'last_check_in_updated': None,
            'messages': [], 'DEVEL_ENV': False, 'now': datetime.now(),
        }
        return parse_jinja_template('file_export/static_export/screen.html', context)

    @classmethod
    def _build_columns(cls, event, screen):
        # Replicates base_screen_user_controller.py:152-191
        # PlayerColumnHandler, BoardColumnHandler, ColumnUsage.SCREEN

    @classmethod
    def export_event(cls, event, storage, tournament_plugin_data):
        """Export all screens for all tournaments to the storage."""
        with storage:
            cls._upload_static_assets(storage)
            tournament_pages = {}
            for tournament in event.tournaments:
                pages = cls._render_tournament_screens(event, tournament, ...)
                for filename, html in pages.items():
                    storage.upload(html.encode('utf-8'),
                                   f'{tournament.sanitized_name}/{filename}')
                tournament_pages[tournament] = pages.keys()
            index_html = cls.render_index(event, tournament_pages)
            storage.upload(index_html.encode('utf-8'), 'index.html')
```

Uses `parse_jinja_template()` from [common/i18n/utils.py](src/common/i18n/utils.py) — works outside request context.

### 3.2 Static Export Templates

**`templates/file_export/static_export/base.html`** — self-contained HTML:
- CSS via relative paths: `{{ static_base_path }}/static/css/base.css`
- Bootstrap CSS + Icons CSS, `base.css`, `user.css`
- jQuery + Bootstrap bundle JS (for tooltips)
- **NO**: HTMX, morphdom, Sortable, select2, WebSocket, modals, sidebar
- **NO**: `url_for()` or `request` references
- Imports `macros.j2` (needed by sub-templates for `column_header`, `column_cell`)
- Copyright footer

**`templates/file_export/static_export/screen.html`** — extends `base.html`:
- Reuses existing sub-templates via `{% include %}`:
  - `user/screen/ranking/set.html` — **no** `url_for`, safe as-is
  - `user/screen/results.html` — **no** `url_for`, safe as-is
  - `user/screen/players/set.html` — **no** `url_for`, safe as-is
  - `user/screen/boards/set.html` — `url_for` guarded by `editable` → `client.can_update_results()` → always `False`
- Skips `input` screen type (interactive)
- Static `<a href>` links between pages instead of HTMX
- Optional `<meta http-equiv="refresh" content="60">` for auto-reload

**`templates/file_export/static_export/index.html`** — tournament listing with links.

### 3.3 Static Assets

Copy subset of `src/web/static/` to `static/` in export:
```
static/
├── css/base.css, css/user.css
├── lib/bootstrap/.../css/bootstrap.min.css + js/bootstrap.bundle.min.js
├── lib/bootstrap-icons/.../font/bootstrap-icons.min.css + fonts/
├── lib/jquery/.../jquery.min.js
└── images/ (flags, logos)
```

Version marker (`static/.version`) avoids re-uploading unchanged assets.

### 3.4 Output Structure

```
export_root/
├── index.html
├── static/css/, lib/, images/
├── open-tournament/
│   ├── boards.html
│   ├── players.html
│   ├── ranking.html
│   └── results.html
└── blitz-tournament/
    └── ...
```

---

## Phase 4: Background Exporter + Auto-Export + UI

### 4.1 Background Exporter (`file_export_background_exporter.py`)

Mirrors [chess_results_background_uploader.py](src/plugins/chess_results/chess_results_background_uploader.py):

```python
class FileExportStatus(IntEnum):
    NEVER=0; EXPORTED=1; CHANGED=2; PENDING=3
    IN_PROGRESS=4; SUCCESS=5; INFO=6; ERROR=7; SETTINGS_ERROR=8

class FileExportBackgroundExporter:
    export_status_messages: dict[str, FileExportResult] = {}
    timeout_threads: dict[str, Timer] = {}

    should_schedule() → checks auto_export, threads, dirty state
    schedule_export() → Timer with configurable delay
    export_tournament() → set_locale, EventLoader, build storage, render, upload, update status
```

### 4.2 Plugin Class (`file_export.py`)

```python
class FileExportPlugin(Plugin[None]):
    static_id() → 'file_export'
    static_name() → _('File Export')
    default_is_enabled → True

    # Hooks:
    get_event_plugin_data_class → FileExportEventPluginData
    get_tournament_plugin_data_class → FileExportTournamentPluginData
    on_tournament_data_updated → schedule background export
    get_nav_upload_items → NavUploadItem with error status
    controllers → [FileExportController]
    event_form_fields_template → 'file_export_event_form_fields.html'
```

### 4.3 Controller (`file_export_controller.py`)

Mirrors [chess_results controller](src/plugins/chess_results/chess_results_event_controller.py):
- `GET /file-export/modal/{event_uniq_id}` — status modal
- `GET /file-export/results/{event_uniq_id}` — status rows (ws auto-refresh)
- `POST /file-export/export/{event_uniq_id}` — export all
- `POST /file-export/export-tournament/{event_uniq_id}/{tournament_id}` — export one

### 4.4 UI Templates

Modeled on [chess_results_upload_modal.html](src/plugins/chess_results/templates/chess_results_upload_modal.html):
- `file_export_upload_modal.html` — status table + "Export now" button
- `file_export_upload_results.html` — per-tournament rows with status, last export time, manual export button
- `file_export_event_form_fields.html` — storage selector dropdown + storage-specific sub-forms that toggle via JS

---

## Key Files Reference

| Purpose | File |
|---|---|
| Extensible storage architecture | [file-export-plugin-plan.md](file-export-plugin-plan.md) |
| Pattern: plugin class | [chess_results.py](src/plugins/chess_results/chess_results.py) |
| Pattern: background uploader | [chess_results_background_uploader.py](src/plugins/chess_results/chess_results_background_uploader.py) |
| Pattern: plugin data | [chess_results/utils.py](src/plugins/chess_results/utils.py) |
| Pattern: upload modal | [chess_results_upload_modal.html](src/plugins/chess_results/templates/chess_results_upload_modal.html) |
| Reuse: column building | [base_screen_user_controller.py:148-191](src/web/controllers/user/base_screen_user_controller.py#L148-L191) |
| Reuse: template rendering | [common/i18n/utils.py](src/common/i18n/utils.py) — `parse_jinja_template()` |
| Reuse: screen sub-templates | [ranking/set.html](src/web/templates/user/screen/ranking/set.html), [results.html](src/web/templates/user/screen/results.html), [players/set.html](src/web/templates/user/screen/players/set.html), [boards/set.html](src/web/templates/user/screen/boards/set.html) |
| Modify: plugin registration | [manager.py:37-45](src/plugins/manager.py#L37-L45) |

## Dependencies

- `boto3>=1.34` — optional, lazy-imported only when S3 storage selected
- `ftplib` — stdlib
- Future: `paramiko>=3.0` for SFTP storage

## Adding a New Storage (Future)

1. Create `storages/new_storage.py` with `NewStorageConfig(StorageConfig)` + `NewStorage(Storage)`
2. Add one line to `storages/__init__.py`: `STORAGES['new'] = NewStorage`
3. Add template `file_export_new_fields.html` for config form
4. Add option to storage selector dropdown

**Zero changes** to `file_export.py`, `utils.py`, renderer, or any other plugin file.

## Verification

1. **Local export**: Configure local path → "Export now" → HTML opens in browser without SharlyChess running
2. **FTP export**: Configure FTP → export → files appear on FTP server
3. **S3 export**: Configure S3/MinIO → export → files served as static website
4. **Auto-export**: Enable → enter result → files update after delay
5. **Static HTML quality**: No `hx-` attributes, no `url_for` errors, CSS/JS loads correctly
6. **New storage extensibility**: Add mock storage + registry line → verify it works
7. **Unit tests**: Storage implementations with temp dirs; renderer column building with fixture data
