# pkg/ddevapp — Subsystem Spec

> Generated for AI consumption. Source: `pkg/ddevapp/` in ddev/ddev.

## 1. File Map

| File | Purpose |
|---|---|
| `pkg/ddevapp/ddevapp.go` | Core `DdevApp` struct, `Init`, `Start`, `Stop`, `Describe`, `Restart`, container management |
| `pkg/ddevapp/config.go` | `NewApp`, `ReadConfig`, `WriteConfig`, `ValidateConfig`, config defaults and migrations |
| `pkg/ddevapp/apptypes.go` | `appTypeFuncs` struct, `appTypeMatrix` map, `DetectAppType`, per-CMS hooks |
| `pkg/ddevapp/addons.go` | Add-on install/remove lifecycle, `InstallDesc`, `AddonManifest`, registry interaction |
| `pkg/ddevapp/router.go` | `StartDdevRouter`, Traefik compose generation, port allocation, hostname resolution |
| `pkg/ddevapp/mutagen.go` | Mutagen sync create/pause/terminate/flush, volume ownership, diagnostics |
| `pkg/ddevapp/backdrop.go` | Backdrop CMS settings creator and detection |
| `pkg/ddevapp/craftcms.go` | Craft CMS settings creator and detection |
| `pkg/ddevapp/drupal.go` | Drupal (6/7/8/9/10/11) settings and detection |
| `pkg/ddevapp/laravel.go` | Laravel settings and detection |
| `pkg/ddevapp/magento.go` | Magento 1/2 settings and detection |
| `pkg/ddevapp/python.go` | Python/Django/Flask settings and detection |
| `pkg/ddevapp/typo3.go` | TYPO3 settings and detection |
| `pkg/ddevapp/wordpress.go` | WordPress settings and detection |
| `pkg/ddevapp/compose_yaml.go` | Docker Compose YAML rendering and template logic |
| `pkg/ddevapp/providerDefault.go` | Default hosting provider implementation |
| `pkg/ddevapp/ports.go` | Port reservation and collision detection |
| `pkg/ddevapp/traefik.go` | Traefik cert generation and per-project config |
| `pkg/ddevapp/hooks.go` | Hook (pre-start, post-start, etc.) processing |

## 2. Key Types

### DdevApp (pkg/ddevapp/ddevapp.go)

Primary struct representing a DDEV project. Fields map directly to `.ddev/config.yaml`
via `yaml:` struct tags. Fields tagged `yaml:"-"` are computed at runtime.

```go
type DdevApp struct {
    // Identity and paths
    Name      string `yaml:"name,omitempty"`
    Type      string `yaml:"type"`           // e.g. "php", "drupal11", "laravel"
    AppRoot   string `yaml:"-"`              // computed: absolute path to project root
    Docroot   string `yaml:"docroot"`
    ConfigPath string `yaml:"-"`             // computed: path to .ddev/config.yaml
    DataDir    string `yaml:"-"`             // computed: .ddev data directory

    // Runtime configuration
    PHPVersion        string `yaml:"php_version"`
    WebserverType     string `yaml:"webserver_type"`
    WebImage          string `yaml:"webimage,omitempty"`
    Database          DatabaseDesc `yaml:"database"`
    NodeJSVersion     string `yaml:"nodejs_version,omitempty"`
    ComposerVersion   string `yaml:"composer_version"`
    ComposerRoot      string `yaml:"composer_root,omitempty"`
    CorepackEnable    bool   `yaml:"corepack_enable"`

    // Networking
    RouterHTTPPort    string   `yaml:"router_http_port,omitempty"`
    RouterHTTPSPort   string   `yaml:"router_https_port,omitempty"`
    AdditionalHostnames []string `yaml:"additional_hostnames"`
    AdditionalFQDNs    []string `yaml:"additional_fqdns"`
    HostDBPort         string `yaml:"host_db_port,omitempty"`
    HostWebserverPort  string `yaml:"host_webserver_port,omitempty"`
    HostHTTPSPort      string `yaml:"host_https_port,omitempty"`
    BindAllInterfaces  bool   `yaml:"bind_all_interfaces,omitempty"`
    ProjectTLD         string `yaml:"project_tld,omitempty"`
    UseDNSWhenPossible bool   `yaml:"use_dns_when_possible"`

    // Performance and sync
    PerformanceMode types.PerformanceMode `yaml:"performance_mode,omitempty"`
    NoProjectMount  bool `yaml:"no_project_mount,omitempty"`

    // Extensibility
    Hooks             map[string][]YAMLTask `yaml:"hooks,omitempty"`
    WebExtraExposedPorts []WebExposedPort   `yaml:"web_extra_exposed_ports,omitempty"`
    WebExtraDaemons      []WebExtraDaemon   `yaml:"web_extra_daemons,omitempty"`
    WebImageExtraPackages []string          `yaml:"webimage_extra_packages,omitempty,flow"`
    DBImageExtraPackages  []string          `yaml:"dbimage_extra_packages,omitempty,flow"`
    WebEnvironment        []string          `yaml:"web_environment"`
    OmitContainers        []string          `yaml:"omit_containers,omitempty,flow"`
    UploadDirs            []string          `yaml:"upload_dirs,omitempty"`
    WorkingDir            map[string]string `yaml:"working_dir,omitempty"`

    // Settings management
    SiteSettingsPath     string `yaml:"-"`
    SiteDdevSettingsFile string `yaml:"-"`
    DisableSettingsManagement bool `yaml:"disable_settings_management,omitempty"`

    // Constraints and metadata
    DdevVersionConstraint string `yaml:"ddev_version_constraint,omitempty"`
    OverrideConfig        bool   `yaml:"override_config,omitempty"`

    // Internal
    ProviderInstance *Provider     `yaml:"-"`
    ComposeYaml      []byte       `yaml:"-"`
    FailOnHookFail   bool         `yaml:"fail_on_hook_fail,omitempty"`
    FailOnHookFailGlobal bool     `yaml:"-"`
    OmitContainersGlobal []string `yaml:"-"`
}
```

### DatabaseDesc (pkg/ddevapp/ddevapp.go)

```go
type DatabaseDesc struct {
    Type    string `yaml:"type"`    // "mariadb", "mysql", "postgres"
    Version string `yaml:"version"` // e.g. "10.11", "8.0", "16"
}
var DatabaseDefault = DatabaseDesc{nodeps.MariaDB, nodeps.MariaDBDefaultVersion}
```

### WebExposedPort and WebExtraDaemon (pkg/ddevapp/ddevapp.go)

```go
type WebExposedPort struct {
    Name             string `yaml:"name"`
    WebContainerPort int    `yaml:"container_port"`
    HTTPPort         int    `yaml:"http_port"`
    HTTPSPort        int    `yaml:"https_port"`
}

type WebExtraDaemon struct {
    Name      string `yaml:"name"`
    Command   string `yaml:"command"`
    Directory string `yaml:"directory"`
}
```

### Site Status Constants (pkg/ddevapp/ddevapp.go)

```go
const (
    SiteRunning       = "running"
    SiteStarting      = "starting"
    SiteStopped       = "stopped"
    SiteDirMissing    = "project directory missing"
    SiteConfigMissing = ".ddev/config.yaml missing"
    SitePaused        = "paused"
    SiteUnhealthy     = "unhealthy"
)
```

## 3. Interface Signatures

### Configuration (pkg/ddevapp/config.go)

```go
func NewApp(appRoot string, includeOverrides bool) (*DdevApp, error)
func (app *DdevApp) ReadConfig(includeOverrides bool) ([]string, error)
func (app *DdevApp) LoadConfigYamlFile(filePath string) error
func (app *DdevApp) WriteConfig() error
func (app *DdevApp) ValidateConfig() error
```

### Lifecycle (pkg/ddevapp/ddevapp.go)

```go
func (app *DdevApp) Init(basePath string) error
func (app *DdevApp) Start() error
func (app *DdevApp) Stop(removeData bool, createSnapshot bool) error
func (app *DdevApp) Restart() error
func (app *DdevApp) Describe(short bool) (map[string]interface{}, error)
func (app *DdevApp) StartAndWait(extraSleep int) error
func (app *DdevApp) StartAppIfNotRunning() error
func (app *DdevApp) StartOptionalProfiles(profiles []string) error
```

### Router (pkg/ddevapp/router.go)

```go
func StartDdevRouter() error
func IsRouterDisabled(app *DdevApp) bool
func RemoveRouterContainer() error
func FindDdevRouter() (*container.Summary, error)
func CheckRouterPorts(activeApps []*DdevApp) error
func AllocateAvailablePortForRouter(start, upTo int) (int, bool)
func GetAvailableRouterPort(proposedPort string, minPort, maxPort int) (string, string, bool)
func generateRouterCompose(activeApps []*DdevApp) (string, error)
func determineRouterHostnames(activeApps []*DdevApp) []string
func determineRouterPorts(activeApps []*DdevApp) []string
```

### Mutagen (pkg/ddevapp/mutagen.go)

```go
func CreateOrResumeMutagenSync(app *DdevApp) error
func SyncAndPauseMutagenSession(app *DdevApp) error
func TerminateMutagenSync(app *DdevApp) error
func PauseMutagenSync(app *DdevApp) error
func ResumeMutagenSync(app *DdevApp) error
func MutagenReset(app *DdevApp) error
func SetMutagenVolumeOwnership(app *DdevApp) error
func MutagenSyncName(name string) string
func GetMutagenVolumeName(app *DdevApp) string
func (app *DdevApp) IsMutagenEnabled() bool
func (app *DdevApp) MutagenStatus() (status string, shortResult string, mapResult map[string]interface{}, err error)
func (app *DdevApp) MutagenSyncFlush() error
func (app *DdevApp) GenerateMutagenYml() error
func CheckMutagenVolumeSyncCompatibility(app *DdevApp) (ok bool, volumeExists bool, info string)
func IsMutagenVolumeMounted(app *DdevApp) (bool, error)
func DiagnoseMutagenConfiguration(app *DdevApp) MutagenDiagnosticResult
```

### Addons (pkg/ddevapp/addons.go)

```go
func GetInstalledAddons(app *DdevApp) []AddonManifest
func GetInstalledAddonNames(app *DdevApp) []string
func InstallAddonFromGitHub(app *DdevApp, addonName, requestedVersion string, verbose bool) error
func InstallAddonFromDirectory(app *DdevApp, extractedDir, repository, version string, verbose bool) error
func InstallAddonFromTarball(app *DdevApp, tarballURL, downloadedRelease, repository string, verbose bool) error
func RemoveAddon(app *DdevApp, addonName string, verbose bool, skipRemovalActions bool) error
func ProcessAddonAction(action string, installDesc InstallDesc, app *DdevApp, verbose bool) error
func ListAvailableAddonsFromRegistry() ([]types.Addon, error)
```

## 4. Data Flow

### Project Lifecycle: config -> init -> start -> describe -> stop

#### Phase 1: Configuration (`ddev config`)

1. `NewApp(appRoot, includeOverrides)` in `config.go` creates `DdevApp{}` with defaults
2. Sets defaults: `PHPVersion=nodeps.PHPDefault`, `WebserverType=nodeps.WebserverDefault`,
   `Database=DatabaseDefault`
3. If `.ddev/config.yaml` exists, calls `app.ReadConfig(includeOverrides)`
4. `ReadConfig` calls `app.LoadConfigYamlFile(app.ConfigPath)` for base config
5. If `includeOverrides=true`, globs `config.*.y*ml` files and loads each in order
6. `LoadConfigYamlFile` reads file, calls `validateHookYAML(source)`, then
   `yaml.Unmarshal(source, app)`
7. `WriteConfig()` copies the app struct, strips default values, calls
   `yaml.Marshal(appcopy)`, appends hook comments, writes to `config.yaml`
8. `WriteConfig` also calls `app.UpdateGlobalProjectList()` to reserve ports

#### Phase 2: Init (`ddev start` pre-container)

1. `app.Init(basePath)` calls `NewApp(basePath, true)` to load config
2. Calls `newApp.ValidateConfig()` which checks: version constraint,
   project name, docroot, hostnames, app type, PHP version, webserver type,
   database type/version, `WebExtraExposedPorts` uniqueness, timezone
3. Checks for existing web containers to detect conflicts

#### Phase 3: Start (`ddev start`)

1. `app.Start()` at line 1489 in `ddevapp.go`
2. Pre-flight checks: Mutagen+hardened incompatibility, Docker Rootless bind mount
   check, internet connectivity
3. Sets port values: `app.RouterHTTPPort = app.GetPrimaryRouterHTTPPort()` etc.
4. Removes Docker network duplicates: `dockerutil.RemoveNetworkDuplicates()`
5. Checks Docker Compose binary: `dockerutil.CheckDockerCompose()`
6. Runs `app.ProcessHooks("pre-start")`
7. Calls `app.WriteDockerComposeYAML()` to render compose files
8. Pulls base images: `PullBaseContainerImages(additionalImages, false)`
9. Validates DB type matches existing volume: `app.GetExistingDBType()`
10. Creates settings files: `app.CreateSettingsFile()`
11. Writes compose YAML again (after settings), adds host entries
12. Builds project images with `app.composeBuild()` (with retry up to
    `composeBuildMaxRetries=3` for BuildKit race conditions)
13. Runs `dockerutil.ComposeCmd` with `up -d` against the full rendered YAML
14. Configures Traefik TLS certs: `configureTraefikForApp(app)`
15. If Mutagen enabled: `app.GenerateMutagenYml()`, checks volume compatibility
    via `CheckMutagenVolumeSyncCompatibility`, calls `CreateOrResumeMutagenSync(app)`,
    then `app.MutagenSyncFlush()`
16. Executes `/start.sh` inside web container via `app.Exec(&ExecOpts{...})`
17. If router not disabled: `StartDdevRouter()`
18. Waits for all containers healthy: `app.WaitByLabels(waitLabels)`
19. Starts `WebExtraDaemons` via `supervisorctl start webextradaemons:*`
20. Runs `app.PostStartAction()` (CMS-specific) then `app.ProcessHooks("post-start")`

#### Phase 4: Describe (`ddev describe`)

1. `app.Describe(short)` at line 219 in `ddevapp.go`
2. Calls `app.ProcessHooks("pre-describe")`
3. Builds `appDesc` map with: name, status, approot, docroot, URLs, database info,
   Mutagen status, router status, services with container details
4. If `short=false`, adds hostnames, performance mode, all URLs, DB connection
   info (user/pass/port), XHProf/XHGui status, installed services with ports

#### Phase 5: Stop (`ddev stop`)

1. `app.Stop(removeData, createSnapshot)` at line 3244 in `ddevapp.go`
2. Clears `EphemeralRouterPortsAssigned`
3. Runs `app.ProcessHooks("pre-stop")`
4. If `createSnapshot=true` and not running, starts app first, then calls
   `app.Snapshot(snapshotName)`
5. If Mutagen enabled: `SyncAndPauseMutagenSession(app)`
6. Removes merged Traefik config via `dockerutil.RunSimpleContainer` executing
   `rm -rf /mnt/ddev-global-cache/traefik/config/<name>_merged.yaml`
7. If `removeData=true`: cleans global cache (certs, traefik config), removes
   Docker volumes, terminates Mutagen sync
8. Runs `docker-compose down` via `dockerutil.ComposeCmd`
9. Runs `app.ProcessHooks("post-stop")`

## 5. Configuration Schema

`.ddev/config.yaml` key fields mapped to `DdevApp` struct fields:

| YAML Key | Struct Field | Type | Default |
|---|---|---|---|
| `name` | `Name` | `string` | directory name |
| `type` | `Type` | `string` | `"php"` |
| `docroot` | `Docroot` | `string` | `""` |
| `php_version` | `PHPVersion` | `string` | `nodeps.PHPDefault` |
| `webserver_type` | `WebserverType` | `string` | `nodeps.WebserverDefault` |
| `database` | `Database` | `DatabaseDesc` | `mariadb:10.11` |
| `nodejs_version` | `NodeJSVersion` | `string` | `nodeps.NodeJSDefault` |
| `composer_version` | `ComposerVersion` | `string` | `nodeps.ComposerDefault` |
| `performance_mode` | `PerformanceMode` | `types.PerformanceMode` | platform-dependent |
| `router_http_port` | `RouterHTTPPort` | `string` | `"80"` |
| `router_https_port` | `RouterHTTPSPort` | `string` | `"443"` |
| `additional_hostnames` | `AdditionalHostnames` | `[]string` | `[]` |
| `additional_fqdns` | `AdditionalFQDNs` | `[]string` | `[]` |
| `hooks` | `Hooks` | `map[string][]YAMLTask` | `nil` |
| `web_extra_exposed_ports` | `WebExtraExposedPorts` | `[]WebExposedPort` | `nil` |
| `web_extra_daemons` | `WebExtraDaemons` | `[]WebExtraDaemon` | `nil` |
| `omit_containers` | `OmitContainers` | `[]string` | `[]` |
| `upload_dirs` | `UploadDirs` | `[]string` | CMS-specific |
| `web_environment` | `WebEnvironment` | `[]string` | `[]` |
| `ddev_version_constraint` | `DdevVersionConstraint` | `string` | `""` |

Config override files (`config.*.yaml`) are loaded in glob order after the base
config. Each is applied via `app.LoadConfigYamlFile()` which unmarshals YAML
on top of the existing struct values.

## 6. Project Type System

### appTypeFuncs (pkg/ddevapp/apptypes.go)

Each project type registers a set of optional callback functions:

```go
type appTypeFuncs struct {
    settingsCreator            func(*DdevApp) (string, error)
    uploadDirs                 func(*DdevApp) []string
    hookDefaultComments        func() []byte
    composerCreateAllowedPaths func(app *DdevApp) ([]string, error)
    appTypeSettingsPaths       func(app *DdevApp)
    appTypeDetect              func(app *DdevApp) bool
    postImportDBAction         func(app *DdevApp) error
    configOverrideAction       func(app *DdevApp) error
    postConfigAction           func(app *DdevApp) error
    postStartAction            func(app *DdevApp) error
    importFilesAction          func(app *DdevApp, uploadDir, importPath, extractPath string) error
    defaultWorkingDirMap       func(app *DdevApp, defaults map[string]string) map[string]string
}
```

### appTypeMatrix (pkg/ddevapp/apptypes.go)

Static map populated in `init()`. Registered types include:
`backdrop`, `bedrock`, `craftcms`, `codeigniter`, `django4`, `drupal`, `drupal6`,
`drupal7`, `drupal8`, `drupal9`, `drupal10`, `drupal11`, `drupal12`, `laravel`,
`magento`, `magento2`, `php`, `python`, `shopware6`, `silverstripe`, `symfony`,
`typo3`, `wordpress`, `generic`.

The `"drupal"` entry is a copy of the latest stable Drupal type with
`appTypeDetect` set to `nil` (not auto-detected, only explicit).

### Detection: DetectAppType (pkg/ddevapp/apptypes.go)

```go
func (app *DdevApp) DetectAppType() string
```

Iterates `appTypeMatrix` keys in sorted order, calling each type's
`appTypeDetect` function. Returns the first match, or `nodeps.AppTypePHP`
as fallback. The `generic` type short-circuits detection (never overridden).

### CMS-specific hooks

Dispatched via `appTypeMatrix` lookup:

- `app.ConfigFileOverrideAction(overrideExistingConfig)` -- allows CMS to set
  recommended DB type (e.g., Drupal 6 needs `php56`). Protects existing DB from
  unwanted upgrades.
- `app.PostStartAction()` -- CMS-specific post-start tasks
- `app.PostConfigAction()` -- runs at end of `ddev config`
- `app.CreateSettingsFile()` -- generates CMS settings (e.g., `settings.php`,
  `wp-config.php`). Calls `PrepDdevDirectory(app)` first, then the type's
  `settingsCreator` function.

### Implementation conventions for new project types

- **`DisableSettingsManagement` check**: Post-start actions that manage settings
  or `.env` files must return early if `app.DisableSettingsManagement` is true.
- **`OmitContainers` check for db**: Types that write database credentials to
  `.env` or settings files must check `slices.Contains(app.OmitContainers, "db")`
  before writing DB_NAME/DB_USER/DB_PASSWORD/DB_HOST. All `.env`-managing types
  (codeigniter, laravel, symfony, bedrock) follow this pattern.
- **`.env` file helpers**: Use `ReadProjectEnvFile()` / `WriteProjectEnvFile()`
  for `.env` management. Do not write `.env` files manually.
- **Test fixtures**: Detection test fixtures in
  `testdata/TestDetectAppType/<type>/` use minimal marker files (empty or 1-2
  lines). Just enough for the detection function to match.
- **Detection order**: `DetectAppType()` iterates alphabetically. If your type
  overlaps with another (e.g., bedrock vs wordpress), the more specific type
  must either sort first or use a more specific marker file.

## 7. Addon System

### Types (pkg/ddevapp/addons.go)

```go
type InstallDesc struct {
    Name                  string            `yaml:"name"`
    ProjectFiles          []string          `yaml:"project_files"`
    GlobalFiles           []string          `yaml:"global_files,omitempty"`
    DdevVersionConstraint string            `yaml:"ddev_version_constraint,omitempty"`
    Dependencies          []string          `yaml:"dependencies,omitempty"`
    PreInstallActions     []string          `yaml:"pre_install_actions,omitempty"`
    PostInstallActions    []string          `yaml:"post_install_actions,omitempty"`
    RemovalActions        []string          `yaml:"removal_actions,omitempty"`
    YamlReadFiles         map[string]string `yaml:"yaml_read_files"`
    Image                 string            `yaml:"image,omitempty"`
}

type AddonManifest struct {
    Name           string   `yaml:"name"`
    Repository     string   `yaml:"repository"`
    Version        string   `yaml:"version"`
    Dependencies   []string `yaml:"dependencies,omitempty"`
    InstallDate    string   `yaml:"install_date"`
    ProjectFiles   []string `yaml:"project_files"`
    GlobalFiles    []string `yaml:"global_files"`
    RemovalActions []string `yaml:"removal_actions"`
}
```

### Install Flow

1. `InstallAddonFromGitHub(app, "owner/repo", requestedVersion, verbose)` parses
   owner/repo, fetches tarball URL via `GetAddonTarballURL`
2. Downloads and extracts to temp directory
3. Reads `install.yaml` into `InstallDesc`
4. Checks `DdevVersionConstraint` via `CheckDdevVersionConstraint()`
5. Installs dependencies recursively via `installAddonRecursive()`
6. Runs `PreInstallActions` via `ProcessAddonAction()`
7. Copies `ProjectFiles` to `.ddev/` and `GlobalFiles` to `~/.ddev/`
8. Runs `PostInstallActions`
9. Creates manifest via `createAddonManifest()` in `.ddev/addon-manifests/<name>/manifest.yaml`

### Remove Flow

1. `RemoveAddon(app, addonName, verbose, skipRemovalActions)` loads all manifests
   via `GatherAllManifests(app)`
2. Checks for dependent add-ons (blocks removal if dependencies exist)
3. Executes `RemovalActions` from manifest via `ProcessAddonAction()`
4. Removes `ProjectFiles` from `.ddev/` (only if `#ddev-generated` signature present)
5. Removes `GlobalFiles` from `~/.ddev/` (same signature check)
6. Warns about orphaned dependencies

### Discovery

- `ListAvailableAddonsFromRegistry()` fetches from the DDEV add-on registry
- Registry has fallback logic in `getAddonRegistryWithFallback()`
- `GetInstalledAddons(app)` reads manifests from `.ddev/addon-manifests/`

## 8. Edge Cases

### Mutagen Sync (pkg/ddevapp/mutagen.go)

- Mutagen is incompatible with `use_hardened_images` -- `Start()` returns error
- `CheckMutagenVolumeSyncCompatibility()` validates that sync session config hash,
  Docker volume label, and session state are consistent. On mismatch, terminates
  sync and removes volume before recreating.
- `NoBindMounts` forces Mutagen enabled on any platform:
  `globalconfig.DdevGlobalConfig.NoBindMounts` triggers Mutagen automatically
- Upload dirs within the Mutagen sync root trigger warnings via
  `app.checkMutagenUploadDirs()`
- `CheckMutagenIgnorePatterns()` warns about `node_modules` directories that
  should be excluded
- `CheckLargeFilesInSync()` warns about large files that slow sync

### Router Conflicts (pkg/ddevapp/router.go)

- `CheckRouterPorts(activeApps)` verifies required ports are available before
  starting the router. Returns error with troubleshooting link on failure.
- `AllocateAvailablePortForRouter(start, upTo)` finds free ports in a range
- `GetEphemeralPortsIfNeeded()` assigns ephemeral ports when configured ports
  are unavailable
- Unhealthy router is killed and recreated: `StartDdevRouter()` checks
  `router.State != "running"` and calls `dockerutil.RemoveContainer()` first
- Let's Encrypt mode extends router wait timeout from 60s to 180s

### Port Collisions

- `ValidateConfig()` checks `WebExtraExposedPorts` for duplicate names,
  duplicate container ports, and duplicate HTTP/HTTPS ports
- `WriteConfig()` calls `globalconfig.ReservePorts(app.Name, portsToReserve)`
  to prevent cross-project collisions
- `globalconfig.CheckHostPortsAvailable(app.Name, portsToReserve)` validates
  before reserving

### Docker Rootless

- `Start()` blocks bind mounts with Docker Rootless:
  `dockerutil.IsDockerRootless() && !globalconfig.DdevGlobalConfig.NoBindMounts`
  returns an error directing users to `ddev config global --no-bind-mounts`

### BuildKit Race Conditions

- `composeBuildMaxRetries = 3` constant controls retry count for
  `docker-compose build` when BuildKit snapshot races occur

### Database Type Migration

- Legacy `mariadb_version` and `mysql_version` fields are auto-migrated to
  `Database` struct in `NewApp()`:
  `if app.MariaDBVersion != "" { app.Database = DatabaseDesc{Type: nodeps.MariaDB, Version: app.MariaDBVersion} }`
- `Start()` checks existing DB volume type against config and blocks mismatches
  via `app.GetExistingDBType()`

### Config Name Override

- `WriteConfig()` detects if name was changed by a `config.*.yaml` override via
  `app.HasConfigNameOverride()` and omits name from the base config file
