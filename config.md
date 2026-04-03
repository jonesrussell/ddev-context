# DDEV Configuration System

Subsystem spec covering `pkg/globalconfig/` and `pkg/config/` (types, remoteconfig, state).

## Layer Rules

**Layer 1 (Config).** Foundation configuration layer.

- **Imports from:** Layer 0 only (util, fileutil, nodeps)
- **Imported by:** Layer 2 (dockerutil), Layer 3 (ddevapp), Layer 4 (cmd/ddev)
- **Never imports:** dockerutil, ddevapp, or any higher layer

## File Map

| File | Purpose |
|---|---|
| `pkg/globalconfig/global_config.go` | `GlobalConfig` struct, `ReadGlobalConfig()`, `WriteGlobalConfig()`, project list CRUD |
| `pkg/globalconfig/remote_config.go` | `RemoteConfig` struct (URLs + intervals), default URL constants |
| `pkg/globalconfig/messages.go` | `MessagesConfig` struct (`TickerInterval int`) |
| `pkg/globalconfig/values.go` | `ValidOmitContainers` map, `ValidXdebugIDELocations`, validation helpers, env-var globals |
| `pkg/globalconfig/errors.go` | `InvalidOmitContainers` error type |
| `pkg/globalconfig/types/types.go` | `RouterTypeTraefik` const, `IsValidRouterType()`, `GetValidRouterTypes()` |
| `pkg/globalconfig/performance_mode.go` | `GlobalConfig.GetPerformanceMode()`, `SetPerformanceMode()`, `IsMutagenEnabled()` |
| `pkg/globalconfig/xhprof_mode.go` | `GlobalConfig.GetXHProfMode()`, `SetXHProfMode()` |
| `pkg/globalconfig/styles.go` | `IsValidTableStyle()`, `GetTableStyleByName()` |
| `pkg/config/types/config_type.go` | `ConfigType`, `ConfigTypeGlobal`, `ConfigTypeProject`; `PerformanceMode`/`XHProfMode` types and constants |
| `pkg/config/types/performance_mode.go` | `ValidPerformanceModeOptions()`, `IsValidPerformanceMode()`, `CheckValidPerformanceMode()`, flag constants |
| `pkg/config/types/xhprof_mode.go` | `IsValidXHProfMode()`, `FlagXHProfModeDefault`, flag constants |
| `pkg/config/types/performance_mode_linux.go` | `GetPerformanceModeDefault()` → `PerformanceModeNone` (build tag) |
| `pkg/config/types/performance_mode_darwin_win.go` | `GetPerformanceModeDefault()` → `PerformanceModeMutagen` (build tag) |
| `pkg/config/remoteconfig/remote_config.go` | `New()` constructor, `remoteConfig` struct, download + local cache logic |
| `pkg/config/remoteconfig/config.go` | `Config` struct with `Local`, URL, intervals; `getLocalSourceFileName()` |
| `pkg/config/remoteconfig/global.go` | `InitGlobal()`/`GetGlobal()` singletons; `InitGlobalSponsorship()`/`GetGlobalSponsorship()` |
| `pkg/config/remoteconfig/state.go` | `state` wrapper with `stateEntry` (UpdatedAt, LastNotificationAt, LastTickerAt, LastTickerMessage, LastSponsorshipAt) |
| `pkg/config/remoteconfig/messages.go` | `ShowNotifications()`, `ShowTicker()` display logic |
| `pkg/config/remoteconfig/sponsorship.go` | `NewSponsorshipManager()`, `sponsorshipManager` struct, `ShowSponsorshipAppreciation()` |
| `pkg/config/remoteconfig/types/remote_config.go` | `RemoteConfig` interface, `RemoteConfigData`, `Message`, `Messages`, `Notifications`, `Ticker`, `Remote` structs |
| `pkg/config/remoteconfig/types/remote_config_storage.go` | `RemoteConfigStorage` interface |
| `pkg/config/remoteconfig/types/messages.go` | `Notification` struct with conditions |
| `pkg/config/remoteconfig/types/sponsorship.go` | `SponsorshipData`, `SponsorshipManager` interface, `GitHubSponsorship`, `InvoicedSponsorship`, `SponsorshipHistoryEntry` |
| `pkg/config/remoteconfig/types/addon.go` | `AddonData`, `Addon`, `FlexibleString` structs for addon registry |
| `pkg/config/remoteconfig/storage/file_storage.go` | `NewFileStorage()` implementing `RemoteConfigStorage` via gob-encoded local file |
| `pkg/config/remoteconfig/storage/addon_storage.go` | `AddonStorage` interface + `NewAddonFileStorage()` gob-backed implementation |
| `pkg/config/remoteconfig/storage/sponsorship_storage.go` | `SponsorshipStorage` interface + `NewSponsorshipFileStorage()` gob-backed implementation |
| `pkg/config/remoteconfig/downloader/jsonc_downloader.go` | `URLJSONCDownloader`, HTTP GET + JSONC parse |
| `pkg/config/state/state_manager.go` | `New(storage)` factory, `stateManager` struct |
| `pkg/config/state/types/state.go` | `State` interface: `Load()`, `Save()`, `Loaded()`, `Changed()`, `Get(key, entry)`, `Set(key, entry)` |
| `pkg/config/state/types/state_storage.go` | `StateStorage` interface: `Read() (RawState, error)`, `Write(RawState) error` |
| `pkg/config/state/storage/yaml/yaml_storage.go` | `YamlStorage` implementing `StateStorage` via YAML file |

## Key Types

### GlobalConfig (`pkg/globalconfig/global_config.go`)

```go
type GlobalConfig struct {
    DeveloperMode                    bool
    FailOnHookFailGlobal             bool
    InstrumentationOptIn             bool
    InstrumentationQueueSize         int
    InstrumentationReportingInterval time.Duration
    InstrumentationUser              string
    InternetDetectionTimeout         int64
    LastStartedVersion               string
    LetsEncryptEmail                 string
    Messages                         MessagesConfig
    MkcertCARoot                     string
    NoBindMounts                     bool
    NoTUI                            bool
    OmitContainersGlobal             []string
    OmitProjectNameByDefault         bool
    PerformanceMode                  configTypes.PerformanceMode
    ProjectTldGlobal                 string
    RemoteConfig                     RemoteConfig
    RequiredDockerComposeVersion     string
    Router                           string
    RouterBindAllInterfaces          bool
    RouterHTTPPort                   string
    RouterHTTPSPort                  string
    RouterMailpitHTTPPort            string
    RouterMailpitHTTPSPort           string
    RouterXHGuiHTTPPort              string
    RouterXHGuiHTTPSPort             string
    ShareDefaultProvider             string
    SimpleFormatting                 bool
    TableStyle                       string
    TraefikMonitorPort               string
    UseDockerComposeFromPath         bool
    UseHardenedImages                bool
    UseLetsEncrypt                   bool
    WSL2NoWindowsHostsMgt            bool
    WebEnvironment                   []string
    XdebugIDELocation                string
    XHProfMode                       configTypes.XHProfMode
    ProjectList                      map[string]*ProjectInfo // deprecated, migrated to project_list.yaml
}
```

### ProjectInfo (`pkg/globalconfig/global_config.go`)

```go
type ProjectInfo struct {
    AppRoot       string   `yaml:"approot"`
    UsedHostPorts []string `yaml:"used_host_ports,omitempty,flow"`
}
```

### RemoteConfig in globalconfig (`pkg/globalconfig/remote_config.go`)

```go
type RemoteConfig struct {
    UpdateInterval     int    `yaml:"update_interval,omitempty"`
    RemoteConfigURL    string `yaml:"remote_config_url,omitempty"`
    SponsorshipDataURL string `yaml:"sponsorship_data_url,omitempty"`
    AddonDataURL       string `yaml:"addon_data_url,omitempty"`
}
```

### MessagesConfig (`pkg/globalconfig/messages.go`)

```go
type MessagesConfig struct {
    TickerInterval int `yaml:"ticker_interval,omitempty"`
}
```

### config/types (`pkg/config/types/config_type.go`, `performance_mode.go`, `xhprof_mode.go`)

```go
// ConfigType distinguishes global vs project config contexts for validation
type ConfigType string
const (
    ConfigTypeGlobal  ConfigType = "global"
    ConfigTypeProject ConfigType = "project"
)

type PerformanceMode = string
const (
    PerformanceModeEmpty   PerformanceMode = ""
    PerformanceModeGlobal  PerformanceMode = "global"  // project-only: inherit from global
    PerformanceModeNone    PerformanceMode = "none"
    PerformanceModeMutagen PerformanceMode = "mutagen"
)

// ValidPerformanceModeOptions returns valid choices for the given ConfigType
func ValidPerformanceModeOptions(configType ConfigType) []PerformanceMode
func IsValidPerformanceMode(performanceMode string, configType ConfigType) bool
func CheckValidPerformanceMode(performanceMode string, configType ConfigType) error
func GetPerformanceModeDefault() PerformanceMode  // OS-specific via build tags

type XHProfMode = string
const (
    XHProfModeEmpty   XHProfMode = ""
    XHProfModePrepend XHProfMode = "prepend"
    XHProfModeXHGui   XHProfMode = "xhgui"
)
func IsValidXHProfMode(xhprofMode string, configType ConfigType) bool
// FlagXHProfModeDefault = "xhgui"
```

### State interface (`pkg/config/state/types/state.go`)

```go
type StateEntryKey = string
type StateEntry = any
type RawState = map[string]any

type State interface {
    Load() error
    Save() error
    Loaded() bool
    Changed() bool
    // Get deserializes key into stateEntry (must be a pointer)
    Get(key StateEntryKey, stateEntry StateEntry) error
    // Set serializes stateEntry under key; marks state changed
    Set(key StateEntryKey, stateEntry StateEntry) error
}

type StateStorage interface {
    Read() (RawState, error)
    Write(RawState) error
}
```

### RemoteConfig interface (`pkg/config/remoteconfig/types/remote_config.go`)

```go
type RemoteConfig interface {
    ShowNotifications()
    ShowTicker()
    ShowSponsorshipAppreciation()
}

type RemoteConfigData struct {
    UpdateInterval int      `json:"update-interval,omitempty"`
    Remote         Remote   `json:"remote"`
    Messages       Messages `json:"messages"`
}
```

### SponsorshipManager interface (`pkg/config/remoteconfig/types/sponsorship.go`)

```go
type SponsorshipManager interface {
    GetSponsorshipData() (*SponsorshipData, error)
    GetTotalMonthlyIncome() float64
    GetTotalSponsors() int
    IsDataStale() bool
}
```

### AddonData / Addon / FlexibleString (`pkg/config/remoteconfig/types/addon.go`)

```go
// FlexibleString unmarshals from JSON string, number, or null
type FlexibleString struct {
    Value string
    IsSet bool
}

type Addon struct {
    Title         string         `json:"title"`
    GitHubURL     string         `json:"github_url"`
    // ... description, user, repo, default_branch, tag_name (FlexibleString), etc.
    TagName       FlexibleString `json:"tag_name"`
}

type AddonData struct {
    UpdatedDateTime     time.Time `json:"updated_datetime"`
    TotalAddonsCount    int
    OfficialAddonsCount int
    ContribAddonsCount  int
    Addons              []Addon
}
```

## Interface Signatures

### Global Config CRUD (`pkg/globalconfig/global_config.go`)

```go
func New() GlobalConfig
func EnsureGlobalConfig()
func ReadGlobalConfig() error
func WriteGlobalConfig(config GlobalConfig) error
func ValidateGlobalConfig() error
func GetGlobalConfigPath() string
func GetGlobalDdevDir() string
func GetGlobalDdevDirLocation() string
func GetGlobalConfigYAML() ([]byte, error)
```

### Project List CRUD (`pkg/globalconfig/global_config.go`)

```go
func ReadProjectList() error
func WriteProjectList(projects map[string]*ProjectInfo) error
func GetProject(projectName string) *ProjectInfo
func SetProjectAppRoot(projectName string, appRoot string) error
func ReservePorts(projectName string, ports []string) error
func RemoveProjectInfo(projectName string) error
func GetGlobalProjectList() map[string]*ProjectInfo
func HostPostIsAllocated(port string) string
func CheckHostPortsAvailable(projectName string, ports []string) error
func GetFreePort(localIPAddr string) (string, error)
```

### Remote Config (`pkg/config/remoteconfig/`)

```go
// remote_config.go
func New(config *Config, stateManager statetypes.State, isInternetActive func() bool) types.RemoteConfig

// global.go — singletons (init-once pattern)
func InitGlobal(config Config, stateManager statetypes.State, isInternetActive func() bool) types.RemoteConfig
func GetGlobal() types.RemoteConfig
func InitGlobalSponsorship(localPath string, stateManager statetypes.State, isInternetActive func() bool, updateInterval int, url string) types.SponsorshipManager
func GetGlobalSponsorship() types.SponsorshipManager

// sponsorship.go
func NewSponsorshipManager(localPath string, stateManager statetypes.State, isInternetActive func() bool, updateInterval int, url string) types.SponsorshipManager

// config.go — Config struct
type Local struct {
    Path string
}
type Config struct {
    Local          Local
    URL            string
    UpdateInterval int
    TickerInterval int
}
```

### State Manager (`pkg/config/state/state_manager.go`)

```go
func New(storage types.StateStorage) types.State
// Returns a stateManager with lazy load; Get() calls Load() on first access
```

### GlobalConfig methods (`pkg/globalconfig/performance_mode.go`, `xhprof_mode.go`)

```go
func (c *GlobalConfig) GetPerformanceMode() types.PerformanceMode
func (c *GlobalConfig) SetPerformanceMode(performanceMode string) *GlobalConfig
func (c *GlobalConfig) IsMutagenEnabled() bool

func (c *GlobalConfig) GetXHProfMode() types.XHProfMode
func (c *GlobalConfig) SetXHProfMode(xhprofMode string) *GlobalConfig
```

### Validation (`pkg/globalconfig/values.go`, `styles.go`, `types/types.go`)

```go
func IsValidOmitContainers(containerList []string) bool
func GetValidOmitContainers() []string
func IsValidXdebugIDELocation(loc string) bool
func IsValidTableStyle(style string) bool
func GetTableStyleByName(name string) table.Style
func IsValidRouterType(router RouterType) bool
func GetValidRouterTypes() []RouterType
```

## Data Flow

### Global Config Loading

```text
EnsureGlobalConfig()                          // pkg/globalconfig/global_config.go
  ├─ DdevGlobalConfig = New()                 // sets defaults from nodeps.* constants
  ├─ DdevProjectList = make(map[string]*ProjectInfo)
  ├─ ReadGlobalConfig()
  │    ├─ os.Stat(GetGlobalConfigPath())      // ~/.ddev/global_config.yaml
  │    │    └─ if missing: WriteGlobalConfig(New()) to create file
  │    ├─ os.ReadFile() + yaml.Unmarshal(&DdevGlobalConfig)
  │    ├─ Post-unmarshal fixups:
  │    │    ├─ readCAROOT() if MkcertCARoot invalid
  │    │    ├─ Clamp InternetDetectionTimeout >= default (3000ms)
  │    │    ├─ Fill empty port fields with nodeps.DdevDefault* constants
  │    │    ├─ Remove deprecated "dba" from OmitContainersGlobal
  │    │    ├─ Set default RemoteConfig URLs if empty
  │    │    └─ Set ProjectTldGlobal default if empty
  │    └─ ValidateGlobalConfig()
  │         ├─ IsValidOmitContainers()
  │         ├─ IsValidRouterType() — forces Traefik if invalid
  │         ├─ IsValidTableStyle() — forces "default" if invalid
  │         └─ IsValidXdebugIDELocation()
  ├─ ReadProjectList()
  │    ├─ if project_list.yaml missing + DdevGlobalConfig.ProjectList populated:
  │    │    migrate from global_config.yaml to project_list.yaml
  │    ├─ yaml.Unmarshal into DdevProjectList
  │    └─ sanitize: remove entries with nil or empty AppRoot
  └─ if SimpleFormatting: set NO_COLOR=1
```

### Remote Config Loading

```text
InitGlobal(config, stateManager, isInternetActive)  // pkg/config/remoteconfig/global.go
  └─ remoteconfig.New(&config, stateManager, isInternetActive)
       ├─ newState(stateManager)              // wraps state.State with stateEntry tracking
       ├─ storage.NewFileStorage(localFile)   // ~/.ddev/.remote-config (gob encoded)
       ├─ loadFromLocalStorage()              // reads cached data from FileStorage
       ├─ downloader.NewURLJSONCDownloader(url)
       └─ updateFromRemote()
            ├─ check isInternetActive()
            ├─ check state.UpdatedAt + getUpdateInterval()
            ├─ urlDownloader.Download(ctx, &remoteConfig)  // HTTP GET + JSONC parse
            ├─ fileStorage.Write(remoteConfig)              // cache locally
            └─ state.save()                                 // persist stateEntry via stateManager.Set + Save()

// GetGlobal() returns the already-initialized singleton (nil if InitGlobal not called)
// Sponsorship is initialized separately via InitGlobalSponsorship()
```

## Configuration Precedence

The merge order during `ReadGlobalConfig()` is:

1. **Compiled defaults** — `New()` sets all fields from `nodeps.*` and `versionconstants.*` constants
2. **File on disk** — `yaml.Unmarshal` into the already-defaulted struct (file values overwrite defaults)
3. **Post-unmarshal fixups** — empty/invalid fields reset to defaults (ports, TLD, remote URLs)
4. **Validation** — `ValidateGlobalConfig()` corrects invalid router type and table style in-place
5. **Remote config** — fetched separately via `GetRemoteConfig()`, does not mutate `GlobalConfig`

For `PerformanceMode`, the OS-specific default is resolved via build tags:

- `pkg/config/types/performance_mode_linux.go`: `GetPerformanceModeDefault()` → `PerformanceModeNone`
- `pkg/config/types/performance_mode_darwin_win.go`: `GetPerformanceModeDefault()` → `PerformanceModeMutagen`

`PerformanceModeGlobal` (`"global"`) is a project-level sentinel meaning "inherit from global config".
It is only valid for `ConfigTypeProject`, not `ConfigTypeGlobal`.

`SetPerformanceMode()` and `SetXHProfMode()` are fluent setters on `GlobalConfig` that validate with `IsValidPerformanceMode()`/`IsValidXHProfMode()` before writing. Invalid values are silently ignored.

Project-level config (in `.ddev/config.yaml`, handled by `pkg/ddevapp/`) can override global
fields like `PerformanceMode`, `OmitContainers`, router ports, and `WebEnvironment`.

## Remote Config

### Architecture

Remote config is a separate subsystem under `pkg/config/remoteconfig/` that downloads
DDEV project announcements, notifications, ticker messages, and sponsorship data from GitHub-hosted
JSONC files.

### Default URLs (`pkg/globalconfig/remote_config.go`)

- `DefaultRemoteConfigURL`: `https://raw.githubusercontent.com/ddev/remote-config/main/remote-config.jsonc`
- `DefaultSponsorshipDataURL`: `https://ddev.com/s/sponsorship-data.json`
- `DefaultAddonDataURL`: `https://addons.ddev.com/addons.json`

### Update Interval Resolution (`remoteConfig.getUpdateInterval()`)

Priority: global config `UpdateInterval` > remote-served `UpdateInterval` > hardcoded 10 hours.

### Storage Layer

- `FileStorage` (`pkg/config/remoteconfig/storage/file_storage.go`): reads/writes `RemoteConfigData` via gob encoding; implements `RemoteConfigStorage`.
- `AddonStorage` (`pkg/config/remoteconfig/storage/addon_storage.go`): typed interface `Read() (*AddonData, error)` / `Write(*AddonData) error`; gob-backed. Constructor: `NewAddonFileStorage(fileName)`. Cache is read once per process.
- `SponsorshipStorage` (`pkg/config/remoteconfig/storage/sponsorship_storage.go`): typed interface `Read() (*SponsorshipData, error)` / `Write(*SponsorshipData) error`; gob-backed. Constructor: `NewSponsorshipFileStorage(fileName)`. Cache is read once per process.

`AddonStorage` and `SponsorshipStorage` have their own typed interfaces (not `RemoteConfigStorage`). All gob-backed files use the same lazy-load/once-per-run pattern.

## State Management

### Interface (`pkg/config/state/types/`)

```go
type StateEntryKey = string
type StateEntry = any
type RawState = map[string]any

type State interface {
    Load() error
    Save() error
    Loaded() bool
    Changed() bool
    Get(key StateEntryKey, stateEntry StateEntry) error  // stateEntry must be a pointer
    Set(key StateEntryKey, stateEntry StateEntry) error
}

type StateStorage interface {
    Read() (RawState, error)
    Write(RawState) error
}
```

### Implementation

- `stateManager` (`pkg/config/state/state_manager.go`): in-memory `RawState` with lazy load.
  `Get()` calls `Load()` on first access; `Set()` marks changed but does not write-through (call `Save()` explicitly).
- `YamlStorage` (`pkg/config/state/storage/yaml/yaml_storage.go`): persists state to YAML at `GetGlobalDdevDir()/.state.yaml`.
- `New(storage)` wires any `StateStorage` into `stateManager`.

### Remote Config State Wrapper (`pkg/config/remoteconfig/state.go`)

The `state` struct wraps `statetypes.State` and persists a `stateEntry` struct under the `"remote_config"` key:

```go
type stateEntry struct {
    UpdatedAt          time.Time `yaml:"updated_at"`
    LastNotificationAt time.Time `yaml:"last_notification_at"`
    LastTickerAt       time.Time `yaml:"last_ticker_at"`
    LastTickerMessage  int       `yaml:"last_ticker_message"`
    LastSponsorshipAt  time.Time `yaml:"last_sponsorship_at"`
}
```

Serialized/deserialized via `stateManager.Get(stateKey, &stateEntry)` / `stateManager.Set(stateKey, stateEntry)`.

## Edge Cases

### Missing Config File

`ReadGlobalConfig()` handles a missing `global_config.yaml` by calling `WriteGlobalConfig(New())`
to create it with defaults. Running as root (uid 0) skips file creation entirely and returns
an empty config with a warning.

### Project List Migration

`ReadProjectList()` detects when `project_list.yaml` is missing but `DdevGlobalConfig.ProjectList`
(the deprecated `project_info` YAML field) is populated. It writes to `project_list.yaml`, then
clears `ProjectList` from `GlobalConfig` and rewrites `global_config.yaml`.

### Global Dir Location Resolution (`GetGlobalDdevDirLocation()`)

Resolution order: `$XDG_CONFIG_HOME/ddev` > `~/.config/ddev` (Linux only, if `~/.ddev` absent) >
`~/.ddev`. `CheckForMultipleGlobalDdevDirs()` warns if both locations exist simultaneously.

### Stale `config.yaml` Guard

`GetGlobalDdevDir()` removes any `config.yaml` found in the global dir (that file belongs in
project `.ddev/` dirs only).

### Remote Config Offline Fallback

If `isInternetActive()` returns false, `updateFromRemote()` is a no-op and the locally cached
`.remote-config` file is used. If both remote and local are unavailable, an empty
`RemoteConfigData{}` is used.

### Write Stripping

`WriteGlobalConfig()` strips default values before writing: `RequiredDockerComposeVersion` if
matching the compiled default, remote config URLs if matching defaults, and `Router` (always
cleared since only Traefik is valid). Additionally, if `PerformanceMode` is empty the effective
OS default is written out; if `XHProfMode` is empty `FlagXHProfModeDefault` (`"xhgui"`) is written.
This ensures the file always reflects the effective values rather than empty strings.
