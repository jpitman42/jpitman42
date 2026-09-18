# Password Strength & Breach Checker

A dependency-free Python CLI that checks a password two ways:

1. **Strength** — offline analysis of length, character variety, common
   patterns, and estimated entropy.
2. **Breach status** — checks the password against
   [HaveIBeenPwned's Pwned Passwords](https://haveibeenpwned.com/Passwords)
   database of 800M+ breached passwords, using **k-anonymity** so your
   real password (and even its full hash) is never sent anywhere.

## Why k-anonymity matters

The naive way to check "has this password leaked?" would be to send the
password (or its hash) to an API. That means trusting a third party with
every password you ever check — a bad trade-off for a security tool.

Instead, this project uses the same technique as Chrome, Firefox, and
1Password:

1. Hash the password locally with SHA-1.
2. Send only the **first 5 characters** of that hash to the API.
3. The API returns every suffix in its database sharing that 5-character
   prefix — typically several hundred unrelated hashes.
4. Check locally whether your full hash is among them.

The API never sees your password. It never even sees your full hash —
just a prefix shared by hundreds of other people's passwords too.

## Quick start

```bash
git clone https://github.com/yourusername/password-checker.git
cd password-checker

# Interactive (recommended) — input is hidden as you type
python3 password_checker.py

# Strength check only, no network call at all
python3 password_checker.py --offline
```

### Example output

```
Enter password to check (input hidden):
------------------------------------------------------------
STRENGTH ANALYSIS
------------------------------------------------------------
Length:            14 characters
Estimated entropy: ~91.8 bits
Checks passed:     6/6
Rating:            Strong

No issues found in offline checks.

------------------------------------------------------------
BREACH CHECK (HaveIBeenPwned, k-anonymity)
------------------------------------------------------------
Good news: this password was NOT found in known breach data.
------------------------------------------------------------
```

## Usage

| Flag              | Description                                                        |
|-------------------|---------------------------------------------------------------------|
| *(no flags)*      | Prompts for a password with hidden input (`getpass`)                |
| `--offline`       | Skip the breach check; strength analysis only, no network call      |
| `--password PW`   | Pass the password directly (see warning below)                      |

> **Warning:** `--password` is provided for scripting/testing convenience,
> but any argument passed on the command line can end up in your shell
> history and in process listings visible to other users on the same
> machine (`ps aux`). Prefer the interactive prompt whenever possible.

## How strength is scored

Six offline checks, each worth one point:

- Length ≥ 12 characters
- Uses 3+ of: lowercase, uppercase, digits, symbols
- Not in a built-in list of extremely common passwords
- No sequential run (`abcd`, `4321`, etc.)
- No repeated-character run (`aaaa`, etc.)
- Estimated entropy ≥ 60 bits

Score → rating: 0–2 Very Weak · 3 Weak · 4 Fair · 5 Good · 6 Strong.
(A password on the common-password list is always rated Very Weak,
regardless of other checks.)

Entropy is estimated as `length × log2(character pool size)`. This is a
simplification — it assumes fully random character selection, which real
human-chosen passwords rarely are — but it's a useful, explainable signal.

## Testing

```bash
python3 -m unittest test_password_checker.py -v
```

Strength tests run for real. Breach-check tests mock the network layer
(via `unittest.mock`) so they run offline and verify the k-anonymity
matching logic — including an explicit test that asserts neither the
plaintext password nor the full hash ever appears in the outgoing request.

## Project structure

```
password-checker/
├── password_checker.py       # Main CLI (strength + breach check)
├── test_password_checker.py  # Unit tests (strength + mocked breach checks)
├── README.md
├── LICENSE
└── .gitignore
```

## Limitations & ideas for extending

- The entropy estimate is a simplification; a stronger approach would use
  a tool like `zxcvbn` (pattern-based strength estimation), at the cost
  of adding a dependency.
- The common-password list is intentionally tiny for readability; a real
  deployment might load the top 10k–100k from a bundled file.
- Could add a `--batch file.txt` mode to check many passwords at once
  (e.g. auditing an exported password list before a migration).
- Could add support for checking against a custom/internal breach list,
  not just HaveIBeenPwned.
