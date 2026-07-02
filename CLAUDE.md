# refhub-paper-drafter

This plugin provides the `refhub-paper-drafter` skill for drafting HCI and visualization research papers.

**Invoke with:** `/refhub-paper-drafter`

The skill takes your scattered notes (local files + RefHub vault queries) and produces a grounded manuscript draft modeled on best practices from award-winning HCI and visualization research.

**Requires:** the `refhub` CLI from `@refhub/cli` and the RefHub Claude plugin — see https://github.com/refhub-io/refhub-cli and https://github.com/refhub-io/refhub-claude.

Use the real RefHub CLI flow in the skill:

```bash
refhub vaults list
refhub tags list --vault <vaultId>
refhub items search --vault <vaultId> --tag <tagId>
refhub export --vault <vaultId> --format json
```

Do not call a `refhub synthesis` command. Any synthesis should be produced by the agent from collected local notes and RefHub item/export data, with claim-level SOURCE MAP provenance.
