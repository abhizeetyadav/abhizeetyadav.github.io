---
author: Abhijeet Yadav
pubDatetime: 2026-07-14T15:44:47+05:45
modDatetime: 2026-07-14T15:44:47+05:45

title: Automating Cloudflare WAF - Blocking Unverified Bot Scanners on Basic Accounts
featured: false
draft: false
tags:
  - cloudflare
  - python
  - security
  - devops
  - waf
canonicalURL: https://smale.codes/posts/automating-cloudflare-waf/
description: How to deploy a robust, tier-aware Cloudflare WAF rule using Python to block unverified bots and scanners on basic accounts.
---

## Table of Contents

- The Problem with Unverified Bots
- Why This Script is Different
- The Python Script
- How to Roll It Out Safely

---

# The Problem with Unverified Bots

If you have a public-facing endpoint, it is a guarantee that automated bots, scrapers, and vulnerability scanners are actively probing your servers. They are constantly hunting for exposed secrets—looking for `.env` files, `.git` directories, `wp-config.php`, and `docker-compose.yml` configurations. 

When you are actively managing containerized environments like Docker or k3s, or running CI/CD pipelines through Jenkins, the last thing you want is junk bot traffic exhausting your server resources. Every malicious request that bypasses the edge means an unnecessary active connection hitting your backend, potentially triggering unwanted pod scaling or bogging down your PostgreSQL and Redis databases.

To stop this at the edge, I wrote a Python deployment script that automatically creates and updates a Cloudflare WAF rule across all your zones. 

---

## Why This Script is Different

There are plenty of Cloudflare API wrappers out there, but many of them fail if you are on a Free or Pro tier, or they fall into common automation traps. I built this script specifically to navigate those pitfalls:

1. **Tier-Aware & Cost-Effective:** Many scripts fail on basic accounts because they rely on the regex `matches` operator (an Enterprise feature) or try to use the `log` action where it isn't entitled. This script uses a chained `contains` expression available on all plans and defaults to `managed_challenge` to safely stop scanners without blocking real users.
2. **Jupyter/Colab Safe:** If you run infrastructure playbooks from notebook environments, hardcoding API tokens is a major security risk. This script uses `getpass` to prompt for hidden input, ensuring your token never ends up saved in shell history or notebook cell source code.
3. **Idempotent Execution:** It uses a fixed description as a state key. Re-running the script updates the existing rule in place rather than blindly appending duplicates and cluttering your Cloudflare dashboard.
4. **Avoids Deprecated Traps:** It completely avoids `cf.threat_score`. Many older tutorials still use this field, but it is deprecated, hardcoded to 0, and will cause your WAF rules to silently fail.
5. **Resilient Network Logic:** Cloudflare caps the API at 1,200 requests per 5 minutes. This script implements a robust `urllib3` retry adapter with exponential backoff to handle 429 rate limits gracefully.

---

## 🐍 The Python Script

Below is the complete deployment script. You can run it directly from your terminal or drop it into a Jupyter Notebook cell.

```python
#!/usr/bin/env python3
"""
Cloudflare WAF — Block Unverified Bot Scanners
================================================

Deploys/updates a single custom WAF rule across every zone (or a chosen
subset) in a Cloudflare account. The rule blocks requests that are NOT from
a verified good bot (Googlebot, Bingbot, etc.) AND are probing for common
secret/config-file paths.
"""

import argparse
import getpass
import os
import sys
import time
from typing import Any, Dict, List, Optional, Tuple

import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

BASE_URL = "https://api.cloudflare.com/client/v4"
RULE_DESCRIPTION = "Block unverified bot scanners (managed by cf_waf_deploy.py)"

BAD_PATH_SIGNATURES = [
    ".env", ".git", "wp-config", "setup.php", ".htpasswd",
    "phpinfo.php", "id_rsa", ".aws/credentials", 
    "docker-compose.yml", "xmlrpc.php", "eval-stdin.php", ".DS_Store",
]

VALID_ACTIONS = ["log", "block", "managed_challenge", "js_challenge", "challenge"]

class CloudflareAPIError(Exception):
    def __init__(self, status_code: int, errors: Any):
        self.status_code = status_code
        self.errors = errors
        super().__init__(f"HTTP {status_code}: {errors}")

def build_expression() -> str:
    path_checks = " or ".join(
        f'http.request.uri.path contains "{p}"' for p in BAD_PATH_SIGNATURES
    )
    return f"(not cf.client.bot and ({path_checks}))"

def get_token() -> str:
    token = os.environ.get("CLOUDFLARE_API_TOKEN", "").strip()
    if token:
        return token

    print("CLOUDFLARE_API_TOKEN is not set in the environment.")
    try:
        token = getpass.getpass("Paste your Cloudflare API token (hidden input): ").strip()
    except Exception:
        token = ""

    if not token:
        raise SystemExit("No token provided.")
    return token

def get_session(token: str) -> requests.Session:
    session = requests.Session()
    retry = Retry(
        total=5, backoff_factor=2,
        status_forcelist=[429, 500, 502, 503, 504],
        allowed_methods=["GET", "POST", "PUT", "DELETE"],
        respect_retry_after_header=True,
    )
    session.mount("https://", HTTPAdapter(max_retries=retry))
    session.headers.update({"Authorization": f"Bearer {token}", "Content-Type": "application/json"})
    return session

def cf_request(session: requests.Session, method: str, path: str, **kwargs) -> Dict[str, Any]:
    resp = session.request(method, f"{BASE_URL}{path}", timeout=30, **kwargs)
    try:
        data = resp.json()
    except ValueError:
        resp.raise_for_status()
        raise RuntimeError(f"Non-JSON response (HTTP {resp.status_code})")

    if not data.get("success", False):
        raise CloudflareAPIError(resp.status_code, data.get("errors"))
    return data

def get_zones(session: requests.Session) -> List[Dict[str, Any]]:
    zones: List[Dict[str, Any]] = []
    page = 1
    while True:
        data = cf_request(session, "GET", "/zones", params={"page": page, "per_page": 50})
        zones.extend(data["result"])
        info = data.get("result_info", {})
        if page >= info.get("total_pages", 1):
            break
        page += 1
    return zones

def get_custom_ruleset_rules(session: requests.Session, zone_id: str) -> List[Dict[str, Any]]:
    try:
        data = cf_request(session, "GET", f"/zones/{zone_id}/rulesets/phases/http_request_firewall_custom/entrypoint")
        return data["result"].get("rules", [])
    except CloudflareAPIError as e:
        if e.status_code == 404:
            return []
        raise

def upsert_rule(session: requests.Session, zone_id: str, expression: str, action: str, dry_run: bool) -> str:
    rules = get_custom_ruleset_rules(session, zone_id)
    idx = next((i for i, r in enumerate(rules) if r.get("description") == RULE_DESCRIPTION), None)

    new_rule = {
        "description": RULE_DESCRIPTION,
        "expression": expression,
        "action": action,
        "enabled": True,
    }

    if idx is not None:
        current = rules[idx]
        if current.get("expression") == expression and current.get("action") == action:
            return "unchanged"
        rules[idx] = new_rule
        outcome = "updated"
    else:
        rules.append(new_rule)
        outcome = "created"

    if dry_run:
        return f"dry-run-{outcome}"

    cf_request(session, "PUT", f"/zones/{zone_id}/rulesets/phases/http_request_firewall_custom/entrypoint", json={"rules": rules})
    return outcome

def enable_bot_fight_mode(session: requests.Session, zone_id: str) -> str:
    try:
        cf_request(session, "PATCH", f"/zones/{zone_id}/bot_management", json={"fight_mode": True})
        return "enabled"
    except CloudflareAPIError as e:
        return f"skipped ({e.errors})"

def run(action: str = "managed_challenge", dry_run: bool = False, yes: bool = False, zones: Optional[List[str]] = None, enable_bfm: bool = False) -> None:
    token = get_token()
    session = get_session(token)
    expression = build_expression()
    
    print("Fetching zones...")
    all_zones = get_zones(session)
    target_zones = all_zones
    
    if zones:
        wanted = set(zones)
        target_zones = [z for z in all_zones if z["name"] in wanted]
    
    if not target_zones:
        print("No matching zones found.")
        return

    print(f"\n{len(target_zones)} zone(s) targeted. action={action} dry_run={dry_run}")
    
    if not dry_run and not yes:
        if input(f"Proceed? [y/N] ").strip().lower() != "y":
            return

    for zone in target_zones:
        try:
            outcome = upsert_rule(session, zone["id"], expression, action, dry_run)
            print(f"OK     {zone['name']}: {outcome}")
        except Exception as e:
            print(f"FAIL   {zone['name']}: {e}")
        time.sleep(0.1)

def main() -> None:
    p = argparse.ArgumentParser()
    p.add_argument("--action", choices=VALID_ACTIONS, default="managed_challenge")
    p.add_argument("--dry-run", action="store_true")
    p.add_argument("--yes", action="store_true")
    p.add_argument("--zone", action="append", default=None)
    args, _ = p.parse_known_args()
    
    run(action=args.action, dry_run=args.dry_run, yes=args.yes, zones=args.zone)

if __name__ == "__main__":
    main()
```

---

## 🚀 How to Roll It Out Safely

When deploying WAF rules, it is always best to observe before you block. 

### 1. Create your API Token
In the Cloudflare dashboard, create a token with `Zone / Firewall Services / Edit` permissions.

### 2. Test Run
Run this to see what will happen without making actual changes:
```bash
python3 cf_waf_deploy.py --action managed_challenge --dry-run
```

### 3. Challenge Mode (Recommended First Step)
This puts a Captcha/JS challenge in front of suspicious requests. It is safe for real users.
```bash
python3 cf_waf_deploy.py --action managed_challenge
```
*Check your Cloudflare Security Events dashboard after 24 hours to review what was caught.*

### 4. Enforce the Block
Once you are confident there are no false positives blocking legitimate traffic, enforce the hard block:
```bash
python3 cf_waf_deploy.py --action block
```

---

## 🏁 Final Thoughts

Keeping bots away from your edge shouldn't require enterprise budgets or brittle scripts. Give it a try, block those unverified scanners, and reclaim your server resources!

**Until next time,** **Happy automating!**