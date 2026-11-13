---
created: 2026-11-13
council-type: standalone
---

# Council Report

Scope: review of the three site prototypes in `prototypes/` and direction for the next iteration. The user's brief: Stage 2 (ESP32) as the base; finished product up front, then scroll through stages, form factors and parts; a PUSH / OPEN channel toggle on the unit; light and dark mode; vintage visual options that would appeal to a grandad in 2026.

## Animal Perspectives

### Bear — Vision & Direction
- The product is the Stage 2 bedside unit; all three prototypes lead with the R1, which is a test bench. Lead with the radio.
- A stage toggle hides half the story. A scroll tells it in order, with the R1 as the "how we proved it" chapter.
- Open channel shifts the product from walkie towards room link. Present it as a deliberate second mode on a physical switch.
- Vintage styling is strategic: the R1 doc already calls for a familiar radio metaphor and adult styling.

### Cat — Risk & Scope
- Open channel contradicts the published rule "mic live only while the button is held". Needs: switch on the unit only, steady red light, spoken open/closed, visible to the other end, Grandad's consent, and rewritten copy.
- The audio design is half duplex with no echo cancelling (820 KB and a third of a core). v1 open mode should be voice-activated, one at a time, mic muted while the speaker plays. Relay floor rules (1.5 s silence, 60 s max) need an open-mode rule.
- Open mode keeps the radio awake (about 180 mA sending, 21 MB per hour): meant for the dock.
- "Finished product" hero must be labelled as a concept image.
- Vintage styles risk low contrast, thin type and hover-only controls. Keep contrast high and body text large.

### Owl — Process & Structure
- Three forks, no index.html, a one-line README, no GitHub Pages. Build one index.html with style options on it.
- Stage progress is stored three different ways. Use one PLAN object.
- The ESP32 doc links to the R1 doc via a private artifact URL; the site should link to repo files only.
- Proposed scroll: radio, try it, two ways to talk, shapes, inside, how we get there, numbers, Grandad's card, questions.
- Keep the shared sound code in one block.

### Puppy — Opportunities & Momentum
- Prototype B's simulator is the strongest idea; put it on the finished unit, straight under the hero.
- A chrome bat-handle toggle for PUSH / OPEN is a natural vintage detail and a one-pin hardware change.
- A glass tuning-dial band with a sliding needle for channels.
- A printable "how to use your radio" card that doubles as the not-an-alarm card.
- Unused images (handheld, pebble back) deserve a "pick a shape" section.

### Rabbit — Resources & Stakeholders
- Stakeholders: Grandad (user, consent), the builder and carer, family on the channels, site readers. Grandad may read it himself, so add a text size control.
- Assets: five concept images, two research docs, image prompts. Missing: a switch image, an R1 photo, updated prompts.
- BOM: toggle switch about NZ$2 to 5; phone-style two-way may need a codec board with an echo reference.
- Dependencies: Google Fonts, speech synthesis (voice and iOS gesture limits), Web Audio. Need fallbacks.
- Voice-activated sending on ESP32 is unresearched; describe open mode as a proposal.

### Wolf — Cohesion & Balance
- Numbers and wording drift between prototypes. One page, one set of numbers, one voice.
- Keep an honest "Stage 1, now" chapter for the R1.
- The page is about Grandad, not for him. Add the printable card written to him.
- Dark mode is inconsistent and has no toggle anywhere.
- Privacy wording must match across the site and both docs.

## Consensus

**Common themes:** lead with the Stage 2 radio and tell the story as a scroll; open channel is wanted but must be fenced; vintage should mean familiar, not hard to read; one page, one data source, one set of numbers.

**Key tensions:** charm against legibility (decorate headings and frames, keep body plain and large); convenience against privacy (hands-free one-at-a-time with physical switch and red light); product-first against honesty (show it, label it a concept); future against present (R1 keeps a smaller honest chapter).

**Overall:** strong research and a winning simulator, but the site is three forks with no home and framed around the test bench. One product-first page with the mode switch designed in and switchable vintage styles in light and dark addresses nearly every point.

## Recommended Next Steps

1. Build `index.html` as one product-first scroll starting with the Stage 2 radio; remove the stage toggle.
2. Add a PUSH / OPEN switch to the unit and simulator, with a "thinking behind the switch" section (v1: hands-free one at a time, unit-only, steady red light, spoken, pauses after 10 quiet minutes, two-person channel only).
3. Offer three vintage styles on the same page, each with light and dark, a visible toggle and a text size control, at high contrast.
4. Keep stage progress in one PLAN object; add the switch to the parts list.
5. Later: update both docs and the image prompts so privacy wording and the switch match, then enable GitHub Pages.
