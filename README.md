# Signing Certificate Expiry Planner

A single-file browser tool that turns a Windows code signing certificate's validity dates into the dates you actually have to act on: when each untimestamped artifact goes dark, when every future renewal falls, and the last safe day to ship a bridge release.

**Live demo:** https://0xelitesystem.github.io/signing-cert-expiry-planner/

This is the arithmetic half of [windows-code-signing-and-smartscreen-reference](https://github.com/0xelitesystem/windows-code-signing-and-smartscreen-reference), which is the prose half: that repo explains what Authenticode, timestamping and SmartScreen reputation do, and this one does the date maths those rules imply. It is also not a generic date subtractor. The catalog already has [date-span-business-days](https://github.com/0xelitesystem/date-span-business-days) for the difference between two dates; this page encodes three specific signing rules on top of that subtraction.

## Use

1. Enter the certificate's `notBefore` and `notAfter` in `YYYY-MM-DD`. If you only know `notBefore` and the issue date, press **Fill notAfter from the cap** and the tool applies the CA/Browser Forum rule that matches the issue date.
2. Enter your release cadence in weeks, the share of the installed base you want carrying the bridge release, and your own per-cycle uptake assumption.
3. Paste your shipped artifacts, one per line, as `name, timestamped`. The flag accepts `yes`/`no`, `y`/`n`, `true`/`false`, `1`/`0`. Lines beginning with `#` are ignored.
4. Set the renewal ladder's start date and how many cycles to project.

You get four outputs:

- **Remaining signing life.** Days left from today (UTC), total validity in days, percent elapsed, and a check of your `notAfter` against the cap that applies to your issue date.
- **Per-artifact go-dark dates.** Untimestamped rows get the exact date they read as unsigned to Windows. Timestamped rows read `unaffected`. If every row is timestamped the page says `0 artifacts affected - timestamping did its job` rather than inventing urgency.
- **The bridge-release deadline**, with the arithmetic printed so you can check it.
- **The renewal ladder**, computed live from your start date: every future renewal date, its cycle length in days, which cap rule applied, and how far it has drifted from the N-year mark.

### The three rules the tool encodes

**1. Without a timestamp, the signature dies with the certificate.** Microsoft, on "Time Stamping Authenticode Signatures":

> "Always time-stamp your Authenticode signatures. Without a time stamp, the signature becomes invalid when the signing certificate expires, and Windows will treat the binary as unsigned. Time stamping ensures long-term signature validity."

and, on the same page:

> "The countersignature method of time stamping implemented below allows for signatures to be verified even after the signing certificate has expired or been revoked."

Source: https://learn.microsoft.com/en-us/windows/win32/seccrypto/time-stamping-authenticode-signatures

Stated in my own words rather than Microsoft's, because it is the part people get wrong: signature validation happens at verify time, on the verifying machine, not at download time. A copy already on a user's disk or on a mirror is re-checked when it is next opened, so it starts failing on the same day a fresh download does. Nothing has to be re-published for that to happen.

**2. The validity cap, and which one applies to you.** CA/Browser Forum, Baseline Requirements for the Issuance and Management of Publicly-Trusted Code Signing Certificates, section 6.3.2:

> "For Code Signing Certificates issued before March 1st, 2026, the validity period MUST NOT exceed 39 months. For Code Signing Certificates issued on or after March 1st, 2026, the validity period MUST NOT exceed 460 days."

Sources: https://github.com/cabforum/code-signing/blob/main/docs/CSBR.md and https://cabforum.org/working-groups/code-signing/requirements/

39 months is a calendar span, not a fixed number of days, so the tool adds 39 months with month-end clamping and reports the day count that results from your dates instead of printing a constant. 460 days is a fixed count and is applied as one. Because 460 days is longer than a year, each renewal under the new cap falls *later* in the calendar than the last, not earlier, and the ladder shows exactly how much later for your dates.

**3. The bridge release, and why the deadline is not the expiry date.** electron-updater checks a downloaded installer against the `publisherName` list compiled into the build the user already has, so a build that only knows the old publisher name will refuse an installer signed with a new certificate whose subject differs. The fix is a release carrying both names, signed while the old certificate is still valid, and it only helps the users who install it. The tool works backwards from `notAfter` using your cadence and your own uptake assumption, and prints the working.

### Scope

Windows Authenticode only. Apple Developer ID is Apple's own PKI with 5-year certificate validity and sits outside the CA/Browser Forum code signing requirements, so none of these caps apply to it.

This tool is independent and unaffiliated. It is not endorsed by, affiliated with, or connected to Microsoft, the CA/Browser Forum, or any certificate authority. The quotations above are short citations of published specifications, attributed inline with their sources. No code from any third party is reproduced here; the rules were read from the published specifications and implemented as date arithmetic.

## Why this exists

I have not bought a code signing certificate yet. This is the arithmetic I wanted in front of me before spending the money, and I could not find it anywhere as a calculator.

Two of the four outputs exist elsewhere as prose you have to apply by hand. The renewal ladder does not exist anywhere I could find, and it is the one nobody can do in their head: under a 460-day cap the renewal date walks about three months forward through the calendar every cycle, so "renew every March" stops being true after the first renewal and the drift compounds. The tool computes the whole ladder from whatever start date you type, including which cycle brings your start month back around and which cycle brings your start day of the month back.

It is one HTML file with no dependency, no build step, no tracker, no network call and no server. Read it before you run it; that is the point of shipping it as one file under MIT.

## Privacy

Everything runs in your browser. Nothing leaves your machine. There is no network request of any kind: no CDN, no font, no analytics, no telemetry, no upload. Certificate dates and artifact names you paste in stay in the tab and are gone when you close it. The only thing written to storage is your light or dark theme choice, in `localStorage`.

## Run locally

```
git clone https://github.com/0xelitesystem/signing-cert-expiry-planner.git
cd signing-cert-expiry-planner
```

Then open `index.html` in a browser. That is enough; the file works straight off disk over `file://`.

If you would rather serve it:

```
python -m http.server 8000
```

and open http://localhost:8000/.

## Build

There is no build. It is one self-contained `index.html` with the CSS and JavaScript inline. No package manager, no bundler, no dependency to install or audit.

## Related

- [windows-code-signing-and-smartscreen-reference](https://github.com/0xelitesystem/windows-code-signing-and-smartscreen-reference) - the prose half: what Authenticode, timestamping and SmartScreen reputation actually do.
- [date-span-business-days](https://github.com/0xelitesystem/date-span-business-days) - plain date and time differences, with none of the signing rules layered on.
- [timezone-and-utc-basics-reference](https://github.com/0xelitesystem/timezone-and-utc-basics-reference) - why this tool does every calculation in UTC.

## License

MIT. See [LICENSE](LICENSE).
