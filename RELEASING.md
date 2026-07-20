# Releasing

Releases are published from GitHub Actions. The workflow uploads the crate,
verifies the archive served by crates.io, and only then creates the matching
`vX.Y.Z` tag and GitHub release. Do not run `cargo publish` or create the tag
locally.

## First publication: 0.1.0

Trusted Publishing cannot create a crate that does not exist on crates.io yet.
The first publication therefore uses a scoped crates.io API token while still
running the complete release from CI.

### Configure crates.io and GitHub

1. Sign in to crates.io with the account that will own `opaque-types`. Verify
   the account email if crates.io requests it.
2. Create a crates.io API token that is allowed to publish a new crate. Give it
   a short expiration because it is needed only for the first release.
3. In **GitHub → opaque-types → Settings → Environments**, create an
   environment named `crates-io`.
4. Add the API token to that environment as a secret named
   `CARGO_REGISTRY_TOKEN`. Do not set the
   `CRATES_IO_TRUSTED_PUBLISHING` variable yet.
5. Optionally add a required reviewer to the environment so publication needs
   explicit approval.

### Prepare and publish 0.1.0

1. Confirm that `Cargo.toml` and `Cargo.lock` contain version `0.1.0` and that
   `CHANGELOG.md` contains a `## 0.1.0` section.
2. Run the release checks locally:

   ```console
   cargo fmt --all -- --check
   cargo clippy --locked --all-targets --all-features -- -D warnings
   RUSTDOCFLAGS="-D warnings" cargo doc --locked --no-deps
   cargo test --locked --all-targets --all-features
   cargo package --locked
   cargo test --manifest-path target/package/opaque-types-0.1.0/Cargo.toml --all-targets --locked
   ```

3. Merge the release-preparation PR into `main` only after CI passes.
4. Open **Actions → Publish to crates.io → Run workflow**.
5. Select `main`, enter `0.1.0` without a `v` prefix, and run the workflow.
6. Approve the `crates-io` environment deployment if approval is required.
7. Confirm that the workflow created all three release artifacts:

   - `opaque-types` 0.1.0 on crates.io;
   - tag `v0.1.0` pointing to the published commit;
   - GitHub release `v0.1.0`.

The workflow verifies that the downloaded crates.io archive records the same
Git commit before it creates the tag. If publication succeeds but a later step
fails, rerun the workflow with `0.1.0`; it resumes without uploading the version
again.

## Switch to Trusted Publishing after 0.1.0

Once 0.1.0 exists on crates.io:

1. Open the `opaque-types` crate's **Settings → Trusted Publishing** page.
2. Add a GitHub Actions publisher with these exact values:

   - repository owner: `milyin`
   - repository: `opaque-types`
   - workflow: `publish.yml`
   - environment: `crates-io`

3. In the GitHub `crates-io` environment, add the variable
   `CRATES_IO_TRUSTED_PUBLISHING` with the value `true`.
4. Delete the `CARGO_REGISTRY_TOKEN` secret from the environment.
5. After one later release succeeds through Trusted Publishing, optionally
   enable Trusted-Publishing-Only mode in the crate settings.

Later publication jobs exchange GitHub's OIDC identity for a short-lived
crates.io token. No permanent crates.io credential remains in GitHub.

## Publish a later version

1. In a release-preparation PR:

   - update the version in `Cargo.toml`;
   - run `cargo check` so `Cargo.lock` records the new version;
   - add the matching `## <version>` section to `CHANGELOG.md`;
   - update the README and API documentation as needed;
   - run the local release checks shown above, substituting the new version in
     the packaged-crate path.

2. Merge the PR into `main` after CI passes.
3. Run **Actions → Publish to crates.io** from `main` with the exact manifest
   version, without the `v` prefix.
4. Confirm the crates.io version, `v<version>` tag, GitHub release, and docs.rs
   documentation.

## Recover from a partial release

Rerunning the workflow with the same version is safe:

- If crates.io does not contain the version, the workflow publishes it.
- If it is already published from the current commit, publication is skipped
  and release creation resumes.
- If it was published from another commit, the workflow stops.
- An existing tag is reused only if it points to the published commit.

Published versions cannot be replaced. Correct a bad release by publishing a
new version; never move a published version's tag.
