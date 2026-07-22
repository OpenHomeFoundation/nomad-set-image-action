# nomad-deploy-action

Composite GitHub Action that rolls a new container image out to a running
[Nomad](https://www.nomadproject.io/) job over
[Tailscale](https://tailscale.com/).

It fetches the live job spec, swaps only the target task's image on every
task group that has it, and resubmits. Everything else in the spec is
round-tripped untouched — so if the job is owned by configuration management
(e.g. an Ansible role that injects secret env vars), a deploy made here never
loses that config and, conversely, must not try to re-render it.

`nomad job run` blocks on the resulting deployment and exits non-zero if the
rollout fails its health checks, so the calling workflow fails loudly.

## Usage

```yaml
jobs:
  deploy:
    name: Deploy to Nomad
    needs: publish
    runs-on: ubuntu-latest
    environment: production
    permissions:
      contents: read
    steps:
      - name: Roll out new image tag
        uses: OpenHomeFoundation/nomad-deploy-action@<sha> # vX.Y.Z
        with:
          image: ${{ needs.publish.outputs.image }}:${{ needs.publish.outputs.version }}
          job: my-nomad-job
          nomad-addr: ${{ secrets.NOMAD_ADDR }}
          nomad-token: ${{ secrets.NOMAD_TOKEN }}
          tailscale-oauth-client-id: ${{ secrets.TS_OAUTH_CLIENT_ID }}
          tailscale-oauth-secret: ${{ secrets.TS_OAUTH_SECRET }}
```

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `image` | yes | — | Fully-qualified image reference to deploy (`registry/name:tag`). |
| `job` | yes | — | Name of the Nomad job to update. |
| `task` | no | `server` | Task whose image is swapped, on every group that contains it. |
| `nomad-addr` | yes | — | Nomad HTTP API address, reached over the tailnet. |
| `nomad-token` | yes | — | Nomad ACL token allowed to submit the job. |
| `nomad-version` | no | `2.0.4` | Nomad CLI version to install. |
| `tailscale-oauth-client-id` | yes | — | Tailscale OAuth client ID used to join the tailnet. |
| `tailscale-oauth-secret` | yes | — | Tailscale OAuth client secret. |
| `tailscale-tags` | no | `tag:ecosystem-device-database-release-action` | ACL tags for the ephemeral runner node. |

## Prerequisites

- A Tailscale OAuth client whose tailnet ACLs allow it to assume
  `tailscale-tags`, and allow the tagged node to reach the Nomad API address
  (typically `:4646` on the target host).
- A Nomad ACL token with `submit-job` on the target job's namespace.
- The job must already exist in Nomad — this action only swaps the image on
  a running job; it never creates one.

## Notes

- Deploy by `:tag` rather than digest if anything reads the tag back from the
  scheduled job (OHF's infrastructure-ansible does, so its converges never
  revert a deploy made here).
- The action fails rather than resubmitting an unchanged spec when no task
  matches `task` — a typo can't silently "deploy" nothing.
