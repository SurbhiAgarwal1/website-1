---
title: Configuring Container Runtimes for Proxy Cache
weight: 40
---

When using Harbor as a proxy cache, you can configure your container runtimes to automatically mirror public registries and pull images through Harbor. This is especially useful for Kubernetes, OpenShift, air-gapped environments, or enterprise setups with restricted outbound access.

The following examples demonstrate how to configure popular container runtimes to use a Harbor Proxy Cache project as a registry mirror. 

{{< note >}}
**Important API compatibility requirement:** When configuring containerd or CRI-O to mirror through Harbor, you must ensure that your configuration enables path overriding (e.g. `override_path = true` in containerd). Without this, the runtimes will request the standard `/v2/` endpoint, but Harbor expects the request to be routed to the specific proxy project endpoint `/v2/<proxy_project_name>/`.
{{< /note >}}

## Containerd

To configure `containerd` to use Harbor as a mirror for `docker.io`, you must use the `hosts.toml` configuration format (v2). 

1. Edit your `/etc/containerd/config.toml` to ensure the `config_path` is set for registry configurations:

```toml
version = 3

[plugins]
  [plugins.'io.containerd.cri.v1.images']
    [plugins.'io.containerd.cri.v1.images'.registry]
      config_path = '/etc/containerd/certs.d'
```

2. Create the directory for the registry you want to mirror (e.g., `docker.io`):
```bash
mkdir -p /etc/containerd/certs.d/docker.io
```

3. Create the `hosts.toml` file (`/etc/containerd/certs.d/docker.io/hosts.toml`) and configure your Harbor proxy cache project as the mirror. In this example, we assume your Harbor instance is `mirror.example.local` and your proxy project is named `docker`.

```toml
server = 'https://docker.io'

[host.'https://mirror.example.local/v2/docker']
  capabilities = ['resolve', 'pull']
  override_path = true
```

{{< note >}}
**Caution regarding `ctr`:** Do not use `ctr` to test your mirror configuration. The `ctr` CLI does not respect the `/etc/containerd/certs.d` directory. Instead, use `crictl` or Kubernetes deployments to verify the configuration.
{{< /note >}}

### Fallback for legacy docker.io clients
If you have tools still resolving to the legacy `registry-1.docker.io`, you can symlink the directory:
```bash
ln -s /etc/containerd/certs.d/docker.io /etc/containerd/certs.d/registry-1.docker.io
```

Restart `containerd` to apply the changes:
```bash
systemctl restart containerd
```

---

## CRI-O

For OpenShift and CRI-O based environments, you can configure the mirror in `/etc/containers/registries.conf`.

1. Open or create `/etc/containers/registries.conf`.
2. Add the `[[registry]]` block for the target registry and configure the Harbor proxy cache as the mirror.

```toml
[[registry]]
prefix = "docker.io"
location = "docker.io"

[[registry.mirror]]
location = "mirror.example.local/docker"
```

{{< note >}}
**TLS & Authentication:** If you are testing with a self-signed certificate on your Harbor instance, you may need to set `insecure = true` in the `[[registry.mirror]]` block, or place your CA certificate in `/etc/containers/certs.d/mirror.example.local/ca.crt`.
{{< /note >}}

Restart CRI-O to apply the changes:
```bash
systemctl restart crio
```
