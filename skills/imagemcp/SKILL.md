---
name: imagemcp
description: >
  Get user info/credits, list available image models, generate images, generate transparent images, execute multicall parallel requests, synthesize SVG vectors, remove backgrounds, upscale images, compress images, edit images, and convert format via ImageMCP Server. ALWAYS check user info/credits first. Use this skill any time the user needs one or more images generated, edited, or processed — including when multiple images are needed at once (use multicall), when a cutout/logo/sticker/UI asset would benefit from a transparent background, when output quality needs to be judged and possibly regenerated, or when file size needs to be optimized via compression. Always expand thin user requests into rich, detailed, context-aware prompts before generating (see Prompt Crafting section) rather than forwarding bare text.
last-updated: 2026-08-08
allowed-tools: Bash(./scripts/imagemcp.js:*)
---

# ImageMCP Server Skill

Fetch user info/credits, list models, generate images, generate transparent images, execute multicall parallel requests, synthesize vector SVGs, remove backgrounds, upscale images, compress image payload sizes, edit images, and convert formats using [ImageMCP Server](https://api.imagemcpserver.com). Run everything through `./scripts/imagemcp.js` (Node.js 18+, zero dependencies). All commands output structured JSON.

> **Authentication Flow & Setup**:
> 1. `npx skills add web5lab/imagemcpserver` (Installs skill without prompting for tokens).
> 2. On first use, if unauthenticated, the CLI outputs: `"ImageMCPServer isn't connected yet. Run npx imagemcp login to connect your account."`
> 3. User runs `npx imagemcp login` to authenticate in the browser, select or generate an API key, and save an encrypted token to `~/.imagemcp/config.json`.
> 4. Skill automatically resolves and uses the stored token.

---

## Project Context & Architecture

**ImageMCP Server** is a unified multi-model AI image generation, vector synthesis, processing, and editing platform designed specifically for AI agents, developers, and workflows.

### Core Capabilities
1. **User Profile & Credit Inspection (`user:info`)**: Always check user profile details, plan, and credit balance first before initiating generation or edit tasks.
2. **Model Listing (`models:list`)**: Access and inspect available OpenRouter and Fal.ai image models (including Google Gemini 2.5 Flash Image, Fal Recraft Vector, Fal Feynobg, Fal Crisp Upscaler, Flux 1.1 Pro, Recraft v3, Ideogram v2, SDXL Turbo, and more).
3. **Text-to-Image Generation (`generate`)**: Synthesize high-resolution visual assets from natural language prompts with aspect ratio control (`1:1`, `16:9`, `9:16`, `4:3`, `3:4`, `21:9`) and visual style presets.
4. **Transparent Image Generation (`generate_transparent`)**: Synthesize an AI image from a prompt and automatically isolate the subject into a transparent PNG cutout using Fal.ai `fal-ai/feynobg`.
5. **Parallel Multicall Execution (`multicall`)**: Execute multiple image generation/processing requests concurrently in parallel via OpenRouter & Fal.ai.
6. **Text to Vector SVG (`text_to_svg`)**: Generate clean, resolution-independent SVG vector graphics code using `fal-ai/recraft/v4.1/text-to-vector`.
7. **Background Removal (`remove_bg`)**: Isolate subjects with transparent alpha PNG cutouts using `fal-ai/feynobg`.
8. **Image Upscaling (`upscale`)**: Super-resolution 4K detail enhancement using `fal-ai/recraft/upscale/crisp`.
9. **Image Compression (`compress`)**: Compress image byte payload size with quality control sliders (10%-95%) and format optimization.
10. **Image Editing & Refinement (`edit`)**: Modify, update, or transform existing images by supplying an input image file or URL (`--image`) alongside edit instructions.
11. **Format Conversion (`convert`)**: Convert image files across `PNG`, `JPG`, `WEBP`, `SVG`, `GIF`, and `BMP` formats.

---

## Quick Reference Guides

| Guide | Use when you need to... |
|-------|-------------------------|
| [`references/setup.md`](references/setup.md) | Configure the API key, API URL, fix authentication errors, or set up environment variables |

---

## Available Commands & Actions

Only tools provided by ImageMCP Server are available. All commands return JSON for clean parsing.

| User Intent | Command |
|-------------|---------|
| "Check user plan / credits / info" | `./scripts/imagemcp.js user:info` |
| "List available image models" | `./scripts/imagemcp.js models:list` |
| "Generate image from prompt" | `./scripts/imagemcp.js generate --prompt "..." --model "google/gemini-2.5-flash-image" --aspect-ratio "16:9"` |
| "Generate transparent image PNG cutout" | `./scripts/imagemcp.js generate_transparent --prompt "Red sports car" --out ./car.png` |
| "Run multiple requests in parallel (Multicall)" | `./scripts/imagemcp.js multicall ./batch_requests.json` |
| "Generate vector SVG graphic" | `./scripts/imagemcp.js text_to_svg --prompt "Minimalist rocket icon" --out ./icon.svg` |
| "Generate image with input image (Image-to-Image)" | `./scripts/imagemcp.js generate --prompt "..." --image ./input.png --out output.png` |
| "Edit existing image / Inpaint" | `./scripts/imagemcp.js edit --image ./input.png --prompt "..." --out edited.png` |
| "Remove background from image" | `./scripts/imagemcp.js remove_bg --image ./input.png --out clean.png` |
| "Upscale image to 4K" | `./scripts/imagemcp.js upscale --image ./input.png --scale 4x --out 4k.png` |
| "Compress image file size" | `./scripts/imagemcp.js compress --image ./input.png --quality 70 --format webp --out compressed.webp` |
| "Convert image format" | `./scripts/imagemcp.js convert --image ./input.png --format webp --out converted.webp` |

---

## Decision Framework: Think Before You Generate

Before calling any generation command, run through this checklist. This is the core "smart" logic of the skill — it decides *what* to call and *how many times*, not just how to call it.

### 1. Do I need one image or several?
Ask: is the user requesting **more than one distinct visual** in the same turn (e.g. "generate 4 icons for my app", "give me 3 variations of this logo", "make a hero image, a background, and an og-image")?

- **Yes → use `multicall`.** Never issue multiple sequential `generate` calls one at a time when the requests are independent — batch them into a single `multicall` JSON array so they run concurrently. This is faster and cheaper in turn-overhead than looping.
- **No, just one image →** use a single `generate` (or `generate_transparent` / `text_to_svg`, see below) call.
- Rule of thumb: **2+ independent image requests in the same task = multicall.**

### 2. Would a transparent background actually help here?
Don't default to transparent PNGs for everything — only reach for `generate_transparent` (or `remove_bg` on an existing image) when the image is meant to sit **on top of other UI/content**, e.g.:
- App icons, logos, stickers, badges, mascots
- Product cutouts for a page/card/thumbnail
- Overlay graphics, watermark-style assets, floating illustrations
- Anything explicitly described as needing to "sit on" a background, a UI, a card, or another image

Do **not** use transparency for:
- Full scenes, photographic backgrounds, wallpapers, hero banners meant to fill a rectangle
- Anything the user describes as a complete picture/scene rather than an isolated subject

If unsure, ask yourself: "would this look right pasted onto a white or colored UI background?" If yes → transparent. If it's a self-contained scene → normal generation.

### 3. Is the result actually good, or should I regenerate?
After every `generate`, `generate_transparent`, or `edit` call, **evaluate the output before presenting it as final** (see "Image Quality Evaluation & Automatic Retry Protocol" below). Judge:
- Did it include everything the prompt asked for (subject, count, pose, text, composition)?
- Is it visually clean — no warped anatomy, garbled text, duplicated limbs, muddy artifacts?
- Does it match the requested style (photorealistic vs vector vs anime vs 3D)?

If it falls short, **don't just present it and apologize** — automatically retry (up to 2-3 times) with a refined prompt and/or a different model per the retry ladder below, *then* show the user the best result. Only ask the user for guidance if retries are exhausted and still unsatisfactory, or if the ask is genuinely ambiguous (e.g. conflicting style requests).

### 4. Does this image need to be optimized for delivery?
After generation is finalized, consider whether the image should be compressed before handing it off:
- Going into a **web page, app bundle, email, or anywhere file size/load time matters** → run `compress` (webp, quality ~65-80 is a good default for photos; higher for graphics/text-heavy images) before delivering.
- A **one-off asset the user will download and edit further**, or something that explicitly needs max fidelity (print, upscale target, further editing) → skip compression, keep it at full quality.
- If the user asks for "optimized", "for web", "small file size", "fast loading", or is clearly building a website/app → compress by default without being asked twice.

---

## Prompt Crafting: Give the Model Rich, Detailed Context

Image model output quality is driven overwhelmingly by prompt quality. Never forward a bare, one-line user request straight into `--prompt` — always expand it into a detailed, unambiguous description first. A vague prompt in, a vague/generic image out.

### Before writing the prompt, gather context
- **Re-read the full conversation**, not just the latest message — the user may have already stated a purpose, audience, brand colors, existing assets, or style preferences earlier that should carry into the prompt.
- **If a reference image exists** (uploaded file, prior generation, or URL), use `--image` for image-to-image or `edit` rather than trying to describe it from scratch in text.
- **If critical details are missing and materially change the output** (e.g. "logo for my company" with no name/industry/colors given), ask a short clarifying question rather than guessing — but don't block on cosmetic details you can reasonably infer (lighting, composition) — infer and state the assumption instead.
- **Infer intended use** from context (app icon vs. hero banner vs. print poster vs. social post) — this should drive aspect ratio, transparency, and level of detail/realism, not just the subject description.

### What a detailed prompt should specify
Build prompts out of these layers, including every layer that's relevant to the request:

1. **Subject** — precisely what/who, including distinguishing details (breed, age, pose, expression, clothing, materials, color, count of objects/people).
2. **Action/composition** — what's happening, camera angle (close-up, wide shot, aerial, eye-level), framing, rule-of-thirds placement, foreground/background relationship.
3. **Setting/environment** — location, time of day, weather, season, background elements — or explicitly "isolated on plain/transparent background" for cutouts.
4. **Lighting** — direction, quality, and mood (soft diffused daylight, dramatic side lighting, golden hour, studio softbox, neon glow, backlit silhouette).
5. **Style/medium** — photorealistic / 3D render / flat vector / watercolor / anime / oil painting / isometric / line art, plus any art-direction reference ("in the style of minimalist Scandinavian design", "corporate flat illustration").
6. **Color palette** — specific colors, brand hex-adjacent descriptions ("deep navy and warm gold"), or explicit contrast/mood ("high contrast, muted pastel").
7. **Technical quality tags** — resolution/detail cues (8k, sharp focus, highly detailed, studio quality) appropriate to photorealistic asks; skip these for flat vector/icon styles where they don't apply.
8. **Negative guidance** — what to avoid, especially for known failure modes: "no extra limbs, no distorted hands, no garbled text, no watermark, no background clutter" — tailor this to the subject (e.g. hands/faces for people, legible text for typography asks).
9. **Text rendering** (only if text/labels are requested) — spell out the exact text in quotes, specify font feel (bold sans-serif, elegant script), and keep the string short — image models render short text far more reliably than long strings.

### Example: expanding a thin request into a detailed prompt
- Thin: `"a red car"`
- Detailed: `"A sleek red sports car parked on a rain-slicked city street at night, three-quarter front angle, neon signs reflecting off the wet asphalt and the car's glossy paint, dramatic low-angle shot, cinematic lighting, photorealistic, 8k, sharp focus, no motion blur, no distorted proportions"`

- Thin: `"app icon for a todo app"`
- Detailed: `"Minimalist flat-vector app icon of a checklist with a bold checkmark, rounded square background, two-tone blue and white color palette, clean geometric shapes, no gradients, no text, centered composition, isolated on transparent background"` (paired with `generate_transparent`)

### Apply the same rigor to `edit` prompts
Edit prompts should state precisely what changes and what stays the same: "Keep the subject, pose, and lighting exactly as-is; change only the background to a snowy mountain resort at dusk, cool blue tones, soft falling snow" — vague edit prompts ("make it winter") are far more likely to alter unintended parts of the image.

### Carry this into retries and multicall
- Every retry in the Auto-Retry Protocol below should *add* specificity (per the layers above), not just resend the same short prompt to a different model.
- In a `multicall` batch, give each entry its own fully fleshed-out prompt — don't share one generic prompt across items and expect the model to infer per-item differences; write each one out in full using the layers above.

---

## Image Quality Evaluation & Automatic Retry Protocol

AI agents using this skill MUST evaluate the quality of generated or edited images before concluding a task.

### 1. Evaluation Criteria
When inspecting generated or edited image results:
- **Prompt Fidelity**: Did the model render all key elements requested in the prompt?
- **Visual Clarity & Quality**: Are there severe visual artifacts, unintended blurriness, anatomical distortion, or missing subjects?
- **Typography & Text Rendering**: If text/labels were requested, is the rendered text readable and accurately spelled?
- **Style Consistency**: Does the output match the specified visual style preset (`photorealistic`, `vector`, `anime`, etc.)?
- **Background Fit** (transparent images only): Is the cutout clean, with no background halos, fringing, or stray pixels around the subject edges?

### 2. Automatic Re-Generation / Retry Strategy
If the generated image is **unsatisfactory**, **low quality**, or **fails to match the prompt**, the AI agent should **automatically try generating again** (up to 2-3 retries) using the following progression:

1. **Refine Prompt Detail**:
   - Expand the prompt using the full layer checklist in "Prompt Crafting" above (subject, composition, setting, lighting, style, palette, quality tags, negative guidance, text handling) — add whichever layers were thin or missing in the previous attempt.
   - Example: Instead of `"a red car"`, use `"a highly detailed sleek red sports car parked on a rainy city street at night with neon reflection, photorealistic 8k, sharp focus"`.

2. **Switch Target Model**:
   - If the default model (`google/gemini-2.5-flash-image`) produces unsatisfactory output, select a model tailored to the visual task:
     - **Photorealism & Realism**: `black-forest-labs/flux-1.1-pro` or `stabilityai/sdxl-turbo`
     - **Vector Art & Graphics**: `recraft-v3`
     - **Text & Typography in Images**: `ideogram-v2`
     - **Fast Iteration**: `google/gemini-2.5-flash-image`

3. **Adjust Parameters**:
   - Modify `--aspect-ratio` (`16:9`, `1:1`, `9:16`) to fit composition bounds better.
   - Specify `--style` explicitly (`photorealistic`, `anime`, `vector`, `3d`).

4. **Stop condition**: after 2-3 retries, present the best of the attempts even if imperfect, and briefly tell the user what's still off and offer to keep iterating — don't loop indefinitely.

---

## Batching Multiple Images: When and How to Use Multicall

Use `multicall` whenever a task naturally decomposes into **multiple independent image operations** — not just multiple generations. It supports generation, transparent generation, background removal, SVG synthesis, and editing requests in the same batch.

**Good multicall triggers:**
- "Generate icons for home, settings, and profile" → 3 `generate_image` (or `generate_transparent_image` if they're app icons) entries in one multicall.
- "Give me 4 style variations of this logo" → 4 `generate_image` entries with different prompts/styles.
- "Create a hero image and an og-image for this landing page" → 2 entries, different aspect ratios.

**Not a multicall case:**
- A single image with an iterative refinement loop (retry protocol above) — that's sequential by nature since each retry depends on judging the previous result.
- Steps that depend on each other's output (e.g. generate → then edit that same output) — these must run in order, not in parallel.

```bash
./scripts/imagemcp.js multicall --json '[
  {"tool":"generate_transparent_image","args":{"prompt":"Minimalist home icon, flat vector style"}},
  {"tool":"generate_transparent_image","args":{"prompt":"Minimalist settings gear icon, flat vector style"}},
  {"tool":"generate_transparent_image","args":{"prompt":"Minimalist profile icon, flat vector style"}}
]'
```

After a multicall batch returns, run the same quality evaluation (above) on **each** result independently — a batch can have some good results and some that need a retry; only regenerate the ones that failed, not the whole batch.

---

## Detailed Command Documentation

### 1. User Info & Model List

#### `user:info` (alias: `me:get`)
Retrieve profile details for the authenticated ImageMCP user including subscription plan and credit balance.

```bash
./scripts/imagemcp.js user:info
```

#### `models:list` (alias: `models:get`)
List all supported image generation models from OpenRouter with priority scores, providers, cost, and supported aspect ratios.

```bash
./scripts/imagemcp.js models:list
```

---

### 2. Image Generation & Transparency

#### `generate` (alias: `image:generate`)
Generate high quality images from text prompts using OpenRouter models.

```bash
# Generate image with specific model and aspect ratio
./scripts/imagemcp.js generate --prompt "Cyberpunk neon city at night" --model "google/gemini-2.5-flash-image" --aspect-ratio "16:9"

# Generate image and save to local path
./scripts/imagemcp.js generate --prompt "Futuristic sports car" --out ./car.png

# Read prompt from file and generate
./scripts/imagemcp.js generate --file ./prompt.txt --out ./output.png
```

**Supported Flags:**
- `--prompt "<text>"`: Text prompt for image generation.
- `--file <filepath>`: Read prompt text from a file.
- `--model <model_id>`: Target model ID (default: `google/gemini-2.5-flash-image`).
- `--aspect-ratio <ratio>`: Aspect ratio (`1:1`, `16:9`, `9:16`, `4:3`, `3:4`, `21:9`).
- `--style <style_name>`: Desired visual style (`photorealistic`, `anime`, `vector`, `3d`, etc.).
- `--image <filepath_or_url>`: Input image for image-to-image synthesis.
- `--out <filepath>`: Download and save the generated output image to a local file.

---

#### `generate_transparent` (alias: `transparent:generate`)
Synthesize an AI image from a text prompt and automatically process background removal using `fal-ai/feynobg` to output a true transparent alpha PNG cutout.

Use this whenever the asset is meant to be composited onto UI, a page, or another image (see Decision Framework step 2) — icons, logos, mascots, product cutouts, stickers, badges.

```bash
# Generate a transparent PNG cutout of a product/object
./scripts/imagemcp.js generate_transparent --prompt "Red vintage sports car isolated" --out ./car.png

# Generate transparent image with specific aspect ratio and model
./scripts/imagemcp.js generate_transparent --prompt "Futuristic robot mascot" --model "black-forest-labs/flux-1.1-pro" --aspect-ratio "1:1" --out ./robot.png
```

**Supported Flags:**
- `--prompt "<text>"`: **(Required)** Detailed text prompt describing the image/subject.
- `--file <filepath>`: Read prompt text from a file.
- `--model <model_id>`: Target model ID (default: `google/gemini-2.5-flash-image`).
- `--aspect-ratio <ratio>`: Aspect ratio (`1:1`, `16:9`, `9:16`, `4:3`, `3:4`, `21:9`).
- `--style <style_name>`: Desired visual style (`photorealistic`, `anime`, `vector`, `3d`, etc.).
- `--out <filepath>`: Download and save the transparent PNG image output to a local file.

---

#### `multicall` (alias: `batch:generate`)
Execute multiple image generation, transparent generation, background removal, SVG synthesis, or editing requests concurrently in parallel via OpenRouter & Fal.ai. See "Batching Multiple Images" above for when to reach for this.

```bash
# Execute batch requests from a JSON file
./scripts/imagemcp.js multicall ./batch_requests.json

# Execute batch requests inline via JSON string flag
./scripts/imagemcp.js multicall --json '[{"tool":"generate_image","args":{"prompt":"Cyberpunk street"}},{"tool":"generate_transparent_image","args":{"prompt":"Futuristic car"}}]'
```

**Sample Batch JSON File (`batch_requests.json`):**
```json
[
  {
    "tool": "generate_image",
    "args": {
      "prompt": "Neon cyberpunk city boulevard at night",
      "aspectRatio": "16:9"
    }
  },
  {
    "tool": "generate_transparent_image",
    "args": {
      "prompt": "Sleek red sports car"
    }
  },
  {
    "tool": "text_to_svg",
    "args": {
      "prompt": "Minimalist rocket logo",
      "style": "logo"
    }
  }
]
```

**Supported Flags:**
- `--file <filepath>` or positional arg: Path to a JSON file containing an array of request objects.
- `--json '<json_str>'`: Inline JSON string containing the requests array.

---

### 3. Image Editing & Refinement Endpoint

#### `edit` (alias: `image:edit`)
Edit, modify, or transform an existing generated image or input image by supplying an image path/URL and prompt instructions.

```bash
# Edit image background or style
./scripts/imagemcp.js edit --image ./photo.jpg --prompt "Change background to a snowy mountain resort" --out ./snowy_photo.png

# Transform image into watercolor sketch
./scripts/imagemcp.js edit --image ./car.png --prompt "Transform into watercolor sketch style" --style "vector" --out ./sketch.png

# Object addition or modification on generated image
./scripts/imagemcp.js edit --image ./room.png --prompt "Add a sleeping golden retriever on the rug" --out ./room_with_dog.png
```

**Supported Flags:**
- `--image <filepath_or_url>`: **(Required)** Input image file path or URL to edit.
- `--prompt "<text>"`: Description of edits, modifications, or target visual transformation.
- `--file <filepath>`: Read prompt text from a file.
- `--model <model_id>`: Target model ID (default: `google/gemini-2.5-flash-image`).
- `--aspect-ratio <ratio>`: Target output aspect ratio (`1:1`, `16:9`, `9:16`, `4:3`, `3:4`, `21:9`).
- `--style <style_name>`: Visual style preset (`photorealistic`, `anime`, `vector`, `3d`, etc.).
- `--out <filepath>`: Save edited image output to a local file path.

---

### 4. Optimization: Compression & Conversion

#### `compress` (alias: `image:compress`)
Compress image byte payload size with quality control sliders (10%-95%) and format optimization. Reach for this any time an image is heading to production/web use (see Decision Framework step 4) — don't leave full-resolution PNGs in a web/app deliverable unless the user needs max fidelity.

```bash
./scripts/imagemcp.js compress --image ./input.png --quality 70 --format webp --out compressed.webp
```

Good defaults:
- Photographic content for web: `webp`, quality `65-80`
- Graphics/UI assets/text-heavy images: keep quality higher (`80-90`) to avoid legible artifacts
- If the destination format isn't specified, `webp` is the best general-purpose choice for size/quality trade-off; use `png` if transparency must be preserved.

#### `convert` (alias: `image:convert`)
Convert image files across `PNG`, `JPG`, `WEBP`, `SVG`, `GIF`, and `BMP` formats.

```bash
./scripts/imagemcp.js convert --image ./input.png --format webp --out converted.webp
```

---

### 5. Configuration Commands

| Command | Description |
|---------|-------------|
| `./scripts/imagemcp.js setup` | Interactive prompt to save API key and URL |
| `./scripts/imagemcp.js config:show` | Display current API key status and URL |
| `./scripts/imagemcp.js config:set-key <key>` | Set API Key in configuration file |
| `./scripts/imagemcp.js config:set-url <url>` | Set API URL in configuration file |

---

## Error Handling & Best Practices

1. **API Key Setup**: Ensure `IMAGEMCP_API_KEY` is exported or configured via `./scripts/imagemcp.js setup`.
2. **API Backend**: Defaults to `https://api.imagemcpserver.com`.
3. **JSON Output**: All commands output valid JSON to stdout for easy parsing in scripts or AI agent pipelines.
4. **Auto-Retry Loop**: When image quality is unsatisfactory, perform up to 2-3 retries automatically by refining the prompt or switching models before asking for user intervention.
5. **Batch Independent Requests**: When 2+ independent images are needed in one task, use `multicall` instead of sequential single calls.
6. **Transparency Is Situational**: Only use `generate_transparent`/`remove_bg` for assets meant to composite onto UI or other content — not for full scenes or backgrounds.
7. **Optimize Before Delivery**: Compress images destined for web/app use; skip compression for assets needing max fidelity or further editing.