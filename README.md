# CodePath AI301 Open Source Contribution

## Contributor

Hi, my name is Ahmed Mohamed. I am an IT student at George Mason University with double concentrations in AI and Cybersecurity. This repository documents my CodePath AI301 Open Source Capstone contribution process.

For additional background, see my [resume](docs/Ahmed_Mohamed_Resume.pdf).

## Selected Issue

I selected Issue #165 from `GRCEngClub/claude-grc-engineering`: Add STARTTLS protocol support to testssl-inspector.

- Issue: https://github.com/GRCEngClub/claude-grc-engineering/issues/165
- Fork: https://github.com/ahmedhxssam/claude-grc-engineering
- Working branch: https://github.com/ahmedhxssam/claude-grc-engineering/tree/fix-165-starttls-testssl
- Pull request: https://github.com/GRCEngClub/claude-grc-engineering/pull/197

## Why I Selected This Issue

This issue fits my AI and Cybersecurity interests because it involves TLS inspection, secure network communication, and GRC/security tooling. The task is scoped as a good first issue and focuses on adding STARTTLS support to an existing connector, which is a practical security feature for scanning email, directory, FTP, and database services.

The issue was also a good fit for my CodePath AI301 Open Source Capstone because it required understanding an existing code path, reproducing the current behavior, planning a small implementation, updating documentation, and submitting a real open-source pull request.

## Current Status

I was vouched by the maintainer for Issue #165 and opened PR #197.

The pull request adds STARTTLS support to the `testssl-inspector` connector by parsing `--starttls=<proto>`, validating supported protocols, applying protocol-specific default ports, preserving explicit ports, passing `--starttls <proto>` through to `testssl.sh`, and updating the related documentation.

## Phase II: Reproduce and Plan

### Reproduction Process

I cloned my fork of the repository, added the upstream repository as a remote, synced my local `main` branch with upstream `main`, and created a dedicated working branch named `fix-165-starttls-testssl`.

Commands used:

```bash
git clone https://github.com/ahmedhxssam/claude-grc-engineering.git
cd claude-grc-engineering

git remote add upstream https://github.com/GRCEngClub/claude-grc-engineering.git
git fetch upstream
git checkout main
git merge upstream/main

git checkout -b fix-165-starttls-testssl
git push -u origin fix-165-starttls-testssl
```

I confirmed that I was on the correct working branch:

```bash
git branch
```

Output:

```text
* fix-165-starttls-testssl
  main
```

I then reproduced the issue by running the scan command with the requested STARTTLS flag:

```bash
node plugins/connectors/testssl-inspector/scripts/scan.js --target=mail.example.com --starttls=smtp --no-docker
```

Actual output before the fix:

```text
[testssl-inspector] unknown flag: --starttls=smtp
```

Exit code:

```text
2
```

This confirmed that the current wrapper rejected `--starttls=smtp` during argument parsing before it could reach `testssl.sh`.

I also tested an unsupported protocol value:

```bash
node plugins/connectors/testssl-inspector/scripts/scan.js --target=mail.example.com --starttls=bogus
echo $?
```

Actual output before the fix:

```text
[testssl-inspector] unknown flag: --starttls=bogus
2
```

This showed that `--starttls` itself was not supported yet, regardless of whether the protocol value was valid or invalid.

### Reproduction Evidence

Working branch: https://github.com/ahmedhxssam/claude-grc-engineering/tree/fix-165-starttls-testssl

Pull request: https://github.com/GRCEngClub/claude-grc-engineering/pull/197

The initial reproduction showed that `--starttls=smtp` failed as an unknown flag. After implementing the fix, I verified that the flag is accepted and reaches runner/tool resolution. Since `testssl.sh` was not installed locally, the command stopped at the expected tool-resolution step instead of performing a live scan.

Post-fix happy path command:

```bash
node plugins/connectors/testssl-inspector/scripts/scan.js --target=mail.example.com --starttls=smtp --no-docker --output=json
echo $?
```

Post-fix output:

```text
[testssl-inspector] testssl.sh not on PATH. Install via: brew install testssl (macOS), apt install testssl.sh (Debian/Ubuntu), or git clone https://github.com/testssl/testssl.sh ~/.local/share/testssl.sh && export PATH="$HOME/.local/share/testssl.sh:$PATH". Or rerun with --docker.
3
```

This confirms that `--starttls=smtp` is no longer rejected as an unknown flag.

I also verified that invalid protocols now fail with the intended usage error:

```bash
node plugins/connectors/testssl-inspector/scripts/scan.js --target=mail.example.com --starttls=bogus --output=json
echo $?
```

Output:

```text
[testssl-inspector] unknown STARTTLS protocol: bogus. Supported protocols: smtp, imap, pop3, ftp, ldap, postgres, mysql, smtps
2
```

### Implementation Plan

#### Understand

The `testssl-inspector` connector previously supported HTTPS endpoint scans but did not expose `testssl.sh` STARTTLS support through the wrapper. The issue requested a new `--starttls=<proto>` flag so users can scan mail, FTP, LDAP, and database services that use STARTTLS.

The expected behavior is that valid STARTTLS protocols should be accepted, passed through to `testssl.sh`, and given a default port when the user does not provide one. Unknown protocols should fail with `EXIT.USAGE`, which is exit code `2`.

#### Match

The relevant logic already existed in:

- `plugins/connectors/testssl-inspector/scripts/scan.js`
- `plugins/connectors/testssl-inspector/commands/scan.md`
- `plugins/connectors/testssl-inspector/README.md`

The script already had a `parseArgs()` function for CLI flags, a `testsslArgs()` function for building the `testssl.sh` command, and runner logic for both Docker and non-Docker execution. The STARTTLS feature fit into those existing patterns.

#### Plan

1. Add a `STARTTLS_PORTS` lookup table for supported protocols and default ports.
2. Extend `parseArgs()` to accept `--starttls=<proto>`.
3. Normalize protocol values to lowercase.
4. Reject unsupported protocols with `EXIT.USAGE`.
5. Add helper logic to append the default port when a STARTTLS target does not include an explicit port.
6. Preserve explicit ports such as `mail.example.com:587`.
7. Pass `--starttls <proto>` into the `testssl.sh` argument list before the target.
8. Wire the STARTTLS argument through both Docker and direct `testssl.sh` runners.
9. Update `commands/scan.md` with the new option and examples.
10. Update the README scope section so it no longer says STARTTLS is unsupported.
11. Verify syntax, fixture contracts, happy-path parsing, explicit port handling, and invalid protocol handling.

#### Implement

The implementation modified three files:

- `plugins/connectors/testssl-inspector/scripts/scan.js`
- `plugins/connectors/testssl-inspector/commands/scan.md`
- `plugins/connectors/testssl-inspector/README.md`

The final diff summary was:

```text
3 files changed, 72 insertions(+), 15 deletions(-)
```

#### Review

I reviewed the diff to make sure only the intended files were changed. The implementation remained scoped to STARTTLS support and did not modify schemas, fixtures, dependencies, or unrelated connector behavior.

#### Evaluate

I ran the required verification commands.

Syntax check:

```bash
node --check plugins/connectors/testssl-inspector/scripts/scan.js
echo $?
```

Result:

```text
0
```

Contract fixture validation:

```bash
bash tests/validate-contract-fixtures.sh
echo $?
```

Result:

```text
All 49 fixture(s) valid.
0
```

Happy-path STARTTLS argument parsing:

```bash
node plugins/connectors/testssl-inspector/scripts/scan.js --target=mail.example.com --starttls=smtp --no-docker --output=json
echo $?
```

Result:

```text
[testssl-inspector] testssl.sh not on PATH. Install via: brew install testssl (macOS), apt install testssl.sh (Debian/Ubuntu), or git clone https://github.com/testssl/testssl.sh ~/.local/share/testssl.sh && export PATH="$HOME/.local/share/testssl.sh:$PATH". Or rerun with --docker.
3
```

Explicit port preservation smoke test:

```bash
node plugins/connectors/testssl-inspector/scripts/scan.js --target=mail.example.com:587 --starttls=smtp --no-docker --output=json
echo $?
```

Result:

```text
[testssl-inspector] testssl.sh not on PATH. Install via: brew install testssl (macOS), apt install testssl.sh (Debian/Ubuntu), or git clone https://github.com/testssl/testssl.sh ~/.local/share/testssl.sh && export PATH="$HOME/.local/share/testssl.sh:$PATH". Or rerun with --docker.
3
```

Unknown protocol path:

```bash
node plugins/connectors/testssl-inspector/scripts/scan.js --target=mail.example.com --starttls=bogus --output=json
echo $?
```

Result:

```text
[testssl-inspector] unknown STARTTLS protocol: bogus. Supported protocols: smtp, imap, pop3, ftp, ldap, postgres, mysql, smtps
2
```

These checks confirm that the new flag is accepted, invalid protocols fail with exit code `2`, and existing fixture contracts still pass.

## Phase II Deliverables Summary

| Deliverable | Status | Link or Location |
|---|---:|---|
| Reproduction steps | Complete | This README |
| Working branch link | Complete | https://github.com/ahmedhxssam/claude-grc-engineering/tree/fix-165-starttls-testssl |
| Solution plan | Complete | This README |
| Pull request | Open | https://github.com/GRCEngClub/claude-grc-engineering/pull/197 |
| Phase II check-in | Ready to submit | CodePath AI301 portal |
