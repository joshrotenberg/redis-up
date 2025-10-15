# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**redis-up** is a CLI tool for quickly spinning up Redis development environments using Docker. It provides a simple, developer-friendly interface for creating various Redis deployment types with sensible defaults and advanced configuration options.

## Common Development Commands

### Build and Test
```bash
# Full test suite with all features
cargo test --lib --all-features
cargo test --all-features

# Build the CLI tool
cargo build --release

# Run the CLI
cargo run -- basic start
cargo run -- --help

# Check compilation
cargo check --all-features

# Run specific test
cargo test --all-features test_name_here -- --exact

# Run with output for debugging
cargo test --all-features -- --nocapture
```

### Code Quality Checks (MUST run before ANY commit)
```bash
# Format check
cargo fmt --all -- --check

# Lint with clippy (zero warnings required)
cargo clippy --all-targets --all-features -- -D warnings

# Run ALL checks in sequence (recommended before pushing)
cargo fmt --all && cargo clippy --all-targets --all-features -- -D warnings && cargo test --lib --all-features && cargo test --all-features
```

### Running redis-up CLI
```bash
# Basic Redis instance
cargo run -- basic start --name my-redis --port 6379

# Redis Stack with modules
cargo run -- stack start --name my-stack

# Redis Cluster
cargo run -- cluster start --masters 3 --replicas 1

# Redis Sentinel
cargo run -- sentinel start --sentinels 3

# Redis Enterprise
cargo run -- enterprise start --nodes 3

# YAML configuration
cargo run -- apply -f examples/basic.yaml

# List running instances
cargo run -- list

# View logs
cargo run -- logs my-redis

# Stop instance
cargo run -- stop my-redis

# Clean up all instances
cargo run -- cleanup
```

## Project Architecture

### Core Components

1. **CLI Layer** (`src/cli.rs`)
   - Built with `clap` derive macros
   - Defines all commands and arguments
   - Provides user-facing interface

2. **Commands** (`src/commands/`)
   - Each Redis deployment type has its own module
   - Common pattern: `handle_action()` function processes CLI args
   - Uses docker-wrapper templates for container orchestration

3. **Configuration** (`src/config.rs`)
   - State management for running instances
   - Stored in `~/.config/redis-up/instances.json`
   - Tracks container IDs, ports, connection info

4. **YAML Support** (`src/commands/yaml.rs`)
   - Declarative configuration format
   - Supports multi-deployment scenarios
   - Infrastructure-as-code for Redis environments

### Directory Structure
```
redis-up/
├── src/
│   ├── main.rs           # CLI entry point
│   ├── lib.rs            # Library exports
│   ├── cli.rs            # Clap CLI definitions
│   ├── config.rs         # State management
│   └── commands/
│       ├── mod.rs        # Command exports
│       ├── basic.rs      # Basic Redis
│       ├── stack.rs      # Redis Stack
│       ├── cluster.rs    # Redis Cluster
│       ├── sentinel.rs   # Redis Sentinel
│       ├── enterprise.rs # Redis Enterprise
│       ├── insight.rs    # RedisInsight integration
│       ├── list.rs       # List instances
│       ├── logs.rs       # View logs
│       ├── cleanup.rs    # Cleanup instances
│       └── yaml.rs       # YAML configuration
├── examples/             # Example YAML configs
├── tests/               # Integration tests
└── .github/workflows/   # CI/CD workflows
```

## Key Design Patterns

### Command Structure
Each deployment type follows this pattern:

```rust
pub async fn handle_action(action: Action, verbose: bool) -> Result<()> {
    match action {
        Action::Start(args) => start(args, verbose).await,
        Action::Stop(args) => stop(args, verbose).await,
    }
}

async fn start(args: StartArgs, verbose: bool) -> Result<()> {
    // 1. Load config
    let mut config = Config::load()?;
    
    // 2. Generate instance name if needed
    let name = args.name.unwrap_or_else(|| 
        config.generate_name(&InstanceType::Basic)
    );
    
    // 3. Use docker-wrapper template
    let container = RedisTemplate::new(&name)
        .port(args.port)
        .password(&password)
        .start()
        .await?;
    
    // 4. Store instance info
    config.add_instance(InstanceInfo { ... });
    config.save()?;
    
    // 5. Display connection info
    println!("Started {} on port {}", name, args.port);
    
    Ok(())
}
```

### Configuration Management
- All instances tracked in `~/.config/redis-up/instances.json`
- Config automatically created on first save
- Each instance stores: name, type, ports, containers, connection info
- Cleanup operations use stored container IDs

### Docker Integration
- Uses `docker-wrapper` crate for Docker CLI interaction
- Templates provide pre-configured Redis containers
- Automatic network creation for multi-container deployments
- Health checks via template `wait_for_ready()` method

## Dependencies

### Core Dependencies
- **docker-wrapper** (0.8.3+): Docker CLI wrapper and templates
- **clap**: CLI argument parsing
- **tokio**: Async runtime
- **serde/serde_json**: Configuration serialization
- **serde_yaml**: YAML configuration support
- **anyhow**: Error handling
- **colored**: Terminal colors
- **rand**: Password generation
- **dirs**: Config directory paths
- **chrono**: Timestamps

### Development Dependencies
- **tempfile**: Temporary directories in tests
- **tokio-test**: Async testing utilities
- **redis**: Redis client for integration tests
- **serial_test**: Serialize tests to avoid port conflicts

## Testing Strategy

### Unit Tests
- Located in `src/` files with `#[cfg(test)]`
- Test configuration management, name generation, etc.
- No Docker required

### Integration Tests
- Located in `tests/` directory
- Require Docker to be running
- Test actual Redis deployments and connections
- Use `#[serial]` to avoid port conflicts

### Running Tests
```bash
# Unit tests only (fast, no Docker)
cargo test --lib

# All tests (requires Docker)
cargo test --all-features

# Specific integration test
cargo test --test deployment_integration
```

## Common Issues and Solutions

### Port Conflicts
- Use unique ports for each deployment
- Default base ports: 6379 (basic), 7000 (cluster), 26379 (sentinel)
- Config tracks used ports to avoid conflicts

### Container Cleanup
```bash
# Clean up all redis-up instances
cargo run -- cleanup

# Manual cleanup
docker ps -a | grep redis-up- | awk '{print $1}' | xargs docker rm -f
```

### Config Directory Missing
- Automatically created by `Config::save()`
- Manual creation: `mkdir -p ~/.config/redis-up`

### Docker Not Available
```bash
# Check Docker status
docker --version
docker ps

# Start Docker Desktop (macOS)
open -a Docker
```

## Release Process

### Conventional Commits
The project uses conventional commits for automatic versioning:
- `feat:` → minor version bump (0.x.0)
- `fix:` → patch version bump (0.0.x)
- `feat!:` or `BREAKING CHANGE:` → major version bump (x.0.0)

### Release Workflow
1. Merge PR to main with conventional commits
2. release-plz automatically creates release PR
3. Merge release PR triggers:
   - Version bump in Cargo.toml
   - CHANGELOG update
   - GitHub release creation
   - crates.io publication

## Feature Flags

Currently no feature flags are used. All features are enabled by default.

## Important Notes

- **Config Location**: `~/.config/redis-up/instances.json`
- **Docker Dependency**: Docker must be running for all operations
- **Port Management**: Tool tracks ports to avoid conflicts
- **State Persistence**: All instance info persisted across restarts
- **Network Isolation**: Each deployment type can use custom networks
- **Password Security**: Generated passwords stored in config and container environment

## Contributing Guidelines

When adding new features:
1. Update CLI definitions in `src/cli.rs`
2. Add command handler in `src/commands/`
3. Update `Config` if new instance types
4. Add integration tests
5. Update README with examples
6. Add example YAML configs if applicable
7. Update CHANGELOG.md

## Troubleshooting

### "Container already exists" errors
```bash
# List existing containers
docker ps -a | grep redis-up

# Remove specific container
docker rm -f container-name

# Or use cleanup command
cargo run -- cleanup
```

### "Port already in use" errors
- Check config: `cat ~/.config/redis-up/instances.json`
- Use different port: `--port 6380`
- Stop conflicting instance: `cargo run -- stop instance-name`

### "Config directory not found"
- Should auto-create, but if not: `mkdir -p ~/.config/redis-up`

### Redis connection issues
- Check container is running: `docker ps`
- Verify port mapping: `docker port container-name`
- Test connection: `redis-cli -p PORT ping`
- Check logs: `cargo run -- logs instance-name`

## Architecture Decisions

### Why docker-wrapper?
- Type-safe Docker CLI interaction
- Pre-built Redis templates with best practices
- Async/await support
- Active maintenance

### Why local config vs. Docker labels?
- Faster lookups
- Works even if containers removed externally
- Easy to inspect and debug
- Survives Docker daemon restarts

### Why YAML configuration?
- Infrastructure-as-code approach
- Easier for multi-deployment scenarios
- Familiar format for DevOps users
- Version-controllable

## Future Enhancements

Potential features for consideration:
- [ ] Docker Compose export
- [ ] TLS/SSL configuration
- [ ] Custom Redis modules loading
- [ ] Backup/restore functionality
- [ ] Multiple profiles/environments
- [ ] Web UI for management
- [ ] Kubernetes manifest export
- [ ] Cloud provider integration (AWS, GCP, Azure)
