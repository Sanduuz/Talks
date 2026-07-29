# Foreword

_I created a presentation of a real life story regarding NFC hacking for a HelSec event in 2026-05-07. This post aims to tell the same story in text, albeit a bit less in detail. This has been written by AI with human touch based on the original slides. The original slides (with more details) are attached at the end of the page._

Before going into the story, it is relevant for the story to know that the author has some microchip implants installed. (This might be another story for some other time.)

# Backstory

When I started my studies at Aalto University in September 2024, I quickly ran into a small but persistent annoyance: getting around campus required an access token.

There were two options presented for the access token. The first was a mobile application. In theory, that sounded convenient. No extra card, no extra thing to remember, just a phone. In practice, it was slow enough to become irritating. Sometimes opening a door took up to 15 seconds. Sometimes the reader did not react at all. The app also wanted permissions I was not excited to grant, including location and Bluetooth.

The second option was an official keycard. That one worked as access cards are supposed to work: tap, open, continue with your day. The downside was that it meant carrying another card (_"oh no, what a big problem..."_).

Obviously I would like to get the official keycard, so I went to ask for a card.

The answer was: _"Use the new mobile application instead."_

Great... That left me with a problem. The app was not reliable enough for daily use, but I could not get the official keycard. Fortunately, there was a third option: Aalto allowed local Helsinki Region Transport (HRT, later HSL: Helsingin seudun liikenne) travel cards to be enrolled as access tokens through the university's Identity Manager.

That solved the immediate usability issue, but it also raised a more interesting question: what exactly was the access-control system trusting?

# Curiosity

My first motivation was practical curiosity:

Could I move my access token from a card into one of my implants and stop carrying the card entirely?

That led to a broader set of questions:

- Could the official access card be cloned?
- Could the HSL travel card be cloned?
- Could a custom card be created from scratch?
- Was the system relying on cryptographic proof from the card, or only on an identifier?

# Research

I started with simple reconnaissance using a Flipper Zero, then moved to a Proxmark3 RDV4.01 for deeper analysis.

Both the official Aalto keycard and the HSL travel card were MIFARE DESFire cards. That mattered, because DESFire is not the kind of legacy card technology where "cloning" usually means dumping everything and writing it to another tag.

DESFire cards are structured around applications and files. Files have access rights. Operations can be protected by keys. Communication can be plaintext, MACed/authenticated, or fully encrypted. In the normal case, if a system uses DESFire properly, cloning a card without the relevant keys should not be practical.

That was the first important result: this was not about breaking DESFire.

The official card behaved like a properly locked-down card. Its useful access-control data required keys I did not have.

The HSL card was more interesting.

The HSL card also used a DESFire application, but parts of the data were readable. Among the readable data was a small application-info field.

That field contained several values, including a card version marker. Old green HSL cards and newer blue HSL cards had different version values. Another byte appeared to be constant across tested cards.

More importantly, the same identifier visible on the physical HSL card also appeared in the readable application-info data. It was also the same identifier used when registering a new HSL card as an access token in Aalto's Identity Manager.

At that point, the shape of the risk changed.

It is unlikely that Aalto would have access to HSL's private keys. This would lead to scenarios where Aalto could issue valid HSL cards. On the other hand, if Aalto doesn't have the keys then the information used in authentication must be unprotected.

# The attack

If Aalto's system only needed the HSL card identifier in the expected place and format, then a custom DESFire card could potentially impersonate an HSL card well enough for campus access enrollment.

That is exactly what testing showed.

Using a blank DESFire card, I recreated the minimum structure the access-control workflow appeared to expect and placed a chosen token identifier into the relevant readable field.

The Identity Manager accepted the resulting card as an HSL card.

This meant two things:

1. A custom DESFire card could be enrolled as an access token.
2. An existing HSL card could be impersonated if its identifier was known.

The second point is the more serious one. The identifier was not a secret. It could be read from the card at close range, and it was also printed on the card. A photo of the back of a card could be enough to reveal it.

Again, this was not a cryptographic break. It was a trust-boundary problem. The system treated a card-presented or user-supplied identifier as if it proved possession of a legitimate token.

# Taking it a step further

After confirming that custom cards could be enrolled, I tried one more simple test:

What happens if I try to enroll a token identifier that is already assigned?

The Identity Manager returned an error saying that a token with that ID had already been assigned or reported lost.

That response created an oracle. By submitting candidate token identifiers, the system revealed whether they corresponded to existing access tokens.

That turned the issue from "I can make my own token ID" into "I can distinguish valid token IDs from invalid ones." In access-control terms, that is a much bigger problem. A valid token could belong to someone with different privileges than my own.

The final finding was therefore:

- Custom HSL-like DESFire cards could be created.
- Existing HSL tokens could be impersonated if their identifier was known.
- The enrollment flow disclosed whether a submitted token identifier was already valid.
- The weakness was in the surrounding identity and enrollment logic, not in DESFire itself.

The core lesson is that identifiers are not authenticators.

An identifier answers "which token is this supposed to be?" It does not answer "is this a genuine token?" or "is the person presenting it authorized to bind it to this account?"

Unless the access-control system can verify something cryptographic about the card, it is not really verifying an HSL card. It is verifying data that looks like HSL card data.

Sometimes digital locksmithing is less about picking the lock and more about noticing which door was never locked in the first place.

# Disclosure and aftermath

I sent an email about the issue to an information security professor on 2024-10-02. I did not receive a response.

On 2024-10-14, Aalto published an article saying that HSL cards would stop functioning as access tokens on the Otaniemi campus on 2024-11-15: https://www.aalto.fi/en/news/the-hsl-card-will-no-longer-function-as-an-access-token-on-the-otaniemi-campus-15112024

Coincidence? I think not.

According to the announcement, HSL card support should have ended on 2024-11-15. In practice, I later observed that HSL cards still worked in 2025 on a random subset of doors.

# Original slides

The original slides can be found [here](./Digital%20locksmithing.pdf).

# More information regarding HSL cards

HSL cards can now be parsed using the proxmark3 client :)

https://github.com/RfidResearchGroup/proxmark3/pull/3388
