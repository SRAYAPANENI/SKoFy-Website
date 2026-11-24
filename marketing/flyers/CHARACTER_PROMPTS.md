# Dodorez 3D cast — image prompts

A **brand-new cast** for the "How Dodorez works" flyer (`cycle.html`) — none of the earlier characters. Generate each image, save it **in this folder** with the exact file name below, and the flyer picks it up (until then: the Dodorez mark in the middle, icons on the steps).

The two **hero** characters replace the Dodorez mark in the middle of the loop; the six **step** characters go on the step cards. They're dressed to stand out on every colour option (yellow, black, cream, navy, sunset, forest, sky).

Works with ChatGPT (image generation), Midjourney, Ideogram, Leonardo or Firefly.

## Specs (every image)

- **Transparent background** (PNG). ChatGPT can do this if you ask; for Midjourney/others, remove the background afterwards (Canva, remove.bg, Photoshop).
- **Portrait, about 1024 × 1536.** Full body, head to feet, centred, nothing cut off, small empty margin all round.
- One character per image (except `step-4`, which is two), facing slightly toward the camera.
- Keep **the same chat / seed** for the whole set if your tool allows it, so the style stays identical.

## Style block — paste at the start of every prompt

> 3D cartoon character, modern stylized Pixar-like render, soft clay-like materials, smooth subsurface skin, big expressive eyes, friendly confident expression, clean studio lighting with soft rim light, vibrant colours — outfits in golden yellow (#FFCE48), coral (#FF6B8A), white, denim and teal, never purple, violet or navy (so they stand out on any of the flyer's backgrounds) — full body from head to shoes, standing, isolated on a transparent background, no text, no logo, no frame, high detail, 4k

**Avoid (negative prompt / "don't"):** purple, violet or navy clothing, realistic photo, uncanny, extra fingers, cut-off feet, cropped head, text, watermark, logo, busy background, shadows on the ground, tools of a plumber or electrician.

## The cast

The people are a mix of ages and backgrounds (US launch), with everyday, current outfits. The jobs are deliberately not plumber or electrician.

| File | Who | Prompt (after the style block) |
|---|---|---|
| `hero-customer.png` | The customer (centre, left, facing right) | A cheerful young man in his mid 20s with wavy dark hair, a white t-shirt under a golden-yellow puffer vest, light denim jeans and white high-top sneakers. He holds his smartphone out in one hand, showing the screen, and points at it with the other hand with a big excited smile, as if saying "found someone!". Body turned three-quarters to the right. |
| `hero-pro.png` | The provider (centre, right, facing left) | A confident young woman professional in her late 20s with a sleek high ponytail and small hoop earrings, a coral cropped jacket over a white top, teal cargo trousers and chunky sneakers, a camera on a strap over her shoulder and a tablet tucked under one arm. She gives a friendly wave with a warm, self-assured smile, ready to work. Body turned three-quarters to the left. |
| `step-1.png` | Ask for anything | A young woman in a yellow raincoat speaking into her phone held near her mouth like a voice message, a cartoon speech bubble with three dots floating beside her, curious happy face. |
| `step-2.png` | Matched in seconds | A young man in a teal t-shirt and denim jacket looking at his phone in surprised delight, small glowing notification bubbles popping out of the screen. |
| `step-3.png` | Pros apply | A woman in her 30s with braids, smart casual blazer, raising her hand enthusiastically and holding up an ID card with a green check mark badge (no readable text). |
| `step-4.png` | Hire & pay safely | Two people shaking hands, a woman customer and a man pro, both smiling, a golden padlock-shield icon floating above their hands. |
| `step-5.png` | Get it done | A friendly tutor/designer in a yellow sweater sitting cross-legged with a laptop on her knees, waving at the laptop screen as if on a video call, a coffee mug beside her. |
| `step-6.png` | Rate each other | An older man (60s) with grey hair and glasses, cardigan and yellow polo, happily holding a big glossy golden 3D star above his head with both hands. |

## Tips

- Generate `hero-customer` first; when you like it, say "same style, same lighting" for the rest.
- If a character comes out cropped, ask for "full body, zoomed out, more empty space around".
- Want different jobs in the pictures (baker, mover, music teacher, pet sitter…)? Swap them into the prompts; keep the style block the same.

## Animate them (optional)

Still PNGs already move on the flyer (they bob and sway). For real animation, turn each **hero** image into a short loop with an image-to-video tool (Runway, Kling, Pika, Luma, Veo) — upload the PNG, then use the prompt below.

**Export:** 4–6 seconds, MP4, the character on a **plain bright green background (#00FF00)** — green-screen style, nothing else in frame, camera not moving. Send me the MP4s; I cut out the green and make them transparent loops for the flyer (and the animated post/story videos).

| File | Animation prompt |
|---|---|
| `hero-customer.mp4` | Gentle idle loop: he glances at his phone, then turns it toward the viewer with an excited smile and a small bounce; soft breathing. Static camera, plain green background, seamless loop, same pose at start and end. |
| `hero-pro.mp4` | Gentle idle loop: she waves hello with a warm smile and a small nod, her ponytail swings slightly; soft breathing. Static camera, plain green background, seamless loop, same pose at start and end. |

Tip: if the tool adds a background, generate the still on green first ("on a plain bright green #00FF00 background" instead of "transparent background"), then animate that.
