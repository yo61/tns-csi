## Decision: Rebuild the tns-csi v0.17.6 image with a source-built open-iscsi 2.1.11 iscsiadm overlay, and publish it to ghcr.io/yo61/tns-csi

## Context:
The stock `bfenski/tns-csi:v0.17.6` image ships `iscsiadm` 2.1.10 (from Alpine).
On Talos, the `iscsi-tools` system extension runs `iscsid` 2.1.11. The
iscsiadm<->iscsid management IPC hangs when the client (2.1.10) and daemon
(2.1.11) versions disagree — see fenio/tns-csi#194. We need a driver image
whose in-container `iscsiadm` matches the node's `iscsid`.

## The rebuild + publish procedure
The full recipe lives in the header of `Dockerfile.tns-csi-oiscsi2.1.11` (the
only file the `robin/1.7.6-built-with-open-iscsi-2.1.11` branch adds on top of
`main`). It builds *around* upstream — the Go driver source comes from the
`fenio/tns-csi` v0.17.6 tag, not from this `yo61/tns-csi` fork. This branch is
just a carrier for the Dockerfile.

```bash
git clone --depth 1 -b v0.17.6 https://github.com/fenio/tns-csi
cp Dockerfile.tns-csi-oiscsi2.1.11 tns-csi/
cd tns-csi
docker buildx build --platform linux/amd64 \
  -f Dockerfile.tns-csi-oiscsi2.1.11 \
  -t ghcr.io/yo61/tns-csi:v0.17.6-oiscsi2.1.11 --push .
```

Three-stage build: (1) build the Go driver unchanged; (2) source-compile
open-iscsi 2.1.11 on musl using Alpine's own APKBUILD recipe, kept
musl-correct with `-DGLOB_ONLYDIR=0` and `-Dno_systemd=true`; (3) upstream
final stage plus an overlay that copies the 2.1.11 `iscsiadm` and its
`libopeniscsiusr` over Alpine's stock 2.1.10. Only `iscsiadm` is overlaid —
`iscsid` runs on the Talos host, not in-container. Two build-time
`iscsiadm --version | grep -q 2.1.11` guards fail the build if the version
doesn't match.

Push target confirmed via GHCR: the `yo61` org owns the `tns-csi` container
package (the personal `robinbowes` account only has `pre-commit-terraform`).

## Alternatives considered:
- Wait for upstream fenio/tns-csi to bump open-iscsi — too slow; blocks the
  Talos cluster now.
- Pin the Talos iscsi-tools extension back to 2.1.10 — fights the node's
  supported version and reverts on the next Talos upgrade.
- Full source rebuild of open-iscsi replacing the whole Alpine package — larger
  blast radius; the surgical iscsiadm overlay is the minimum that fixes the IPC.

## Reasoning:
Overlaying only the client binary is the smallest change that makes the
client<->daemon IPC versions agree, while leaving the rest of the upstream
image (and the Go driver) byte-for-byte identical to v0.17.6.

## Trade-offs accepted:
- We now maintain a fork image and must re-run this build on every upstream
  driver release until the fix lands upstream.
- The overlay mixes an Alpine 2.1.10 package with a source-built 2.1.11
  binary + lib; acceptable because only `iscsiadm` is used in-container.

## Supersedes: (none — first decision on this topic)
