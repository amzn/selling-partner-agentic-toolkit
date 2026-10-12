# Seller Central URLs by marketplace

Maps the **marketplaceId the skill is invoked for** to that marketplace's Seller Central host,
so the skill can build a correct region-specific link to the seller's Payments page:

```
<Seller Central host>/amazonsell/payments
```

**Why this table exists:** Seller Central accounts are partitioned by region (North America /
Europe / Far East). A host from the wrong region does **not** show the seller's data, so the link
must be built from the marketplaceId being closed — never hardcoded to one host.

**How to use:** look up the invoked `marketplaceId` below, take its host, and append
`/amazonsell/payments`. If the `marketplaceId` is **not in this table** (or is unknown at
runtime), do **not** guess a host — instead tell the seller to open **Payments** in their Seller
Central account (no link).

| marketplaceId | Country | Seller Central host | Payments link |
|---|---|---|---|
| A2EUQ1WTGCTBG2 | Canada | https://sellercentral.amazon.ca | https://sellercentral.amazon.ca/amazonsell/payments |
| ATVPDKIKX0DER | United States | https://sellercentral.amazon.com | https://sellercentral.amazon.com/amazonsell/payments |
| A1AM78C64UM0Y8 | Mexico | https://sellercentral.amazon.com.mx | https://sellercentral.amazon.com.mx/amazonsell/payments |
| A2Q3Y263D00KWC | Brazil | https://sellercentral.amazon.com.br | https://sellercentral.amazon.com.br/amazonsell/payments |
| A1RKKUPIHCS9HS | Spain | https://sellercentral-europe.amazon.com | https://sellercentral-europe.amazon.com/amazonsell/payments |
| A1F83G8C2ARO7P | United Kingdom | https://sellercentral-europe.amazon.com | https://sellercentral-europe.amazon.com/amazonsell/payments |
| A13V1IB3VIYZZH | France | https://sellercentral-europe.amazon.com | https://sellercentral-europe.amazon.com/amazonsell/payments |
| A1805IZSGTT6HS | Netherlands | https://sellercentral.amazon.nl | https://sellercentral.amazon.nl/amazonsell/payments |
| A1PA6795UKMFR9 | Germany | https://sellercentral-europe.amazon.com | https://sellercentral-europe.amazon.com/amazonsell/payments |
| APJ6JRA9NG5V4 | Italy | https://sellercentral-europe.amazon.com | https://sellercentral-europe.amazon.com/amazonsell/payments |
| A2NODRKZP88ZB9 | Sweden | https://sellercentral.amazon.se | https://sellercentral.amazon.se/amazonsell/payments |
| A1C3SOZRARQ6R3 | Poland | https://sellercentral.amazon.pl | https://sellercentral.amazon.pl/amazonsell/payments |
| ARBP9OOSHTCHU | Egypt | https://sellercentral.amazon.eg | https://sellercentral.amazon.eg/amazonsell/payments |
| A33AVAJ2PDY3EV | Turkey | https://sellercentral.amazon.com.tr | https://sellercentral.amazon.com.tr/amazonsell/payments |
| A2VIGQ35RCS4UG | United Arab Emirates | https://sellercentral.amazon.ae | https://sellercentral.amazon.ae/amazonsell/payments |
| A17E79C6D8DWNP | Saudi Arabia | https://sellercentral.amazon.com.sa | https://sellercentral.amazon.com.sa/amazonsell/payments |
| A21TJRUUN4KGV | India | https://sellercentral.amazon.in | https://sellercentral.amazon.in/amazonsell/payments |
| A19VAU5U5O7RUS | Singapore | https://sellercentral.amazon.sg | https://sellercentral.amazon.sg/amazonsell/payments |
| A39IBJ37TRP1C6 | Australia | https://sellercentral.amazon.com.au | https://sellercentral.amazon.com.au/amazonsell/payments |
| A1VC38T7YXB528 | Japan | https://sellercentral.amazon.co.jp | https://sellercentral.amazon.co.jp/amazonsell/payments |

**Sources**
- Seller Central hosts by country: Amazon Seller Central URLs (developer docs).
- Marketplace IDs: internal Marketplace ID wiki (w.amazon.com/index.php/MarketplaceID), cross-checked
  against SP-API Marketplace IDs (developer docs).
- A row is included only when **both** a verified marketplaceId and a Seller Central host are
  known. A marketplace with a known Seller Central host but no host coverage here (or vice versa)
  falls through to the no-link "open Payments" fallback so the skill never emits an unverified host.
