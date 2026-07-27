# Remote-to-remote container image transfer

Use `crane copy` on an SSH host to copy an OCI/Docker image between two registries without pulling it into the local Docker daemon. Image layers pass through the SSH host's network and memory/temporary buffers; they do not occupy the local Docker image store.

## Required information

Prepare the following values before starting:

- SSH host alias, for example `st-pubcld-cci0`.
- Full source image reference: registry, project/namespace, repository, and tag or digest.
- Full destination image reference.
- Credentials with pull permission on the source and push/create-repository permission on the destination.
- Whether private CA certificates or insecure HTTP registries are involved.
- Remote host OS and architecture. The example below assumes Linux `x86_64`.

Example:

```text
Source: registry.cn-sh-01.sensecore.cn/devsft-ccr-2/verl_blenderrl:<tag>
Target: registry.ms-sc-01.maoshanwangtech.com/ms-ccr/verl_blenderrl:<tag>
SSH:    st-pubcld-cci0
```

## 1. Log in locally

Run these commands in WSL. `--password-stdin` keeps passwords out of shell history:

```bash
read -rp 'Source registry username: ' SRC_USER
read -rsp 'Source registry password: ' SRC_PASS
echo
printf '%s' "$SRC_PASS" | docker login \
  --username "$SRC_USER" --password-stdin \
  registry.cn-sh-01.sensecore.cn
unset SRC_PASS

read -rp 'Destination registry username: ' DST_USER
read -rsp 'Destination registry password: ' DST_PASS
echo
printf '%s' "$DST_PASS" | docker login \
  --username "$DST_USER" --password-stdin \
  registry.ms-sc-01.maoshanwangtech.com
unset DST_PASS
```

This stores credentials in `~/.docker/config.json`. If it uses a credential helper instead of inline `auth` entries, log in directly on the remote host or generate a temporary Docker config by another secure mechanism.

## 2. Install a temporary crane binary remotely

The remote host does not need Docker. Download the static archive locally and upload only the tool:

```bash
SSH_REMOTE='st-pubcld-cci0'
REMOTE_DIR='/tmp/registry-copy-tool'
TOOL_ARCHIVE="$(mktemp --suffix=.tar.gz)"

curl -fL --retry 3 \
  -o "$TOOL_ARCHIVE" \
  https://github.com/google/go-containerregistry/releases/latest/download/go-containerregistry_Linux_x86_64.tar.gz

ssh "$SSH_REMOTE" "mkdir -m 700 '$REMOTE_DIR'"
scp "$TOOL_ARCHIVE" "$SSH_REMOTE:$REMOTE_DIR/tool.tar.gz"
ssh "$SSH_REMOTE" \
  "tar -xzf '$REMOTE_DIR/tool.tar.gz' -C '$REMOTE_DIR' crane && chmod 700 '$REMOTE_DIR/crane'"
```

If the remote host can reach GitHub, the download and extraction can instead run directly there.

## 3. Send only the required credentials

Do not copy the entire Docker config when it contains credentials for unrelated registries. Filter it to the two required entries and create a mode-0600 temporary config remotely:

```bash
SRC_REGISTRY='registry.cn-sh-01.sensecore.cn'
DST_REGISTRY='registry.ms-sc-01.maoshanwangtech.com'

jq --arg src "$SRC_REGISTRY" --arg dst "$DST_REGISTRY" \
  '{auths: {($src): .auths[$src], ($dst): .auths[$dst]}}' \
  ~/.docker/config.json |
ssh "$SSH_REMOTE" \
  "umask 077; dd of='$REMOTE_DIR/config.json' status=none"
```

Never put passwords directly in command-line arguments: they may appear in shell history or process listings.

## 4. Copy the image

Set exact source and destination references. Keeping the repository name and tag is conventional but not required:

```bash
TAG='cu129_lightllm_qwen35_sandbox_megatron0.14.0_vllm0.13.0R3_torch2.9.0_fa3_te2.5.0_260715'
SRC="$SRC_REGISTRY/devsft-ccr-2/verl_blenderrl:$TAG"
DST="$DST_REGISTRY/ms-ccr/verl_blenderrl:$TAG"

SRC_DIGEST="$(ssh "$SSH_REMOTE" \
  "DOCKER_CONFIG='$REMOTE_DIR' '$REMOTE_DIR/crane' digest '$SRC'")"

ssh "$SSH_REMOTE" \
  "DOCKER_CONFIG='$REMOTE_DIR' '$REMOTE_DIR/crane' copy '$SRC' '$DST'"
```

Run long copies in `tmux`, `screen`, or a detached service if the SSH connection is unreliable. `crane copy` skips blobs already present at the destination and preserves the image digest when the registries support the same manifest.

## 5. Verify the result

Read the destination digest independently and compare it with the source:

```bash
DST_DIGEST="$(ssh "$SSH_REMOTE" \
  "DOCKER_CONFIG='$REMOTE_DIR' '$REMOTE_DIR/crane' digest '$DST'")"

printf 'source: %s\ntarget: %s\n' "$SRC_DIGEST" "$DST_DIGEST"
test "$SRC_DIGEST" = "$DST_DIGEST"
```

Only treat the transfer as successful when `crane copy` exits successfully and both digests match.

## 6. Clean up temporary credentials and tools

```bash
ssh "$SSH_REMOTE" \
  "rm -f '$REMOTE_DIR/config.json' '$REMOTE_DIR/crane' '$REMOTE_DIR/tool.tar.gz' && rmdir '$REMOTE_DIR'"
rm -f "$TOOL_ARCHIVE"
unset SRC_USER DST_USER SRC_DIGEST DST_DIGEST
```

The workflow copies by default and leaves the source image intact. For a true move, delete the source only after digest verification and after checking the registry's manifest-deletion semantics; deleting a digest may affect multiple tags that reference it.

## Troubleshooting

- `404 repository not found`: verify the complete project/namespace and repository path. A valid login does not prove that the path is correct or visible to that account.
- `401 unauthorized`: confirm the temporary config contains both registry host keys and that the account has the required pull/push permissions.
- `certificate signed by unknown authority`: install the private CA on the remote host. Avoid disabling TLS verification as a permanent fix.
- Target tag appears only at the end: this is normal. Registries receive blobs first and the manifest/tag is committed after all required layers finish.
- Transfer is slow: traffic still passes through the SSH host. Place it near both registries and check bandwidth, egress policy, and cross-region charges.
- Multi-architecture images: `crane copy` copies the remote image/index. Verify the destination digest to ensure the complete index was retained.
