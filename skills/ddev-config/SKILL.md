---
name: ddev:config
description: Use when modifying pkg/config/ or pkg/globalconfig/ — global config, layered config, remote config, state management
---

## Scope

This skill covers global configuration, remote configuration, and state management across two package trees.

### Key Files

- `pkg/globalconfig/global_config.go` -- GlobalConfig struct, read/write/validate, project list
- `pkg/globalconfig/remote_config.go` -- RemoteConfig struct and default URLs
- `pkg/globalconfig/values.go` -- Default value constants
- `pkg/globalconfig/performance_mode.go` -- Performance mode resolution
- `pkg/globalconfig/styles.go` -- Table styling presets
- `pkg/globalconfig/messages.go` -- MessagesConfig for notification control
- `pkg/config/remoteconfig/remote_config.go` -- Remote config fetcher and updater
- `pkg/config/remoteconfig/messages.go` -- Notification/ticker display logic with conditions
- `pkg/config/remoteconfig/sponsorship.go` -- Sponsorship data management
- `pkg/config/remoteconfig/global.go` -- Global singleton accessors
- `pkg/config/remoteconfig/state.go` -- Remote config state persistence
- `pkg/config/state/state_manager.go` -- Generic state manager with in-memory cache
- `pkg/config/state/types/state.go` -- State interface definition
- `pkg/config/state/types/state_storage.go` -- StateStorage interface definition
- `pkg/config/state/storage/yaml/yaml_storage.go` -- YAML file storage backend
- `pkg/config/types/performance_mode.go` -- PerformanceMode type and platform-specific defaults

## Key Interfaces

### GlobalConfig Struct

```go
type GlobalConfig struct {
    DeveloperMode                    bool                        `yaml:"developer_mode,omitempty"`
    FailOnHookFailGlobal             bool                        `yaml:"fail_on_hook_fail"`
    InstrumentationOptIn             bool                        `yaml:"instrumentation_opt_in"`
    InstrumentationQueueSize         int                         `yaml:"instrumentation_queue_size,omitempty"`
    InstrumentationReportingInterval time.Duration               `yaml:"instrumentation_reporting_interval,omitempty"`
    InstrumentationUser              string                      `yaml:"instrumentation_user,omitempty"`
    InternetDetectionTimeout         int64                       `yaml:"internet_detection_timeout_ms"`
    LastStartedVersion               string                      `yaml:"last_started_version"`
    LetsEncryptEmail                 string                      `yaml:"letsencrypt_email"`
    Messages                         MessagesConfig              `yaml:"messages,omitempty"`
    MkcertCARoot                     string                      `yaml:"mkcert_caroot"`
    NoBindMounts                     bool                        `yaml:"no_bind_mounts"`
    NoTUI                            bool                        `yaml:"no_tui,omitempty"`
    OmitContainersGlobal             []string                    `yaml:"omit_containers,flow"`
    OmitProjectNameByDefault         bool                        `yaml:"omit_project_name_by_default,omitempty"`
    PerformanceMode                  configTypes.PerformanceMode `yaml:"performance_mode"`
    ProjectTldGlobal                 string                      `yaml:"project_tld"`
    RemoteConfig                     RemoteConfig                `yaml:"remote_config,omitempty"`
    RequiredDockerComposeVersion     string                      `yaml:"required_docker_compose_version,omitempty"`
    Router                           string                      `yaml:"router,omitempty"`
    RouterBindAllInterfaces          bool                        `yaml:"router_bind_all_interfaces"`
    RouterHTTPPort                   string                      `yaml:"router_http_port"`
    RouterHTTPSPort                  string                      `yaml:"router_https_port"`
    RouterMailpitHTTPPort            string                      `yaml:"mailpit_http_port,omitempty"`
    RouterMailpitHTTPSPort           string                      `yaml:"mailpit_https_port,omitempty"`
    RouterXHGuiHTTPPort              string                      `yaml:"xhgui_http_port,omitempty"`
    RouterXHGuiHTTPSPort             string                      `yaml:"xhgui_https_port,omitempty"`
    ShareDefaultProvider             string                      `yaml:"share_default_provider,omitempty"`
    SimpleFormatting                 bool                        `yaml:"simple_formatting"`
    TableStyle                       string                      `yaml:"table_style"`
    TraefikMonitorPort               string                      `yaml:"traefik_monitor_port,omitempty"`
    UseDockerComposeFromPath         bool                        `yaml:"use_docker_compose_from_path,omitempty"`
    UseHardenedImages                bool                        `yaml:"use_hardened_images"`
    UseLetsEncrypt                   bool                        `yaml:"use_letsencrypt"`
    WSL2NoWindowsHostsMgt            bool                        `yaml:"wsl2_no_windows_hosts_mgt"`
    WebEnvironment                   []string                    `yaml:"web_environment"`
    XdebugIDELocation                string                      `yaml:"xdebug_ide_location"`
    XHProfMode                       configTypes.XHProfMode      `yaml:"xhprof_mode,omitempty"`
}
```

### Global Variables and Constants

```go
const DdevGlobalConfigName = "global_config.yaml"
const DdevProjectListFileName = "project_list.yaml"

var DdevGlobalConfig GlobalConfig
var DdevProjectList map[string]*ProjectInfo

type ProjectInfo struct {
    AppRoot       string   `yaml:"approot"`
    UsedHostPorts []string `yaml:"used_host_ports,omitempty,flow"`
}
```

### GlobalConfig Functions

```go
func ReadGlobalConfig() error
func WriteGlobalConfig(config GlobalConfig) error
func ValidateGlobalConfig() error
func GetGlobalDdevDir() string
func GetGlobalDdevDirLocation() string
func IsValidOmitContainers(containerList []string) bool
func GetFreePort(localIPAddr string) (int, error)
func ReservePorts(projectName string, ports []string) error
func IsProjectConfigured(project string) bool
```

### RemoteConfig (in globalconfig)

```go
const DefaultRemoteConfigURL = "https://raw.githubusercontent.com/ddev/remote-config/main/remote-config.jsonc"
const DefaultSponsorshipDataURL = "https://ddev.com/s/sponsorship-data.json"
const DefaultAddonDataURL = "https://addons.ddev.com/addons.json"

type RemoteConfig struct {
    UpdateInterval     int    `yaml:"update_interval,omitempty"`
    RemoteConfigURL    string `yaml:"remote_config_url,omitempty"`
    SponsorshipDataURL string `yaml:"sponsorship_data_url,omitempty"`
    AddonDataURL       string `yaml:"addon_data_url,omitempty"`
}
```

### Remote Config Package (`pkg/config/remoteconfig/`)

```go
// Singleton management
func InitGlobal(config Config, stateManager statetypes.State, isInternetActive func() bool) types.RemoteConfig
func GetGlobal() types.RemoteConfig
func InitGlobalSponsorship(localPath string, stateManager statetypes.State,
    isInternetActive func() bool, updateInterval int, url string) types.SponsorshipManager
func GetGlobalSponsorship() types.SponsorshipManager

// Factory
func New(config *Config, stateManager statetypes.State, isInternetActive func() bool) types.RemoteConfig

// Condition system for message filtering
func AddCondition(name, description string, conditionFunc func() bool)
func ListConditions() (conditions map[string]string)
```

### RemoteConfig Interface

```go
type RemoteConfig interface {
    ShowNotifications()
    ShowTicker()
    ShowSponsorshipAppreciation()
}
```

### Remote Config Data Types

```go
type RemoteConfigData struct {
    UpdateInterval int      `json:"update-interval,omitempty"`
    Remote         Remote   `json:"remote,omitempty"`
    Messages       Messages `json:"messages,omitempty"`
}

type Messages struct {
    Notifications Notifications `json:"notifications"`
    Ticker        Ticker        `json:"ticker"`
}

type Notifications struct {
    Interval int       `json:"interval"`
    Infos    []Message `json:"infos"`
    Warnings []Message `json:"warnings"`
}

type Message struct {
    Message    string   `json:"message"`
    Title      string   `json:"title,omitempty"`
    Conditions []string `json:"conditions,omitempty"`
    Versions   string   `json:"versions,omitempty"`
}

type RemoteConfigStorage interface {
    Read() (RemoteConfigData, error)
    Write(RemoteConfigData) error
}
```

### State Management (`pkg/config/state/`)

```go
// Factory
func New(storage types.StateStorage) types.State

// State interface
type State interface {
    Load() error
    Save() error
    Loaded() bool
    Changed() bool
    Get(StateEntryKey, StateEntry) error  // StateEntry must be a pointer
    Set(StateEntryKey, StateEntry) error  // StateEntry must NOT be a pointer
}

// Type aliases
type StateEntry = interface{}
type StateEntryKey = string
type RawState = map[string]any

// Storage interface
type StateStorage interface {
    Read() (RawState, error)   // Returns empty RawState (not error) if no state exists
    Write(RawState) error
    TagName() string           // Returns struct tag name (e.g., "yaml")
}
```

### State Manager Internals

```go
type stateManager struct {
    storage      types.StateStorage
    state        types.RawState
    stateChanged bool
    stateLoaded  bool
}
```

## Architecture

### Global Config Read/Write Flow

```
ReadGlobalConfig()
  -> Reads ~/.ddev/global_config.yaml via yaml.Unmarshal into DdevGlobalConfig
  -> Applies defaults for empty fields (ports, TLD, remote config URLs)
  -> Removes deprecated values (e.g., "dba" from OmitContainers)
  -> Calls ValidateGlobalConfig()

WriteGlobalConfig(config)
  -> Calls ValidateGlobalConfig()
  -> Strips default values before writing (e.g., default compose version, default URLs)
  -> Clears Router field (only one router exists now)
  -> Marshals to YAML and writes to ~/.ddev/global_config.yaml
  -> Appends image version comments
```

### Remote Config Update Flow

```
InitGlobal(config, stateManager, isInternetActive)
  -> Creates remoteConfig instance
  -> loadFromLocalStorage() -- reads cached JSONC from disk
  -> If internet active and stale: updateFromRemote()
    -> Downloads from RemoteConfigURL via downloader.JSONCDownloader
    -> Writes to local storage via RemoteConfigStorage
    -> Updates state timestamps via stateManager

ShowNotifications() / ShowTicker()
  -> Checks interval against last shown time (from state)
  -> Filters messages by conditions (registered via AddCondition())
  -> Filters by version constraints
  -> Displays via table writer with configured style
```

### State Management Flow

```
state := yaml.NewState(filepath.Join("config", "state.yml"))

state.Get("my_key", &myStruct)
  -> ensureLoaded() -> storage.Read() if not loaded
  -> mapstructure decode from RawState into typed struct

state.Set("my_key", myStruct)
  -> ensureLoaded()
  -> Updates in-memory RawState
  -> Sets stateChanged = true

state.Save()
  -> If stateChanged: storage.Write(state)
  -> Resets stateChanged = false
```

### Config Layering (Project Context)

Global config values flow into project config via `DdevApp`:

1. `ReadGlobalConfig()` populates `globalconfig.DdevGlobalConfig`
2. `DdevApp.ReadConfig()` loads `.ddev/config.yaml`
3. Fields like `FailOnHookFailGlobal`, `OmitContainersGlobal` are copied from global to app
4. Project `config.*.yaml` overrides are merged via `mergeAdditionalConfigIntoApp()`

## Common Mistakes

- **Writing defaults to config file**: `WriteGlobalConfig` strips default values before writing. If you add a new field with a default, add the stripping logic or the config file will contain unnecessary entries.
- **Forgetting `ValidateGlobalConfig()`**: Both read and write call validation. New fields with constraints must be added to the validation function.
- **State.Get requires pointer, State.Set requires non-pointer**: `state.Get("key", &myStruct)` vs `state.Set("key", myStruct)`. Passing wrong pointer semantics causes silent failures.
- **StateStorage.Read must not error on missing file**: The contract requires returning an empty `RawState` (not an error) when no state file exists yet.
- **Remote config URL defaults**: Empty URL fields get populated with defaults during `ReadGlobalConfig()`. Custom URLs must be set after reading, or they will be overwritten.
- **Port fields are strings**: `RouterHTTPPort`, `RouterHTTPSPort`, `TraefikMonitorPort` are all strings despite representing numeric ports.

## Testing Patterns

```go
func TestGlobalConfig(t *testing.T) {
    // Save and restore global state
    origConfig := globalconfig.DdevGlobalConfig
    t.Cleanup(func() {
        globalconfig.DdevGlobalConfig = origConfig
        err := globalconfig.WriteGlobalConfig(origConfig)
        require.NoError(t, err)
    })

    // Modify and test
    globalconfig.DdevGlobalConfig.RouterHTTPPort = "8080"
    err := globalconfig.WriteGlobalConfig(globalconfig.DdevGlobalConfig)
    require.NoError(t, err)

    err = globalconfig.ReadGlobalConfig()
    require.NoError(t, err)
    require.Equal(t, "8080", globalconfig.DdevGlobalConfig.RouterHTTPPort)
}

// State manager tests use testify suites
func TestStateTestSuite(t *testing.T) {
    suite.Run(t, new(StateTestSuite))
}
```

- Always save/restore `DdevGlobalConfig` in tests
- Use `require` over `assert` for all assertions
- State tests use testify suite pattern
- Set `DDEV_NO_INSTRUMENTATION=true`

## Related Specs

- `pkg/ddevapp/config.go` -- Consumes GlobalConfig for project defaults (see `ddev:core-app` skill)
- `pkg/ddevapp/router.go` -- Reads router ports and TLD from GlobalConfig
- `pkg/config/remoteconfig/storage/` -- File-based storage implementations
- `pkg/config/remoteconfig/downloader/` -- JSONC download logic
- `pkg/config/remoteconfig/types/` -- All interface and data type definitions
