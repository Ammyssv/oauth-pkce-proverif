# ProVerif Models: Authorization Code Replay Prevention and Session Binding in OAuth 2.0 with PKCE

This repository contains the ProVerif models and outputs for the paper

> Souksavay Sivisay, Yuichi Futa, Hiroyuki Okazaki, and Takehiko Mieno, "Formal Verification of Authorization Code Replay Prevention and Session Binding in OAuth 2.0 with PKCE Using ProVerif."

It contains every model whose results are reported in the paper, the command to run each one, and the full ProVerif output. Each output includes the attack trace of every query that returns false.

## Requirements

- [ProVerif 2.05](https://bblanche.gitlabpages.inria.fr/proverif/)

## Repository structure

```
README.md
section6/                       Main model and its variants (Section 6, Tables 3 and 4)
section7-3-replay-resistant/    Replay-resistant channel and client retries (Section 7.3, Table 5)
section7-3-shared-key-tls/      Alternative TLS abstraction with one shared key (Section 7.3)
```

Each model `<name>.pv` has its output `<name>.txt` in the same folder.

## How to run

```
proverif section6/main.pv > section6/main.txt
```

Replace the path with any other model. The results of all queries are listed under `Verification summary` at the end of each output.

For a reachability query `query event(E).`, ProVerif reports `not event(E) is false` when the event is reachable, and `not event(E) is true` when it is not.

## Section 6: main model and variants (Tables 3 and 4)

All models in this folder include the request identifier `rid` used by properties B1 and B2. The main model has 29 security queries and 17 reachability queries (R1 to R17).

| File | Change from the main model | Results that differ from the main model |
|---|---|---|
| `main` | (main model, Sections 4 and 5) | none: all 29 security queries are true, and all reachability queries are reachable |
| `no_used_code_check` | Lookup and record of the used code removed (Lines 3–12 of Listing 4) | P0, Q-SU and Q7 are false (token-request replay, Section 6.3); R4 is not reachable |
| `no_pkce` | PKCE check `s256(codeVerifier) = originalChallenge` replaced by `true` (Line 12 of Listing 3) | Secrecy of the access token, Q2, Q5+, Q6+, Q16+, S1, S2, S3 and S5 are false (Section 6.4) |
| `no_lookup` | Lookup (`get`) removed (Line 3 of Listing 4); record and restriction of Listing 5 kept | R4 is not reachable; all security queries remain true |
| `no_restriction` | Restriction of Listing 5 (atomicity assumption) removed; lookup and record kept | P0, Q-SU and Q7 are false (two requests pass the lookup before either records the code, Section 6.2); R4 cannot be proved |
| `negative_control_record_before_pkce` | Lookup and record (Lines 3–12 of Listing 4) moved before the PKCE check | B1 and B2 are false (a request with a wrong verifier consumes the code, Section 6.2) |

## Section 7.3: replay-resistant channel and client retries (Table 5)

These models start from the main model or from `no_used_code_check` and add:

- **Replay-resistant channel:** an event `TLSRecordAccepted` and a restriction stating that the server accepts each encrypted part of a token request at most once. This models the TLS record layer, which rejects a record it has already accepted (RFC 8446, Appendix E.2).
- **Client retries (`retry` files only):** the legitimate client may send its token request any number of times, each time on a new connection with a fresh response key.

| File | Client retries | Used-code check | Single use (P0, Q-SU, Q7) | R4 |
|---|---|---|---|---|
| `rrc_with_check` | no | yes | true | cannot be proved |
| `rrc_without_check` | no | no | true | not reachable |
| `rrc_retry_with_check` | yes | yes | true | reachable |
| `rrc_retry_without_check` | yes | no | **false** | not reachable |

All other security queries are true in all four models.

## Section 7.3: alternative TLS abstraction with one shared key

These three models model TLS by symmetric encryption under one key `tlsKey` that all sessions share and the attacker does not know. They are an **earlier version of the model**, written before TLS was modeled per connection. Besides the TLS abstraction, they differ from the main model as follows:

- the used-code check comes before the PKCE check;
- the `redirect_uri` is not bound into the code;
- they use the earlier queries Q5 and Q6, and do not include S5, B1, B2 or the request identifier `rid`.

The earlier queries Q5 and Q6 count any response to the attacker, including an error, as an attack. They are therefore false in all three models by construction, and the paper replaces them with Q5+, Q6+ and Q16+.

| File | Session identifier check | PKCE | Results |
|---|---|---|---|
| `shared_key_with_session_id_check` | on | on | all queries true except Q5 and Q6 |
| `shared_key_without_session_id_check` | off | on | S1, S2 and S3 are false: a response for one session is forwarded to another session of the same client, which decrypts it with the shared key |
| `shared_key_no_pkce` | on | off | same results as `shared_key_with_session_id_check`: the attacker cannot open its own connection, so it cannot use an intercepted code |

These two effects come from the shared-key abstraction, not from the protocol. The main model avoids both by modeling TLS per connection (Section 7.3).

## Notes

- All outputs were produced with ProVerif 2.05 from exactly the model files in this repository.
