# Dry run: make-site-multilingual + translate-site on the Hugo starter

Request: "Make this site multilingual — French and German." Default `en` at the root, Rosey + RCC, visitor locale picker, French translated now, German next week. Built with Hugo 0.166, Rosey 2.3.10, rosey-cloudcannon-connector 2.0.1. Nothing committed.

## What I did, by doc section

| Step | Doc section followed | What I did |
| ---- | -------------------- | ---------- |
| 1 | `SKILL.md` → `setup.md` Phase 1 | Audited the layouts, content and data files. No Bookshop. Output dir `public/`. One page-builder page (`content/_index.md`), 11 posts, blog paginated at 6 (so `/blog/page/2/` exists), tags taxonomy. |
| 2 | `hugo/overview.md` § Choose a setup | No `languages` block, the site owns its layouts, no per-file bodies wanted, so **setup C (Rosey only)**. |
| 3 | Phase 1 step 4, step 5 | Default language at the root. **German is held back**: step 4 says every locale with a locale file gets published, so I only passed `--locales fr`. No `de.json`, and `de` isn't in the picker. |
| 4 | Phase 2, `hugo/overview.md` C table | Ran `npx -y rosey-cloudcannon-connector init --yes --locales fr --build-dir public`. Then made the three fixes Phase 2 asks for: added `rm -rf ./_untranslated_site`, added the fix script, and moved the existing `pagefind` step to the end (Phase 4 § Existing postbuild steps). Added `locales` to `collection_groups` (Phase 5d). |
| 5 | Phase 5c → `cloudcannon-configuration` § Do this first | Downloaded the schemas to `.cloudcannon/migration/`, gitignored `*.schema.json`, and checked the keys I added (`instance_value`, the `translate` icon, the `disable_*` keys). `cloudcannon validate`: the config passes. |
| 6 | Phase 3 / `tagging.md` / overview § Rules for B and C | **baseof**: `$root` from `.Path`, `data-rosey-root` on `<main>`, a `data-rcc` boundary around the skip link, header, main and footer, the RCC import, and the Phase 9 script. **Head**: `{root}:page_title` on `<title>`; `{root}:page_description` on the meta description via `data-rosey-attrs-explicit`. Pages with no `seo.page_description` share the global `page_description` key. I also put the same keys on `og:title` and `og:description` (my own decision). **Nav/footer**: `data-rosey-ns="nav"`/`"footer"`, content-as-key links, hamburger and menu `aria-label`s with attrs-explicit, the copyright statement. I skipped the social icon labels and logo alts because they're brand names. **Blocks**: `data-rosey-ns` from `_uuid` on each partial's root (hero, hero-video, left-right, text-block, both buttons). `markdownify` became `page.RenderString (dict "display" "block")`. Alt text goes through a `rosey_alt` param on `processed-image`. **Blog**: per-item `data-rosey-root` from `.Path` in listings and Recent Posts, with the leaf key `title`. Dates keyed by ISO date under `dates`. A `tag-label` helper (with SEO and WYSIWYG overrides) and a `tag-link` partial, under a `tag-labels` root. `absURL` tag links became `relURL`. `pagination` root with new `aria-label`s on the arrow links. 404 heading and link. |
| 7 | §3g seeding | Seeded `_uuid` into `content/_index.md` (6 items, buttons included) with a line-level script I wrote, anchored on `_name`, run twice to check it's idempotent. Added `_uuid:` to all 6 structure values, to the hero block in `.cloudcannon/schemas/page.md`, and added a global hidden `_uuid` input with `instance_value: UUID`. |
| 8 | Phase 5f | Added `CLOUDCANNON_SYNC_PATHS=/rosey/` to `.cloudcannon/initial-site-settings.json`. It still has to be set in the live site's settings (see manual checks). |
| 9 | Phase 9 + overview § Locale picker | `layouts/partials/locale-picker.html` (from the overview), `params.locales` in `hugo.yaml` (en, fr), placed in the header `<nav>` outside the editable array, with CSS in `_header.css`. No `ENV_CLIENT` guard, because the header isn't a re-rendered component. |
| 10 | troubleshooting § Code samples break | Reproduced the entity bug on a clean build: `&lt;` count on `/fr/` dropped from 6 to 0, 8 to 0 and 1 to 0 on the three posts with escaped markup. **Gated the body key**: `data-rosey="body"` only when `.Content` has no `&lt;`. Those 3 post bodies stay English on `/fr/`. |
| 11 | Phase 6 / 6a | Ran the 6a assertions (below), then the full postbuild. |
| 12 | `translate-site` Part 1 | `prepare-translation.mjs` → translated 146 entries → `merge-translation.mjs`. |
| 13 | (own decision) | Added `--cleanDestinationDir` to `build:hugo`. See friction #1. |

## Build and postbuild results

- `npm run build`: clean, no warnings. 61 pages and 4 paginator pages.
- `.cloudcannon/postbuild` (sourced, as CloudCannon does): `write-locales` 146 keys; manifest and client written; `fix-rosey-pages [fr]: 63 generated pages, 35 added to sitemap.xml`; Pagefind found 2 languages (en, fr) with 11 pages each.
- A second build and postbuild changed nothing (0 added, 0 removed) and left no stale `fr/` in `_untranslated_site/`.
- `npx @cloudcannon/cli validate`: `cloudcannon.config.yml` is valid. `initial-site-settings.json` fails on keys the starter already had (`preserveOutput`, `hugoVersion`…), which predate this work.

## What I verified in `public/` and how

Using grep and node over the post-processed `public/`:

- **6a on `rosey/base.json`**: 146 keys. 0 dotted, 0 `undefined` or empty segments, 0 originals with `<svg`/`<!--`, every UUID segment exists in content. The only colon-less keys are `page_description` (deliberately shared) and `skip_to_main_content` (outside `<main>`). `_rcc/locales.json` is `{"locales":["fr"]}`. `_rcc/client.mjs` exists. No `data-rosey-ns="undefined"` in the output.
- **`/fr/index.html`**: `lang="fr"`, `content-language`, `hreflang` for en and fr, French meta description, canonical `https://…/fr/`. Hero, left-right headings and text, and image alts are French. Brand buttons and the hero heading ("Hugo Starter") are kept as-is. Nav and footer are French where it makes sense. Internal links are `/fr/…`. Picker: English → `/`, Français → `/fr/`.
- **`/fr/blog/tailwind/`** (a keyed body with code blocks): French body, Chroma code blocks intact, French title, description, canonical, tag chips (`/fr/tags/…`), dates (`6 mai 2025`) and Recent Posts.
- **`/fr/blog/icons/`, `/editable-regions/`, `/processed-images/`** (escaped markup, body unkeyed): the `&lt;` count matches `/` (6/6, 8/8, 1/1). The body is English; everything else on the page is French.
- **`/fr/blog/page/2/`**: same keys as page 1 (`blog:title`, per-post roots), French titles, dates and tags, canonical `…/fr/blog/` (the fix script localized it), "Page précédente" aria-label.
- **`/fr/tags/seo/`**: heading "SEO" (shared `tag-labels:seo` key), `<title>` "SEO | …", shared French description, canonical `/fr/tags/seo/`. It lists the same posts as `/tags/seo/` with French titles.
- **Site-wide**: no `/fr/fr/` or `/en/en/`. Every `/fr/` canonical is a `/fr/` URL. `sitemap.xml` has 35 `/fr/` entries. Alias redirects (`/fr/blog/page/1/`) point at `/fr/blog/`. No `og:url` is emitted. `/fr/404.html` and `/fr/categories/` titles are French.

## Friction

| # | Doc file + section | What it said | What actually happened / what was missing | What I did |
| - | ------------------ | ------------ | ----------------------------------------- | ---------- |
| 1 | `setup.md` Phase 6 step 6; Phase 2 "Fix the postbuild" | "Run `.cloudcannon/postbuild` after a fresh build", and add `rm -rf ./_untranslated_site` before the `mv`. | **Hugo doesn't clean its destination.** After one local postbuild, `public/` holds Rosey's output, `fr/` included. The next `npm run build` writes over it and leaves the stale `public/fr/`, the `mv` carries it into `_untranslated_site/fr/`, and Rosey then treats those pages as **SSG-built locale pages**: it "respects existing content", so changes don't reach `/fr/`, `content-language` and `hreflang` tags are duplicated, and `fix-rosey-pages` stops counting them as generated. This sent my first entity-bug check wrong: the body looked untouched because Rosey never injected it. No doc mentions it, and `npm run build` looks like a fresh build to anyone. CloudCannon builds start clean, so only local testing is hit. | Added `--cleanDestinationDir` to `build:hugo`. The docs should say this in the Hugo C table and Phase 6 (or add `rm -rf public` to the local loop). |
| 2 | `troubleshooting.md` § Code samples break; `hugo/overview.md` setup C | Options: translate those posts as files (Phase 8 / setup B), or key prose elements one by one and leave the code unkeyed. | Neither works here. The user declined per-file bodies (so not B). Hugo renders `.Content` as one rich text region, and §3c says to tag the region, not its contents, so there's no per-element route. There's no Hugo recipe for C, and no guidance on telling a user who asked to "translate everything" that some bodies will stay English. | Gated the key with `{{ if not (strings.Contains .Content "&lt;") }}`. 3 of 11 post bodies stay English on `/fr/`. A user needs telling. |
| 3 | `hugo/overview.md` § Markdown fields and `data-type` | Don't pair `markdownify` with `block`; use `.Page.RenderString (dict "display" "block") .text`. | In page-builder partials `.` is the block dict, so `.Page` doesn't exist. You need the `page` function. This **contradicts** `cloudcannon-visual-editing/hugo/visual-editing-reference.md`, which prescribes `markdownify` for `block` regions. It also changes the English HTML: single-paragraph subheadings gain a `<p>`. Whether `page.RenderString` works in the editor's WASM renderer is untested (`page` itself is supported there). | Used `{{ .x \| page.RenderString (dict "display" "block") }}` in hero, hero-video, left-right and text-block. Listed as a manual check. |
| 4 | `setup.md` Phase 1 step 4; `translate-site/SKILL.md` § When not to use | Hold back an untranslated locale by leaving it out of `--locales`. Adding a locale to the build "is make-site-multilingual". | make-site-multilingual has **no "add a locale later" section**. For German next week a user must change five places: `write-locales --locales fr,de` (once, or in the postbuild), the fix script's `--locales`, `data_config.locales_de`, `params.locales` for the picker, and a translate-site run. `init` also hard-codes `--locales fr` into the postbuild's `write-locales`, although Phase 4 says the flag is only needed on the first run. | Left a comment in the postbuild and `hugo.yaml`. Listed the five steps below. |
| 5 | `hugo/overview.md` § Listings show other pages' keys | "Use the same leaf key (`title`) the post page uses for its own heading." | Assumes the post heading and the listing title are the same field. Here the post `<h1>` renders `post_hero.heading`, an inline-editable field, while listings render `.Title`. They're equal in all 11 posts today, but an edit to one in the Visual Editor gives one key two different originals. | Used `title` for both as instructed. Flagged as a risk. |
| 6 | `hugo/overview.md` § Taxonomies; `tagging.md` §3i | One helper for "every chip, heading, and listing". The overview only shows chips. | Hugo's own term `.Title` ("Seo", "Editable Regions") drives the term page `<h1>`, the `/tags/` listing and the `<title>`. Without extra work each term gets a second key (`tags/seo:title`) next to `tag-labels:seo`, and the "Seo" misspelling becomes a translation source. | Sent the term-page `<h1>` and `/tags/` items through the helper (23 duplicate keys removed). Head titles stay per term, so the English tab still reads "Seo \| …". |
| 7 | `tagging.md` § Seeding `_uuid` | Rules for a line-level, idempotent pass, plus "cover structure defaults and data files". | No script is provided, so every user writes one. Schema files (`.cloudcannon/schemas/*.md`) also contain `content_blocks` but aren't on the coverage list. | Wrote a 30-line Node seeder anchored on `_name`. Added `_uuid:` to the schema hero block. |
| 8 | `hugo/overview.md` § Visitor-Facing Locale Picker | Build links from `.RelPermalink`; the script highlights the active link by pathname. | On `/blog/page/2/` `.RelPermalink` is `/blog/`, so switching language drops you on page 1 and no link is highlighted. The picker's `aria-label="Language"` is also the script's selector, so it can't be translated and stays English on `/fr/`. | Kept as documented. Minor. |
| 9 | `translate-site/locale-files.md` § Whitespace | "`write-locales` preserves whitespace (`" Blog "`); preserve it." | `base.json` keeps the template whitespace, but `write-locales` **trims** `original`/`_base_original`/`value`, and the task file shows trimmed text. The advice is moot. | None needed. |
| 10 | `setup.md` Phase 4 § What `rosey build` rewrites | "`--base-url` makes hreflang hrefs absolute; it changes nothing else." | Neutral. Search engines expect fully qualified `hreflang` URLs, and the docs don't say whether to use the flag. | Left it off (the `baseURL` is a preview domain). A decision a user needs to make. |
| 11 | `rosey-cloudcannon-connector init` output | Next steps point at `docs/ssg-setup.md`. | That path isn't in the project (it's the package's repo). Harmless. | None. |

### How the `translate-site` scripts behaved

- `prepare-translation.mjs --locale fr`: classified 146 entries as untranslated, with no translation-memory hits and no tone examples (expected on a first run). It wrote `rosey/locales/.translation-task-fr.json`. That file sits inside the CloudCannon `locales` collection path, but it's a dotfile and is deleted on merge. Originals in the task file are trimmed.
- `merge-translation.mjs --locale fr`: merged 146 entries with **no HTML warnings** (that includes the Tailwind body, whose Chroma code block I copied by hand). It recorded 28 brand or identical entries in `rosey/translate-site-keep.json` and deleted the task file. The rebuild picked everything up, `write-locales` reported 0 added or removed, and no entry shows as stale.
- Smooth overall. The only manual part was the translating itself.

### Adding German next week (from friction #4)

1. `npx rosey-cloudcannon-connector write-locales --source rosey --dest public --locales fr,de` once after a build, or change `--locales fr` to `fr,de` in `.cloudcannon/postbuild`.
2. Change the fix script line in the postbuild to `--locales fr,de`.
3. Add `locales_de: { path: rosey/locales/de.json }` to `data_config`.
4. Add `{ code: de, label: Deutsch }` to `params.locales` in `hugo.yaml`.
5. Translate with `translate-site` (`--locale de`), then rebuild.

## Manual checks for CloudCannon (after pushing)

1. Set `CLOUDCANNON_SYNC_PATHS=/rosey/` in the site's environment variables. `initial-site-settings.json` only applies to a new site. After a build, confirm `rosey/base.json` and `rosey/locales/fr.json` sync back.
2. Open the home page in the Visual Editor. Confirm the RCC locale switcher appears and the visitor picker is hidden. Switch to FR and check the hero, left-right blocks, buttons, nav and footer show French inside the `data-rcc` boundary.
3. In FR, edit a hero heading, a markdown block (subheading or left-right text) and a button label. Save, **reload, and confirm each edit survived**. Check the markdown block isn't permanently stale (amber badge).
4. Confirm the hero, left-right and text-block markdown regions still render and edit in the **default** language. They now use `page.RenderString` instead of `markdownify`, and the editor's WASM renderer is untested with it. Check single-paragraph subheadings still look right with their new `<p>`.
5. Add a new content block and a new button in the editor. Confirm each gets a `_uuid` and a distinct `data-rosey-ns`. Then reorder blocks and confirm the French text follows its block.
6. Create a new page from the default schema. Confirm the hero block gets a `_uuid` (the schema has an empty `_uuid:`).
7. Open a blog post in the Visual Editor in FR. Check the body region (keyed on 8 posts) edits as one rich region, and that the 3 code-sample posts show English bodies (expected).
8. Open the Locales collection (under "Translations"). Check head keys (`*:page_title`, `*:page_description`, `page_description`) and alt text can be edited there, since they aren't on the page.
9. On the live `/fr/` pages, check the picker (styling at desktop and mobile widths), the French Pagefind search UI on `/fr/blog/`, and what the host serves for a missing `/fr/…` URL (Rosey wrote `/fr/404.html`, but the host may only serve `/404.html`).
10. Commit `rosey/base.json` (the Phase 6b baseline), `rosey/locales/fr.json` and `rosey/translate-site-keep.json`.
