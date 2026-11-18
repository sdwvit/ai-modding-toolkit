# The Zone Kit support desk forms, and the mod.io API

Verified upstream with several Xbox build requests and 2 bug reports.
Adapted from the Stalker 2 modding harness (github.com/noahsemus/stalker2-modding-harness, MIT, copyright Noah Semus); this part was contributed there by sdwvit.

The user fills in and submits these forms. You prepare the values below as text the user can paste.

## Where the form is
The Zone Kit support desk is `https://zonekit-support.stalker2.com`. The user's tickets are under Support > Tickets
(`/en-US/support/tickets`), and the new-ticket form is the create page there (`.../support/tickets/create`). Older
Zendesk form links no longer work.

## Source zip for a mod with Wwise audio
Voiced dialogs, radio or SFX: the cooked event `.uasset`s alone are not enough to build Xbox soundbanks, and one such
build came back with subtitles and no voice. Also ship the mod's `S2_WwiseProject` folder in the plugin folder, and the
source WAVs at the path the `.wproj` Originals setting resolves to from the kit root (for example
`O/Voices/<language>/<Mod>/`). Before uploading, list the zip and check that the `.wwu` files and WAVs are in it.
The Wwise project and the Originals are GSC's content (stalker2-audio): this zip goes only to GSC for the Xbox
request. Don't share its link anywhere else, and take the file down once the Xbox build is on mod.io.

## mod.io description links
The description is a rich-text editor whose auto-linker cuts a typed URL at `?` or `-` (a link ended up pointing at
the wrong page). Paste links as `<a href>` anchors in HTML, or check every link after saving.

## Xbox build request form
| Field (`*` = required) | Value |
|---|---|
| Category (top dropdown; pick it first, the mod fields appear after) | `New mod / Mod update` |
| Name*, Email* | only shown when signed out: the user's own name and contact e-mail, typed by the user (never store them in a repo or a file) |
| Subject* | `XBOX Mod Cooking Request: <Mod>` |
| Description* | for example `Requesting Xbox cooking for the mod "<Mod>" by <author>. The zip is linked in the Google Drive field. Thanks!` |
| Your Name (how to address you) | the author's name |
| URL to Google Drive with a mod* | the zip link |
| URL to mod's page on mod.io | the mod's page URL as mod.io shows it, plus `#description` (form: `https://mod.io/g/<game name>/m/<slug>#description`, game name `stalker2`) |
| Malicious files or inappropriate content check* | a checkbox: an attestation |
| Reference confirmation* | a text field (it was a checkbox), for example `Confirmed: the mod has no references to missing assets.` |
| Attachments | none |

- **The two attestations are the user's statements.** Show the user the prepared form values and let them tick,
  type and press Submit themselves.
- Signed in: Name / Email disappear, a CCs field appears above the Drive field, and Submit opens the new ticket
  (its number is in the page address): note the number in the project log. Signed out: only "Thank you for
  contacting us.", and the confirmation goes by e-mail.
- Opened directly by URL, the create page can look signed out (no avatar, Name / Email shown) while the ticket still
  lands on the account; judge sign-in from the tickets list.
- GSC replies on the ticket once the Xbox build is on mod.io.

What the user will see while filling it in:
- Picking the category reloads the field list (a spinner). If the form collapses without the mod fields, reload the
  page.
- Typing the Subject opens an "Are you looking for this?" list that pushes the Description down.
- The Description is a rich-text editor: `/` opens a command list, and a line starting with `- ` becomes a list
  that keeps the dash on the following items (type list items without dashes).
- Never resubmit a request that went through, even to fix its subject.

## Bug report form
Same site, **Category `Support request`**: Subject, Operating System & Version, Hardware Specifications (CPU / RAM
/ GPU), Description, **Issue Type `Bug`** (or `Question`), attachments up to 20 MB (larger files as a shareable
link). Submitting with an attachment takes about 30 s and opens the new ticket.
- Write it as a reproducible game or kit bug: what you did, what happened, what should happen, kit and game version,
  logs.
- Replying on a ticket: "Add message" opens the editor, paste, **Update**; with no attachment it asks "Attachment
  Missing" > **Send Anyway**.

## mod.io API
- Game id **5761** (`stalker2`). The REST API base comes with the user's mod.io API access; calls carry the user's
  OAuth token. The user runs any call that needs it; you never see the token.
- Create a mod: `POST /games/<game id>/mods` (name, summary, description HTML, logo).
- Edit: `POST /games/<game id>/mods/<mod id>` (`visible=1` makes it public).
- Add a file: `POST /games/<game id>/mods/<mod id>/files`, multipart with `filedata` (the zip), `version`,
  `changelog`, `platforms[]`, `active=true`.
- The mod's public URL: `profile_url` from `GET /games/<game id>/mods/<mod id>`.
- **Unverified**: upstream's uploads tagged the PC file `platforms[] = windows, xboxseriesx`; whether `xboxseriesx`
  is needed on the PC file is not known.
