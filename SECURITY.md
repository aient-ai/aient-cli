# Security

Report suspected vulnerabilities privately to
[security@aient.ai](mailto:security@aient.ai). Do not include credentials,
customer data, or exploit details in a public GitHub issue.

## Release identity

`aient-ai/aient-cli` is a binary-only public distribution repository. Aient
builds releases from the private `haf/glimt` repository with GitHub Actions,
then mirrors the resulting bytes without rebuilding or re-signing them.

The expected Sigstore identity for a release tag is therefore:

```text
https://github.com/haf/glimt/.github/workflows/aient-cli-release.yml@refs/tags/aient-cli-v<VERSION>
```

with OIDC issuer:

```text
https://token.actions.githubusercontent.com
```

Verification must use that exact identity. Do not replace it with a permissive
regular expression. The SLSA source URI is likewise `github.com/haf/glimt`.
The private repository identity is public in the transparency and provenance
records; private source content is not.

GitHub's generated source archives contain only this repository's public
documentation and are not part of the verified Aient CLI release.
