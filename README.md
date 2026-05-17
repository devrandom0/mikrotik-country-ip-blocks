# mikrotik-country-ip-blocks

Pre-split IPv4 address blocks by country, formatted for MikroTik firewall address-lists.

## How it works

A GitHub Actions workflow ([`.github/workflows/update-ip-blocks.yml`](.github/workflows/update-ip-blocks.yml)) runs every Monday at 02:00 UTC (and on every push to `countries.txt`). For each country code listed in [`countries.txt`](countries.txt) it:

1. Downloads aggregated IPv4 ranges from [ipverse/country-ip-blocks](https://github.com/ipverse/country-ip-blocks)
2. Skips regeneration if the source hasn't changed (checksum comparison)
3. Splits the list into chunks of 450 CIDRs and writes them as `<cc>/<cc>_NN.rsc` files
4. Commits and pushes only when something changed

## Repository layout

```
countries.txt          — one ISO 3166-1 alpha-2 country code per line
<cc>/
  .checksum            — SHA-256 of the last downloaded source (used to skip unchanged countries)
  count.txt            — total number of IP ranges across all chunks
  <cc>_01.rsc          — plain CIDR list, chunk 1
  <cc>_02.rsc          — chunk 2 …
update-ir-ip-range     — MikroTik RouterOS script (see below)
```

Currently tracked: `ir` (Iran).

## Adding a country

Add its ISO 3166-1 alpha-2 code (lowercase) to `countries.txt` and push. The workflow runs automatically on that push and generates the `.rsc` files.

## MikroTik import script

[`update-ir-ip-range`](update-ir-ip-range) is a RouterOS script that:

- Fetches the remote `.checksum` first — skips the entire update if the data hasn't changed
- Fetches `count.txt` and verifies the imported range count matches; warns on mismatch and resumes incomplete imports
- Discovers `.rsc` files via the GitHub API, fetches each one, and upserts every CIDR into a named firewall address-list
- Tags entries with the remote checksum (`auto-<country>-<checksum>`) so stale entries from previous runs are cleaned up automatically
- Supports an `excludeList` to skip specific CIDRs
- Supports a `removeList` flag to wipe the address-list instead of updating
- Supports a `forceUpdate` flag to bypass the checksum skip
- Supports an `enableLogs` flag for verbose per-entry logging

### Quick start

Paste the script into **System → Scripts** on your MikroTik router (or run it from the terminal). Adjust the top variables if needed:

```routeros
:local country     "ir"      # country subdirectory / address-list name
:local listName    "ir"      # firewall address-list to write into
:local excludeList ({})      # CIDRs to skip, e.g. {"5.22.0.0/17"; "5.56.128.0/21"}
:local removeList  false     # set true to delete the list instead of updating
:local forceUpdate false     # set true to re-import even if checksum is unchanged
:local enableLogs  false     # set true for verbose per-entry log messages
```

The script requires outbound HTTPS access from the router to `api.github.com` and `raw.githubusercontent.com`.

## Sources

IPv4 data: [ipverse/country-ip-blocks](https://github.com/ipverse/country-ip-blocks) (aggregated, updated weekly).
