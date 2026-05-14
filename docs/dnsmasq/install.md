# dnsmasq Install

`dnsmasq` is used for local development hostnames like `app.localhost`, `api.localhost`, or `foo.bar.localhost`.

## macOS Setup

Install `dnsmasq` with Homebrew:

```bash
brew install dnsmasq
```

Update the `dnsmasq` config:

- Apple Silicon: `/opt/homebrew/etc/dnsmasq.conf`
- Intel: `/usr/local/etc/dnsmasq.conf`

Add:

```conf
address=/localhost/127.0.0.1
```

Start the service:

```bash
sudo brew services start dnsmasq
```

`sudo brew services start dnsmasq` registers `dnsmasq` with `launchd`, so it should start automatically on system startup.

Tell macOS to use `dnsmasq` for `.localhost` names:

```bash
sudo mkdir -p /etc/resolver
echo 'nameserver 127.0.0.1' | sudo tee /etc/resolver/localhost
```

## Linux Setup

The preferred setup on Linux is usually to let NetworkManager run `dnsmasq`.

Install `dnsmasq`:

```bash
# Ubuntu / Debian
sudo apt install dnsmasq

# Fedora / RHEL
sudo dnf install dnsmasq
```

Enable the NetworkManager `dnsmasq` plugin:

```bash
sudo mkdir -p /etc/NetworkManager/conf.d
sudo tee /etc/NetworkManager/conf.d/00-use-dnsmasq.conf >/dev/null <<'EOF'
[main]
dns=dnsmasq
EOF
```

Add a local `.test` domain rule:

```bash
sudo mkdir -p /etc/NetworkManager/dnsmasq.d
sudo tee /etc/NetworkManager/dnsmasq.d/dev-test.conf >/dev/null <<'EOF'
address=/.test/127.0.0.1
EOF
```

Restart NetworkManager:

```bash
sudo systemctl restart NetworkManager
```

When NetworkManager is configured with `dns=dnsmasq`, NetworkManager starts and manages the local `dnsmasq` instance on startup.

## Verifying

Test that a local `.test` hostname resolves to `127.0.0.1`:

```bash
dig app.test
ping app.test
```

On Linux, you can also verify with:

```bash
resolvectl query app.test
```

If the setup is working, any `.test` hostname should resolve to `127.0.0.1`.
