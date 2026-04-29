# NAXOS Protocol Analysis (Handout 1)

This document collects notes on the attacks discovered by removing single arguments from the key derivation function `h2` of the NAXOS protocol.

## Base Protocol
The original function is:
- **Initiator**: `h2(< Y^~lkI, pkR^exI, Y^exI, $I, $R >)`
- **Responder**: `h2(< pkI^exR, X^~lkR, X^exR, $I, $R >)`

If the function is intact, Tamarin mathematically verifies the key secrecy (`key_secrecy`).

---

## Case 1: Removal of the 1st Argument
- **Initiator uses**: `h2(< pkR^exI, Y^exI, $I, $R >)`
- **Responder uses**: `h2(< X^~lkR, X^exR, $I, $R >)`
- **Tamarin Result**: ATTACK FOUND (7 steps)
- **Explanation (Neutral Element Attack on Responder)**: 
  The adversary intercepts the initiator's message and sends the neutral element `X = 1` to the responder. As a result, both `X^~lkR` and `X^exR` become `1`. 
  The responder computes the key as `h2(< 1, 1, $I, $R >)`. The adversary, knowing the public values `1`, `$I`, and `$R`, can compute exactly the same hash and obtain the correct session key.
  The removed argument (`pkI^exR` and symmetrically `Y^~lkI`) prevented this attack by forcing the use of a secret component that the adversary could not compute even if `X=1`.

---

## Case 2: Removal of the 2nd Argument
- **Initiator uses**: `h2(< Y^~lkI, Y^exI, $I, $R >)`
- **Responder uses**: `h2(< pkI^exR, X^exR, $I, $R >)`
- **Tamarin Result**: ATTACK FOUND (7 steps)
- **Explanation (Neutral Element Attack on Initiator)**: 
  This is the exact symmetric attack of Case 1, but applied against the Initiator. The adversary intercepts the responder's message and sends the neutral element `Y = 1` to the initiator. 
  Both `Y^~lkI` and `Y^exI` become `1`. The initiator computes the key as `h2(< 1, 1, $I, $R >)`, which the adversary can trivially compute. The removed argument `pkR^exI` was responsible for tying the key to the responder's public key, preventing this flaw.

---

## Case 3: Removal of the 3rd Argument
- **Initiator uses**: `h2(< Y^~lkI, pkR^exI, $I, $R >)`
- **Responder uses**: `h2(< pkI^exR, X^~lkR, $I, $R >)`
- **Tamarin Result**: VERIFIED (22 steps)
- **Explanation**: 
  Interestingly, Tamarin proves that basic key secrecy is still maintained! Why? Because the remaining arguments (`Y^~lkI` and `pkR^exI`) are still mathematically hard for the adversary to compute, as they require knowing the long-term keys of the honest parties.
  **However**, removing the ephemeral-ephemeral component (`Y^exI` / `X^exR`) destroys **Perfect Forward Secrecy (PFS)**. If the adversary compromises the long-term keys *in the future*, they would be able to decrypt past sessions. The current `key_secrecy` lemma does not test for PFS, which is why it reports "Verified".

---

## Case 4: Removal of the 4th Argument (Identity I)
- **Initiator uses**: `h2(< Y^~lkI, pkR^exI, Y^exI, $R >)`
- **Responder uses**: `h2(< pkI^exR, X^~lkR, X^exR, $R >)`
- **Tamarin Result**: VERIFIED (22 steps)
- **Explanation**: 
  The `key_secrecy` lemma is still verified because the cryptographic material (the Diffie-Hellman shares) is still secret. The adversary cannot compute the hash without the secrets.
  **However**, removing the identities from the key derivation function usually opens the door to **Unknown Key-Share (UKS) attacks**. In a UKS attack, the secrecy is maintained, but the *authentication* is broken (e.g., Alice thinks she is talking to Bob, but Bob thinks he is talking to Eve). Since our lemma only checks secrecy and not authentication, Tamarin reports it as verified.

---

## Case 5: Removal of the 5th Argument (Identity R)
- **Initiator uses**: `h2(< Y^~lkI, pkR^exI, Y^exI, $I >)`
- **Responder uses**: `h2(< pkI^exR, X^~lkR, X^exR, $I >)`
- **Tamarin Result**: VERIFIED (22 steps)
- **Explanation**: 
  Exactly the same reasoning as Case 4. Secrecy is maintained because the Diffie-Hellman secrets are intact, but removing the responder's identity from the hash makes the protocol vulnerable to Unknown Key-Share (UKS) authentication attacks, which are not covered by this specific `key_secrecy` lemma.
