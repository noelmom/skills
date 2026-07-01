# Ubuntu Docker SSH Keyboard-Interactive Test Server

This guide creates a disposable Ubuntu Docker SSH server for testing SSH `keyboard-interactive` authentication, including PAM-backed password prompts.

It is useful when validating client implementations such as `swift-nio-ssh` keyboard-interactive support.

## What this tests

Expected SSH authentication flow:

```text
Client -> SSH_MSG_USERAUTH_REQUEST keyboard-interactive
Server -> SSH_MSG_USERAUTH_INFO_REQUEST
Client -> SSH_MSG_USERAUTH_INFO_RESPONSE
Server -> SSH_MSG_USERAUTH_SUCCESS
```

A successful server log should include:

```text
Accepted keyboard-interactive/pam for testuser
monitor_child_preauth: user testuser authenticated by privileged process
```

## 1. Start an Ubuntu container

```bash
docker run -it --rm \
  --name ssh-test \
  -p 2222:22 \
  ubuntu:24.04
```

This maps host port `2222` to the container SSH port `22`.

## 2. Install and configure OpenSSH inside the container

Run these commands inside the Ubuntu container:

```bash
apt update
apt install -y openssh-server libpam-modules

useradd -m testuser
echo "testuser:testpass" | chpasswd

mkdir -p /run/sshd

cat >/etc/ssh/sshd_config <<'EOF'
Port 22
PermitRootLogin no
PasswordAuthentication yes
KbdInteractiveAuthentication yes
UsePAM yes
ChallengeResponseAuthentication yes
Subsystem sftp /usr/lib/openssh/sftp-server
EOF

/usr/sbin/sshd -D -ddd -e
```

Leave this terminal open. The `-ddd -e` flags keep `sshd` in the foreground and print verbose debug logs directly to the terminal.

## 3. Test from another terminal

From the Docker host, connect with OpenSSH and force keyboard-interactive auth:

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

## 4. Test from another machine on the LAN

Use the Docker host IP instead of `127.0.0.1`:

```bash
ssh -vvv \
  -p 2222 \
  -o PreferredAuthentications=keyboard-interactive \
  -o PubkeyAuthentication=no \
  testuser@192.168.1.176
```

Replace `192.168.1.176` with your Docker host IP.

## 5. Point your Swift client at the test server

Use:

```text
host: 192.168.1.176
port: 2222
username: testuser
keyboard-interactive response: testpass
```

If OpenSSH succeeds but your Swift client fails, the Docker server is working and the issue is likely in the client implementation.

## Troubleshooting

### Connection closes on localhost

If the log shows:

```text
Connecting to localhost [::1] port 2222
Connection closed by ::1 port 2222
```

Force IPv4:

```bash
ssh -4 -p 2222 testuser@127.0.0.1
```

### Confirm Docker port mapping

```bash
docker ps | grep ssh-test
```

For the Ubuntu container, the mapping should look like:

```text
0.0.0.0:2222->22/tcp
```

### Difference from LinuxServer OpenSSH image

The LinuxServer image listens on container port `2222`, so its mapping is usually:

```bash
-p 2222:2222
```

The Ubuntu container in this guide runs OpenSSH on container port `22`, so the mapping is:

```bash
-p 2222:22
```

### See server-side protocol logs

Because `sshd` is running with:

```bash
/usr/sbin/sshd -D -ddd -e
```

You should see debug logs directly in the container terminal, including keyboard-interactive/PAM authentication events.

## Cleanup

If the container is still running in another terminal:

```bash
docker rm -f ssh-test
```

If you used `--rm`, Docker will remove it automatically when it exits.
