---
description: Set up the virtual secretary (Fio tokens, configuration)
thinking: enabled
---

Help the user set up their virtual secretary. Walk through each step interactively.

## Step 1: Create the secretary directory

Run this command to create the data directory:

```bash
mkdir -p ~/.secretary
```

## Step 2: Fio API tokens

The user needs to generate API tokens in Fio internetbanking:
1. Log into https://ib.fio.cz
2. Go to Nastaveni -> API
3. Click "Pridat novy token"
4. Select "Sledovani uctu a zadavani platebnich prikazu"
5. Authorize via SMS
6. Wait 5 minutes, then copy the token

Ask the user for:
- **Business account token** and account number (format: 123456/2010)
- **Personal account token** and account number

Save to `~/.secretary/fio-tokens.json`:
```json
{
  "business": {
    "token": "<user-provided-token>",
    "accountNumber": "<user-provided-account-number>"
  },
  "personal": {
    "token": "<user-provided-token>",
    "accountNumber": "<user-provided-account-number>"
  }
}
```

## Step 3: Configuration

Ask the user if they want to change any defaults, then save `~/.secretary/config.json`:
```json
{
  "autoPayLimit": 5000,
  "defaultAccount": "business"
}
```

## Step 4: Initialize empty data files

```bash
echo '[]' > ~/.secretary/trusted_vendors.json
echo '[]' > ~/.secretary/pending_payments.json
```

## Step 5: Verify Gmail MCP

Test that Gmail MCP is working by fetching the user's profile. If it fails, guide them through Gmail OAuth setup per SETUP.md.

## Step 6: Verify Fio API

Test both Fio tokens by fetching account info:
```bash
curl -s "https://fioapi.fio.cz/v1/rest/last/<token>/transactions.json" | head -20
```

If either fails, help the user troubleshoot (wrong token, token not yet active, etc.)

## Step 7: Summary

Print a summary of what was configured and suggest running `/secretary` to do a first inbox pass.
