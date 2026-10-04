**English** | [中文](README.zh-CN.md)

# dsh-ui-personalization

A personalization plugin for the [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (dsh) client: it replaces the account row at the bottom of the sidebar with your own avatar, nickname and account balance, and can additionally hide the brand mark at the top of the sidebar. dsh does not provide any of this itself; the plugin mounts on slots dsh reserves, as a client-side cordis bundle.

Built against dsh `0.2.0-rc.2` (same client build version); not tested on other versions.

## Install

```shell
dsh plugin --profile <your profile> add github:Mgeeeeee/dsh-ui-personalization
```

Refresh the page after installing; if it does not take effect, restart the profile once. `dsh plugin` passes straight through to pnpm, so the target can also be an absolute path to a local directory:

```shell
dsh plugin --profile <profile> add /absolute/path/dsh-ui-personalization
```

This plugin is distributed from GitHub only; there is no npm release. The "Add plugin" dialog in the settings panel works too — paste the full address above. Entering just `dsh-ui-personalization` fails with "not in the npm registry" — that failure is deliberate: the old name `dsh-personalization` on npm belongs to a plugin by another author, and installing by name would install the wrong thing.

An agent inside dsh can install it directly as well: have it run `plugin_manager install_bundle <absolute path to the package directory>`.

## Usage

1. The avatar row at the bottom of the sidebar shows your avatar and nickname, with that account's balance at the right end; clicking it opens the settings panel.
2. **Settings → Personalization**: a page in the settings panel's left-hand navigation, listed after "Agent presets". The page is split into two cards by function, each starting with its own switch:
   - **Identity row**: Custom identity (switch) → Avatar (click the avatar to pick an image; on hover or keyboard focus a pencil icon appears on the avatar and a cross in its top-right corner; click the cross to restore the default) → Nickname
   - **Interface elements**: Hide top brand mark (switch)

The avatar is scaled locally to a 128px square and stored in localStorage; the nickname applies immediately. When the sidebar is collapsed into an icon rail, the nickname and balance are not shown (that row keeps only the avatar).

## Slots it takes over

| Where | Slot | Notes |
|---|---|---|
| Bottom of the sidebar | `settings.launcher` (single) | Overrides the built-in account row with `priority: -1`; turning "Custom identity" off unregisters it and the original returns unchanged |
| Top-of-sidebar brand | `sidebar.brand.mark`, `sidebar.brand.name` (single) | While "Hide top brand mark" is on they mount on top and render nothing; turning it off hands them back |
| Settings panel navigation | `settings.section` (list, id `ui-personalization`, order `25`) | One navigation entry per registration; `label` is a thunk that follows the language |

All three mount through `ctx.slots.inject`: they appear when the slot declaration appears and are torn down automatically when it disappears. Registrations controlled by a switch are mounted or unregistered as the state changes.

**The height of the brand row** is hard-coded by the shell (60px, and not through a token), so blanking the two slots above it leaves a gap. The plugin injects one extra rule for this, attached to the document only while the switch is on:

```css
[class*="_logoRow"]:not(:has(button)){height:0;min-height:0;margin:0;padding:0;overflow:hidden}
```

This is the only place where the plugin reaches outside its own components. It matches on a fragment of the shell's CSS-modules local name and refuses rows containing a button — on Windows the collapse button sits in that row, and the guard leaves it alone there. If the shell renames the local class, the rule stops applying on its own, and the result falls back to "content blanked but an empty row left behind".

**The settings panel cannot be opened at a given page**: the shell exposes no "open section X" interface, so clicking the avatar can only open the settings panel, and which page it lands on is decided by the shell's own selection state.

## Balance

The number at the right end of that sidebar row comes from the account Remote, along the same path as the official account page:

```
ctx.remote.account.getBalance({ version, locale, timezoneOffsetSeconds })
```

- **What it counts**: top-up balance + grant balance, summed when the currency is the same; it follows the official account page's rules for two decimals and for amounts under one cent (`<¥0.01`).
- **Refresh**: every frame of the account state stream (sign-in, sign-out and state changes each push one), on plugin activation, and when the settings panel is opened; plus a fallback every 3 minutes while the page is visible. Credentials do not become available until the first frame of the state stream, so the read follows the stream.
- **Nothing shown when it cannot be read**: signed out, offline, request refused, or a deployment without the account Remote — that cell in the sidebar simply does not render, rather than showing a wrong number.

## Data

All state lives in `localStorage['dsh.personalization.v1']` (the key keeps the old name from the `dsh-personalization` era before the rename, so changing the package name does not lose saved avatars and nicknames):

```json
{ "enabled": true, "nickname": "示例", "avatar": "data:image/png;base64,...", "hideBrand": false }
```

`enabled` controls the custom identity, `hideBrand` controls the top brand mark; the two are independent. Purely client-side state: it does not go into the session log, does not go to the cloud, and the balance is never written to disk; changing browsers or clearing the cache loses it.

## Known limitations

1. **There is no entry point for "Sign out".** Besides the avatar and the phone number, the built-in account row also carries a dropdown menu (Settings / Feedback / Sign out). `settings.launcher` is a single-occupant slot, so taking it over means taking over the whole menu, and the sign-out implementation lives inside the system plugin. When you need it, turn "Custom identity" off on this page and the original account row and menu come back immediately. Official settings themselves are unaffected; the "Settings" row at the bottom of the sidebar still works as before.
2. **The client version is hard-coded.** The account API requires a client build version (the `x-client-version` header); the shell inlines it into each bundle at build time, and there is no readable global at runtime, so the `CLIENT_VERSION` constant at the top of `client.js` must be updated by hand along with the dsh version. If the balance stops showing after a dsh upgrade, change that line first.
3. **The brand row is collapsed by a rule matching a class-name fragment**, see above. It goes quietly stale when the shell changes structure (leaving blank space, with no error).
4. **The navigation icon is the same as "General settings".** The shell maps icons by page id in a fixed table, unknown ids fall back to the default gear, and the registration itself has no icon field.
5. **The two affordances on the avatar appear only on hover or keyboard focus** (the pencil, and the cross in the top-right corner). Touch screens have no hover and therefore no such hint — clicking the avatar still picks an image, but the cross has to be guessed by position. That is the price paid for keeping the interface clean.

## Uninstall

```shell
dsh plugin --profile <your profile> remove dsh-ui-personalization
```

An agent inside dsh can run `plugin_manager remove_bundle dsh-ui-personalization` instead.

After uninstalling, the sidebar returns to the system account row and the settings navigation entry disappears; the record in localStorage is not cleared — delete that key if you want it gone.

One trap: with a `link:` install, uninstalling does not delete the symlink under the profile's `node_modules/`; it is left dangling, so delete it by hand afterwards.

## Development

| File | Purpose |
|---|---|
| `client.js` | The browser half, all of the logic |
| `index.js` | The host half, currently empty |
| `cordis.patch.yml` | Inserts this plugin's line into cordis |
| `locale/en.json`, `locale/zh.json` | Title and description for the plugin card and detail page |
| `icon.svg` | Plugin card icon |
| `THIRD-PARTY.md` | List of styles and markup copied from components that ship with dsh |

The browser half registers through `window.__ModuleLoader__.load({ id, factory })`, where `id` must equal the package name; `react` is taken from the page's module table and need not be bundled. Style class names always use the `dshprf_` prefix, and colors and spacing use dsh theme tokens (`--dsw-*`). Only two kinds of color are hard-coded: the letter background of the avatar (a pattern color derived from the nickname), and black-background-white-content on overlays — the scrim `#00000073`, the cross badge `#000000b3`, and their white icons. The latter deliberately do not follow the theme: the scrim sits on top of a photo and must be dark with light content under both themes.

With a `link:` install, refreshing the page after a change is enough; if you installed the GitHub copy, changes made in the workshop do not get in, and upgrading means uninstalling and installing again. Restart dsh if the new code does not take effect.

The plugin's own version number is `PLUGIN_VERSION` at the top of `client.js`; the settings footer displays it, and both sides must match `version` in `package.json` — the structure check blocks a release when they disagree. Do not confuse it with `CLIENT_VERSION` in the same file: that one is the shell's build version, used only for the balance request header.

## Acknowledgements

This plugin manages to look like part of dsh because [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) is open source: the slot mechanism, cordis bundles and plugin_manager all come from it; the styles, markup and balance display rules on screen all follow its own components (item by item in [THIRD-PARTY.md](THIRD-PARTY.md)). Thanks to DeepSeek for opening the project up.

## License

[MIT](LICENSE). Some styles and markup come from other sources, listed in [THIRD-PARTY.md](THIRD-PARTY.md).
