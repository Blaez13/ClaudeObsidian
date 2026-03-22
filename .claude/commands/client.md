# /client — Client Lookup or Creation

Look up or create a client profile.

Usage: /client [client name]

Instructions:
1. Search `10-stellaris-ridge/clients/` for a folder matching the client name
2. If found: read their `brief.md` and present a summary:
   - Who they are and what they do
   - What we're doing or have done for them
   - Current project status
   - Last contact / last audit date
   - Key contacts and notes
3. If NOT found: create a new client folder and brief
   - Ask for minimum required info: business name, industry, location,
     website URL, and what they need from Stellaris Ridge
   - Create `10-stellaris-ridge/clients/[slug]/brief.md` from the template
     at `90-templates/client-brief.md`
   - Add client to `10-stellaris-ridge/_active-projects.md`

Client folder slug format: lowercase, hyphens (e.g., "joe's pizza" → "joes-pizza")
