---
name: ddev:docker
description: Use when modifying pkg/dockerutil/ or pkg/docker/ — Docker client, compose operations, container inspection, volume management
---

## Scope

This skill covers two packages that handle all Docker interactions in DDEV.

### Key Files

- `pkg/dockerutil/docker_compose.go` -- Compose command execution, version management, image pulling
- `pkg/dockerutil/containers.go` -- Container CRUD, inspection, exec, health checks, copy operations
- `pkg/dockerutil/docker_manager.go` -- Docker client singleton, context/host detection
- `pkg/dockerutil/volumes.go` -- Volume CRUD, size inspection, file operations within volumes
- `pkg/dockerutil/networks.go` -- Network creation, deduplication, alias management
- `pkg/dockerutil/images.go` -- Image existence checks, label-based search, removal
- `pkg/dockerutil/providers.go` -- Docker Desktop, Colima, OrbStack, Podman detection
- `pkg/dockerutil/requirements.go` -- Buildx checks, Docker auth, version requirements
- `pkg/dockerutil/host_docker_internal.go` -- host.docker.internal resolution across platforms
- `pkg/docker/` -- Image tag helper functions (`GetWebImage`, `GetDBImage`, `GetRouterImage`)

## Key Interfaces

### Docker Compose

```go
type ComposeCmdOpts struct {
    ComposeFiles []string
    ComposeYaml  *types.Project
    Profiles     []string
    Action       []string
    Progress     bool          // Add dots every second while running
    Timeout      time.Duration
    ProjectName  string        // Optional, set via -p flag
    Env          []string
}

func ComposeCmd(cmd *ComposeCmdOpts) (string, string, error)
func ComposeWithStreams(cmd *ComposeCmdOpts, stdin io.Reader, stdout io.Writer, stderr io.Writer) error
func CreateComposeProject(yamlStr string) (*types.Project, error)
func GetDockerComposeVersion() (string, error)
func GetLiveDockerComposeVersion() (string, error)
func DownloadDockerComposeIfNeeded() (bool, error)
func DownloadDockerCompose() error
func PullImages(images []string, pullAlways bool) error
func Pull(image string) error
```

### Container Operations

```go
// Discovery
func FindContainerByName(name string) (*container.Summary, error)
func FindContainerByLabels(labels map[string]string) (*container.Summary, error)
func FindContainersByLabels(labels map[string]string) ([]container.Summary, error)
func FindContainersWithLabel(label string) ([]container.Summary, error)
func GetDockerContainers(allContainers bool) ([]container.Summary, error)
func GetAppContainers(sitename string) ([]container.Summary, error)

// Inspection
func InspectContainer(name string) (container.InspectResponse, error)
func GetContainerStateByName(name string) (container.ContainerState, error)
func GetContainerHealth(c *container.Summary) (string, string)
func GetContainerEnv(key string, c container.Summary) string
func GetPublishedPort(privatePort uint16, c container.Summary) int
func GetBoundHostPorts(containerID string) ([]string, error)
func GetRouterNetworkAliases(containerID string) ([]string, error)
func ContainerName(c *container.Summary) string
func TruncateID(id string) string
func ValidatePort(port interface{}) error

// Lifecycle
func ContainerWait(waittime int, labels map[string]string) (string, error)
func ContainersWait(waittime int, labels map[string]string) error
func RemoveContainer(id string) error
func RemoveContainersByLabels(labels map[string]string) error
func GetContainerUser() (uidStr string, gidStr string, username string)

// Running
func RunSimpleContainer(image string, name string, cmd []string, entrypoint []string,
    env []string, binds []string, uid string, removeContainerAfterRun bool,
    detach bool, labels map[string]string, portBindings network.PortMap,
    healthConfig *container.HealthConfig) (containerID string, out string, returnErr error)
func RunSimpleContainerExtended(name string, config *container.Config,
    hostConfig *container.HostConfig, removeContainerAfterRun bool,
    detach bool) (containerID string, out string, returnErr error)

// Exec and Copy
func Exec(containerID string, command string, uid string) (string, string, error)
func CopyIntoContainer(srcPath string, containerName string, dstPath string, exclusion string) error
func CopyFromContainer(containerName string, containerPath string, hostPath string) error
func GetContainerNames(containers []container.Summary, excludeContainerNames []string,
    removePrefix string) []string
```

### Docker Manager (Singleton)

```go
func GetDockerClient() (context.Context, client.APIClient, error)
func GetDockerClientInfo() (system.Info, error)
func GetDockerContextNameAndHost() (string, string, error)
func GetDockerIP() (string, error)
func IsRemoteDockerHost() bool
func ResetDockerHost(host string) error
func GetServerVersion() (client.ServerVersionResult, error)
func GetDockerVersion() (string, error)
func GetDockerAPIVersion() (string, error)
func GetCLIPlugins() ([]manager.Plugin, error)
func GetCLIPlugin(name string) (manager.Plugin, error)
```

### Volume Management

```go
func CreateVolume(volumeName string, driver string, driverOpts map[string]string,
    labels map[string]string) (volume.Volume, error)
func RemoveVolume(volumeName string) error
func VolumeExists(volumeName string) bool
func VolumeLabels(volumeName string) (map[string]string, error)
func CopyIntoVolume(sourcePath string, volumeName string, targetSubdir string,
    uid string, exclusion string, destroyExisting bool) error
func PurgeDirectoryContentsInVolume(volumeName string, subdirs []string, uid string) error
func ListFilesInVolume(volumeName string, subdir string) ([]string, error)
func RemoveFilesFromVolume(volumeName string, subdir string, files []string) error
func GetVolumeSize(volumeName string) (int64, string, error)
func ParseDockerSystemDf() (map[string]VolumeSize, error)

type VolumeSize struct {
    // contains size fields
}
```

### Network Management

```go
func EnsureNetwork(name string, netOptions client.NetworkCreateOptions) error
func EnsureDdevNetwork()
func NetworkExists(netName string) bool
func NetExists(name string) bool
func FindNetworksWithLabel(label string) ([]network.Summary, error)
func RemoveNetwork(netName string) error
func RemoveNetworkWithWarningOnError(netName string)
func RemoveNetworkDuplicates(netName string)
func IsErrNotFound(err error) bool
```

### Image Helpers

```go
// pkg/dockerutil/images.go
func ImageExistsLocally(imageName string) (bool, error)
func FindImagesByLabels(labels map[string]string, danglingOnly bool) ([]image.Summary, error)
func RemoveImage(tag string) error

// pkg/docker/ (image tag resolution)
func GetWebImage() string
func GetDBImage(dbType string, dbVersion string) string
func GetSSHAuthImage() string
func GetRouterImage() string
func GetXhguiImage() string
```

### Provider Detection

```go
func IsDockerDesktop() bool
func IsColima() bool
func IsOrbStack() bool
func IsLima() bool
func IsPodman() bool
func IsRootless() bool
func IsWSL2() bool
```

### Host Docker Internal

```go
type HostDockerInternal struct {
    // Platform-specific host.docker.internal resolution
}

func GetHostDockerInternal() *HostDockerInternal
func ResetHostDockerInternal()
```

### Requirements Checking

```go
func CheckDockerAuth() error
func CheckBuildx() error
func GetBuildxLocation() (string, error)
func GetRequiredDockerVersion() string
```

## Architecture

### Docker Client Lifecycle

The Docker client is managed as a singleton via `docker_manager.go`:

1. `getDockerManagerInstance()` initializes the client once
2. All public functions (`GetDockerClient`, `GetDockerIP`, etc.) go through this singleton
3. The manager detects the Docker context and host automatically
4. `ResetDockerHost(host)` allows switching Docker hosts at runtime

### Compose Command Execution

```
ComposeCmd(opts) or ComposeWithStreams(opts, stdin, stdout, stderr)
  -> Resolves docker-compose binary path
  -> Builds command args from ComposeFiles + Action + Profiles
  -> Sets ProjectName via -p flag if specified
  -> Executes with optional Progress dots (every 1s)
  -> Returns stdout, stderr, error
```

### Container Wait Pattern

`ContainerWait` polls for container health status:

1. Finds container by labels
2. Polls every 500ms checking `GetContainerHealth()`
3. Returns log output on timeout for debugging
4. Used by `DdevApp.Start()` to verify web/db containers are healthy

### Network Architecture

- All DDEV containers join the `ddev_default` network via `EnsureDdevNetwork()`
- `RemoveNetworkDuplicates()` handles Docker's occasional duplicate network creation
- Router container gets network aliases for each project hostname

## Common Mistakes

- **Not checking `IsErrNotFound(err)`**: Many Docker API calls return "not found" errors that should be handled gracefully, not treated as fatal.
- **Forgetting `allContainers` parameter**: `GetDockerContainers(false)` returns only running containers. Use `true` to include stopped/paused ones.
- **Ignoring Podman differences**: `IsPodman()` returns true on Podman setups. Some Docker API behaviors differ (rootless, volume handling).
- **Hard-coding `docker-compose`**: Always use `ComposeCmd` which resolves the correct binary path and handles the bundled docker-compose.
- **Missing cleanup in `RunSimpleContainer`**: Set `removeContainerAfterRun=true` for ephemeral containers, or they accumulate.
- **Network name assumptions**: Always use `EnsureDdevNetwork()` rather than assuming `ddev_default` exists.

## Testing Patterns

```go
func TestContainerOperations(t *testing.T) {
    if os.Getenv("DDEV_NO_INSTRUMENTATION") == "" {
        t.Setenv("DDEV_NO_INSTRUMENTATION", "true")
    }

    // Use RunSimpleContainer for test containers
    id, out, err := dockerutil.RunSimpleContainer(
        "busybox:latest",
        "test-container-"+t.Name(),
        []string{"sleep", "60"},
        nil,   // entrypoint
        nil,   // env
        nil,   // binds
        "",    // uid
        true,  // removeContainerAfterRun
        true,  // detach
        nil,   // labels
        nil,   // portBindings
        nil,   // healthConfig
    )
    require.NoError(t, err)
    require.NotEmpty(t, id)

    t.Cleanup(func() {
        _ = dockerutil.RemoveContainer(id)
    })

    // Verify container exists
    container, err := dockerutil.FindContainerByName("test-container-" + t.Name())
    require.NoError(t, err)
    require.NotNil(t, container)
}
```

- Always use `require` over `assert`
- Clean up containers and volumes in `t.Cleanup()`
- Set `DDEV_NO_INSTRUMENTATION=true`
- Use unique container names per test (include `t.Name()`)

## Related Specs

- `pkg/ddevapp/` -- Primary consumer of dockerutil (see `ddev:core-app` skill)
- `pkg/ddevapp/router.go` -- Uses ComposeCmd for router lifecycle
- `pkg/versionconstants/` -- Defines image tags used by `pkg/docker/`
- `containers/` -- Dockerfiles for ddev-webserver, ddev-dbserver, ddev-traefik-router
