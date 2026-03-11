---
name: ddev:core-app
description: Use when modifying pkg/ddevapp/ — project lifecycle, configuration, Docker orchestration, addons, project types
---

## Scope

This skill covers the `pkg/ddevapp/` package, which is the core of DDEV project management.

### Key Files

- `pkg/ddevapp/ddevapp.go` -- `DdevApp` struct, lifecycle methods (Start, Stop, Restart, Describe, Exec)
- `pkg/ddevapp/config.go` -- Config reading/writing, YAML merge, validation
- `pkg/ddevapp/addons.go` -- Add-on install, removal, manifest management
- `pkg/ddevapp/apptypes.go` -- Project type detection and settings (PHP, Node.js, etc.)
- `pkg/ddevapp/router.go` -- Router compose generation, port management, Traefik config
- `pkg/ddevapp/mutagen.go` -- Mutagen sync session lifecycle
- `pkg/ddevapp/backdrop.go`, `drupal.go`, `laravel.go`, `wordpress.go` -- CMS-specific settings writers

## Key Interfaces

### DdevApp Struct (abbreviated)

```go
type DdevApp struct {
    Name                      string                `yaml:"name,omitempty"`
    Type                      string                `yaml:"type"`
    AppRoot                   string                `yaml:"-"`
    Docroot                   string                `yaml:"docroot"`
    PHPVersion                string                `yaml:"php_version"`
    WebserverType             string                `yaml:"webserver_type"`
    WebImage                  string                `yaml:"webimage,omitempty"`
    RouterHTTPPort            string                `yaml:"router_http_port,omitempty"`
    RouterHTTPSPort           string                `yaml:"router_https_port,omitempty"`
    XdebugEnabled             bool                  `yaml:"xdebug_enabled"`
    NoProjectMount            bool                  `yaml:"no_project_mount,omitempty"`
    AdditionalHostnames       []string              `yaml:"additional_hostnames"`
    AdditionalFQDNs           []string              `yaml:"additional_fqdns"`
    Database                  DatabaseDesc          `yaml:"database"`
    PerformanceMode           types.PerformanceMode `yaml:"performance_mode,omitempty"`
    FailOnHookFail            bool                  `yaml:"fail_on_hook_fail,omitempty"`
    Hooks                     map[string][]YAMLTask `yaml:"hooks,omitempty"`
    UploadDirs                []string              `yaml:"upload_dirs,omitempty"`
    WorkingDir                map[string]string     `yaml:"working_dir,omitempty"`
    OmitContainers            []string              `yaml:"omit_containers,omitempty,flow"`
    WebImageExtraPackages     []string              `yaml:"webimage_extra_packages,omitempty,flow"`
    DBImageExtraPackages      []string              `yaml:"dbimage_extra_packages,omitempty,flow"`
    NodeJSVersion             string                `yaml:"nodejs_version"`
    ConfigPath                string                `yaml:"-"`
    DataDir                   string                `yaml:"-"`
    // ... many more fields
}
```

### Lifecycle Methods

```go
func (app *DdevApp) Init(basePath string) error
func (app *DdevApp) Start() error
func (app *DdevApp) StartAndWait(extraSleep int) error
func (app *DdevApp) StartOptionalProfiles(profiles []string) error
func (app *DdevApp) StartAppIfNotRunning() error
func (app *DdevApp) Restart() error
func (app *DdevApp) Stop(removeData bool, createSnapshot bool) error
func (app *DdevApp) Pause() error
func (app *DdevApp) Describe(short bool) (map[string]interface{}, error)
func (app *DdevApp) GetType() string
```

### Execution Methods

```go
func (app *DdevApp) Exec(opts *ExecOpts) (string, string, error)
func (app *DdevApp) ExecWithTty(opts *ExecOpts) error
func (app *DdevApp) ExecOnHostOrService(service string, cmd string) error
func (app *DdevApp) Logs(service string, follow bool, timestamps bool, tailLines string) error
func (app *DdevApp) CaptureLogs(service string, timestamps bool, tailLines string) (string, error)
```

### Container and Port Methods

```go
func (app *DdevApp) FindContainerByType(containerType string) (*container.Summary, error)
func (app *DdevApp) GetPublishedPort(serviceName string) (int, error)
func (app *DdevApp) GetPublishedPortForPrivatePort(serviceName string, privatePort uint16) (publicPort int, err error)
func (app *DdevApp) Wait(requiredContainers []string) error
func (app *DdevApp) WaitForServices() error
func (app *DdevApp) GetMaxContainerWaitTime() int
func (app *DdevApp) DockerEnv() map[string]string
```

### Configuration Methods

```go
func (app *DdevApp) ReadConfig(includeOverrides bool) ([]string, error)
func (app *DdevApp) LoadConfigYamlFile(filePath string) error
func (app *DdevApp) WriteConfig() error
func (app *DdevApp) mergeAdditionalConfigIntoApp(configPath string) error
func (app *DdevApp) ValidateConfig() error
func (app *DdevApp) GetConfigPath(filename string) string
func (app *DdevApp) GenerateWebserverConfig() error
func (app *DdevApp) GeneratePostgresConfig() error
func (app *DdevApp) CheckExistingAppInApproot() error
```

### Active Project Discovery

```go
func GetActiveApp(siteName string) (*DdevApp, error)
func GetActiveAppRoot(siteName string) (string, error)
func GetActiveProjects() []*DdevApp
```

### Add-on Types and Functions

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

func GetInstalledAddons(app *DdevApp) []AddonManifest
func GetInstalledAddonNames(app *DdevApp) []string
func GetInstalledAddonProjectFiles(app *DdevApp) []string
func ProcessAddonAction(action string, installDesc InstallDesc, app *DdevApp, verbose bool) error
```

### Router Functions

```go
func StartDdevRouter() error
func RemoveRouterContainer() error
func FindDdevRouter() (*container.Summary, error)
func IsRouterDisabled(app *DdevApp) bool
func RouterComposeYAMLPath() string
func FullRenderedRouterComposeYAMLPath() string
func CheckRouterPorts(activeApps []*DdevApp) error
func PushGlobalTraefikConfig(activeApps []*DdevApp) error
func ClearRouterHealthcheck() error
```

### Mutagen Functions

```go
func CreateOrResumeMutagenSync(app *DdevApp) error
func ResumeMutagenSync(app *DdevApp) error
func GetMutagenConfigFile(app *DdevApp) string
func GetMutagenConfigFilePath(app *DdevApp) string
func GetMutagenConfigFileHash(app *DdevApp) (string, error)
func GetMutagenVolumeLabel(app *DdevApp) (string, error)
func MutagenSyncName(appName string) string
```

### Site Status Constants

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

var DatabaseDefault = DatabaseDesc{nodeps.MariaDB, nodeps.MariaDBDefaultVersion}
```

## Architecture

### Project Start Flow

1. `app.Init(basePath)` -- Initializes DdevApp, sets paths
2. `app.ReadConfig(includeOverrides)` -- Loads `.ddev/config.yaml` then merges `config.*.y*ml` files in glob order
3. `app.WriteConfig()` -- Persists any computed defaults
4. `app.Start()` -- Orchestrates the full startup:
   - Validates config
   - Generates Docker Compose YAML from templates
   - Runs `pre-start` hooks
   - Calls `dockerutil.ComposeCmd()` with `up --build -d`
   - Starts Mutagen sync if `PerformanceMode == "mutagen"`
   - Starts router via `StartDdevRouter()`
   - Waits for container health checks
   - Runs `post-start` hooks

### Config Layering

Config files merge in this order (later overrides earlier):

1. `~/.ddev/global_config.yaml` (global defaults)
2. `.ddev/config.yaml` (project config)
3. `.ddev/config.*.yaml` (override files, in glob order)
4. Environment variable overrides

### Router Architecture

- Router runs as a separate Docker Compose project (`ddev-router`)
- Uses Traefik for routing HTTP/HTTPS to project containers
- Compose file generated from `router_compose_template.yaml` embedded asset
- User overrides via `~/.ddev/router-compose.*.yaml` glob
- Detects port and hostname changes to decide whether to recreate or just push config
- Let's Encrypt support increases wait timeout from 60s to 180s

### Add-on Lifecycle

1. `install.yaml` parsed into `InstallDesc`
2. `PreInstallActions` executed (shell or PHP with strict mode)
3. `ProjectFiles` and `GlobalFiles` copied
4. `PostInstallActions` executed
5. `AddonManifest` written to `.ddev/addon-metadata/<name>/manifest.yaml`
6. Removal reads manifest and runs `RemovalActions`

## Common Mistakes

- **Forgetting config overrides**: `ReadConfig(true)` merges `config.*.yaml` files. Testing with `ReadConfig(false)` skips overrides and produces different behavior.
- **Modifying AppRoot directly**: `AppRoot` is tagged `yaml:"-"` and derived from `Init()`. Setting it manually leads to path mismatches.
- **Not checking `IsRouterDisabled()`**: Codespaces and devcontainers disable the router. Code that assumes router presence will fail in those environments.
- **Ignoring `composeBuildMaxRetries`**: Docker Compose builds retry up to 3 times for BuildKit snapshot race conditions. Custom compose logic must handle this.
- **Port string types**: Router ports (`RouterHTTPPort`, `RouterHTTPSPort`) are strings, not ints. Use `strconv` for comparisons.

## Testing Patterns

```go
func TestSomeDdevFeature(t *testing.T) {
    assert := asrt.New(t)
    _ = assert // if needed

    app := &DdevApp{}
    err := app.Init(testDir)
    require.NoError(t, err)

    // Always clean up
    t.Cleanup(func() {
        err := app.Stop(true, false)
        require.NoError(t, err)
    })

    err = app.Start()
    require.NoError(t, err)

    // Verify container is running
    container, err := app.FindContainerByType("web")
    require.NoError(t, err)
    require.NotNil(t, container)
}
```

- Prefer `require` over `assert` for all assertions
- Use `t.Cleanup()` to stop apps after tests
- Set `DDEV_NO_INSTRUMENTATION=true` in test environments
- Use `GOTEST_SHORT=true` to limit test matrix iterations

## Related Specs

- `pkg/dockerutil/` -- Docker operations used by DdevApp (see `ddev:docker` skill)
- `pkg/globalconfig/` -- Global config merged into project config (see `ddev:config` skill)
- `pkg/versionconstants/` -- Docker image tags referenced during Start
- `pkg/nodeps/` -- Constants for container names, default ports, TLD
