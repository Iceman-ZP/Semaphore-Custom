# Semaphore Custom Build

Custom build of Semaphore UI based on upstream Semaphore v2.19.7.

Repository:
- Custom: Iceman-ZP/Semaphore-Custom
- Upstream: semaphoreui/semaphore

Base upstream commit:
- e9dc41a1

Current custom branch:
- custom/ansible-vars

## Changes

### 1. Ansible variable transport

Commit:

    aef087a3 Fix Ansible secret vars JSON transport

Semaphore originally passed Variable Group secret variables to Ansible as separate:

    --extra-vars name=value

This breaks values containing spaces and can corrupt keys and other complex values.

The custom build merges Ansible secret variables into the existing JSON extra-vars payload.

Result:

    --extra-vars '{"variable":"full value"}'

This preserves spaces and special characters and keeps secret variable precedence.

Environment-type secrets are not included in the Ansible extra-vars JSON.

### 2. Multiline variable values

Commit:

    dc2f1423 Support multiline secret variables in UI

The Variable Group Extra Variables secret-value field was changed from a single-line text field to a textarea.

This allows multiline values such as:

- OpenSSH private keys
- SFTP private keys
- PuTTY key files
- certificates
- other multiline Ansible variables

Newline characters are preserved through:

    UI
    -> Semaphore API
    -> encrypted storage
    -> decryption
    -> Environment.Secrets
    -> JSON --extra-vars
    -> Ansible

### 3. Compact multiline field

Commit:

    6eb76b16 Keep multiline secret fields compact

The multiline textarea uses a fixed compact height instead of auto-growing with the entire key.

This prevents large private keys from expanding the Variable Group table.

## Verified behavior

Tested successfully with:

- values containing spaces
- SSH public keys
- multiline PuTTY private-key data
- trailing newline
- Ansible JSON extra-vars transport

The end-to-end Semaphore task completed successfully with the multiline value preserved.

## Build

From the repository root:

    sudo docker build \
      --progress=plain \
      -f Dockerfile.custom \
      -t local/semaphore:v2.19.7-ansiblevars-6eb76b16 \
      .

Verify the embedded version:

    sudo docker run --rm \
      --entrypoint /usr/local/bin/semaphore \
      local/semaphore:v2.19.7-ansiblevars-6eb76b16 \
      version

Expected commit in output:

    6eb76b16

## Runtime base

The runtime image is pinned to:

    semaphoreui/semaphore:v2.19.7@sha256:8886d55da349f6d600f66f2c6dd2be01a3a5b4265542b04cb6c10ae47e4306d2

## Deployment

Set the Semaphore service image in docker-compose.yml to:

    local/semaphore:v2.19.7-ansiblevars-6eb76b16

Then recreate only the Semaphore service:

    sudo docker compose up -d \
      --no-deps \
      --force-recreate \
      big-bear-semaphore

Verify:

    sudo docker inspect big-bear-semaphore \
      --format 'Image={{.Config.Image}} Status={{.State.Status}}'

## Upstream updates

The official Semaphore repository is configured as:

    upstream https://github.com/semaphoreui/semaphore.git

The custom repository is:

    origin https://github.com/Iceman-ZP/Semaphore-Custom.git

Upstream changes can therefore be fetched without mixing them with the custom repository:

    git fetch upstream

Custom patches should be reviewed and reapplied/rebased when moving to a newer Semaphore release.
