# domainIQ API

The domainIQ API gives programmatic access to the tools on
[domainiq.com](https://www.domainiq.com): domain, registrant, email and IP
reports, reverse lookups (DNS, IP, MX, keyword), bulk WHOIS and DNS, domain
monitoring, Certificate Transparency data, domain continuity checks and
lookalike searches.

> **Base URL:** `https://domainiq-public-api.intelium.com/api2`
>
> All examples in this document use this endpoint. Requests are standard HTTP
> `GET` or `POST` calls with parameters in the query string or form body. The
> API key goes in the `X-API-Key` request header.

---

## Table of Contents

- [Access & Licensing](#access--licensing)
- [Authentication](#authentication)
- [Fallback Endpoint](#fallback-endpoint)
- [Request Format](#request-format)
- [Response Format](#response-format)
- [Errors](#errors)
- [Queued Requests](#queued-requests)
- [API Calls](#api-calls)
  - [Summary reports](#summary-reports)
    - [domain_report](#domain_report--domain-report)
    - [name_report](#name_report--registrant-name-report)
    - [organization_report](#organization_report--registrant-organization-report)
    - [email_report](#email_report--registrant-email-report)
    - [ip_report](#ip_report--ip-address-report)
  - [Reverse lookups](#reverse-lookups)
    - [domain_search](#domain_search--domain-keyword-search)
    - [reverse_search](#reverse_search--registrant-email-and-organization-wildcard-search)
    - [reverse_dns](#reverse_dns--reverse-dns-search)
    - [reverse_ip](#reverse_ip--reverse-ip-search)
    - [reverse_mx](#reverse_mx--reverse-mx-search)
  - [Bulk lookups](#bulk-lookups)
    - [bulk_dns](#bulk_dns--bulk-dns-lookup)
    - [bulk_whois](#bulk_whois--bulk-whois-lookup)
    - [bulk_whois_ip](#bulk_whois_ip--bulk-domain-ip-whois-lookup)
  - [Domain services](#domain-services)
    - [whois](#whois--whois-record)
    - [dns](#dns--dns-record-lookup)
    - [categorize](#categorize--domain-categorization)
    - [snapshot](#snapshot--domain-screenshot)
    - [snapshot_history](#snapshot_history--historical-screenshots)
    - [ct_names](#ct_names--certificate-transparency-names)
    - [cert_history](#cert_history--certificate-history)
  - [Domain history and lookalikes](#domain-history-and-lookalikes)
    - [domain_continuity](#domain_continuity--domain-continuity-check)
    - [lookalike_search](#lookalike_search--lookalike-search)
  - [Monitoring](#monitoring)
    - [Reading monitors](#reading-monitors)
    - [Managing monitors](#managing-monitors)
  - [Account](#account)
    - [limits](#limits--account-limits)
    - [queue](#queue--queued-request-status-and-results)
- [Related Services](#related-services)

---

## Access & Licensing

- The API requires a current, paid domainIQ account. All paid account types
  include API access.
- The API exposes the same features as the website and uses the **same
  account limits**. A query made through the API counts against your daily,
  monthly or count-based limits exactly as it would on the site.
- Some services (for example `reverse_search`) are only enabled for certain
  account types. [Contact us](https://www.domainiq.com/contact) to enable
  them.
- Use the [`limits`](#limits--account-limits) call, or your
  [Limits page](https://www.domainiq.com/account_limits), to see what remains
  on your account.

---

## Authentication

Every request must carry your API key in the `X-API-Key` request header. The
header name is case-insensitive. The key is 32 alphanumeric characters and is
shown on your [API page](https://www.domainiq.com/account_api), where you can
also change it. Changing the key immediately deactivates the previous one.

```
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://domainiq-public-api.intelium.com/api2?service=domain_report&domain=google.com"
```

Requests without a valid header are rejected with HTTP `400` (see
[Errors](#errors)). This endpoint does **not** accept the key as a query
parameter. Keep the key out of URLs, logs and shared code; treat it like a
password.

---

## Fallback Endpoint

The same API is also served at `https://www.domainiq.com/api`. Requests,
parameters and responses are identical. Two differences:

| | `domainiq-public-api.intelium.com/api2` | `www.domainiq.com/api` |
|---|---|---|
| `X-API-Key` header | required | accepted |
| `key` query parameter | rejected | accepted (header wins if both are sent) |

If you cannot reach one endpoint, retry the other with the same request.

---

## Request Format

### Required parameters

| Parameter | Description |
|-----------|-------------|
| `service` | The call you are making, for example `domain_report`, `whois`, `reverse_ip`. One per request. |

Plus the `X-API-Key` header. Everything else depends on the service and is
documented under [API Calls](#api-calls).

### Optional parameters (all services)

| Parameter | Description |
|-----------|-------------|
| `output_mode` | `xml` (default), `json`, `csv` or `ini`. If a service does not support the requested mode it falls back to XML. See [Response Format](#response-format). |
| `queued` | Set to `1` to run the call in the background and poll for the result. See [Queued Requests](#queued-requests). |
| `as_file` | Set to `1` to receive the response as a file download named `API.<service>.<timestamp>.<ext>`. Useful in a browser. |

### GET vs POST

Parameters may be sent in the query string (`GET`) or as a form body
(`POST`). Use `POST` for large inputs such as bulk domain lists.

### List parameters

Services that accept several items (`domains`, `ips`, `items`, ...) take them
in one parameter separated by `>>`:

```
domains=red.com>>green.com>>blue.com
```

**Example** — WHOIS for `google.com` as JSON:

```
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://domainiq-public-api.intelium.com/api2?service=whois&domain=google.com&output_mode=json"
```

---

## Response Format

The default response is XML. Pass `output_mode` to choose another format:

| Mode | Description |
|------|-------------|
| `xml` | Default. One root element per service (`<result>`, `<results>`, `<reports>`, ...). |
| `json` | Same structure as the XML, as a JSON object. |
| `csv` | Comma-separated, fields quoted with `"`. Supported by the services marked "CSV" below. The reverse and bulk lookups default to CSV. |
| `ini` | INI-file layout. Any service that supports CSV also supports INI. Each row gains a `record_index` field starting at 1. |

Successful responses have no envelope; the root element is the data. A
response that contains an `error` key is a failure (see [Errors](#errors)).

Sample XML response, `whois` for `google.com`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<result>
	<domain>google.com</domain>
	<status>ServerUpdateProhibited,ServerTransferProhibited,ClientDeleteProhibited,...</status>
	<registrar>MARKMONITOR INC.</registrar>
	<registrar_url>http://www.markmonitor.com</registrar_url>
	<ns_1>NS1.GOOGLE.COM</ns_1>
	<ns_2>NS2.GOOGLE.COM</ns_2>
	<creation_date>1997-09-15</creation_date>
	<expiration_date>2020-09-14</expiration_date>
	<update_date>2011-07-20</update_date>
	<emails>abusecomplaints@markmonitor.com,contact-admin@google.com,dns-admin@google.com</emails>
	<phones>1.2083895740,1.6502530000,...</phones>
	<registrant>Dns Admin</registrant>
	<raw>
		Domain Name: GOOGLE.COM
		Registrar: MARKMONITOR INC.
		...
	</raw>
</result>
```

> Responses are sent with `Cache-Control: no-store`. Cache on your side if
> you need to.

---

## Errors

A failed request returns HTTP `400` and a single `error` field in the
requested output mode. **Check for `error` before reading the data.**

| HTTP status | `error` | Cause |
|-------------|---------|-------|
| `400` | `Invalid or missing X-API-Key header: must be exactly 32 alphanumeric (A-Z a-z 0-9) characters.` | The header is absent or malformed. Sent as JSON regardless of `output_mode`. |
| `400` | `Invalid key: ...` | The key is well-formed but does not belong to an active account. |
| `400` | `Unknown service: ...` | `service` is not a supported call. |
| `400` | `No service name specified` | `service` is missing. |
| `400` | `Service must be run in queued mode: ...` | The service only runs with `queued=1`. |
| `400` | *(service message)* | A per-service problem: missing parameter, daily limit reached, input too large, and so on. |

JSON error:

```json
{"error":"Invalid key: 0123456789abcdef0123456789abcdef"}
```

XML error:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<error>Unknown service: nothing</error>
```

Some services add fields alongside `error`. `domain_search` reports a
concurrency limit this way:

```xml
<error>
	<error>Too many other concurrent domain searches. Please wait for the other searches to complete.</error>
	<concurrent_error>1</concurrent_error>
	<concurrent_sessions>3</concurrent_sessions>
</error>
```

Retry after a short wait when you see a concurrency or limit error. Do not
retry a `400` caused by bad input without changing the request.

---

## Queued Requests

Some calls take more than a few seconds, depending on the data involved. Add
`queued=1` to any such call and the API returns at once with a hash you can
poll. Most services support queued mode; `limits` and `queue` do not.

**1. Start the request**

```
https://domainiq-public-api.intelium.com/api2?service=domain_report&domain=google.com&queued=1
```

```xml
<queue>
	<queued>1</queued>
	<hash>g1aqpj6c7qdcoe4yiqi9f359vzgu73uu</hash>
</queue>
```

**2. Poll for status**

```
https://domainiq-public-api.intelium.com/api2?service=queue&action=status&hash=g1aqpj6c7qdcoe4yiqi9f359vzgu73uu
```

```xml
<queue>
	<status>Complete</status>
	<output_mode>xml</output_mode>
	<last_update>2016-12-29 14:55:52</last_update>
	<result_size>231269</result_size>
</queue>
```

**3. Fetch the result once `status` is `Complete`**

```
https://domainiq-public-api.intelium.com/api2?service=queue&action=results&hash=g1aqpj6c7qdcoe4yiqi9f359vzgu73uu
```

The response is exactly what the original call would have returned, in the
`output_mode` you gave the original call.

> `status` responses come back as XML or JSON only. If the original call asked
> for CSV or INI, the `status` call falls back to XML; the `results` call still
> returns CSV or INI.

---

## API Calls

Every example below needs the `X-API-Key` header. It is left out of the URLs
for brevity.

### Summary reports

Single-item reports for a domain, registrant name, organization, email
address or IP address. XML and JSON output. These correspond to the report
pages on the site and return many fields, most of them self-explanatory.

#### domain_report — Domain report

| Parameter | Description |
|-----------|-------------|
| `domain` | The domain name. |
| `cached` | Optional. Set to `1` to return the cached report if one exists. Only useful on accounts configured with a higher limit for cached-only reports. |

```
https://domainiq-public-api.intelium.com/api2?service=domain_report&domain=google.com
https://domainiq-public-api.intelium.com/api2?service=domain_report&domain=google.com&cached=1
```

#### name_report — Registrant name report

| Parameter | Description |
|-----------|-------------|
| `name` | The registrant's name. |

```
https://domainiq-public-api.intelium.com/api2?service=name_report&name=Jeph+Smythe
```

#### organization_report — Registrant organization report

| Parameter | Description |
|-----------|-------------|
| `organization` | The registrant organization. |

```
https://domainiq-public-api.intelium.com/api2?service=organization_report&organization=My+Company
```

#### email_report — Registrant email report

| Parameter | Description |
|-----------|-------------|
| `email` | The registrant email address. |

```
https://domainiq-public-api.intelium.com/api2?service=email_report&email=someone@somewhere.com
```

#### ip_report — IP address report

| Parameter | Description |
|-----------|-------------|
| `ip` | The IP address. |

```
https://domainiq-public-api.intelium.com/api2?service=ip_report&ip=1.2.3.4
```

---

### Reverse lookups

Find sets of domains by keyword, registrant, nameserver, IP or mail server.
These match the services on the site's [Tools page](https://www.domainiq.com/tools).

#### domain_search — Domain keyword search

Find registered domains matching one or more keywords, with summarized
registrant data. All output modes; **defaults to CSV**.

**Restrictions**

- Maximum **3 concurrent** `domain_search` calls per account. A fourth
  returns the `concurrent_error` response shown in [Errors](#errors).

**Keyword parameters**

| Parameter | Description |
|-----------|-------------|
| `keyword` | One keyword. For several, index the parameter: `keyword[0]`, `keyword[1]`, ... |
| `condition` | How the keyword must match. `contains` (default), `starts`, `ends`, `does_not_contain`, `does_not_start_with`, `does_not_end_with`. `ends` applies to the SLD (the part before the TLD). Index it to pair with a keyword: `condition[1]` goes with `keyword[1]`. |
| `match` | With several keywords: `any` (default) or `all`. |

**Result parameters**

| Parameter | Description |
|-----------|-------------|
| `count_only` | If present, return only the number of matches. Re-run without it to get the rows; the result is cached. |
| `exclude_dashed` | `1` to drop domains containing `-`. |
| `exclude_numbers` | `1` to drop domains containing digits. |
| `exclude_idn` | `1` to drop IDN domains (`xn--`). |
| `min_length`, `max_length` | Match length limits, excluding the TLD. |
| `min_create_date`, `max_create_date` | WHOIS creation date range, `YYYY-MM-DD`. |
| `limit` | Maximum rows. Defaults to your account's maximum. |

**Examples**

All domains containing `glengarry`:

```
https://domainiq-public-api.intelium.com/api2?service=domain_search&keyword=glengarry
```

Count only:

```
https://domainiq-public-api.intelium.com/api2?service=domain_search&keyword=glengarry&count_only=1
```

Domains that start with `baseball` or contain `basketball`:

```
https://domainiq-public-api.intelium.com/api2?service=domain_search&keyword[0]=baseball&condition[0]=starts&keyword[1]=basketball&condition[1]=contains&match=any
```

Domains that start with `first` and end with `bank`:

```
https://domainiq-public-api.intelium.com/api2?service=domain_search&keyword[0]=first&condition[0]=starts&keyword[1]=bank&condition[1]=ends&match=all
```

Sample CSV columns:

| domain | tld | sld_length | whois_registrant | whois_emails | whois_age | dns |
|--------|-----|------------|------------------|--------------|-----------|-----|
| glengarryouthouses.com | com | 19 | Julia Lucio | jmlucio@versacore.info | 1 | NS671.VERSACOREHOSTING.COM |
| glengarrybrewing.com | com | 17 | David Hladky | dhladky@gmail.com | 0 | NS1.APLUS.NET |

Count-only output:

```xml
<count>196</count>
```

#### reverse_search — Registrant, email and organization wildcard search

Wildcard search over registrant names, email addresses or organizations.
Returns each match with the number of domains tied to it. XML and JSON.

**This service is only enabled for certain account types.**
[Contact us](https://www.domainiq.com/contact) to turn it on.

| Parameter | Description |
|-----------|-------------|
| `type` | `name`, `email` or `org`. |
| `search` | The phrase. For names, strip spaces and punctuation (`garysmith`). Emails may contain letters, digits, `-`, `_`, `.` and `@`. |
| `match` | `contains` (default), `begins` or `ends`. |

Returns up to 100 results sorted by domain count, from a result set capped at
10,000. If the search is too broad the response says so; narrow it. `begins`
is much faster than `contains` or `ends`, which can take a minute or more.

```
https://domainiq-public-api.intelium.com/api2?service=reverse_search&type=name&search=garysmith
https://domainiq-public-api.intelium.com/api2?service=reverse_search&type=email&search=@microsoft.com&match=ends
```

```xml
<results>
	<message>27 total results found</message>
	<items>
		<item><item>GARY SMITH</item><domain_count>3196</domain_count></item>
		<item><item>GARY SMITH JR</item><domain_count>72</domain_count></item>
		...
	</items>
</results>
```

#### reverse_dns — Reverse DNS search

All domains hosted on a nameserver. All output modes; **defaults to CSV**.

| Parameter | Description |
|-----------|-------------|
| `domain` | A nameserver hostname (`ns1.google.com`) or just its domain (`yahoo.com`). |

```
https://domainiq-public-api.intelium.com/api2?service=reverse_dns&domain=science.com
```

Columns: `domain`, `ip`, `whois_registrant`, `whois_registrant_domain_count`,
`whois_email`, `whois_email_domain_count`, `whois_registrar`.

#### reverse_ip — Reverse IP search

All domains on an IP address, subnet, block, range, or on the IP a domain
resolves to. All output modes; **defaults to CSV**.

| Parameter | Description |
|-----------|-------------|
| `type` | `ip`, `subnet` (1.1.1.1–1.1.1.255), `block` (1.1.1.1–1.1.255.255), `range`, or `domain` (resolved to its IP). |
| `data` | The IP (`12.34.56.78`), subnet (`12.34.56`), block (`12.34`), range (`12.34.56.78-12.34.56.90`) or domain name. |

```
https://domainiq-public-api.intelium.com/api2?service=reverse_ip&type=ip&data=4.3.2.1
https://domainiq-public-api.intelium.com/api2?service=reverse_ip&type=range&data=4.3.2.1-4.3.2.21
https://domainiq-public-api.intelium.com/api2?service=reverse_ip&type=subnet&data=4.3.2.0
https://domainiq-public-api.intelium.com/api2?service=reverse_ip&type=domain&data=sampledomain.com
```

Columns: `domain`, `ip`, `whois_registrant`, `whois_registrant_domain_count`,
`whois_email`, `whois_email_domain_count`, `whois_registrar`, `create_date`.

#### reverse_mx — Reverse MX search

All domains using a mail server, by hostname, IP, subnet, block or range. All
output modes; **defaults to CSV**.

| Parameter | Description |
|-----------|-------------|
| `type` | `hostname`, `ip`, `subnet`, `block` or `range`. |
| `data` | The MX hostname, IP, subnet, block or range (same forms as `reverse_ip`). |
| `recursive` | If present, also follow MX hostnames found along the way to discover more domains. |

```
https://domainiq-public-api.intelium.com/api2?service=reverse_mx&type=hostname&data=msgin.vvv.facebook.com
https://domainiq-public-api.intelium.com/api2?service=reverse_mx&type=range&data=4.3.2.1-4.3.2.21
```

Columns: `domain`, `mx_ip`, `mx_hostname`, `whois_registrant`,
`whois_registrant_domain_count`, `whois_email`, `whois_email_domain_count`,
`whois_registrar`, `create_date`.

---

### Bulk lookups

Bulk calls move a lot of data and can take a while. Send input with `POST`
and consider [`queued=1`](#queued-requests) for large batches. Each has a
daily limit on your account; check [`limits`](#limits--account-limits).

#### bulk_dns — Bulk DNS lookup

DNS records for many domains at once. **CSV only.**

| Parameter | Description |
|-----------|-------------|
| `domains` | Domain names separated by `>>`. |
| `type` | Record types, comma-separated: `A`, `AAAA`, `CNAME`, `MX`, `NS`, `TXT`, `SOA`. Default `NS`. |

```
https://domainiq-public-api.intelium.com/api2?service=bulk_dns&domains=red.com>>green.com>>blue.com
```

| domain | dns_1 | dns_2 |
|--------|-------|-------|
| red.com | ns2.p09.dynect.net | ns1.p09.dynect.net |
| blue.com | ns2.digitalocean.com | ns1.digitalocean.com |

#### bulk_whois — Bulk WHOIS lookup

WHOIS data for many domains or IP addresses. **CSV output**, about 47
columns covering the whole record.

| Parameter | Description |
|-----------|-------------|
| `type` | `live` (default, slowest, freshest), `registry` (registry data only, no contacts; only matters for COM/NET and other thin-registry TLDs), or `cached` (fast, full data, up to 30 days old). |
| `domains` **or** `ips` | Items separated by `>>`. A batch holds domains or IPs, not both, because the columns differ. IPv4 and IPv6 accepted. |

Additional daily credits and custom limits are available;
[contact us](https://www.domainiq.com/contact).

```
https://domainiq-public-api.intelium.com/api2?service=bulk_whois&domains=red.com>>green.com>>blue.com
```

| domain | status | registrar | creation_date | expiration_date | update_date |
|--------|--------|-----------|---------------|-----------------|-------------|
| blue.com | CLIENTTRANSFERPROHIBITED | DOMAIN.COM, LLC | 11/7/1996 | 10/29/2019 | 2/20/2015 |
| red.com | CLIENTDELETEPROHIBITED | DNC HOLDINGS, INC. | 10/27/1992 | 10/26/2021 | 11/20/2013 |

#### bulk_whois_ip — Bulk domain IP WHOIS lookup

Resolves domains to IPs, then returns ownership and geolocation for those
IPs, plus MX and DNS. **CSV output.** This is not the same as `bulk_whois`
with `ips`; use that to get WHOIS records for IP addresses you already have.

| Parameter | Description |
|-----------|-------------|
| `domains` | Domain names separated by `>>`. |

```
https://domainiq-public-api.intelium.com/api2?service=bulk_whois_ip&domains=red.com>>blue.com>>white.com
```

| domain | ip | mx_hostname | mx_ip | dns | whois_ip_isp | whois_ip_org |
|--------|----|-------------|-------|-----|--------------|--------------|
| black.com | 72.26.208.183 | mail.black.com. | 72.26.208.183 | ns.voxel.net. | Voxel Dot Net, Inc. | Voxel Dot Net, Inc. |

---

### Domain services

#### whois — WHOIS record

Live WHOIS lookup for one domain or IP, parsed plus raw. Uses your daily
WHOIS limit.

| Parameter | Description |
|-----------|-------------|
| `domain` **or** `ip` | What to look up. |
| `full` | `1` for the thoroughly parsed version shown on a domain report. Without it you get a simpler parse of the current record. In full mode, missing fields are filled from past records unless `current_only=1`. |
| `current_only` | `1` to use only the current record in `full` mode. |

```
https://domainiq-public-api.intelium.com/api2?service=whois&domain=google.com
https://domainiq-public-api.intelium.com/api2?service=whois&domain=google.com&full=1
```

See [Response Format](#response-format) for a sample.

#### dns — DNS record lookup

Live DNS records for a hostname or domain. XML and JSON.

| Parameter | Description |
|-----------|-------------|
| `q` | The hostname or domain. |
| `type` / `types` | Optional. One type, or several comma-separated. Default: all. Supported: `A`, `A6`, `AAAA`, `CNAME`, `HINFO`, `MX`, `NAPTR`, `NS`, `PTR`, `SOA`, `SRV`, `TXT`. |

```
https://domainiq-public-api.intelium.com/api2?service=dns&q=www.science.com
```

```xml
<results>
	<item><host>redirectserver.aaas.org</host><class>IN</class><ttl>884</ttl><type>A</type><ip>198.151.217.186</ip></item>
	<item><host>www.science.com</host><class>IN</class><ttl>884</ttl><type>CNAME</type><target>redirectserver.aaas.org</target></item>
</results>
```

#### categorize — Domain categorization

Assigns a category to one or more domains. Uses your daily WHOIS limit.

| Parameter | Description |
|-----------|-------------|
| `domain` / `domains` | One domain, or several separated by `>>`. |

```
https://domainiq-public-api.intelium.com/api2?service=categorize&domain=freshdonuts.com
```

```xml
<result>
	<item><domain>freshdonuts.com</domain><category>Food -- Baked Goods</category></item>
</result>
```

#### snapshot — Domain screenshot

A current or recently cached screenshot of a domain's home page.

| Parameter | Description |
|-----------|-------------|
| `domain` | The domain. |
| `full` | `1` for a full-page image. Default is a 250×125 thumbnail. |
| `no_cache` | `1` to force a fresh capture. |
| `raw` | `1` to return the PNG/JPEG bytes directly instead of a URL. |
| `width`, `height` | Thumbnail size in pixels, 100–640 each. Default 250×125. |

```
https://domainiq-public-api.intelium.com/api2?service=snapshot&domain=msn.com
https://domainiq-public-api.intelium.com/api2?service=snapshot&domain=msn.com&full=1&raw=1
```

```xml
<snapshot>http://www.domainiq.com/cache/snapshots/apisnap.msn.com.14f1.png</snapshot>
```

#### snapshot_history — Historical screenshots

Archived screenshots for a domain, with thumbnail and full-size URLs.

| Parameter | Description |
|-----------|-------------|
| `domain` | The domain. |
| `width`, `height` | Thumbnail size, 100–640. Default 250×125. |
| `limit` | Number of snapshots. Default 10, maximum 50. |

```
https://domainiq-public-api.intelium.com/api2?service=snapshot_history&domain=msn.com
```

```xml
<snaps>
	<item>
		<domain>msn.com</domain>
		<format>jpeg</format>
		<date>2012-08-22 16:42:57</date>
		<img_url_thumb>https://www.domainiq.com/snap?domain=MSN.COM&amp;width=250&amp;height=125&amp;img=...</img_url_thumb>
		<img_url_full>https://www.domainiq.com/snap?domain=MSN.COM&amp;full=1&amp;img=...</img_url_full>
	</item>
	...
</snaps>
```

#### ct_names — Certificate Transparency names

Every FQDN that has appeared in a CT-logged certificate for a registrable
domain: the apex, `www`, wildcards and any other cert-bearing name. This is
not full subdomain enumeration. Each name has first/last seen dates,
certificate count and the most recent issuing CA. All output modes;
**defaults to JSON**.

| Parameter | Description |
|-----------|-------------|
| `domain` | The domain. Normalized to its registrable domain. |

```
https://domainiq-public-api.intelium.com/api2?service=ct_names&domain=domainiq.com
```

```json
{"domain":"domainiq.com","results":[
  {"domain":"domainiq.com","first_seen":"2015-06-02","last_seen":"2025-11-14",
   "cert_count":214,"last_issuer":"Let's Encrypt","is_wildcard":false},
  {"domain":"www.domainiq.com","first_seen":"2015-06-02","last_seen":"2025-11-14",
   "cert_count":214,"last_issuer":"Let's Encrypt","is_wildcard":false}
]}
```

#### cert_history — Certificate history

Certificate issuance for a registrable domain rolled up across all its
FQDNs: one row per issuing CA with validation level (DV vs OV/EV), total
certificates and first/last seen dates. All output modes; **defaults to
JSON**.

| Parameter | Description |
|-----------|-------------|
| `domain` | The domain. Normalized to its registrable domain. |

```
https://domainiq-public-api.intelium.com/api2?service=cert_history&domain=domainiq.com
```

```json
{"domain":"domainiq.com",
 "history":[
   {"issuer_id":45,"issuer_cn":"R3","issuer_org":"Let's Encrypt",
    "certs":210,"dv_certs":210,"subject_o":null,
    "org_label":"DV — no organization",
    "first_seen":"2019-02-14","last_seen":"2025-11-14"},
   {"issuer_id":12,"issuer_cn":"RapidSSL SHA256 CA","issuer_org":"GeoTrust Inc.",
    "certs":4,"dv_certs":4,"subject_o":null,
    "org_label":"DV — no organization",
    "first_seen":"2015-06-02","last_seen":"2016-07-02"}
 ],
 "ca_switches":[]}
```

---

### Domain history and lookalikes

#### domain_continuity — Domain continuity check

Is a domain as old as it looks? For each domain: first registration date,
whether it dropped and was registered again, owner, registrar and nameserver
changes, time out of the zone, UDRP cases, and a verdict with plain-English
`reasons` and a dated `timeline`.

Verdicts: `re_registered`, `changed_hands`, `no_change`, `not_enough_history`.
`changed_hands` means the registrant changed while the creation date did
not. Usually a sale, sometimes the same owner under a new name or agent.

Owner changes are only visible while WHOIS shows the owner, so most changes
after 2018 are hidden by privacy or redaction; `owner_hidden` flags that.
`history_from` is where our records for the domain begin; before it,
`no_change` only means there is nothing to compare.

JSON (default), XML and CSV. In JSON, flags are `1`/`0`, `in_zone` is
`yes`/`no`/`unknown`, missing values are empty strings. Each domain counts
once against your daily Domain Continuity Check limit.

| Parameter | Description |
|-----------|-------------|
| `domain` | One domain, **or** |
| `domains` | Up to 1,000 domains separated by `>>`. |

```
https://domainiq-public-api.intelium.com/api2?service=domain_continuity&domains=example.com>>example.net
```

```json
{"results":[
  {"domain":"example.com","verdict":"re_registered","verdict_label":"Re-registered",
   "first_registered":"2003-11-17","current_created":"2024-06-03","registrations":2,
   "registration_dates":["2003-11-17","2024-06-03"],"last_reregistered":"2024-06-03",
   "owner_changes":0,"last_owner_change":"","last_owner_change_after":"",
   "owner_hidden":1,"owner_visible_until":"2016-02-01",
   "registrar_changes":1,"last_registrar_change":"2024-06-10",
   "ns_changes":2,"last_ns_change":"2019-03-11",
   "zone_gaps":[{"left":"2024-02-19","returned":"2024-06-04","days":106}],
   "left_zone":"","in_zone":"yes","udrp_cases":0,"last_udrp_case":"",
   "history_from":"2010-04-01","snapshots":5,
   "reasons":["First registered 17 November 2003, then registered again 3 June 2024.",
              "It was out of the zone for 3 months just before that, from 19 February 2024 to 4 June 2024."],
   "timeline":[{"date":"2003-11-17","type":"registered","label":"First registered"}]}
]}
```

#### lookalike_search — Lookalike search

Registered domains that look like a brand: typos (missing, extra, swapped or
adjacent letters), look-alike and international characters, one-bit errors,
added hyphens, the brand plus a word such as `login` or `support`, the brand
on other TLDs, and, for brands of five or more characters, other names that
contain it. Coverage is the zone files we collect, which leaves out most
country-code TLDs.

Each result has the owner (when WHOIS shows one), registration date,
nameservers, mail servers, parking, UDRP cases, the domain continuity
verdict, and a `score` from 0 to 100 with a `level` of `high`, `medium`,
`low` or `yours`. `factors` lists what built the score as `[reason, points]`
pairs. The score is a way to decide what to look at first, not a finding
that a domain is malicious.

International names come back as `xn--...` in `domain`. In JSON, flags are
`1`/`0`, `in_zone` is `yes`/`no`/`unknown`, missing values are empty
strings. A lookalike is `yours` only when registered with the same email as
one of your `own` domains; a matching organization name shows in `same_org`
but does not change the level, because anyone can type one.

JSON (default), XML and CSV. Each call counts once against your daily
Lookalike Finder limit.

| Parameter | Description |
|-----------|-------------|
| `brand` | A brand name or one of its domains: `acme` or `acme.com`. |
| `own` | Optional. Domains the brand owns, separated by `>>`. Excluded from results; lookalikes sharing their registrant email get level `yours`. |
| `limit` | Optional. Most results to return, highest score first. Up to 1,000 or your plan's limit. |

```
https://domainiq-public-api.intelium.com/api2?service=lookalike_search&brand=acme&own=acme.com&limit=100
```

```json
{"brand":"acme","total":412,"shown":100,"tlds":61,"contains_note":"",
 "results":[
  {"domain":"acme-login.com","ext":"com",
   "brand":"acme","technique":"combo_risky","technique_label":"Brand plus a login or payment word",
   "group":"combo","created":"2026-09-18","registrar":"NameCheap, Inc.","owner":"",
   "owner_hidden":1,"owned":0,"same_org":0,"ns":["ns1.cloudflare.com","ns2.cloudflare.com"],
   "ns_provider":"cloudflare.com","in_zone":"yes","parked":"","mx":["mx1.privateemail.com"],
   "has_mx":1,"udrp_cases":0,"continuity":"not_enough_history",
   "continuity_label":"Not enough history","first_registered":"2026-09-18",
   "score":93,"level":"high",
   "factors":[["Brand plus a login or payment word on .com",46],["Can receive email",26],
              ["Registered in the last 30 days",18],["Owner hidden",3]]}
 ]}
```

---

### Monitoring

All monitoring calls use `service=monitor` with an `action` parameter. XML
and JSON output. Monitors are grouped into **reports**; each report holds
**items** (the things being watched); each day's detected changes form a
**change** record.

#### Reading monitors

**`action=list`** — your active monitoring reports. Equivalent to the
[Monitor page](https://www.domainiq.com/monitor).

```
https://domainiq-public-api.intelium.com/api2?service=monitor&action=list
```

```xml
<reports>
	<item>
		<report>163</report>
		<type>Keyword</type>
		<name>My Keywords</name>
		<item_count>1</item_count>
		<last_alert_date>2015-02-18 18:10:04</last_alert_date>
	</item>
	...
</reports>
```

**`action=report_items`** — items inside a report.

| Parameter | Description |
|-----------|-------------|
| `report` | A `report` id from `action=list`. |

```
https://domainiq-public-api.intelium.com/api2?service=monitor&action=report_items&report=163
```

```xml
<items>
	<item>
		<item>51541</item>
		<type>Keyword</type>
		<monitor_name>My Keywords</monitor_name>
		<data>science</data>
		<monitor_start_date>2014-10-29</monitor_start_date>
		<last_alert_date>2015-02-18 18:10:04</last_alert_date>
	</item>
	...
</items>
```

**`action=report_summary`** — daily change counts for a report or one item.

| Parameter | Description |
|-----------|-------------|
| `report` | The report id. |
| `item` | Optional. Restrict to one item id. |
| `range` | Optional. Only the last N days. |

```
https://domainiq-public-api.intelium.com/api2?service=monitor&action=report_summary&report=163
https://domainiq-public-api.intelium.com/api2?service=monitor&action=report_summary&report=163&item=51541
https://domainiq-public-api.intelium.com/api2?service=monitor&action=report_summary&report=163&range=15
```

```xml
<changes>
	<item>
		<change>67</change>
		<keyword>science</keyword>
		<domains_added_count>37</domains_added_count>
		<alert_date>2014-10-29</alert_date>
	</item>
	...
</changes>
```

**`action=report_changes`** — the domains behind one daily change.

| Parameter | Description |
|-----------|-------------|
| `report` | The report id. |
| `change` | A `change` id from `report_summary`. |

```
https://domainiq-public-api.intelium.com/api2?service=monitor&action=report_changes&report=163&change=67
```

```xml
<change_details>
	<item>
		<domain>ameresciences.com</domain>
		<registrar>eNom</registrar>
		<registrant>Whois Agent</registrant>
		<emails>jmpcyqrmgq@whoisprivacyprotect.com</emails>
		<dns>DNS1.NAME-SERVICES.COM</dns>
		<create_date>2014-10-26</create_date>
		<report_date>2014-10-29</report_date>
	</item>
	...
</change_details>
```

#### Managing monitors

**`action=report_create`** — create a report.

| Parameter | Description |
|-----------|-------------|
| `type` | `domain`, `dns`, `registrant`, `organization`, `email`, `registrar`, `keyword`, `ip` or `domain ip`. See the [Add Monitor](https://www.domainiq.com/monitor_add) page for what each watches. |
| `name` | The report's name. |
| `email_alert` | Optional. `0` to leave this report out of the daily alert email. Default on. |

The number of reports is limited by your account. On success you get the new
report id.

```
https://domainiq-public-api.intelium.com/api2?service=monitor&action=report_create&type=domain&name=CorporateDomains
```

```xml
<result>
	<success>1</success>
	<report_id>112233</report_id>
</result>
```

**`action=report_item_add`** — add items to a report. The report's type must
match the item type, except that `email` and `registrant` items may share a
report.

| Parameter | Description |
|-----------|-------------|
| `report_id` | The report to add to. |
| `type` | `domain`, `dns`, `registrant`, `organization`, `email`, `registrar`, `ip`, `domain_ip` or `keyword`. |
| `item` / `items` | Items separated by `>>`. Format depends on `type`. |

Per-type details:

| `type` | Items | Extra parameters |
|--------|-------|------------------|
| `domain` | Domain names. | **Required** `domain_alert`: comma-separated set of `emails`, `registrar`, `dns`, `create date`, `expire date`, `update date`. Accepts filters. |
| `dns` | Nameservers. Subdomains are stripped; monitored by root domain. | |
| `registrant` | Registrant names. | |
| `organization` | Organization names. | |
| `email` | Registrant email addresses. | |
| `registrar` | Registrar names. | Accepts filters. |
| `ip` | IP addresses. | |
| `domain_ip` | Domain names, for IP-change monitoring. | |
| `keyword` | Keywords. | **Required** `match_types` (`contains`, `starts`, `ends`, one per keyword, `>>`-separated) and `join_types` (`AND` or `OR`, one per keyword). Optional `use_typos` (boolean), `typo_strength` (4–41, required with `use_typos`), `include_added` (default true), `include_deleted` (default true). Accepts filters. |

**Filter parameters**, for the types marked above. All filters are
inclusive: a result must satisfy every filter given.

| Parameter | Description |
|-----------|-------------|
| `filter_tld_in`, `filter_tld_out` | Accept or reject by TLD. |
| `filter_keyword_in`, `filter_keyword_out` | Accept or reject by keyword anywhere in the name. |
| `filter_registrar_in`, `filter_registrar_out` | By registrar name. |
| `filter_dns_in`, `filter_dns_out` | By nameserver. |
| `filter_email_in`, `filter_email_out` | By registrant email. |
| `filter_registrant_in`, `filter_registrant_out` | By registrant name. |
| `exclude_dash` | Reject domains containing `-`. |
| `exclude_num` | Reject domains containing digits. |
| `exclude_idn` | Reject IDN domains (`xn--`). |

`filter_tld_in=net&filter_keyword_in=science&filter_keyword_out=top` matches
only `.net` domains containing `science` and not containing `top`.

```
https://domainiq-public-api.intelium.com/api2?service=monitor&action=report_item_add&report_id=112233&type=domain&item=domainiq.com&domain_alert=emails,registrar,dns
```

```xml
<success>1</success>
<affected_items>1</affected_items>
```

**`action=enable_typos` / `action=disable_typos`** — turn typo detection on
or off for a keyword item.

| Parameter | Description |
|-----------|-------------|
| `report_id` | The report containing the item. |
| `item_id` | The item id (from `report_items`). |
| `strength` | Optional, `enable_typos` only. 5–41; controls how many typo variants are generated (20 gives roughly 17–22). Default 41. |

```
https://domainiq-public-api.intelium.com/api2?service=monitor&action=enable_typos&report_id=112233&item_id=223344&strength=21
https://domainiq-public-api.intelium.com/api2?service=monitor&action=disable_typos&report_id=112233&item_id=223344
```

**`action=modify_typo_strength`** — change the strength on an item that
already has typos enabled. Same parameters as `enable_typos`.

```
https://domainiq-public-api.intelium.com/api2?service=monitor&action=modify_typo_strength&report_id=112233&item_id=223344&strength=41
```

**`action=report_item_delete`** — remove one item. **Permanent.** All
history for the item is deleted.

| Parameter | Description |
|-----------|-------------|
| `item_id` | The item id. |

```
https://domainiq-public-api.intelium.com/api2?service=monitor&action=report_item_delete&item_id=223344
```

**`action=report_delete`** — remove a whole report and every item in it.
**Permanent.**

| Parameter | Description |
|-----------|-------------|
| `report_id` | The report id. |

```
https://domainiq-public-api.intelium.com/api2?service=monitor&action=report_delete&report_id=332211
```

```xml
<success>1</success>
<affected_items>10</affected_items>
```

---

### Account

#### limits — Account limits

Your account's limits and what remains on each. XML, JSON, CSV and INI.
Does not support queued mode.

```
https://domainiq-public-api.intelium.com/api2?service=limits&output_mode=json
```

Each entry carries `title`, `allowed` and `remaining`; `unlimited` where no
cap applies.

#### queue — Queued request status and results

See [Queued Requests](#queued-requests).

| Parameter | Description |
|-----------|-------------|
| `hash` | The hash returned when the queued call started. |
| `action` | `status` or `results`. |

---

## Related Services

- **[EstiBot](https://github.com/domainret/estibot-api)** — domain appraisals,
  lead generation, drop lists, zone diffs, search volume and site info.
- **[DomainPlex](https://www.domainplex.com)** — enterprise API services.

---

*Code samples (PHP, Python, and more) will be added to this repository over time.*
