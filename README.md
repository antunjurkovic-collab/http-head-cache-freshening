# HEAD Freshening and Stored-Response Association

**Update, 12 November 2026:** [Qualification and implementation update](addendum-2026-11-12.md).
A local Bouine test reproduced validator-free HEAD freshening; it does not
establish an RFC defect. The v1.0 report and release remain unchanged.

Antun Jurkovikj | Report date: 28 October 2026 | Version 1.0

[Read the report](report.md).

This informational research report examines an HTTP cache operation: using a
successful upstream HEAD response to update a stored GET response while
retaining its content. It is not an Internet-Draft, an IETF consensus document,
or a proposed replacement cache algorithm.

## Main questions

- Can a HEAD response without validators renew freshness for old content under
  the non-conflict reading of RFC 9111 Section 4.3.5?
- What does a matching strong entity tag establish, and what remains dependent
  on the meaning of each field?
- How do changed or omitted Vary fields affect the association with the stored
  request?

The report presents the competing interpretation of the validator condition
and explains why its candidate restrictions are not complete repairs.

## Evidence and limits

The report separates specification reasoning, illustrative protocol traces,
retained synthetic model results, and observations from three source snapshots.
It establishes no positive deployed-cache reproduction of the central trace,
general performance benefit, or first-discovery claim.

The traces and public-source citations are included. The underlying models,
research notes, and retained result files are not included, so this is not a
reproducible test-suite release. Reported file hashes identify retained
artifacts; they do not authenticate execution dates or independent execution.

The report contains its evidence limits and prior-work attribution in
Sections 6 and 7.

## License

Copyright (c) 2026 Antun Jurkovikj.

The original text of this README, the report, and the addendum is licensed under
[Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).
See [LICENSE](LICENSE) for the full terms.

Quoted third-party material is not relicensed by this notice and retains its
applicable terms and attribution. Referenced external works and undistributed
research artifacts are not covered by this license grant.
