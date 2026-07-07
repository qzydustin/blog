+++
title = "Cell Phone Plans Worth Keeping and How to Get Them"
date = 2022-10-08T16:20:35-07:00

[taxonomies]
categories = ["Mobile"]
tags = ["wireless", "t-mobile", "prepaid", "qci", "porting"]
+++

The cheapest cell plans in America are not advertised. They are grandfathered prepaid lines T-Mobile no longer sells, a $10/month unlimited tablet line, and a pay-per-use number you can keep alive for about a dollar a year.

<!--more-->

## Network Priority and Line Management 

### Number Basics

- **Wireless status (Y/N):** banks, Google, and many signup flows reject "wireless N" (VoIP or landline) numbers. The status follows the plan and cannot be set manually. Check any number at [freecarrierlookup.com](https://freecarrierlookup.com). Every T-Mobile keeper line below is wireless Y.
- **Porting:** moving a number between carriers is routine. Get the account number and PIN/passcode from the old carrier and hand them to the new one. Do not cancel the old line; the new carrier pulls the number through.
- **Do Not Call:** register at [donotcall.gov](https://www.donotcall.gov) to cut telemarketing.
- **Caller ID:** set by the provider; some charge for it, some do not.

### Porting Out a Number Without Losing the Line

Porting a number out normally cancels the line and destroys the plan. The workaround: change the number on the keeper line, add a cheap relay line, move the old number onto the relay, then port out from there.

For T-Mobile: contact **T-Life chat** and ask the rep to change the number on the keeper line. The original number drops into the carrier's dormant pool. Add a watch line (around $5/month, skips the $35 activation fee, does not count against the 12-voice-line cap), then have the rep recover the original number onto the watch line. Port out from the watch line; only the watch line closes.

Other carriers follow similar mechanics: contact customer service, request a number change on the keeper line, add a low-cost line, recover the old number onto it, then port out from the relay line.

Credentials for the port-out (T-Mobile): the **account number** (Manage → Account → Account details), a **transfer PIN** (Profile / Security; six digits, valid four days), and the **billing ZIP**.

### QCI Explained

QCI (QoS Class Identifier, 5QI on 5G) is the network tag that decides whose traffic goes first when a tower is congested. A lower number means higher priority. The key distinction: **deprioritization is not throttling**. A deprioritized line runs at full speed unless the tower is actually congested.

On **Verizon**, most consumers sit at QCI 8 ("premium data"), the normal priority tier; prepaid and most MVNOs drop to QCI 9. On **AT&T**, the top tier is QCI 6 (FirstNet and business plans), with postpaid consumers at QCI 7–8 and MVNOs at QCI 9. AT&T's network absorbs congestion more gracefully, so even QCI 9 stays usable. On **T-Mobile**, postpaid concentrates at QCI 6: Magenta, T-Mobile Prepaid, and the $10 Business Tablet line. Mid-tier MVNOs land at QCI 7, hotspot lines at QCI 8, and deprioritized traffic at QCI 9.

The same QCI number does not mean the same thing across carriers. A QCI 8 line on Verizon is normal priority; a QCI 8 line on T-Mobile is heavily deprioritized. That is why a grandfathered T-Mobile postpaid line at QCI 6 still beats a newer plan in congestion performance, and why a cheap MVNO on Verizon can feel worse than one on AT&T at the same nominal tier.

## The God-Tier Plans

### T-Mobile Legacy Pay As You Go ($0/month)

- **The deal.** $0 monthly fee, pay-per-use at $0.10/min voice, $0.10/SMS, $0.25/MMS. Wireless status Y, with eSIM, Wi-Fi Calling, paid roaming, and Caller ID.
- **Gold status.** Accumulate $100 in refills on a legacy PayGo account and you reach Gold, after which **any refill of any amount extends the balance a full 365 days**. The classic keep-alive was a $10 annual refill. After T-Mobile migrated legacy prepaid to a new billing platform, the minimum refill dropped to $1, so a Gold line now lives on about a dollar a year.
- **Availability.** Officially Gold is retired and reserved for accounts on the legacy Pay As You Go plan before August 17, 2014, but those legacy accounts still reach and keep Gold in 2025. The $0 plan itself is not fully closed either: a rep can still open one (confirmed in 2025), even though self-service funnels you to Connect or the $3 PayGo. What is unconfirmed is whether a freshly rep-opened $0 line can then climb to Gold by refilling $100; T-Mobile's official stance is no, but no public data point confirms or contradicts it.
- **Keep-alive.** Refill manually before the expiration date. A Gold line takes $1 a year; a non-Gold line takes $1 a quarter (90 days), since refills under $100 expire in 90 days. Do not enable AutoPay: the $0 monthly rent rolls the due date forward every day, which AutoPay cannot handle and will break.
- **How to get one.** Buy an already-Gold legacy account on eBay, or ask a rep to open a $0 PayGo line. A non-Gold legacy account can reach Gold by refilling $100.

### T-Mobile Pay As You Go ($3/month)

- **The deal.** $3/month covers a combined pool of 30 minutes or 30 messages; overage about $0.10/min, $0.10/SMS, $0.25/MMS. The 30 units do not roll over. A flat, predictable bill, unlike the $0 legacy PayGo where you nurse a balance. Same T-Mobile network, with eSIM, Wi-Fi Calling, paid roaming, and Caller ID.
- **How to get one.** This is the tier any T-Mobile monthly prepaid line drops into when it cannot renew. Activate a cheap monthly prepaid plan (Connect is the usual starting point), fund the account so the balance sits below the monthly price but at least $3, and let the renewal fail. At the 30-day cycle the line automatically converts to this $3/month plan and draws $3 per cycle. Keep the balance at or above $3; once a renewal cannot cover even the $3, the line moves toward suspension and deactivation.

### T-Mobile Business Tablet Line ($10/month)

- **The deal.** $10/month, all taxes and regulatory fees included. Officially a T-Mobile Business Tablet plan; the $10 edition is SOC code **ZB10HSTI**, with a later $15 replacement carrying ZTP10HTI. On the device: unlimited 5G/4G LTE, no hard cap, no overages, video at 480p. Tethering: 10GB of high-speed hotspot, then 3G-speed.
- **Advantage: QCI 6.** The line runs at QCI 6, the same priority tier as T-Mobile's flagship Magenta and Magenta MAX postpaid. Heavy users past 50GB in a month may be deprioritized during congestion, but the line is never hard-throttled.
- **International roaming.** 5GB of high-speed data in Mexico and Canada, then 256 kbps; unlimited 2G-speed (256 kbps) roaming across 210+ countries. In mainland China it works with no extra settings and no VPN, because T-Mobile's roaming already reaches Google, Gmail, and the rest of the blocked western stack. Speed is modest (messaging and email fine, streaming not).
- **How to activate.** Visit a Costco T-Mobile kiosk or a corporate (COR) store. Provide your SSN to open a T-Mobile Business account (no actual business or EIN required; expect a soft credit check, so lift any credit freeze beforehand). Hand the rep the SOC code **ZB10HSTI** and ask a manager to waive the $35 activation fee. The line is labeled "tablet" but runs in a phone for data and SMS. eSIM is supported.

### T-Satellite (Starlink): $10/month

- **The deal.** $10/month for 50GB of high-speed data (QCI 6) on T-Mobile's 5G/4G network plus Starlink satellite messaging and data off-grid. Standalone line with a new number and no voice. Anyone can open one, including AT&T and Verizon customers. For existing T-Mobile lines, satellite is a $10/month add-on, or free on Experience Beyond and Go5G Next. The online signup page lists $15/month, but a $5/month discount applies automatically.
- **The satellite side.** Off-grid, the line falls back to Starlink for SMS, location sharing, picture and audio messages, select apps, satellite data, and texts-to-911. It works on 60+ phone models. The satellite service is U.S.-only. No hotspot support without workarounds.
- **How to activate.** For a new standalone line, sign up online at [t-mobile.com](https://www.t-mobile.com/coverage/satellite-phone-service/choose-your-satellite-service), select "I'm new to T-Mobile," and complete activation with eSIM. No SSN or credit check required. Existing T-Mobile customers can add it in the T-Life app or call 1-800-937-8997. No activation fee. Must activate on a phone; iPads are not supported.

