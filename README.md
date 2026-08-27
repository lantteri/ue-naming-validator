# Naming Validator

Enforce asset naming conventions in Unreal Engine.
Map asset types to prefix/suffix rules in Project Settings, get violations reported when assets are saved, and fix them with a one-click rename.
Supports Unreal Engine 5.6, 5.7 and 5.8. Editor-only. Nothing is added to your packaged game.

---

## Table of Contents

- [Installation](#installation)
- [Enabling the Plugin](#enabling-the-plugin)
- [Quick Start](#quick-start)
- [Settings Reference](#settings-reference)
- [Working with Rules](#working-with-rules)
- [Worked Examples: Which Rule Wins](#worked-examples-which-rule-wins)
- [How Violations Are Reported](#how-violations-are-reported)
- [The One-Click Rename Fix](#the-one-click-rename-fix)
- [Controlling What Gets Checked](#controlling-what-gets-checked)
- [Advanced: Regex Overrides](#advanced-regex-overrides)
- [Sharing Rules With Your Team](#sharing-rules-with-your-team)
- [Default Rules](#default-rules)
- [Troubleshooting](#troubleshooting)
- [Support](#support)

---

## Installation

### From Fab

1. Open the **Epic Games Launcher** and go to **Unreal Engine → Library**.
2. Find **Naming Validator** in your **Fab Library** and click **Install to Engine**.
3. Choose the engine version you want to install it to.

The plugin is installed to your engine and is available to every project using that engine version.

### Manual installation (per project)

If you prefer to install into a single project instead:

1. Close the Unreal Editor.
2. Create a `Plugins` folder in your project root if it does not already exist.
3. Copy the `NamingValidator` folder into it, so you end up with:

```
MyProject/
└── Plugins/
    └── NamingValidator/
        └── NamingValidator.uplugin
```

4. Reopen the project. If your project is a C++ project you may be prompted to rebuild. Accept.

---

## Enabling the Plugin

1. In the Unreal Editor, open **Edit → Plugins**.
2. Search for **Naming Validator**.
3. Tick the checkbox next to it.
4. Click **Restart Now** when prompted.

![The Plugins browser with Naming Validator enabled](Images/enable-plugin.png)

> **Note:** Naming Validator depends on Epic's built-in **Data Validation** plugin. It is enabled automatically alongside Naming Validator — you do not need to enable it yourself.

After the restart, the plugin is active and already enforcing a sensible default convention. You can start using it immediately, or customise the rules first.

---

## Quick Start

1. Open **Edit → Project Settings**.
2. In the left panel, scroll to the **Plugins** section and select **Naming Validator**.
3. Leave everything at its defaults for now.
4. Go to the Content Browser, rename any Texture asset to something that does *not* start with `T_`, and save it.
5. The **Message Log** opens with a validation error and a **Rename to '…'** link. Click it.

![Project Settings showing the Naming Validator page](Images/project-settings.png)
![Project Settings showing the Naming Validator page](Images/project-settings2.png)

That is the whole loop: configure rules once, then the editor tells you when something drifts and offers to fix it.

---

## Settings Reference

All settings live under **Project Settings → Plugins → Naming Validator**.

### Validation

| Setting | Description |
| --- | --- |
| **Enable Validation** | Master switch. Turn this off to disable all naming checks without uninstalling the plugin. |
| **Violation Severity** | Whether a naming violation is reported as an **Error** or a **Warning**. See [How Violations Are Reported](#how-violations-are-reported) for what this changes. |
| **Ignored Paths** | Content folders to skip entirely. Subfolders of an ignored path are skipped too. |
| **Validate Plugin Content** *(advanced)* | Off by default. When off, only assets under `/Game/` are checked. Turn it on to also check content that lives inside plugins. |

> Advanced settings are hidden behind the small arrow at the bottom of the category. Click it to reveal them.

### Naming Rules

| Setting | Description |
| --- | --- |
| **Rules** | The list of naming rules. See [Working with Rules](#working-with-rules). |
| **Reset Rules to Defaults** | A button that discards your rule list and restores the shipped defaults. This cannot be undone, so export or note your custom rules first. |

---

## Working with Rules

A rule says: *assets of this type must be named like this.*

Each entry in the **Rules** array has the following fields:

| Field | Description |
| --- | --- |
| **Summary** | Read-only. A compact recap such as `Texture2D - T_*` shown as the collapsed row title, so you can scan the list without expanding every entry. It updates automatically. |
| **Asset Class** | The asset type the rule applies to — `Texture2D`, `StaticMesh`, `Blueprint`, and so on. |
| **Prefix** | Text the asset name must start with, for example `T_`. Leave empty for no prefix requirement. |
| **Suffix** | Text the asset name must end with, for example `_Inst`. Leave empty for no suffix requirement. |
| **Enabled** | Untick to switch a single rule off without deleting it. Disabled rules show `(disabled)` in their summary. |
| **Regex Override** *(advanced)* | See [Advanced: Regex Overrides](#advanced-regex-overrides). |

![An expanded rule showing Asset Class, Prefix and Suffix](Images/rule-fields.png)

### Adding a rule

1. Click the **+** button on the **Rules** array.
2. Expand the new entry.
3. Set **Asset Class** to the type you want to govern.
4. Fill in **Prefix**, **Suffix**, or both.

### Prefix and suffix matching is case-sensitive

`T_Rock` satisfies a `T_` prefix. `t_Rock` does not. This is deliberate — inconsistent capitalisation is usually the exact thing you are trying to stamp out.

### How a rule is chosen for an asset

When an asset could match several rules, the **most specific rule wins**. Specificity is measured by how close the rule's class is to the asset's own class in the inheritance chain.
For example, with both a `Blueprint → BP_` rule and a `UserWidget → WBP_` rule active, a Widget Blueprint is governed by the `WBP_` rule, because `UserWidget` is the closer ancestor.
Blueprints are matched against the class they generate, so a Blueprint deriving from `GameModeBase` picks up your `GameModeBase` rule rather than the generic `Blueprint` rule. Specialised Blueprint asset types such as Editor Utility Blueprints and Editor Utility Widgets are matched on the asset type itself, so they can have their own rules.

If no rule matches an asset's type at all, the asset is not checked.

---

## Worked Examples: Which Rule Wins

Blueprints are the case where rule selection is least obvious, because a Blueprint has *two* class identities: the asset itself (always some kind of `Blueprint`) and the class it generates. Naming Validator looks at both, and the three examples below cover every path it can take.
The rule is always the same underneath: **the closest ancestor wins.** These examples just show where "closest" is measured from.

### Example 1 — A gameplay Blueprint gets `BP_`

You create a Blueprint deriving from `Actor` and name it `Door`.

| | |
| --- | --- |
| Asset class | `Blueprint` |
| Generated class | derives from `Actor` |
| Rule chosen | `Actor → BP_` |
| Result | `Door` is reported, suggested name `BP_Door` |

Both the `Blueprint → BP_` rule and the `Actor → BP_` rule could apply here. The plugin resolves against the **generated class**, so the `Actor` rule is the one that matches — which is what you want, since it is the Blueprint's actual gameplay identity.
This is why the shipped defaults give `Blueprint` and `Actor` the same `BP_` prefix. Whichever path resolution takes, the answer is the same and nothing surprising happens.

### Example 2 — An Editor Utility Widget gets `EUW_`, not `WBP_`

You create an Editor Utility Widget and name it `AssetRenamer`.

| | |
| --- | --- |
| Asset class | `EditorUtilityWidgetBlueprint` |
| Generated class | derives from `EditorUtilityWidget`, which derives from `UserWidget` |
| Candidate rules | `UserWidget → WBP_`, `EditorUtilityWidget → EUW_` |
| Rule chosen | `EditorUtilityWidget → EUW_` |
| Result | suggested name `EUW_AssetRenamer` |

An Editor Utility Widget *is* a User Widget, so the `WBP_` rule genuinely applies — it is just further away. `EditorUtilityWidget` is an exact match on the generated class, `UserWidget` is one step up the chain, so the closer one wins and you get `EUW_`.

The practical takeaway: your editor tooling stays visually separate from your runtime UI in the Content Browser, without you having to carve anything out by hand.

> A plain **Editor Utility Blueprint** takes the other path. Its asset class is `EditorUtilityBlueprint`, which is an exact match for the shipped `EditorUtilityBlueprint → EUB_` rule, so it is decided on the asset class rather than the generated one. The generic `Blueprint` rule is deliberately skipped when a more specific Blueprint asset type has its own rule — otherwise every Editor Utility Blueprint would just be told to use `BP_`.

### Example 3 — A custom Actor class overrides the inherited rule

This is the pattern to reach for when one family of Actors deserves its own prefix.

Say your project has a C++ class `AInteractableDoor` that derives from `AActor`, and you want every Blueprint of it prefixed `BPI_` rather than the generic `BP_`.

**Add a rule:**

| Field | Value |
| --- | --- |
| Asset Class | `InteractableDoor` |
| Prefix | `BPI_` |

**What happens now:**

| Blueprint | Derives from | Rule chosen | Required name |
| --- | --- | --- | --- |
| `FrontDoor` | `InteractableDoor` | `InteractableDoor → BPI_` | `BPI_FrontDoor` |
| `Crate` | `Actor` | `Actor → BP_` | `BP_Crate` |
| `SlidingDoor` | a Blueprint deriving from `BPI_FrontDoor` | `InteractableDoor → BPI_` | `BPI_SlidingDoor` |

`FrontDoor` matches **both** rules — `InteractableDoor` is an exact match, `Actor` is its parent. The exact match is closer, so `BPI_` wins and the inherited `BP_` rule is ignored for this branch of the hierarchy. `Crate` is unaffected and still uses `BP_`.
Note the third row: the rule applies to the whole subtree. A Blueprint child of a Blueprint that derives from `InteractableDoor` still resolves to `BPI_`, because `InteractableDoor` remains an ancestor no matter how many Blueprint layers you stack on top.
This also means you never need to write one rule per subclass. Put the rule on the base class and every descendant inherits it, then add a narrower rule only where you want to override.

> **Tip:** if you add a custom class rule and it does not seem to take effect, the usual cause is that the class is not the one you think. Check the Blueprint's **Parent Class** in the Class Settings toolbar and confirm it matches the class named in your rule.

---

## How Violations Are Reported

Assets are validated in two situations:

- **When you save an asset**, if *Validate on Save* is enabled. This is a Data Validation setting, found under **Project Settings → Editor → Data Validation**.
- **When you run a validation pass manually**, via **Tools → Validate Assets**, or by right-clicking a folder in the Content Browser and choosing **Validate Assets in Folder**.

Results appear in the **Message Log** window (**Window → Message Log → Asset Check**). A violation reads like this:

> `'my_texture' does not follow the naming convention for Texture2D: it must start with 'T_'.`

![A naming violation in the Message Log](Images/message-log.png)

### Error vs Warning

The **Violation Severity** setting changes how strict this is:

- **Error** — the asset is marked as failing validation. This is what you want if you run validation as part of a build or submission check, since a failing asset can block the pass.
- **Warning** — the violation is still reported in the Message Log with the same rename fix, but the asset is treated as valid. Use this when you are introducing the plugin to an existing project and do not want to break anyone's workflow on day one.

A good adoption path is to start on **Warning**, clean up the backlog, then switch to **Error** to keep it clean.

---

## The One-Click Rename Fix

Every violation message includes a clickable **Rename to '…'** link with a suggested name.
Clicking it opens Unreal's standard asset rename flow, which updates references and handles redirectors for you — exactly as if you had renamed the asset by hand in the Content Browser.

The suggested name is built like this:

- If the name is missing the required prefix, the prefix is added to the front.
- If the name already starts with a prefix belonging to a *different* rule, that wrong prefix is **replaced** rather than stacked. So a Static Mesh named `T_Rock` is suggested as `SM_Rock`, not `SM_T_Rock`.
- If the name is missing the required suffix, the suffix is appended.

A few cases produce no fix link, and must be renamed by hand:

- Rules that use a **Regex Override**, because a pattern does not describe a single correct name.
- Rules with neither a prefix nor a suffix set.
- Cases where an asset with the suggested name already exists in the same folder. Here the fix link appears but reports that the name is taken.

---

## Controlling What Gets Checked

Some content should never be validated. Naming Validator skips the following:

- **Engine content** (`/Engine/`) and script paths — always skipped, unconditionally.
- **Plugin content** — skipped unless **Validate Plugin Content** is turned on. Only `/Game/` content is checked by default.
- **Ignored Paths** — any folders you list, plus everything beneath them.

**Ignored Paths** is the one you will use most. Typical entries:

- `/Game/Developers` — personal scratch folders
- `/Game/ThirdParty` — purchased assets you do not want to rename
- `/Game/Prototypes` — work that has not earned a convention yet

To add one, click **+** on the **Ignored Paths** array and use the folder picker to choose a content directory.

![Adding an entry to Ignored Paths](Images/ignored-paths.png)

> Paths are matched on folder boundaries, so ignoring `/Game/Dev` does **not** accidentally ignore `/Game/DevTools`.

---

## Advanced: Regex Overrides

When a prefix and suffix are not expressive enough, a rule can instead require the name to match a regular expression.

Set **Regex Override** on a rule (it is under the advanced section of the rule) and the **Prefix** and **Suffix** fields are ignored for that rule.

The pattern must match the **entire** asset name, not just part of it.

### Examples

| Goal | Pattern |
| --- | --- |
| `T_` prefix plus a required size suffix | `^T_.+_(512\|1024\|2048)$` |
| A prefix and a two-digit variant number | `^SM_.+_[0-9]{2}$` |
| Prefix, and no lowercase letters anywhere | `^T_[A-Z0-9_]+$` |

> **Remember:** rules using a regex do not offer a one-click rename, because the plugin cannot know which of the many valid names you intended. The violation is still reported, with the pattern quoted in the message.
If a pattern fails to compile, it is reported in the **Output Log** as soon as you finish editing the field, under the `LogNamingValidator` category — so a typo surfaces while you are still looking at Project Settings rather than on your next save.

---

## Sharing Rules With Your Team

Naming Validator's settings are stored in your project at:

```
YourProject/Config/DefaultEditor.ini
```

Commit that file to source control and every team member picks up the same conventions automatically. There is nothing per-user to configure.
This also means the rules travel with the project, not with the machine — a new hire cloning the repo is validating against your conventions on their first save.

---

## Default Rules

The plugin ships with a rule set based on the widely used Unreal community naming convention. You can edit, disable or delete any of these, and restore them at any time with **Reset Rules to Defaults**.

**Blueprints**

| Asset Type | Prefix |
| --- | --- |
| Blueprint | `BP_` |
| Actor | `BP_` |
| UserWidget | `WBP_` |
| AnimInstance | `ABP_` |
| GameModeBase | `GM_` |
| BlueprintFunctionLibrary | `BPFL_` |

**Editor utilities**

| Asset Type | Prefix |
| --- | --- |
| EditorUtilityWidget | `EUW_` |
| EditorUtilityBlueprint | `EUB_` |

**Meshes and animation**

| Asset Type | Prefix |
| --- | --- |
| StaticMesh | `SM_` |
| SkeletalMesh | `SK_` |
| Skeleton | `SKEL_` |
| PhysicsAsset | `PHYS_` |
| AnimSequence | `AS_` |
| BlendSpace | `BS_` |
| AnimMontage | `AM_` |
| Rig | `Rig_` |

**Materials and textures**

| Asset Type | Prefix |
| --- | --- |
| Material | `M_` |
| MaterialInstanceConstant | `MI_` |
| MaterialFunction | `MF_` |
| MaterialParameterCollection | `MPC_` |
| Texture2D | `T_` |
| TextureCube | `TC_` |
| TextureRenderTarget2D | `RT_` |
| PhysicalMaterial | `PM_` |

**Audio**

| Asset Type | Prefix |
| --- | --- |
| SoundWave | `S_` |
| SoundCue | `SC_` |
| SoundClass | `SCL_` |
| SoundAttenuation | `ATT_` |

**Data**

| Asset Type | Prefix |
| --- | --- |
| DataTable | `DT_` |
| CurveBase | `C_` |
| CurveTable | `CT_` |
| DataAsset | `DA_` |
| UserDefinedEnum | `E_` |
| UserDefinedStruct | `F_` |

**Effects and cinematics**

| Asset Type | Prefix |
| --- | --- |
| NiagaraSystem | `NS_` |
| NiagaraEmitter | `NE_` |
| LevelSequence | `LS_` |

**Media**

| Asset Type | Prefix |
| --- | --- |
| MediaSource | `MS_` |
| MediaPlayer | `MP_` |
| MediaOutput | `MO_` |
| MediaProfile | `MPR_` |

---

## Troubleshooting

**Nothing happens when I save an asset.**

Check, in order:

1. **Enable Validation** is ticked in Project Settings → Plugins → Naming Validator.
2. *Validate on Save* is enabled under **Project Settings → Editor → Data Validation**.
3. The asset is not inside an **Ignored Path**, and is under `/Game/` unless **Validate Plugin Content** is on.
4. A rule actually covers that asset type. Assets with no matching rule are never reported.

**My asset is reported even though the name looks right.**
Prefix and suffix matching is case-sensitive. Confirm the capitalisation matches the rule exactly, including the underscore.

**The wrong rule is being applied.**
The most specific rule wins, measured by inheritance distance. If a Widget Blueprint is being asked for `BP_` instead of `WBP_`, confirm the `UserWidget` rule exists and is **Enabled**.

**The Rename link says the name is already taken.**
Another asset in the same folder already uses the suggested name. Rename or move one of them manually.

**My regex never matches.**
The pattern must match the whole name. Anchor it with `^` and `$`, and check the **Output Log** for a compile error under `LogNamingValidator`.

**Reset Rules to Defaults wiped my custom rules.**
That button is destructive and has no undo. Restore `Config/DefaultEditor.ini` from source control if you have it.

---

## Support

Found a bug, or want a feature? Open an issue:

- **Issues:** https://github.com/lantteri/ue-naming-validator/issues
- **Documentation:** https://github.com/lantteri/ue-naming-validator

---

## License
This documentation is Copyright 2026 lantteri. All Rights Reserved.

Naming Validator is a commercial plugin distributed through Fab and is licensed under Epic Games' Fab license terms, accepted at the time of purchase.
This repository contains documentation only. It does not grant any license to the plugin or its source code.
