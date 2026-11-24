# Dodorez "Ride" campaign v2 — image prompts

A **new look** for "As easy as booking a ride. Verified like it matters." —
built to stop the scroll on Instagram, not to match the yellow flyers.

**What changes from the old flyers:**

| Old | New |
|---|---|
| Yellow gradient background | **Night city**, deep midnight, with warm light |
| Characters standing still, smiling at the camera | **Mid-action**: walking in, stepping out, arriving |
| Eye-level, flat studio light | **Low-angle hero shots**, dramatic rim light, light trails |
| Yellow everywhere | Yellow kept **only as the accent**: the badge, the route, the key word |
| Soft toy-like Pixar style | **Semi-realistic stylised 3D**: grown-up and premium |

Works with ChatGPT image generation, Midjourney, Ideogram, Leonardo or Firefly.

## Palette (for the flyer, not the prompts)

| Name | Hex | Use |
|---|---|---|
| Midnight | `#0B1320` | Background |
| Night blue | `#13233A` | Panels, cards |
| Dodorez yellow | `#FFCE48` | **Accent only**: the "Verified" badge, the route line, one key word |
| Signal teal | `#1EE3C1` | Check marks, "on the way" |
| Warm white | `#FFF8E7` | Headline text |
| Hot coral | `#FF5A5F` | Very rarely, one small detail |

## Specs

- **Characters:** transparent background PNG, portrait about **1024 × 1536**, full body, head to shoes, nothing cut off, a small empty margin all round.
- **Background plates:** no transparency, **1080 × 1350** (post) and **1080 × 1920** (story).
- Keep the **same chat or seed** for the whole set so the style matches. Make `pro-arrival` first; once you like it, say "same style, same lighting" for the rest.

## Style block — paste at the start of every character prompt

> Semi-realistic stylized 3D character render, grown-up proportions, sculpted
> detail, cinematic night lighting: strong warm golden rim light from behind and
> one side, cool blue fill light, soft glow on the face, subtle film grain,
> shallow depth of field, premium advertising look, confident and real, full body
> from head to shoes, isolated on a transparent background, no text, no logo, no
> brand names, high detail, 4k

**Avoid (negative prompt / "don't"):** cute toy look, chibi, big baby eyes, flat
lighting, plain studio background, purple, text, letters, logos, brand names,
watermarks, extra fingers, cut-off feet, cropped head, uniforms with logos.

## The cast

A mix of ages and backgrounds (US launch). Clothes are current and everyday,
with workwear that looks real, not costume.

| File | Who | Prompt (after the style block) |
|---|---|---|
| `v2-pro-arrival.png` | **The key image**: the pro arriving, like a ride pulling up | A confident Latino tradesman in his early 30s, short dark hair, light stubble, navy work jacket over a grey henley, dark work trousers, worn leather boots, a heavy canvas tool bag in one hand. He walks **toward the camera** mid-stride, seen from a **low angle**, chin up, calm half-smile. Warm golden light behind him as if from car headlights, a faint glowing yellow shield-with-a-check badge clipped to his jacket pocket (no text). |
| `v2-customer-phone.png` | The customer, booking | A woman in her late 20s, Black, natural hair in a high puff, oversized cream knit sweater, wide-leg jeans, house slippers. She stands at her front door at night, **looking down at her phone**, the screen lighting her face warm yellow, relieved smile, one hand on the door frame. Three-quarter view facing right. |
| `v2-pro-developer.png` | Online work | A South Asian man in his mid 20s, round glasses, black hoodie, slim joggers, white sneakers. He walks forward with an **open laptop held in one arm**, the screen glowing teal on his face, focused, slight smile. Low angle, mid-stride. |
| `v2-pro-mechanic.png` | Mobile mechanic | A woman mechanic in her 30s, white, auburn hair tied back, rolled-up sleeves on a charcoal work shirt, dark work trousers, gloves tucked in her back pocket. She holds a **torque wrench resting on her shoulder**, the other hand wiping grease off with a rag, confident grin. Three-quarter view facing left. |
| `v2-pro-delivery.png` | Pickup & delivery | A young East Asian man in his early 20s, black bomber jacket, cargo pants, high-top sneakers, crossbody bag. He carries a **taped cardboard parcel under one arm** and walks briskly toward the camera, checking his phone in the other hand. Low angle, light trail streaks behind him. |
| `v2-pro-gas.png` | Plumbing & gas (the camera-test trade) | A Black man in his 40s, trimmed beard, high-visibility safety vest over a dark work jacket, safety glasses pushed up on his head, holding a **gas leak detector** in one hand. He kneels on one knee as if checking a pipe at floor level, looking up at the camera, serious and trustworthy. |

## Props — same style, transparent background

| File | Prompt (after the style block, minus the person) |
|---|---|
| `v2-badge.png` | A glossy 3D **shield badge** with a bold check mark, Dodorez yellow (#FFCE48) with a thin dark rim, softly glowing, floating at a slight angle, a few small light sparks around it. **No text.** Square, 1024 × 1024. |
| `v2-pin.png` | A glossy 3D **map pin** in Dodorez yellow with a small **toolbox icon** inside, a soft glowing pulse ring at its base. **No text.** Square, 1024 × 1024. |

## Background plates — no transparency

| File | Prompt |
|---|---|
| `v2-bg-street-portrait.png` (1080 × 1350) and `v2-bg-street-story.png` (1080 × 1920) | Cinematic night street in a calm American suburb, low camera angle from the middle of the road, wet asphalt reflecting warm streetlights, **long golden light trails** from passing cars curving through the scene like a route on a map, soft bokeh house lights, deep midnight blue sky (#0B1320), **the centre and top third left calm and dark for a headline**, premium car-ad look, no people, no text, no signs, no logos. |

## How they'll be used

1. **Post (1080 × 1350):** the street plate behind; `v2-pro-arrival` large on the right, walking in; a phone-style card on the left (map, the yellow route, "On the way · 6 min", the badge); the headline in warm white with **"Verified"** in yellow.
2. **Story (1080 × 1920):** the same scene, taller, with the three steps under it.
3. **A carousel series**, one trade per slide, so the feed stops looking repetitive: developer → mechanic → delivery → gas pro. Each uses the same headline and badge with a different pro arriving.

## Tips

- If a character is cut off, ask for "full body, zoomed out, more empty space around".
- If the style drifts cute, add "grown-up proportions, realistic hands, no big eyes".
- Midjourney: add `--ar 2:3 --style raw` for characters, and `--ar 4:5` / `--ar 9:16` for the plates.
- Save every image **in this folder** with the exact file name; I'll build the flyer from them.
