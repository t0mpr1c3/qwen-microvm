# qwen-microvm

Run [Qwen 3.6 27B](https://ollama.com/library/qwen3.6:27b) in an isolated NixOS microVM via [microvm.nix](https://github.com/microvm-nix/microvm.nix) (QEMU+KVM). Your project directory is mounted read-write at `/work` inside the guest via virtiofs — no root required.

Qwen starts automatically on boot inside an Ollama console. Exiting Ollama shuts down the VM.

## Prerequisites

- [Nix](https://nixos.org/) with flakes enabled
- KVM support (`/dev/kvm`)

## Quick start

```sh
# Build and run with current directory mounted at /work
make vm.run

# Mount a specific project directory
WORK_DIR=/path/to/project make vm.run
```

## Usage from anywhere

### `nix run` (no install)

```sh
# From the repo directory
WORK_DIR=. nix run

# From a local checkout
WORK_DIR=/path/to/project nix run /path/to/this/repo

# Directly from git
WORK_DIR=. nix run github:t0mpr1c3/ollama-microvm
```

### Install to PATH

```sh
nix profile add github:t0mpr1c3/ollama-microvm

# Now available everywhere
WORK_DIR=/path/to/project ollama-run
```

### As a flake input

Add as a dependency in another project's `flake.nix`:

```nix
{
  inputs.ollama-vm.url = "github:t0mpr1c3/ollama-microvm";

  outputs = { nixpkgs, ollama-vm, ... }:
    let system = "x86_64-linux"; in {
      devShells.${system}.default = nixpkgs.legacyPackages.${system}.mkShell {
        packages = [ ollama-vm.packages.${system}.vm ];
      };
    };
}
```

Then `nix develop` gives you `microvm-run` in the shell.

## How it works

### virtiofs (host directory sharing)

The host `WORK_DIR` is shared into the VM at `/work` using virtiofs. A `virtiofsd` daemon is started automatically as a systemd user service (`ollama-vm-virtiofsd-<id>`, where `<id>` is derived from the work directory path) — no root or sudo needed. It runs unprivileged in a user namespace with UID/GID translation so files created inside the VM are owned by your host user.

Each work directory gets its own virtiofsd instance, so multiple VMs can run in parallel on different projects. The daemon persists between VM restarts for fast re-launches. Manage it with:

```sh
systemctl --user list-units 'ollama-vm-virtiofsd-*'
systemctl --user stop ollama-vm-virtiofsd-<id>
```

### Sandboxing

The VM provides strong isolation from the host:

- **Filesystem** — only `/work` is shared; everything else is VM-local and ephemeral
- **Processes** — completely isolated (separate kernel)
- **Network** — QEMU user-mode NAT; the VM can reach the internet but can't bind host ports

To let Qwen run fully autonomously inside the VM (no permission prompts), append `--experimental-yolo` to the `ollama` invocation in `flake.nix`.

### Shutting down

Exiting the Ollama prompt automatically powers off the VM.

## Customization

### Exposing ports

No ports are forwarded by default. To expose ports, edit `flake.nix`:

```nix
microvm.qemu.extraArgs = [
  "-netdev" "user,id=usernet,hostfwd=tcp::8080-:8080"
  "-device" "virtio-net-device,netdev=usernet"
];
networking.firewall.allowedTCPPorts = [ 8080 ];
```

Rebuild with `make vm`.

### VM specs

| Resource | Default  |
|----------|----------|
| RAM      | 20000 MB |
| vCPUs    | 4        |
| Network  | User-mode (SLiRP) |
| Work dir | Host directory via virtiofs (read-write) |
