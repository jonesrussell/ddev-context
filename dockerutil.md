# Subsystem Spec: pkg/dockerutil

Package `pkg/dockerutil` wraps the Docker Engine API and Docker Compose CLI
for all container, volume, network, and image operations in DDEV.

## Layer Rules

**Layer 2 (Docker).** Docker infrastructure layer.

- **Imports from:** Layer 0 (util, fileutil, nodeps), Layer 1 (globalconfig, versionconstants)
- **Imported by:** Layer 3 (ddevapp) only. **Never import dockerutil from cmd/ddev/cmd/ (Layer 4).** CLI commands should use DdevApp methods which internally delegate to dockerutil.
- **Never imports:** ddevapp or any higher layer

## File Map

| File | Purpose |
|---|---|
| `pkg/dockerutil/docker_manager.go` | Singleton `dockerManager` struct; `GetDockerClient()`, version/context/IP helpers |
| `pkg/dockerutil/docker_compose.go` | `ComposeCmdOpts`, `ComposeCmd()`, `ComposeWithStreams()`, compose download, `CreateComposeProject()` |
| `pkg/dockerutil/containers.go` | Container find/inspect/exec/copy/wait/run/remove operations; `NoHealthCheck` var |
| `pkg/dockerutil/volumes.go` | Volume CRUD, `CopyIntoVolume()`, `ParseDockerSystemDf()`, `VolumeSize` type |
| `pkg/dockerutil/networks.go` | Network ensure/remove/duplicate-cleanup; `NetName` const (`ddev_default`) |
| `pkg/dockerutil/images.go` | `ImageExistsLocally()`, `FindImagesByLabels()`, `RemoveImage()` |
| `pkg/dockerutil/providers.go` | `IsDockerDesktop()`, `IsColima()`, `IsLima()`, `IsRancherDesktop()`, `IsOrbStack()`, `IsPodman()`, `IsRootless()`, `IsPodmanRootless()`, `IsDockerRootless()`, `IsSELinux()` detection |
| `pkg/dockerutil/host_docker_internal.go` | `HostDockerInternal` type; platform-specific `host.docker.internal` IP resolution |
| `pkg/dockerutil/requirements.go` | `DockerVersionMatrix` type; version/buildx/compose/auth checks |
| `pkg/docker/images.go` | Image tag helpers (`GetWebImage()`, `GetDBImage()`, `GetRouterImage()`, `GetSSHAuthImage()`, `GetXhguiImage()`) -- separate package |

## Key Types

### dockerManager (unexported singleton in `docker_manager.go`)

```go
type dockerManager struct {
    goContext         context.Context
    apiClient         client.APIClient
    cli               *command.DockerCli
    dockerContextName string
    host              string
    hostIP            string
    hostIPErr         error
    info              system.Info
    serverVersion     client.ServerVersionResult
    cliPlugins        []manager.Plugin
    cliPluginsErr     error
}
```

Initialized once via `sync.Once` in `getDockerManagerInstance()`. All public
`Get*` functions delegate to this singleton.

### ComposeCmdOpts (`docker_compose.go:27`)

```go
type ComposeCmdOpts struct {
    ComposeFiles []string
    ComposeYaml  *types.Project
    Profiles     []string
    Action       []string
    Progress     bool
    Timeout      time.Duration
    ProjectName  string
    Env          []string
}
```

### Other Exported Types

- **`HostDockerInternal`** (`host_docker_internal.go`): `IPAddress string`, `ExtraHost string`, `Message string`
- **`DockerVersionMatrix`** (`requirements.go`): `APIVersion`, `Version`, `BuildxVersion`, `PodmanVersion`, `ComposeVersionConstraint` (all `string`)
- **`VolumeSize`** (`volumes.go`): `Name string`, `SizeBytes int64`, `SizeHuman string`
- **`NoHealthCheck`** (`containers.go`): `var NoHealthCheck = container.HealthConfig{...}` -- passed to disable health checks

## Interface Signatures

### Docker Client (`docker_manager.go`)

- `GetDockerClient() (context.Context, client.APIClient, error)`
- `GetDockerClientInfo() (system.Info, error)`
- `GetDockerContextNameAndHost() (string, string, error)`
- `GetDockerIP() (string, error)`
- `IsRemoteDockerHost() bool`
- `ResetDockerHost(host string) error`
- `GetServerVersion() (client.ServerVersionResult, error)`
- `GetDockerVersion() (string, error)`
- `GetDockerAPIVersion() (string, error)`
- `GetCLIPlugins() ([]manager.Plugin, error)`
- `GetCLIPlugin(name string) (manager.Plugin, error)`

### Compose Operations (`docker_compose.go`)

- `ComposeCmd(cmd *ComposeCmdOpts) (string, string, error)` -- returns stdout, stderr, error
- `ComposeWithStreams(cmd *ComposeCmdOpts, stdin io.Reader, stdout io.Writer, stderr io.Writer) error`
- `GetDockerComposeVersion() (string, error)`
- `GetLiveDockerComposeVersion() (string, error)`
- `DownloadDockerComposeIfNeeded() (bool, error)`
- `DownloadDockerCompose() error`
- `CreateComposeProject(yamlStr string) (*types.Project, error)`
- `PullImages(images []string, pullAlways bool) error`
- `Pull(image string) error`

### Container Operations (`containers.go`)

- `FindContainerByName(name string) (*container.Summary, error)`
- `FindContainerByLabels(labels map[string]string) (*container.Summary, error)`
- `FindContainersByLabels(labels map[string]string) ([]container.Summary, error)`
- `FindContainersWithLabel(label string) ([]container.Summary, error)`
- `InspectContainer(name string) (container.InspectResponse, error)`
- `GetContainerStateByName(name string) (container.ContainerState, error)`
- `GetDockerContainers(allContainers bool) ([]container.Summary, error)`
- `GetAppContainers(sitename string) ([]container.Summary, error)`
- `GetContainerHealth(c *container.Summary) (string, string)`
- `GetContainerEnv(key string, c container.Summary) string`
- `GetPublishedPort(privatePort uint16, c container.Summary) int`
- `GetContainerUser() (uidStr string, gidStr string, username string)`
- `GetContainerNames(containers []container.Summary, excludeContainerNames []string, removePrefix string) []string`
- `ContainerWait(waittime int, labels map[string]string) (string, error)`
- `ContainersWait(waittime int, labels map[string]string) error`
- `ContainerName(c *container.Summary) string`
- `Exec(containerID string, command string, uid string) (string, string, error)`
- `CopyIntoContainer(srcPath string, containerName string, dstPath string, exclusion string) error`
- `CopyFromContainer(containerName string, containerPath string, hostPath string) error`
- `RunSimpleContainer(image, name string, cmd, entrypoint, env, binds []string, uid string, removeContainerAfterRun, detach bool, labels map[string]string, portBindings network.PortMap, healthConfig *container.HealthConfig) (containerID string, out string, returnErr error)`
- `RunSimpleContainerExtended(name string, config *container.Config, hostConfig *container.HostConfig, removeContainerAfterRun bool, timeout time.Duration) (containerID string, out string, returnErr error)` -- **signature changed**: `detach bool` replaced by `timeout time.Duration`; `timeout=0` means detach
- `RemoveContainer(id string) error`
- `RemoveContainersByLabels(labels map[string]string) error`
- `GetBoundHostPorts(containerID string) ([]string, error)`
- `GetRouterNetworkAliases(containerID string) ([]string, error)`

### Volume Operations (`volumes.go`)

- `RemoveVolume(volumeName string) error`
- `VolumeExists(volumeName string) bool`
- `VolumeLabels(volumeName string) (map[string]string, error)`
- `CreateVolume(volumeName string, driver string, driverOpts map[string]string, labels map[string]string) (volume.Volume, error)`
- `CopyIntoVolume(sourcePath, volumeName, targetSubdir, uid string, excludeDirs string, destroyExisting bool) error`
- `ParseDockerSystemDf() (map[string]VolumeSize, error)`
- `GetVolumeSize(volumeName string) (int64, string, error)`
- `ListFilesInVolume(volumeName string, subdir string) ([]string, error)` -- lists filenames in a volume subdirectory
- `RemoveFilesFromVolume(volumeName string, subdir string, files []string) error` -- removes specific files from a volume subdirectory
- `PurgeDirectoryContentsInVolume(volumeName string, subdirs []string, uid string) error` -- removes all files inside subdirs but keeps the directories (preserves inotify watches)

### Network Operations (`networks.go`)

- `EnsureNetwork(name string, netOptions client.NetworkCreateOptions) error`
- `EnsureDdevNetwork()` -- creates `ddev_default` bridge network or fatals
- `NetExists(name string) bool`
- `NetworkExists(netName string) bool`
- `FindNetworksWithLabel(label string) ([]network.Summary, error)`
- `RemoveNetwork(netName string) error`
- `RemoveNetworkDuplicates(netName string)`
- `IsErrNotFound(err error) bool`

### Requirements (`requirements.go`)

- `CheckDockerVersion(dockerVersionMatrix DockerVersionMatrix) error`
- `CheckDockerCompose() error`
- `CheckDockerBuildxVersion(dockerVersionMatrix DockerVersionMatrix) error`
- `CheckAvailableSpace()`
- `CheckDockerAuth() error`

## Data Flow: Compose Up/Down

1. `ddevapp.DdevApp.Start()` calls `app.WriteDockerComposeYAML()` which renders all
   compose files into a single YAML at `app.DockerComposeFullRenderedYAMLPath()`.
2. `ddevapp` gathers compose file list via `app.ComposeFiles()` (`ddevapp.go:1308`).
3. Start calls `dockerutil.EnsureDdevNetwork()` and `dockerutil.RemoveNetworkDuplicates()`
   (`ddevapp.go:1529-1532`).
4. Start invokes `dockerutil.ComposeCmd(&ComposeCmdOpts{ComposeFiles: []string{renderedYAML}, Action: []string{"up", ...}})` (`ddevapp.go:1454`).
5. `ComposeCmd()` (`docker_compose.go:90`) wraps `ComposeWithStreams()`, capturing stdout/stderr.
6. `ComposeWithStreams()` (`docker_compose.go:40`) builds CLI args from `ComposeCmdOpts`:
   `-f file1 -f file2 --profile X -p projectName action...`, then shells out to the
   `docker compose` binary (downloaded if needed via `DownloadDockerComposeIfNeeded()`).
7. For stop: `ddevapp.DdevApp.Stop()` calls `ComposeCmd()` with `Action: []string{"down"}` (`ddevapp.go:1848`).
8. For exec: `ddevapp.ExecOpts` dispatches to either `ComposeWithStreams()` (interactive)
   or `ComposeCmd()` (captured) at `ddevapp.go:2539-2544`.

## Docker Compose Generation

- `DdevApp` holds `ComposeYaml *composeTypes.Project` (`ddevapp.go:151`), populated
  after `WriteDockerComposeYAML()`.
- `app.ComposeFiles()` (`ddevapp.go:1308`) collects all `.ddev/docker-compose.*.yaml` files.
- Rendered YAML path: `app.DockerComposeFullRenderedYAMLPath()` -- a single merged file.
- `ComposeCmd` receives this path via `ComposeCmdOpts.ComposeFiles`.
- `CreateComposeProject(yamlStr string)` (`docker_compose.go:313`) parses raw YAML into
  a `*types.Project` for programmatic inspection (used in `compose_yaml.go:75`).

## Container Inspection

- **By name**: `FindContainerByName("ddev-mysite-web")` (`containers.go:152`)
- **By labels**: `FindContainerByLabels(map[string]string{"com.ddev.site-name": "mysite"})` (`containers.go:192`)
- **Health**: `GetContainerHealth(c)` returns `(status, healthDetail)` (`containers.go:470`)
- **State**: `GetContainerStateByName(name)` returns `container.ContainerState` (`containers.go:180`)
- **All app containers**: `GetAppContainers(sitename)` filters by `com.ddev.site-name` label (`containers.go:530`)
- **Wait for healthy**: `ContainerWait(waittime, labels)` polls until healthy or timeout (`containers.go:270`); `ContainersWait()` waits for multiple containers (`containers.go:340`)

## Edge Cases: Docker Provider Differences

### Provider Detection (`providers.go`)

All detection functions use `GetDockerClientInfo()` (system.Info) unless noted.

- `IsDockerDesktop()`: checks `info.OperatingSystem` prefix `Docker Desktop` or `info.Name` contains `docker-desktop`
- `IsColima()`: checks `info.Name` prefix `colima`
- `IsLima()`: checks `info.Name` prefix `lima`, excluding names containing `rancher-desktop`
- `IsRancherDesktop()`: checks `info.OperatingSystem` prefix `Rancher Desktop` or `info.Name` contains `rancher-desktop`
- `IsOrbStack()`: checks `info.OperatingSystem` prefix `OrbStack` or `info.Name` contains `orbstack`
- `IsPodman()`: detected via `GetServerVersion()` components -- looks for component named `Podman Engine`
- `IsRootless()`: checks `info.SecurityOptions` for `name=rootless`
- `IsPodmanRootless()`: `IsRootless() && IsPodman()`
- `IsDockerRootless()`: `IsRootless() && nodeps.IsLinux() && !IsPodman() && !IsLima()`
- `IsSELinux()`: checks `info.SecurityOptions` for `name=selinux`

### host.docker.internal Resolution (`host_docker_internal.go`)

`GetHostDockerInternal()` returns platform-specific `HostDockerInternal`:

| Scenario | ExtraHost | IPAddress |
|---|---|---|
| Docker Desktop (any OS) | *(built-in)* | *(auto)* |
| Colima | `host-gateway` | `192.168.5.2` |
| Rancher Desktop / Lima | `host-gateway` | Docker bridge gateway IP |
| Native Linux (no WSL2) | `host-gateway` | Docker bridge gateway IP |
| WSL2 + Docker Desktop | *(built-in)* | *(auto)* |
| WSL2 + non-Desktop engine | `host-gateway` | Windows host IP from routing table |
| WSL2 mirrored mode | `host-gateway` | Windows reachable IP via PowerShell |
| xdebug_ide_location override | `host-gateway` | User-specified IP |

`ResetHostDockerInternal()` clears the cached singleton -- call after mutating global config (e.g., in tests).

### Error Handling Patterns

- `IsErrNotFound(err)` (`networks.go`) wraps `cerrdefs.IsNotFound` for Docker 404 errors
- Volume-in-use errors in `RemoveVolume()` return container names for user action
- `CheckAvailableSpace()` warns when Docker disk < `nodeps.MinimumDockerSpaceWarning` KB
- All Docker API calls go through `GetDockerClient()` which returns errors if the
  singleton failed to initialize (Docker daemon unreachable)
