---
name: google-oauth2-playground-auth
description: Fetch a Google Workspace OAuth access token and refresh token for a given set of API scopes. Use this skill whenever a Google API operation requires authentication, an access token needs to be obtained or refreshed, or scopes need to be authorized before calling a Google Workspace API.
---

# Google OAuth2 Playground Auth

Fetch a Google Workspace OAuth access and refresh token using:

```bash
nix --extra-experimental-features "nix-command flakes" shell nixpkgs#nodejs -c npx -- github:dbertouille/google-oauth2-playground-auth -s 'SCOPES'
```

The `-c` flag passes the command to the nix shell, ensuring Node.js is available without requiring a global installation. `SCOPES` is a space-separated list of OAuth scope URLs.

## Usage

When a skill or task needs a Google API token, run the auth command with the required scopes:

```bash
# Single scope
nix --extra-experimental-features "nix-command flakes" shell nixpkgs#nodejs -c npx -- github:dbertouille/google-oauth2-playground-auth -s 'https://www.googleapis.com/auth/drive.readonly'

# Multiple scopes (space-separated, quoted)
nix --extra-experimental-features "nix-command flakes" shell nixpkgs#nodejs -c npx -- github:dbertouille/google-oauth2-playground-auth -s 'https://www.googleapis.com/auth/drive.readonly https://www.googleapis.com/auth/gmail.readonly'
```

The command will open a browser for the OAuth consent flow and print the resulting `access_token` and `refresh_token` to stdout.

## Common Scopes

| Service | Scope |
|---------|-------|
| Drive (read-only) | `https://www.googleapis.com/auth/drive.readonly` |
| Drive (full) | `https://www.googleapis.com/auth/drive` |
| Gmail (read-only) | `https://www.googleapis.com/auth/gmail.readonly` |
| Gmail (full) | `https://www.googleapis.com/auth/gmail.modify` |
| Calendar (read-only) | `https://www.googleapis.com/auth/calendar.readonly` |
| Calendar (full) | `https://www.googleapis.com/auth/calendar` |
| Sheets (read-only) | `https://www.googleapis.com/auth/spreadsheets.readonly` |
| Sheets (full) | `https://www.googleapis.com/auth/spreadsheets` |
| Docs (read-only) | `https://www.googleapis.com/auth/documents.readonly` |
| Docs (full) | `https://www.googleapis.com/auth/documents` |

## When to Use

- Before any Google API call that requires user authorization
- When an existing token has expired and a new one is needed
- When adding new scopes that were not included in a previous auth flow

## Notes

- Run the command directly with Bash — the browser will open on the user's machine automatically. Do not ask the user to run it themselves.
- Tokens are printed to stdout; capture and use them for subsequent API calls. Never log or display the access token or refresh token values.
- Request only the scopes actually needed — follow the principle of least privilege.
