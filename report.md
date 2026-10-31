# HEAD Freshening and Stored-Response Association

## Findings, candidate restrictions, and limits of the available evidence

Author: Antun Jurkovikj

Report date: 28 October 2026

Version: 1.0

Status: Informational research report; not a standards proposal

License: [CC BY 4.0](LICENSE) for original text. Quoted third-party material
is excluded; see the [license scope](README.md#license).

## Abstract

A cache can receive current metadata without receiving replacement content.
This report examines what associates that metadata with a retained GET response
when the updating exchange is a successful HEAD request. Its central finding
is conditional: under a non-conflict reading of RFC 9111's validator test, a
validator-free HEAD can renew freshness for old bytes even though the origin
now serves different bytes of the same length. A reading that requires at
least one matching validator prevents that trace. The report asks for an
explicit treatment of the zero-validator case.

The report also examines candidate restrictions and their limits. Requiring a
matching strong entity tag addresses representation-data identity, but does
not establish every field's applicability or every stored request association.
An additional check using effective Vary detects some incompatible associations;
it cannot recover an undisclosed selection dimension. These are distinct
claims, not a single completed repair.

Evidence consists of specification analysis, retained synthetic model results,
and a bounded source study. No positive deployed-cache reproduction of the
central HEAD-update trace is established. The report makes no claim of
widespread corruption, measured performance benefit, or first discovery.

## 1. Scope and analytical distinctions

The operation under study is an upstream `200` response to HEAD updating a
previously stored GET response while its content remains stored. It is not
merely serving HEAD from a cached GET, suppressing a GET body, fetching GET
upstream for a HEAD client, or processing a `304` response.

[RFC 9111 Section 4.3.5](https://www.rfc-editor.org/rfc/rfc9111.html#section-4.3.5)
defines this maintenance operation; Section 3.2 governs its field updates.
[RFC 9110 Section 9.3.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.3.2)
defines HEAD and permits specified field omissions.

This analysis separates three questions:

| Question | Evidence needed in the examples below |
|---|---|
| Are the retained representation data associated with the updating response? | More than equal length or the absence of a conflicting validator. |
| Does a particular incoming field describe the data or message to which it will be attached? | The field's own input and applicability, not just representation identity. |
| Does the retained original request association remain supported after selection metadata changes? | Relevant request-field information, not just a matching entity tag. |

These are analytical distinctions, not new HTTP requirements. The examples
assume an ordinary cache partition, a fixed target URI, and no transformation,
authorization change, Range processing, or applicable stale-serving permission
unless stated otherwise. They are paper traces, not captured network traffic.
Model records and source observations are identified separately from protocol
reasoning.

## 2. Central finding: validator-free renewal of freshness

### 2.1 A complete trace

Consider one cacheable resource at `https://example.test/item`, with no
negotiation. All requests use the same selection context. Displayed payloads
are exactly four ASCII octets; no trailing newline is part of either payload.

1. At 12:00:00, a GET stores this response:

   ```http
   HTTP/1.1 200 OK
   Date: Tue, 27 Oct 2026 12:00:00 GMT
   Cache-Control: max-age=0
   Content-Type: text/plain
   Content-Length: 4

   AAAA
   ```

   Neither `ETag` nor `Last-Modified` is present. The entry becomes stale.

2. The origin changes the selected content to `BBBB`, also four octets.
   At 12:00:10 the cache forwards a client's unconditional HEAD to the
   origin, which returns:

   ```http
   HTTP/1.1 200 OK
   Date: Tue, 27 Oct 2026 12:00:10 GMT
   Cache-Control: max-age=600
   Content-Type: text/plain
   Content-Length: 4
   ```

   There is no response content and again no validator. The length correctly
   describes the current hypothetical GET response, not the cache's `AAAA`.

3. Interpret the validator condition as requiring every received validator
   to match its stored counterpart. No validator was received, so there is
   no conflicting validator; the lengths also agree. The cache takes the
   update branch, installs the supplied freshness metadata, and retains
   `AAAA`.

4. A matching GET shortly afterwards can receive `AAAA` as fresh, although
   a GET reaching the origin would receive `BBBB`.

The new lifetime is deliberately much longer than the short interval in this
trace. The outcome does not depend on assuming that an update resets every
internal age timestamp. Equal length has supplied no evidence distinguishing
the changed-content case from an otherwise identical unchanged-content case.

Equal length is not necessary for this interpretation: if HEAD legitimately
omits `Content-Length`, Section 4.3.5 performs no length comparison. The trace
includes it to show that a successful length comparison is insufficient.

### 2.2 Interpretation and strongest counterargument

[RFC 9111 Section 4.3.5](https://www.rfc-editor.org/rfc/rfc9111.html#section-4.3.5),
paragraph 3, conditions its recommended update on "matching values for any
received validator fields" and agreement of any received length; otherwise it
recommends treating the entry as stale. Paragraph 1 explains that
HEAD maintenance can be useful "due to no validators being present in the
stored response". The central interpretation here treats the validator test
conjunctively, with no affirmative match required when the received set is empty.

The strongest alternative is that matching values presupposes at least one
received validator. Under that reading, HEAD without validators can detect a
change or leave an entry stale, but cannot renew its freshness. That prevents
this trace. The wording does not state the non-empty requirement explicitly,
and the no-validator rationale leaves the distinction important to implementers.

The finding is therefore conditional but consequential: the non-conflict
reading admits renewal without evidence of unchanged data; the alternative
requires an explicit account of the zero-validator branch. A conservative
cache can avoid the trace. It does not follow that all conforming caches
exhibit it, or that a particular deployed cache is vulnerable.

This is also not a claim that cached content must always equal current origin
content. The distinctive step is renewal after the old freshness grant has
expired, using an exchange that cannot distinguish these two four-octet bodies.

## 3. Weak association and timestamp provenance

A weak validator can associate distinct representation data. That alone does
not establish that every newly supplied descriptive field applies to the older
bytes. Conversely, it does not make every weakly validated update wrong:
retaining an acceptable earlier representation and preserving dependent fields
can be useful. The relevant question is the particular update and its inputs.
See [RFC 9110 Section 8.8.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-8.8.1).

An additional counterexample concerns an earlier candidate considered in this
work: admitting matching `Last-Modified` as positive HEAD-update evidence when
inferred strong under RFC 9110 Section 8.8.2.2. The retained model includes a
stored state with this history:

- A weak `304` update replaced `Date` but omitted `Last-Modified`.
- The cache kept the old content and modification time.
- The resulting fields show `Last-Modified` at 10:00 and `Date` at 11:00,
  despite the association with the retained bytes having been weak.
- A later HEAD repeats the modification time and weak entity tag.

Applying a time-gap inference to the reconstructed pair can make the old
timestamp appear stronger without adding evidence about the retained bytes.
This is a provenance limitation of that proposed admission test, not a claim
that modification dates can never be strong under
[RFC 9110 Section 8.8.2.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-8.8.2.2).

The HEAD model takes the weak-304 history as fixture input; it does not execute
the preceding `304` exchange. Accordingly, this result is not end-to-end
implementation evidence for that history or a general adjudication of `304`
processing. It explains why the candidate restriction tested below does not
promote a matching modification date into a positive HEAD-update witness.

The timestamp-provenance argument is manual analysis. The model does not
calculate timestamp strength or evaluate the Date/Last-Modified gap; it
refuses the update in fixture 04c under its fixed strong-ETag admission rule.

## 4. What a strong entity tag does not settle

### 4.1 A field can have a different input

Suppose stored GET content is `BASE` and an accurate strong entity tag remains
unchanged. HEAD still carries no message content. If a `Content-Digest` value
is supplied for that empty content, it is not a digest of the retained `BASE`.
The representation match does not make those two digest inputs equal.
[RFC 9530 Sections 2 and 3](https://www.rfc-editor.org/rfc/rfc9530.html#section-2)
distinguish content from representation digests; its
[HEAD example](https://www.rfc-editor.org/rfc/rfc9530.html#appendix-B.2)
illustrates the distinction directly.

The author's individual Internet-Draft
[Maintaining HTTP Integrity Fields in Cached and Cache-Generated Responses](https://datatracker.ietf.org/doc/html/draft-jurkovikj-http-integrity-cache-00#section-5.1),
`-00`, published on 20 August 2026, already applies this distinction to empty
HEAD content in Sections 5.1, 6.2, and 12.1. Its field-specific proposals are
work in progress, not adopted requirements.

This establishes a limit on a strong-tag-only assurance, not proof that every
cache copies the digest incorrectly. Section 3.2's update exceptions and
field-specific processing remain relevant. No generic preserve, recompute, or
remove policy for all metadata is established by the HEAD admission model.

### 4.2 New Vary can expose an unsupported request association

Consider this separate synthetic sequence:

1. A GET with `Accept-Language: en` stores `BASE`, strong ETag `"A"`, length
   four, a short freshness lifetime, and no `Vary`. Initially the origin
   serves `BASE` regardless of language preference.
2. After expiry, the origin changes selection: absent `Accept-Language`
   selects `BASE`/`"A"`; `en` selects `NEWW`/`"B"`.
3. A separate client's unconditional HEAD has no `Accept-Language`. Its
   upstream response contains `200`, ETag `"A"`, length four,
   `Vary: Accept-Language`, and a new freshness lifetime. It has no body.
4. Under the old stored metadata, the GET entry was a candidate for that
   HEAD. A matching-strong-tag admission test passes. If the update installs
   the new fields but retains the original `en` association, a later `en`
   request can select freshly marked `BASE` even though the HEAD described
   the absent-language selection.

Both strong tags are accurate. The missing evidence concerns the retained
request association, not a collision between tags or an incorrectly strong
validator. This is a correction-neighbor trace for the proposed strong-only
gate, not a separately reproduced deployment failure.

A candidate additional restriction checks the effective Vary before installing
the update. It compares every nominated field of the retained original request
with the actual upstream HEAD request, using applicable matching semantics.
Unknown information is not treated as known absence. A mismatch, wildcard or
uninterpretable Vary, or unavailable required information causes refusal of
the whole update, without migrating the entry to another request association.
A supplied Vary replaces the old value;
omission retains it. This restriction was modeled, not adopted as an RFC rule.

### 4.3 Omitted Vary is an information limit

Now change the preceding example so HEAD omits `Vary`. The necessary premise
is specific: this origin determines the selection-dependency metadata only
during content generation, making the HEAD omission permissible, while it can
obtain accurate selected-representation ETag and length separately.

With neither old nor supplied Vary, the candidate comparison has no nominated
field to check. It admits the update. The cache cannot tell whether the origin
still serves `BASE` for everyone or has introduced the unreported language
selection. The model preserves this as a limitation case, not a repair success.

The strongest counterargument succeeds against a broader defect claim:
permitted cached reuse is not a guarantee of current origin selection, and a
cache cannot enforce an unexpressed preference. This example therefore does
not establish cache nonconformance or a new erratum. It refutes a stronger
claim that the proposed gates always establish the current selection for the
stored original request. Even a present Vary field would not, by itself,
prove completeness of all selection dependencies.

## 5. Candidate restrictions are choices, not completed repairs

The central finding and the more conservative design response are different:

- Requiring at least one received matching validator closes the empty-set
  branch in Section 2. It does not, by itself, settle weak metadata association.
- Requiring a matching strong entity tag is a broader candidate restriction.
  The model also checks every received validator and any received length for
  conflicts. Modification dates remain consistency checks, not positive
  witnesses in that candidate.
- The effective-Vary comparison adds a check of available request-association
  information. It does not establish field applicability or reveal omissions.

Refusing an update does not require deleting the stored body, prevent delivery
of the current HEAD response, or automatically grant permission to serve stale
content to a GET client. Other retrieval and reuse decisions remain separate.

The strong-only policy deliberately gives up weak, modification-date-only,
and validator-free HEAD optimizations. Additional retrieval cost and the value
of those optimizations have not been measured. More selective field-preserving
approaches remain alternatives; this report does not establish that blanket
refusal is necessary or proportionate. Compatibility and mixed-implementation
behavior remain unresolved.

## 6. Evidence: what was modeled and what was not observed

### 6.1 Retained model results

Two author-retained model records support bounded design analysis:

| Record | Retained result | Limit |
|---|---|---|
| Strong-association corpus, revision 2, dated 25 August 2026 | 38 expected outcomes: six strong, sixteen weak, four validator-free, seven conflicts, five non-selections. | Preliminary selection and applicable update fields are supplied premises; no complete cache or preceding 304 processing is executed. |
| Effective-Vary successor, dated 28 September 2026 | 37 new single-candidate fixtures: thirteen admitted and twenty-four refused or not selected. Also a two-candidate case, four invalid-fixture controls, and five detected seeded faults. | Parsed inputs, bounded matching, abstract freshness/index state, and supplied field applicability; no complete HTTP engine. |

The successor also reproduces the 38 historical result projections. That is
a regression bridge, not 38 additional independent tests. Its six inherited
positive cases have explicitly added synthetic request/Vary premises. The
omitted-Vary case is an expected admission that demonstrates a limitation.

These records show agreement between custom models and their expected results.
They do not establish independently designed oracles, normative correctness,
wire interoperability, production impact, or performance. No model was rerun
in preparing this report.

### 6.2 Bounded implementation source observations

A source study dated 20 August 2026 did not identify the target HEAD-200-to-
stored-GET metadata update in the inspected paths of three exact revisions:

| Source snapshot | Reported path |
|---|---|
| [Apache Traffic Server, `1372b5ef`](https://github.com/apache/trafficserver/blob/1372b5efba2508e82252799e1242bd1b7ab121f5/src/proxy/http/HttpTransact.cc) | Ordinary 304 header updates are present; the inspected HEAD-200 path deletes the existing object instead of freshening its retained GET content. |
| [Squid, `e38e170b`](https://github.com/squid-cache/squid/blob/e38e170b44632509f2f8307408eb583ecd15a9ec/src/client_side_reply.cc) | HEAD can find a stored GET; successful HEAD-200 handling invalidates/replaces rather than performing the target merge. The study also traced `src/store.cc` and `src/http.cc`. |
| [Vinyl Cache, `ce97c406`](https://code.vinyl-cache.org/vinyl-cache/vinyl-cache/src/commit/ce97c4069feba29c92cd70630740272d8ac33d99) | The default cacheable path fetches GET upstream and suppresses content for the HEAD client; the inspected custom-HEAD test separates the variant. |

[Vinyl Cache continues the former Varnish Cache FOSS project](https://vinyl-cache.org/organization/on_vinyl_cache_and_varnish_cache.html);
the table identifies the exact inspected revision, not every Varnish-branded
release or distribution.

These are reported source-reading results, not runtime measurements or an
audit of every configuration. No deployment probe followed, and no maintainer
confirmation is claimed. Their absence result neither proves universal absence
of HEAD freshening nor establishes why implementers chose those paths. In
particular, these implementations must not be cited as positive reproductions
of the central trace. The author's published
[Integrity Cache -00, Section 13.1](https://datatracker.ietf.org/doc/html/draft-jurkovikj-http-integrity-cache-00#section-13.1)
also records the lack of a documented deployment known to the author for this
path and requests implementation reports.

### 6.3 Evidence availability and identity

The underlying research notes, models, and retained outputs are not distributed
with this report. The protocol traces above are self-contained; readers cannot
independently reproduce the reported model runs from this document alone.
The following sizes and SHA-256 values were recomputed from the two retained
result files on 28 October 2026. This was an identity check, not a model rerun:

- 25 August revision 2, 35,958 bytes:
  `84e82df2d9ab478e4892c91d14c7943b8de59416e5b6bb5b8f6bb1b54e49672e`.
- 28 September successor, 181,352 bytes:
  `360fc581f9c2889781568b5a058f22361b8bed51b3b6d8b64048a3ff16a5b1f0`.

Hashes identify the retained files; they do not independently authenticate
execution dates, runtime identity, authorship priority, or experimental
independence. Dates on private records are not public disclosure dates.

## 7. Prior work and attribution

HEAD cache maintenance is long-standing, not a newly invented mechanism.
[RFC 2068 Section 9.4](https://www.rfc-editor.org/rfc/rfc2068.html#section-9.4)
already described it. The HTTPbis
[2013 clarification](https://github.com/httpwg/http-core/commit/731f12bb3f8a0eccec3a48d57359cbce399cf1e4)
and [RFC 7234 Section 4.3.5](https://www.rfc-editor.org/rfc/rfc7234.html#section-4.3.5)
are relevant predecessors. This history weighs against presenting the issue
as an accidental new omission in RFC 9111.

The [2012 proposal for ticket 227](https://lists.w3.org/Archives/Public/ietf-http-wg/2012JanMar/0768.html)
qualified comparisons with "when present"; [issue 659](https://github.com/httpwg/http-core/issues/659)
later described matching received validators. These are relevant historical
context, not explicit decisions on the zero-validator case.

[Integrity Cache -00, Section 6.2](https://datatracker.ietf.org/doc/html/draft-jurkovikj-http-integrity-cache-00#section-6.2)
limits weak, modification-date, length, and validator-free evidence for
installing a new Integrity member. It leaves RFC 9111's HEAD update/invalidation
conditions unchanged. Its field-specific restrictions are therefore distinct
from Section 2's question about renewing freshness. Those public observations
must be distinguished from the additional HEAD-admission and request-association
analysis assembled here.

The October exchange between
[Roy Fielding](https://lists.w3.org/Archives/Public/ietf-http-wg/2026OctDec/0035.html)
and [Daniel Franke](https://lists.w3.org/Archives/Public/ietf-http-wg/2026OctDec/0040.html)
examines related metadata and Vary questions during `304` updates. Fielding's
dependent-field objection is a relevant counterargument; Franke reports
implementation tests. Their exchange does not supply a successful HEAD-200
reproduction, establish consensus on a remedy, or validate the restrictions
modeled here. This report does not adopt Franke's tests as its own evidence.

The contribution of this report is the explicit separation of the
validator-free trace, timestamp-provenance concern, field-input boundary,
request-association check, and omission limit, with evidence bounded for each.
No exhaustive novelty search or first-discovery claim is made.

## 8. Conclusion

Validator-free HEAD freshening deserves a precise explanation of whether
absence of conflict is sufficient to renew a stored response's freshness.
The illustrated path exposes the consequence of that interpretation without
assuming a malicious origin or a deployed exploit.

The candidate strong-tag restriction and effective-Vary check address
different parts of the association problem. Neither establishes a complete
solution, and their costs remain unmeasured. The available evidence supports
a focused specification discussion and bounded analytical findings, not a
claim that ordinary caches generally corrupt responses or that a normative
replacement is ready.
