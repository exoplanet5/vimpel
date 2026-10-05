# Vimpel high-orbit space-object catalogue — archive and field notes

This repository holds an unmodified archive of the weekly orbit newsletters published by
JSC "Vimpel Interstate Corporation" and the Keldysh Institute of Applied Mathematics (KIAM) on
<http://spacedata.vimpel.ru/>, together with notes on what the catalogue is for, how its files are
laid out, how it behaves in practice, and what we learned about it while using it to investigate
two GEO events in July 2026 (LDPE-1 and Intelsat 805).

| | |
|---|---|
| Data | `catalog_database/orbits.YYYYMMDD.txt` — one file per weekly issue |
| Coverage | 448 issues, 2018-01-01 → 2026-09-21 (every issue the portal lists) |
| Size | 3.80 million rows, 19 596 distinct objects, 414 MB |
| Growth | 2 863 objects per issue (Jan 2018) → 13 889 (Sep 2026) |

Statements marked **(publisher)** come from the portal's own description. Everything else is
measured from the archive in this repository or from our own analysis, and should be read as
empirical observation, not as documented behaviour.

## 1. Goal of the catalogue

**(publisher)** The newsletters publish orbits of space-debris objects in high orbits — mainly
orbits with periods above 200 minutes, i.e. the geostationary region and highly elliptical
orbits — that were found by optical observation and are largely missing from public sources such
as space-track.org. The stated purpose is to supplement those sources so that spacecraft operators
have more complete information on objects that may approach their satellites.

The orbits are produced by the data-analysis centre of near-Earth space monitoring of JSC Vimpel
jointly with KIAM, from optical measurements of Vimpel's own network and partners (ASC, Roscosmos,
KIAM, ISTP SB RAS, INASAN).

What the catalogue is not: it carries no object names, no international designators and no NORAD
numbers. Objects are known only by Vimpel's own number. A separate portal file (`datefirst.txt`,
not archived here) lists the NORAD number for objects that later appeared on space-track.

## 2. File format

Each file is plain text, one object per line, 15 comma-separated fixed-width fields, no header:

```
   2, 35207,06112006,21092026 233629,   5, 13066.1, 29.651, 10.073,0.466925,359.7,294.952,3.89e-01, 18.8, 0.3, 54
   4, 90002,18062026,22092026 044338,  67, 31059.2,  4.712,  5.037,0.694566,358.2,127.932,1.19e+01, 16.4,27.9, -
```

| # | Field | Unit / format | Notes |
|---|---|---|---|
| 1 | Sequence number | – | Row number within the issue; not an identifier |
| 2 | Object number (SO) | integer | Constant for an object across issues |
| 3 | Date of first measurement | `DDMMYYYY` | |
| 4 | Reference time of the elements | `DDMMYYYY HHMMSS`, UTC | See "epoch" below |
| 5 | Gap | days | Time since the orbit was last updated with measurements |
| 6 | Semi-major axis | km | |
| 7 | Inclination | deg | |
| 8 | Right ascension of ascending node | deg | |
| 9 | Eccentricity | – | |
| 10 | Argument of latitude at the reference time | deg | |
| 11 | Argument of perigee | deg | |
| 12 | Area-to-mass ratio (AMR) | m²/kg | Average of the values fitted from drag and radiation pressure |
| 13 | Magnitude | mag | Median, reduced to zero phase angle and 40 000 km, diffuse sphere |
| 14 | σ_t, along-track timing uncertainty | minutes | 0.5 confidence level; `-` if not below 100 |
| 15 | σ_x, cross-track position uncertainty | km | 0.5 confidence level; `-` if not below 1000 |

**(publisher)** conventions:

- **Elements** are osculating Keplerian elements in the inertial frame of epoch J2000.
- **Epoch**: each orbit is given at the time of the ascending-node passage closest to 24:00 UTC of
  the issue date.
- **Force model** used to produce the orbits: geopotential to degree and order 8, Sun and Moon as
  point masses (DE-405), atmosphere (GOST R 25645.166-2004) and solar radiation pressure.
- **Real errors**: for most objects the actual position error at the reference time is within
  twice σ_t and σ_x.
- **Ephemerides** (J2000 Cartesian, 10-minute step, one week ahead) are published separately for
  objects with σ_t below 20 minutes. They are not archived here.
- **Cadence**: weekly.

A minimal parser:

```python
from datetime import datetime

def parse_line(line):
    p = [x.strip() for x in line.split(",")]
    if len(p) != 15:
        return None
    num = lambda s: None if s in ("-", "") else float(s)
    return dict(
        so_id=int(p[1]),
        first_meas=datetime.strptime(p[2], "%d%m%Y"),
        epoch=datetime.strptime(p[3], "%d%m%Y %H%M%S"),
        gap=float(p[4]), a=float(p[5]), inc=float(p[6]), raan=float(p[7]),
        e=float(p[8]), u=float(p[9]), argp=float(p[10]), amr=float(p[11]),
        mag=num(p[12]), sig_t=num(p[13]), sig_x=num(p[14]))
```

True anomaly is `u − argp`; with that, fields 6–11 give a full state vector in J2000 at `epoch`.

## 3. Manners — how the catalogue behaves in practice

These are measured over all 448 issues unless a date is given.

### Publication

- Issues are dated on Mondays (445 of 448; three in 2018 fall on other days). Nine weeks have no
  issue on the portal: 2018-07-09, 2019-09-02, 2020-01-13, 2020-03-16, 2020-06-08, 2020-06-29,
  2024-12-09, 2025-10-06 and 2025-10-13.
- An issue can appear on the portal a week or more after its nominal date.
- Weekly turnover is small: a median of 21 objects added and 4 dropped per issue. More than half of
  all objects appear in at least 100 issues.

### Epoch and elements

- The reference time falls within about ±12 hours of 24:00 UTC of the issue date for
  near-geosynchronous objects, and within ±20 hours for 99.8 % of all rows.
- The argument of latitude is within 0.5° of the node for most rows (values such as 359.7 or 0.1).
  It departs by several degrees only for near-equatorial orbits, where the node is poorly defined.
- Because the elements are osculating and in J2000, they differ systematically from space-track
  TLEs, which are mean elements in the TEME frame. Compare positions, not elements (section 4).

### Most entries are stale

The gap field matters more than anything else in the file.

| Gap since last measurement | Share of rows | Median σ_t | σ_t missing |
|---|---|---|---|
| 0–7 days | 21 % | 0.1 min | 0 % |
| 8–30 days | 10 % | 0.7 min | 1 % |
| 31–180 days | 20 % | 5.0 min | 8 % |
| over 180 days | 49 % | 21 min | 41 % |

- The median gap over the whole archive is 167 days; in 2026 it is about a year, and only 13 % of
  entries were measured in the previous week.
- An untracked object stays in the catalogue and its gap grows by 7 each week; its orbit is a
  propagation of the last fit. The gap drops again when the object is re-observed.
- σ_t is missing (not below 100 minutes) in 22 % of rows and σ_x (not below 1000 km) in 51 %.
- Only 59 % of rows meet the publisher's σ_t < 20 minutes criterion for an ephemeris.

### Object numbers

- The object number is stable across issues, and 99 % of objects show no holes in their run of
  issues.
- The number appears to encode the orbital period at first cataloguing: the leading digits are the
  period in minutes and the last two are a serial within that period. For 95 % of objects the
  leading digits are within 5 minutes of the period in the object's first issue (70 % within
  1 minute). This is not documented by the publisher. Example: SO 163103 has a period of
  1 632 minutes.
- Numbers are not infallible: associations change (section 4), and the date of first measurement
  was revised for 307 objects.

### First measurement versus first publication

- An object enters the newsletter with a date of first measurement in the past. The delay from
  first measurement to first publication has a median of 7 days (interquartile range 4–11 days),
  and exceeds 30 days for 10 % of objects.
- So a new issue can add objects whose first-measurement dates are weeks old. Counting "objects
  first measured by date X" gives a number that keeps growing in later issues.

### Physical parameters

- AMR spans 4 × 10⁻⁶ to 550 m²/kg with a median near 1 m²/kg; half of all rows exceed 1 and 13 %
  exceed 10. The population is dominated by high-AMR debris.
- AMR and magnitude are running estimates. They are unchanged from one week to the next in about
  90 % of rows, but occasionally jump (AMR by more than a factor of 2 in 0.1 % of weekly steps).
- Magnitudes range from 6.8 to 22.8 with a median of 17.1.

### What is in it (issue 2026-09-21, 13 889 objects)

| Region | Objects |
|---|---|
| GEO region (period 1300–1600 min, e < 0.25) | 3 974 |
| Molniya-type (period 600–800 min, e > 0.5) | 3 070 |
| Other GTO / HEO (e > 0.5, perigee below 2000 km) | 1 048 |
| Period ≤ 225 min | 144 |
| Everything else (MEO, GNSS region, super-GEO, …) | 5 653 |

About 1 % of objects have periods below the stated 200-minute focus.

## 4. What the LDPE-1 / Intelsat 805 analysis taught us about the catalogue

**The case.** In late July 2026 the space-track TLEs of LDPE-1 (NORAD 49818) showed an abrupt
orbit change, and in early August a cloud of new objects appeared in the Vimpel catalogue nearby.
Using issues 2026-05-04 to 2026-09-07 together with space-track TLEs, we back-propagated the
Vimpel orbits numerically (J2, Sun and Moon, radiation pressure scaled by the Vimpel AMR). Our
analysis indicates that the cloud belongs to Intelsat 805 (NORAD 25371), which fragmented on
2026-07-29 at about 19:50–20:10 UTC, and that LDPE-1 had a separate, much smaller event around
2026-07-27. These are our own results, not confirmed by an operator. The lessons about the
catalogue are below.

### Strengths

- **It is early.** Fragments that space-track first catalogued on 08-18 and 09-07…09-09 had Vimpel
  first-measurement dates of 08-04…08-06, a lead of two to five weeks.
- **It is deep.** By the 09-07 issue we counted 114 Vimpel objects converging on Intelsat 805
  against 11 catalogued by space-track (61 on 08-10, 91 on 08-17).
- **Well-tracked orbits are accurate.** Nine of the 11 space-track fragments match a Vimpel object
  to 29–68 km in position at the Vimpel epoch.
- **Fresh, low-AMR orbits propagate well.** Objects with AMR below 0.3 m²/kg and σ_t ≤ 0.2 minutes
  converged on the parent's track to an ensemble RMS of 88–135 km after two to three weeks of
  back-propagation, which fixed the breakup time to within about 20 minutes. After 40 days the
  RMS was 419 km.
- **Members persist.** All 91 objects identified on 08-17 were still present three issues later,
  with a median week-to-week change in semi-major axis of 1–1.7 km.
- **AMR is usable.** It separates the population cleanly: objects above 10 m²/kg drifted by
  0.5–3° in node and inclination per week, as radiation pressure predicts, while low-AMR objects
  followed the gravitational baseline.

### Limits and traps

- **No identities.** Finding which Vimpel object is which satellite takes a position-level match
  against propagated TLEs. Matching on elements alone is unreliable because of the frame and
  mean-versus-osculating differences.
- **Associations can be wrong.** SO 144189 matched LDPE-1's TLEs in May and June, jumped to a
  semi-major axis near 43 000 km in July on a track that connects to nothing, was lost, and was
  re-acquired on 08-16 at 42 713 km. One object number carried three unrelated orbits.
- **The catalogues disagree on what exists.** The post-event orbit of LDPE-1 in the TLEs
  (42 505 km) has no Vimpel object within 817 km in the 08-31 and 09-07 issues. Two space-track
  fragments of Intelsat 805 have no Vimpel counterpart either. Neither catalogue is a superset of
  the other.
- **The core arrives first.** Low-velocity fragments were catalogued within days; fragments far
  from the parent orbit were still arriving about six weeks later. Any completeness or momentum
  estimate from an early issue is biased toward slow fragments.
- **Orbits get re-fitted.** One member's semi-major axis changed by 149 km between issues with no
  event, and one candidate's AMR was revised from 0.23 to 2.09 m²/kg along with its eccentricity.
  One candidate dropped out of the catalogue.
- **High-AMR orbits do not back-propagate.** With a simple cannonball radiation-pressure model and
  the catalogue's average AMR, objects above about 1 m²/kg lost convergence over 40 days,
  especially once the GEO eclipse season began. Conclusions about event times should rest on
  low-AMR objects only.
- **Uncertainties are optimistic if used as 1σ.** σ_t and σ_x are 0.5-confidence values. Treating
  them as standard deviations in a Monte Carlo probably understates errors by about 1.5×.
- **Short arcs constrain velocity unevenly.** With two-week arcs, along-track and normal velocity
  changes were recovered to a few m/s, radial ones only to tens of m/s.
- **Size and mass are order-of-magnitude.** Sizes from magnitude (diffuse sphere) and masses from
  size and AMR summed to roughly the whole parent's mass while the parent was still in orbit.

### Working rules we settled on

1. Filter before analysing: keep rows with a small gap (a week or two) and a defined, small σ_t.
2. Identify objects by position: propagate the TLE with SGP4, convert TEME to J2000, and compare
   with the Vimpel state at the Vimpel epoch. A match is tens of kilometres.
3. Track an object across several issues before trusting it; check that its number carries one
   continuous orbit.
4. For event reconstruction use only low-AMR, well-tracked objects, and expect membership counts
   to rise for weeks.
5. Treat AMR, magnitude and σ values as estimates that are revised between issues.

## 5. Source and attribution

The catalogue files are redistributed as downloaded from <http://spacedata.vimpel.ru/>, where they
are available to registered users under the portal's user agreement. **(publisher)** A reference
to the site is mandatory when the data are used in any publication. The data are prepared jointly
by JSC "Vimpel Interstate Corporation" and the Keldysh Institute of Applied Mathematics.

The portal also publishes per-object ephemerides, `datefirst.txt` (Vimpel number ↔ NORAD number
and first-appearance dates) and `duplicates.txt` (merged object numbers). They are not part of
this archive.
