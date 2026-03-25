> **Question:** Can you review once more the possible names for the project? We are currently proposing idea-edu-africa. These should be available on Github and should allow me to publish the static web page on it. You should also check if a domain name for that name is still available so we can publish the site under that name in the future.

# Organisation name review

## Availability results

| Name | GitHub | GitHub Pages URL | Domain (.org) | Notes |
|------|--------|-----------------|---------------|-------|
| `idea-edu-africa` | ✅ Available | idea-edu-africa.github.io | ✅ Likely available | Current proposal |
| `idea-africa` | ❌ Taken | — | ❌ Taken | Real Kenyan AI/education org, 53 repos, website idea-africa.org |
| `ideaafrica` | ❌ Taken | — | ❌ Registered since 2016 | — |
| `idea-edu` | ❌ Taken | — | Not checked | — |
| `idea-digital-africa` | ✅ Available | idea-digital-africa.github.io | ✅ Likely available | — |
| `open-idea-africa` | ✅ Available | open-idea-africa.github.io | Not checked | — |
| `idea-schools` | ✅ Available | idea-schools.github.io | ✅ Likely available | — |
| `idea-for-africa` | ✅ Available | idea-for-africa.github.io | Not checked | — |

Domain checks used who.is WHOIS. "Likely available" = no WHOIS data returned, which
for .org reliably indicates an unregistered domain.

---

## Important conflict: idea-africa already exists

The GitHub org `idea-africa` is a real Kenyan organisation in the AI and education
space. They have 53 repositories and a live website at `idea-africa.org`. Even though
their name does not block ours directly, any name we choose that is close to
"idea-africa" risks confusion with them — especially for donors, partners, and field
workers searching online.

This is worth taking seriously. Our name should be clearly distinct from theirs.

---

## The candidates worth considering

### `idea-edu-africa` (current proposal)
**GitHub:** ✅ available — **Domain:** ✅ idea-edu-africa.org likely available

The most descriptive option. All three elements are explicit: Initiative (IDEA),
Education (edu), Africa. Clear to any external reader what the organisation does.
Downside: hyphenated three-word names are slightly awkward to say aloud and to type on
a phone. Not a fatal flaw for an internal org name.

### `idea-schools`
**GitHub:** ✅ available — **Domain:** ✅ idea-schools.org likely available

Shorter and more human. "Schools" is concrete and accessible — meaningful to donors,
field partners, and teachers without needing to decode acronyms. Loses the explicit
Africa reference in the name, but that can live in the website content and description.
Better as a public-facing brand; slightly less precise as a technical identifier.

### `idea-digital-africa`
**GitHub:** ✅ available — **Domain:** ✅ idea-digital-africa.org likely available

Replaces "edu" with "digital" — arguably more accurate (the project is about digital
infrastructure, not just education content). But longer, and "digital Africa" is a
crowded phrase used by many initiatives.

---

## Recommendation

**Keep `idea-edu-africa` for the GitHub org**, but register `idea-schools.org` as the
public website domain.

The reasoning:
- The GitHub org name is a technical identifier used by developers and in repo URLs.
  Precision matters more than catchiness there. `idea-edu-africa` is clear and
  unambiguous.
- The website domain is a public-facing brand asset. `idea-schools.org` is shorter,
  warmer, and more memorable for donors, supporters, and field partners.
- Both names are available. Owning both gives flexibility — the website can live at
  `idea-schools.org` while the GitHub org stays `idea-edu-africa`.
- `idea-schools.github.io` could serve as a staging URL before the custom domain is set up.

If you prefer a single name for everything: `idea-edu-africa` is the safest choice
(fully available, clearly descriptive, no naming conflicts). `idea-schools` would be the
better brand name but loses the Africa context.

---

## Action
- Confirm org name before creating the GitHub organisation. GitHub does allow org
  renaming, but redirects from the old name only hold as long as nobody else claims it —
  and all local git remotes, CI configs, and hardcoded references need updating. Renaming
  is possible but disruptive; better to settle it before embedding the name in many places.
- Register the chosen domain name — .org registration costs roughly €10–15/year
- If going with the split approach: register both `idea-edu-africa` (GitHub) and
  `idea-schools.org` (domain)
