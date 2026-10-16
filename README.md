# SF commercial filing research methods

A bounded Python workflow for selecting public San Francisco commercial permit records and joining the agency's recorded firm associations, with a free five-record example from an October 16, 2026 snapshot. This repository documents the method, rather than promising available contracts or current decision-makers.

## Sources and selection

The source datasets are [building permits i98e-djp9](https://data.sfgov.org/resource/i98e-djp9.json) and [permit firm associations 3pee-9qhc](https://data.sfgov.org/resource/3pee-9qhc.json), published by San Francisco Department of Building Inspection through DataSF. Their inspected metadata specified PDDL; the collector checks that license identifier before retrieving records, and stops if it changes. Selection and joining are our modifications, with no government affiliation or endorsement.

The example window is October 10 through October 16, 2026, inclusive. Selection requires a primary address, a proposed use of office, retail sales, food/beverage handling or health studios/gym, a filed/approved/issued status and no completed date. The collector checks filing and selected event dates against the window, rejects duplicate permit numbers, preserves recorded architect and contractor roles separately, and excludes empty or Owner Builder placeholders. Source loading timestamps are retained without the same window check. A recorded contractor firm is not proof of general-contractor status, decision authority, an open bid or an available contract.

The collector deliberately selects firm names and roles rather than personal names, phone numbers or emails. It stops instead of publishing a truncated pack when the fixed 100-permit or 1,000-association bound is exceeded. The source can later be corrected, so a new run is a new observation rather than an exact reconstruction of this snapshot.

## Five actual example records

| Address | Filed | Stage | Use | Recorded firms | Source |
| --- | --- | --- | --- | --- | --- |
| 2675 Geary Bl | 2026-10-14 | issued | retail sales | contractor: Western Electric S Llc | [Official record](https://data.sfgov.org/resource/i98e-djp9.json?%24where=permit_number%3D%27202610149463%27) |
| 81 Jefferson St | 2026-10-14 | filed | retail sales | No firm recorded | [Official record](https://data.sfgov.org/resource/i98e-djp9.json?%24where=permit_number%3D%27202610149443%27) |
| 211 Sutter St | 2026-10-14 | issued | office | contractor: Bruce E. Boyle Construction, Inc. | [Official record](https://data.sfgov.org/resource/i98e-djp9.json?%24where=permit_number%3D%27202610149429%27) |
| 2043 Fillmore St | 2026-10-14 | filed | food/beverage hndlng | architect: Min/Day | [Official record](https://data.sfgov.org/resource/i98e-djp9.json?%24where=permit_number%3D%27202610149418%27) |
| 82 14th St | 2026-10-14 | filed | food/beverage hndlng | architect: Frost-Tsuji Architects; contractor: Cookline Construction Inc | [Official record](https://data.sfgov.org/resource/i98e-djp9.json?%24where=permit_number%3D%27202610149416%27) |

These examples include minor work, alterations and revisions. Applicant estimates are not verified contract values, and the selection is not a census of all commercial work. Dataset freshness and recorded firm associations must be checked for the intended use.

## Reproduce a bounded observation

Save the Python source below as `collect.py`, then use Python 3 with no third-party packages:

```sh
python3 collect.py --start 2026-10-10 --end 2026-10-16 --output sf-observation
```

The output directory must not already exist. The collector writes selected API responses, a retrieval ledger with URLs, timestamps and SHA-256 hashes, and `pack.json` with accepted records, exclusions and limitations. API failures remain failures, with no generated substitute data. The collector makes one bounded retrieval attempt per endpoint, so rerun intentionally into a fresh directory after reviewing a failure rather than assuming continuous operation.

## Related dated-pack pilot

[SF Filing Desk](https://csv-preflight-validation.onrender.com) provides a free five-record preview of a completed 25-record October 16 snapshot, including 17 records with recorded firms and 14 with contractor-role firms. The proposed finite pilot prices the formatted CSV, JSON and HTML artifact at $19 once, with a maximum of three purchases; check the external page for current checkout availability. Source data remains freely available, while the artifact covers selection, joining and formatting. It is a fixed research snapshot with no phones, emails or verified available bids, and no customer outcome or profit is established.

## Collector source

```python
"""Collect a bounded licensed commercial filing cohort, never inferred bids."""
import argparse
import datetime as dt
import hashlib
import json
from pathlib import Path
import urllib.error
import urllib.parse
import urllib.request

BASE = 'https://data.sfgov.org'
PERMITS = 'i98e-djp9'
CONTACTS = '3pee-9qhc'
FIELDS = ('permit_number,street_number,street_name,street_suffix,description,'
          'status,status_date,filed_date,issued_date,completed_date,existing_use,'
          'proposed_use,estimated_cost,primary_address_flag,data_as_of,'
          'data_loaded_at,record_id')
USES = ('office', 'retail sales', 'food/beverage hndlng', 'health studios & gym')


def fetch(root, name, url, ledger):
    entry = {'file': name, 'url': url,
             'retrieved_utc': dt.datetime.now(dt.timezone.utc).isoformat()}
    ledger.append(entry)
    try:
        req = urllib.request.Request(url, headers={'User-Agent': 'FitoutResearch/1.0'})
        with urllib.request.urlopen(req, timeout=25) as response:
            raw = response.read(2_000_001)
            entry['http_status'] = response.status
        if len(raw) > 2_000_000:
            raise ValueError('Response exceeds fixed two MB ceiling')
        value = json.loads(raw)
        (root / name).write_bytes(raw)
        entry.update(bytes=len(raw), sha256=hashlib.sha256(raw).hexdigest())
        return value
    except Exception as exc:
        entry['error'] = type(exc).__name__ + ': ' + str(exc)
        raise
    finally:
        (root / 'retrieval.json').write_text(json.dumps(ledger, indent=2) + '\n')


def resource(dataset, query):
    return BASE + '/resource/' + dataset + '.json?' + urllib.parse.urlencode(query)


def qualifies(row, start, end):
    try:
        filed = dt.date.fromisoformat(row.get('filed_date', '')[:10])
    except ValueError:
        return False
    for key in ('status_date', 'issued_date', 'completed_date', 'data_as_of'):
        if row.get(key):
            try:
                observed = dt.date.fromisoformat(row[key][:10])
            except ValueError:
                return False
            if observed > end or (key == 'data_as_of' and observed < filed):
                return False
    return (start <= filed <= end and row.get('primary_address_flag') == 'Y'
            and row.get('proposed_use', '').lower() in USES
            and row.get('status', '').lower() in ('filed', 'approved', 'issued')
            and not row.get('completed_date') and bool(row.get('permit_number')))


def normalize(rows, contacts, start, end):
    groups = {}
    for c in contacts:
        if c.get('role', '').lower() not in ('architect', 'contractor'):
            continue
        if c.get('from_date'):
            try:
                if dt.date.fromisoformat(c['from_date'][:10]) > end:
                    continue
            except ValueError:
                continue
        if c.get('to_date'):
            try:
                if dt.date.fromisoformat(c['to_date'][:10]) <= end:
                    continue
            except ValueError:
                continue
        firm = c.get('firm_name', '').strip()
        if not firm or firm.lower() in ('owner builder', 'owner', 'n/a'):
            continue
        groups.setdefault(c.get('permit_number'), set()).add((c['role'], firm))
    seen, accepted, rejected = set(), [], []
    for row in rows:
        key = row.get('permit_number')
        if not qualifies(row, start, end) or key in seen:
            rejected.append({'record_id': row.get('record_id'),
                             'reason': 'Outside declared cohort or duplicate permit'})
            continue
        seen.add(key)
        item = {k: v for k, v in row.items() if k in FIELDS.split(',')}
        item['recorded_firms'] = [{'role': role, 'firm': firm}
                                  for role, firm in sorted(groups.get(key, set()))]
        item['source_url'] = resource(PERMITS, {'$where': "permit_number='" + key + "'"})
        item['bid_availability'] = 'Not established by this record'
        accepted.append(item)
    return accepted, rejected


def collect(root, start, end):
    root.mkdir(parents=True, exist_ok=False)
    ledger = []
    for dataset in (PERMITS, CONTACTS):
        meta = fetch(root, dataset + '-metadata.json',
                     BASE + '/api/views/' + dataset + '.json', ledger)
        if meta.get('licenseId') != 'PDDL':
            raise ValueError('Reuse license changed or absent; stop collection')
    uses = ','.join("'" + use + "'" for use in USES)
    where = (f"filed_date >= '{start}T00:00:00' AND "
             f"filed_date < '{end + dt.timedelta(days=1)}T00:00:00' AND "
             f"primary_address_flag='Y' AND lower(proposed_use) in ({uses}) AND "
             "lower(status) in ('filed','approved','issued') AND completed_date is null")
    rows = fetch(root, 'permits.json', resource(PERMITS, {
        '$select': FIELDS, '$where': where,
        '$order': 'filed_date DESC,record_id DESC', '$limit': 101}), ledger)
    if len(rows) > 100:
        raise ValueError('Declared cohort exceeds fixed cap; no incomplete paid pack')
    if not rows:
        raise ValueError('No records; do not manufacture a deliverable')
    numbers = sorted({r['permit_number'] for r in rows})
    if any(not n.isdigit() for n in numbers):
        raise ValueError('Unexpected permit identifier; stop before query construction')
    identifiers = ','.join("'" + n + "'" for n in numbers)
    contacts = fetch(root, 'firms.json', resource(CONTACTS, {
        '$select': 'permit_number,role,firm_name,from_date,to_date,data_as_of,data_loaded_at',
        '$where': f'permit_number in ({identifiers}) AND firm_name is not null AND to_date is null',
        '$limit': 1001}), ledger)
    if len(contacts) > 1000:
        raise ValueError('Contact cap exceeded; no incomplete join')
    accepted, rejected = normalize(rows, contacts, start, end)
    document = {'scope': 'SF commercial permit filing research, not available bids',
                'window': {'start': str(start), 'end': str(end)},
                'retrieved_utc': dt.datetime.now(dt.timezone.utc).isoformat(),
                'source': 'DataSF, San Francisco Department of Building Inspection',
                'license': 'PDDL', 'permit_dataset': PERMITS, 'firm_dataset': CONTACTS,
                'selection': where, 'records': accepted, 'excluded_records': rejected,
                'limitations': ['No phone or email supplied', 'Not a census of all commercial work',
                                'Filing does not establish bidding or contract availability',
                                'Includes alterations and revisions, not only new tenant fit-outs',
                                'Each record retains its own source update dates; no real-time claim',
                                'Recorded costs are applicant estimates, not verified contract values',
                                'No government affiliation or endorsement',
                                'Data can be corrected and is only current at the stated snapshot']}
    (root / 'pack.json').write_text(json.dumps(document, indent=2) + '\n')
    print(json.dumps({'path': str(root), 'records': len(accepted),
                      'excluded': len(rejected), 'paid_customers': 0}))


if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--start', required=True, type=dt.date.fromisoformat)
    parser.add_argument('--end', required=True, type=dt.date.fromisoformat)
    parser.add_argument('--output', required=True, type=Path)
    args = parser.parse_args()
    if not args.start <= args.end <= dt.date.today():
        parser.error('Invalid or future collection window')
    collect(args.output, args.start, args.end)
```
