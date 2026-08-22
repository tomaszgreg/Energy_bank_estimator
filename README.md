# Home Battery Tariff Calculator

A single-file web app that compares Polish household electricity tariffs — with and without
a battery — using your own hourly meter data.

Everything runs in the browser. Open `index.html` and go. There is no build step, no server,
no dependency to install, and your meter readings never leave the device.

## What it answers

You have hourly meter exports and you want to know whether switching from a flat G11 tariff to
a two-zone G12 tariff pays off, and whether adding a 14 kWh or 28 kWh battery on top of that
pays off. The app computes the actual bill for each option, hour by hour, over as many months
as you feed it.

It compares up to seven configurations side by side:

| Row | Tariff | Battery | Night pre-heating |
|---|---|---|---|
| G11 | flat | — | — |
| G12 | two-zone | — | — |
| G12 + A kWh | two-zone | variant A | — |
| G12 + B kWh | two-zone | variant B | — |
| G12 + noc | two-zone | — | yes |
| G12 + A kWh + noc | two-zone | variant A | yes |
| G12 + B kWh + noc | two-zone | variant B | yes |

Battery capacities and power ratings are editable, so "A" and "B" are whatever you want to
evaluate.

## Input data

Hourly exports from the **ePGE** customer portal — one file per month, `.xls` or `.xlsx`.
Load several at once; each month gets its own card and the totals accumulate.

The app reads the `Dane` sheet and uses the `En. Czynna zbilansowana` rows, which carry the
hourly-netted balance in columns `H01`–`H24`. Positive values are imports, negative values are
exports. This is the same basis the utility bills on, so no averaging or interpolation is
involved anywhere in the calculation.

> Files exported by ePGE carry an `.xls` extension but are actually XLSX (a ZIP archive)
> inside. Genuine legacy BIFF `.xls` files are not supported — open one in a spreadsheet
> program and save it as `.xlsx` first.

## Price data

Surplus energy is settled hour by hour against **RCE**, the Polish balancing-market price.
Three ways to supply it:

1. **Fetch from PSE** — pulls `rce-pln` straight from `api.raporty.pse.pl` for every loaded
   month. Prices are published in 15-minute resolution and averaged to hourly, because
   prosumer meters report hourly (Art. 4b(11) of the RES Act).
2. **Upload a file** — a CSV or XLSX report downloaded manually from `raporty.pse.pl`.
   The parser is tolerant about column order and separators.
3. **Fallback price** — a single flat value from the settings, used for any hour with no data.
   Useful for what-if scenarios, and it makes the app fully usable offline.

Fetched prices can be cached so you do not re-download them every session. See
[Where prices are stored](#where-prices-are-stored).

## How the model works

### Tariff zones

G12 cheap-zone hours follow the seasonal split:

- **Winter** (1 Oct – 31 Mar): 22:00–06:00 and 13:00–15:00
- **Summer** (1 Apr – 30 Sep): 22:00–06:00 and 15:00–17:00

The season is derived from the date inside the file, so mixed-year data sets work correctly.

### Battery

The battery keeps two separate balances: energy that came from the PV array and energy bought
from the grid. This matters because only PV-origin energy may legally be exported by
a prosumer, and the model enforces that.

- **The house always comes first.** Discharge covers household load before anything else, and
  it draws down grid-origin energy first so the PV pool stays available for export.
- **Night charging looks ahead.** The battery buys only as much as the coming expensive block
  will actually need beyond own production. Without this it would sit full of grid energy at
  dawn with nowhere to put the morning's PV.
- **Export splits into two tranches.** The *reserve* — what the house will consume before PV
  refills the battery — is sold only when the price beats self-consumption. The *spare* — the
  rest, which in summer is most of the pack — is sold at the best-paying hour before the next
  recharge, with no profitability threshold, because its alternative is being dumped into
  tomorrow's midday price trough.
- **The break-even is computed, not guessed.** With the default rates, selling beats
  self-consumption above roughly 1010 PLN/MWh.

Round-trip efficiency, charge/discharge power and a PV headroom reserve are all configurable.

### Prosumer deposit

Exported energy is valued at `kWh × RCE × 1.23` and credited to a deposit that offsets the
energy portion of later bills — never the distribution charges.

The deposit is modelled as FIFO buckets with age. Each month ages them, expires anything past
12 months, refunds up to 20% of the expired bucket's original value, then spends the oldest
remaining credit. Without expiry the balance grows without bound on a PV-heavy profile and the
annual figures stop meaning anything.

Whether the deposit reduces the gross or the net energy value is a toggle, since practice
varies between suppliers.

### Year looping

With all twelve months loaded, **Loop the year** feeds December's closing state — deposit
balance and battery charge — back into January and re-runs until the annual bill stabilises,
usually after three or four passes.

This matters more than it sounds. A cold start makes January and February pay full price with
a nonexistent summer behind them; on a PV-heavy profile the first pass can overstate the
annual bill threefold.

### Night pre-heating

Two numbers per month: how many kWh per day you can move out of expensive hours by heating
domestic hot water and the floor slab harder at night, and how much extra you burn for it in
standing losses.

Energy is taken proportionally from expensive hours that actually show consumption — where PV
is running there is nothing to move, so no artificial export is created — and added evenly
across cheap-zone hours. If a given day has less expensive-zone load than you asked to shift,
only what exists is moved and the losses scale down with it.

Shifting and storage overlap heavily: both push consumption into the cheap zone, so their
savings do not add up. The side-by-side rows exist precisely to show that.

## Output

- **Per-month cards** — one table per month plus a day × hour heat map of grid exchange, with
  a tab per configuration so you can see the battery flatten the expensive-zone columns.
- **Period totals** — the same table shape, aggregated.
- **Monthly bill chart** — one line per configuration. Configurations differing only by night
  pre-heating share a colour and use a dashed line, so the gap between a solid and dashed line
  of the same colour is the value of shifting in that month.

## Where prices are stored

Fetched RCE prices can be kept in two places:

- **Browser storage** — automatic, restored on the next visit. Requires a normal origin. If
  you open `index.html` straight from the file system or from an Android downloads provider,
  the origin is opaque and storage is blocked; the app detects this and says so.
- **A JSON file** — the *Save prices to file* button writes `ceny-rce.json`, which you reload
  through the same picker as CSV reports. This always works.

Serving the repository through GitHub Pages gives a proper `https://` origin, which makes
browser storage work and is the easiest way to use the app on a phone.

## Privacy

- No cookies, no analytics, no tracking of any kind.
- No external resources are loaded at startup — system fonts, and an XLSX reader built into
  the page.
- Meter data is parsed locally and is never uploaded or written to disk.
- The only outbound request is to the PSE price API, and only when you press the button.
- The only thing that may be persisted is RCE prices, opt-in and clearable.

## Defaults

Shipped values are the 2026 PGE Obrót and PGE Dystrybucja tariffs for a three-phase connection
on monthly settlement, in PLN/kWh net:

| Component | G11 | G12 day | G12 night |
|---|---|---|---|
| Energy | 0.4982 | 0.5656 | 0.3718 |
| Variable network charge | 0.3469 | 0.4014 | 0.0765 |
| Quality + RES + cogeneration | 0.0435 | 0.0435 | 0.0435 |

Fixed monthly charges, net: network 9.98 (G11) / 14.39 (G12), subscription 4.50, capacity
24.05. VAT 23%.

**Check these against your own invoice**, particularly the fixed network charge, which depends
on the number of phases, and the capacity charge, which is banded by annual consumption. Every
field is editable and results recompute as you type.

## Requirements

Any current browser. The built-in XLSX reader uses the native
`DecompressionStream('deflate-raw')` API: Chrome 80+, Edge 80+, Firefox 113+, Safari 16.4+.

## Licence

GNU General Public License, version 2 — see [`LICENSE`](LICENSE) or
<https://www.gnu.org/licenses/old-licenses/gpl-2.0.html>.

`SPDX-License-Identifier: GPL-2.0-only`

Copyright © 2026 Tomasz Gregorek

## Disclaimer

This is an estimate, not a quotation and not a bill forecast. It models published tariffs and
a simplified battery control strategy. Real results depend on your inverter's actual
scheduling capability, battery derating, metering details and your supplier's settlement
practice. Verify anything you intend to spend money on.
