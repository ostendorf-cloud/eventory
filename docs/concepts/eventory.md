# Eventory

Inventory and rental management for event technology, with a customer-facing rental shop.

## Problem

- The stock of gear is tracked outside any system, so what is in which case is knowledge, not data.
- Nothing records what is committed to a future job, so a double booking is only noticed on the
  loading ramp.
- Handing gear out and taking it back happens on a phone in a venue, where a desktop tool and a
  spreadsheet are both unusable.
- A rental enquiry has to be answered by checking availability by hand before a price can be named.
- Items have inspection and maintenance obligations whose due dates live in no calendar.
- When gear is subhired or sold on, nothing connects the movement to an invoice.

## Goals

- Every physical unit is identifiable by list, QR code, manufacturer barcode, serial number or
  NFC tag, and all five resolve to the same record.
- Containers nest: an item sits loose, in a case, or in a rack, and a container is itself a unit
  that can be moved with everything inside it.
- A job carries a date range, and availability for any item over any range is computed from the
  jobs committed to it.
- A conflict is surfaced when it is created, not when the van is loaded.
- Checking out and checking in works one-handed on a phone and on an iPad, including in a cellar
  with no signal, syncing when the connection returns.
- A customer can assemble a request from the rentable stock and receive a quotation.
- A quotation is calculated from the day rate and the chargeable days of the job.
- A discount can be given per line and on the whole job, as a percentage or an amount, and the
  reason is recorded with it.
- A confirmed job produces an offer or invoice in the accounting system without retyping,
  carrying the discounts as given rather than as an already-reduced price.
- Different people hold different rights: warehouse, office, read-only, owner.
- A tenant with an accounting system connected reads its customers from there; one without it
  keeps customers in Eventory.
- Inspection and service due dates are visible before the gear is committed to a job.
- A second tenant can be added without a data migration.

## Non goals

- Crew scheduling, vehicle loading and transport planning. Named now so the data model does not
  block them, not built.
- Being a CRM. Customers exist as counterparties, not as a sales pipeline.
- Accounting. Eventory produces documents through a port; solobill owns the ledger.
- A native mobile application in the first version.
- Automatic rental pricing beyond a day rate per chargeable day. Week rates, scaling and package
  prices wait; a discount covers the cases they would have handled.
- Subhire from other providers. Named so the movement model does not block it, not built.
- A bulk import tool. Around two hundred assets is an afternoon of typing, not a feature.
- Barcode label printing hardware integration beyond producing printable sheets.

## Approach

- Go API and Vue SPA on govue, Postgres, one Helm chart, images to ghcr. Public, behind login.
- Four model layers, kept distinct because conflating them is what makes rental tools rigid:
  - product, a type of gear with its specification, rental rate and rentable flag
  - asset, one physical unit with its own identifiers, condition and history
  - container, an asset that holds assets, so cases and racks need no separate model
  - location, where something is when it is not on a job
- Bulk goods without individual identity, such as cable and consumables, are a product with a
  counted quantity rather than an asset per unit, under the same availability engine.
- Identification is a table of identifiers of different kinds pointing at one asset, so QR,
  barcode, serial and NFC are rows, not columns, and a replaced label is a new row.
- Availability is a function of product, quantity and date range over committed job lines. One
  engine answers the warehouse view, the conflict warning and the shop, so they cannot disagree.
- A movement is an append-only row: an asset left for a job, came back, went to service, was sold.
  Current location is derived from movements, never stored and updated in place.
- The mobile surface is a PWA with an offline queue. Scans and movements are recorded locally and
  replayed, each carrying a client-generated id so a replay cannot double-count.
- Web NFC exists only in Chrome on Android. The identifier model is ready for NFC everywhere, the
  reader is wired where the browser supports it, and iOS keeps list, QR and barcode until a
  companion exists.
- The rental shop is a second, unauthenticated frontend over the same API. It reads the rentable
  catalogue and the availability engine and writes a draft offer. Nothing a shop visitor does
  commits stock.
- The shop resolves its tenant from the request host, so shop.ostendorf.tech and a later tenant's
  own domain reach the same handler.
- Accounting is a port with one adapter for solobill. The port speaks documents and customers, not
  solobill's schema, so a second adapter is a package and not a refactor.
- Customers are read through that port when one is connected, and solobill is then the source of
  truth. With no accounting system connected, a local customer store behind the same interface
  serves them, so the rest of the application never knows which case it is in.
- Authorisation is role-based with the permission checked in the service layer, not the handler,
  so the API, the shop and any later client share one answer.
- Every table carries `tenant_id` with Postgres row-level security beneath the application.

## Alternatives

- Buying easyjob, Rentman or EventWorx: the capability is right and the price is not, which is
  the reason this project exists.
- Modelling cases as a field on an item rather than as assets: rejected because a case is itself
  rented, inspected and moved, and the field would grow into the same model with worse edges.
- Storing current location on the asset: rejected because history is what answers "where was this
  when it broke", and a derived value cannot drift from its own log.
- A native app for scanning: rejected for the first version because a PWA covers QR and barcode
  on both platforms, and only NFC on iOS would gain from it.
- Letting the shop write directly into jobs: rejected because an unauthenticated surface must not
  reach anything that commits stock.
- Generating invoices inside Eventory: rejected because two systems issuing numbered documents is
  how numbering goes wrong.

## Open questions

- Which inspection regimes actually apply, and whether due dates block a commitment or only warn.
- Whether the shop asks for a customer account or only an email address.
- What the first release must do when the accounting system is unreachable at the moment a job is
  confirmed.
- Whether a shop visitor who is already a solobill customer is recognised, and on what.
- How a day rate is applied to a multi-day job: every day charged, or a convention of the trade
  that the first release should follow.
