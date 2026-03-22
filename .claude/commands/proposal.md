# /proposal — Client Proposal Generator

Generate a client-ready proposal document.

Usage: /proposal [client name]

Instructions:
1. Read the client brief at `10-stellaris-ridge/clients/[client-name]/brief.md`
2. If an audit exists at `10-stellaris-ridge/audits/`, reference it
3. Use `90-templates/proposal.md` as the structure
4. Make it specific to THIS client — reference their actual situation, not generic copy
5. Include specific services, pricing ranges, and timeline
6. End with a clear next step / CTA

Save output to:
- `10-stellaris-ridge/proposals/[client-name]-proposal-[date].md`
- Note the file path in `10-stellaris-ridge/_active-projects.md`

Quality standard: This goes to a client. Run through Sage-level quality check before
saving (would this embarrass us? is it specific enough? does it solve THEIR problem?).
