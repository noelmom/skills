# Ubuntu Docker SSH Keyboard-Interactive Test Server

Use this skill when you need a clean OpenSSH test server for validating SSH `keyboard-interactive` authentication, especially when testing client implementations such as `swift-nio-ssh`.

This setup uses an Ubuntu Docker container with OpenSSH, PAM, and `sshd` running in verbose debug mode.

## Goal

Create a temporary SSH server that supports keyboard-interactive authentication and confirms this packet flow:

```text
Client -> SSH_MSG_USERAUTH_REQUEST keyboard-interactive
Server -> SSH_MSG_USERAUTH_INFO_REQUEST
Client -> SSH_MSG_USERAUTH_INFO_RESPONSE
Server -> SSH_MSG_USERAUTH_SUCCESS
```

## Start Ubuntu Container

Run this on the Docker host:

```bash
docker run -it --rm \
  --name ssh-test \
  -p 2222:22 \
  ubuntu:24.04
```

The mapping `-p 2222:22` exposes container SSH port `22` as host port `2222`.

## Install and Configure OpenSSH

Inside the container, run:

```bash
apt update
apt install -y openssh-server libpam-modules

useradd -m testuser
echo "testuser:testpass" | chpasswd

mkdir -p /run/sshd

cat >/etc/ssh/sshd_config <<'EOF_SSHD'
Port 22
PermitRootLogin no
PasswordAuthentication yes
KbdInteractiveAuthentication yes
UsePAM yes
ChallengeResponseAuthentication yes
Subsystem sftp /usr/lib/openssh/sftp-server
EOF_SSHD
```

## Start sshd in Debug Mode

Still inside the container, run:

```bash
/usr/sbin/sshd -D -ddd -e
```

Leave this terminal open. The SSH server logs will print directly to the screen.

## Test with OpenSSH Client

From another terminal on the Docker host, run:

```bash
ssh -4 -vvv \
  -p 2222 \
  -o PreferredAuthentications=keyboard-interactive \
  -o PubkeyAuthentication=no \
  testuser@127.0.0.1
```

Password:

```text
testpass
```

Use `-4` and `127.0.0.1` to avoid `localhost` resolving to IPv6 `::1` during testing.

## Expected Client-Side Debug Lines

A successful keyboard-interactive test should show lines similar to:

```text
Next authentication method: keyboard-interactive
userauth_kbdint
we sent a keyboard-interactive packet, wait for reply
```

If the server sends the challenge correctly, the client should receive packet type `60`:

```text
receive packet: type 60
```

Then after the response is accepted:

```text
Authenticated to 127.0.0.1
```

## Expected Server-Side Debug Lines

In the container terminal running `sshd -D -ddd -e`, success should include:

```text
Accepted keyboard-interactive/pam for testuser from 172.17.0.1 port <port> ssh2
monitor_child_preauth: user testuser authenticated by privileged process
```

These lines confirm:

- The server accepted keyboard-interactive authentication.
- PAM handled the prompt.
- The supplied password was accepted.
- Authentication succeeded.

## Test from a Swift SSH Client

Point the Swift client to the Docker host IP and mapped port:

```text
host: 127.0.0.1 or <docker-host-lan-ip>
port: 2222
username: testuser
keyboard-interactive response: testpass
```

If testing from another machine on the LAN, use the Docker host IP, for example:

```text
host: 192.168.1.176
port: 2222
username: testuser
response: testpass
```

A working Swift implementation should handle:

```text
SSH_MSG_USERAUTH_INFO_REQUEST  = packet 60
SSH_MSG_USERAUTH_INFO_RESPONSE = packet 61
SSH_MSG_USERAUTH_SUCCESS       = packet 52
```

## Troubleshooting

### Docker Port Mapping

Confirm the container is running and mapped correctly:

```bash
docker ps | grep ssh-test
```

Expected mapping for Ubuntu:

```text
0.0.0.0:2222->22/tcp
```

Do not use `-p 2222:2222` for this Ubuntu setup. That mapping was only needed for some prebuilt OpenSSH images that run sshd on container port `2222`.

### Connection Closes Immediately

If you see:

```text
kex_exchange_identification: Connection closed by remote host
```

Check that `sshd` is actually running inside the container:

```bash
docker exec -it ssh-test bash
ps aux | grep sshd
```

Start it again if needed:

```bash
/usr/sbin/sshd -D -ddd -e
```

### IPv6 localhost Issue

If logs show:

```text
Connecting to localhost [::1] port 2222
```

Use IPv4 explicitly:

```bash
ssh -4 -p 2222 testuser@127.0.0.1
```

### Keyboard-Interactive Advertised but No Challenge Sent

If the client shows:

```text
Authentications that can continue: publickey,keyboard-interactive
userauth_kbdint: disable: no info_req_seen
Permission denied (publickey,keyboard-interactive)
```

The server advertised `keyboard-interactive` but did not send `SSH_MSG_USERAUTH_INFO_REQUEST` packet `60`.

For this Ubuntu setup, verify:

```bash
grep -Ei 'Kbd|Challenge|Password|UsePAM' /etc/ssh/sshd_config
```

Expected:

```text
PasswordAuthentication yes
KbdInteractiveAuthentication yes
UsePAM yes
ChallengeResponseAuthentication yes
```

### Password

The default test account is:

```text
username: testuser
password: testpass
```

Reset it inside the container if needed:

```bash
echo "testuser:testpass" | chpasswd
```

## Cleanup

If the container was started with `--rm`, it is removed automatically when stopped.

To stop it from another terminal:

```bash
docker rm -f ssh-test
```

## Notes

This setup is intentionally simple and temporary. It is meant for local development and protocol testing, not production use.
