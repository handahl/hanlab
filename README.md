> The following text has been written (per my instructions and refinement) by Google Gemini, which has been equally a great help and a great pain in my journey of learning linux, containers and devops, gitops, cloud native.
 
han

*Taipei, 2026-02-13*

---

# A homelab, rootless

A partial look into my modular, containerized homelab environment built with **Podman Quadlets** and **systemd user-level units**.

This stack is developed and validated on **immutable Fedora-based systems** (specifically **Universal Blue** images: bazzite-dx and ucore / fcos). It treats the host as a minimal, immutable foundation, delegating all service logic to rootless user-space containers managed by systemd.

## Design Principles

* **Immutable-Native**: Designed for OS environments where /usr is read-only. Configuration is centralized in $HOME.  
* **Rootless Execution**: All containers run in the user's unprivileged namespace to maximize security.  
* **Declarative Infrastructure**: Networks, volumes, and pods are defined as Quadlet files; systemd handles the transient unit generation.  
* **Resource Isolation**: Logical grouping via homelab.slice for aggregate resource accounting and constraints.

## 1. Host Preparation

On immutable systems, ensure your user environment is prepared for persistent service execution.

### Enable User Linger

Allows user-space services to initialize at boot and persist after the final session logout.

loginctl enable-linger $USER

### Enable Podman User Socket

Required for API-driven observability tools (e.g., Dozzle, Beszel) to communicate with the runtime.

systemctl --user enable --now podman.socket

## 2. Deployment Workflow

### Directory Structure

Quadlet files (.container, .network, .volume, .pod) should be placed in or symlinked to:

~/.config/containers/systemd/

### Applying Configurations

Whenever a Quadlet definition is modified or added, reload the user daemon to regenerate the underlying systemd units:

systemctl --user daemon-reload

### Service Control

Once generated, units are managed via standard systemd verbs:

* **Start/Stop**: systemctl --user [start|stop|restart] <name>.service  
* **Status**: systemctl --user status <name>.service  
* **Inventory**:  
  * systemctl --user list-units | grep -E 'container|pod|volume'  
  * systemctl --user list-unit-files

[!NOTE]

**Naming Convention**: It is recommended to use a consistent prefix (e.g., hl-) for filenames to facilitate easy filtering across the systemd stack.

## 3. Runtime Inspection

While systemd manages the lifecycle, use Podman to inspect the internal state of the container engine:

podman ps -a          # Container status  
podman pod ps         # Pod groupings  
podman network ls     # Virtual networks  
podman volume ls      # Persistent volumes

## 4. Observability and Troubleshooting

### Logs**

The systemd journal is the primary source of truth for both container output and lifecycle events.

# Follow logs for a specific unit  
journalctl --user -f -u <name>.service

# Inspect recent execution failures  
journalctl --user -xe

### SELinux (Immutable Hosts)

Given the Fedora/FCOS base of Bazzite and uCore, SELinux is active by default. If a container fails with exit codes 126 or 127 (Permission Denied):

1. **Verify Volume Labels**: Ensure mount points use :Z (private) or :z (shared) relabeling options in the .container file.  
2. **Inspect Denials**:  
   sudo ausearch -m AVC -ts recent

3. **Policy Exceptions**: If standard relabeling is insufficient:  
   sudo ausearch -m AVC -ts recent | audit2allow -M homelab_local  
   sudo semodule -i homelab_local.pp

### Resource Accounting

Monitor the aggregate impact of the lab via the custom slice:

systemctl --user status homelab.slice

## 5. Repository Structure

* /quadlets: Definitions for containers, images, networks, and volumes.  
* /configs: Template/stub configuration files (e.g., Traefik, Kanidm).  
* /systemd-units: Custom .slice files for resource management.  
* /scripts: Utility scripts for maintenance and deployment.

## License

GNU GPL v3.0