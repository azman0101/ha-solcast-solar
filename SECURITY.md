# Security Policy

This policy applies to the azman0101-maintained fork. The current `main` branch
receives best-effort maintenance; there are no fork releases at the time of the
initial adoption review. The inherited integration version `4.6.3` is an upstream
baseline, not a promise to support upstream releases.

## Reporting a vulnerability

Use this repository's **Security → Report a vulnerability** option if available.
If private reporting is unavailable, open an issue at
https://github.com/azman0101/ha-solcast-solar/issues with a high-level description
and ask for a private reporting channel before sending sensitive details.

Do not publish API keys, credentials, precise coordinates or unredacted logs.
Logs are useful but are not a prerequisite for reporting a security concern.
The `solcast_solar.get_options` action currently returns API keys unredacted;
treat its response and any automation traces containing it as sensitive.

Report vulnerabilities in Home Assistant or Solcast itself through those
projects' own security channels. Fork-specific reports belong in this repository.
