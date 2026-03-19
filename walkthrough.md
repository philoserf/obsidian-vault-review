# Vault Review Walkthrough

*2026-03-19T18:34:13Z by Showboat 0.6.1*
<!-- showboat-id: b783bd0e-3bd9-4d2a-98d0-3d37c716ca5c -->

## Overview

Vault Review is an Obsidian plugin that helps users systematically review every note in their vault. It takes a **snapshot** of all markdown files, then lets the user work through them one at a time — marking each as reviewed, opening random unreviewed files, and tracking progress via a status bar and settings panel.

**Key technologies:** TypeScript, Obsidian Plugin API, Bun (build + test), Biome (lint/format).

**Entry point:** `src/main.ts` — the entire plugin lives in a single 651-line file.

**Build output:** `main.js` (CJS, minified), `manifest.json`, `styles.css`.

## Architecture

### Directory layout

```bash
cat <<'HEREDOC'
.
├── src/
│   ├── main.ts            # Plugin source (all classes)
│   └── main.test.ts       # Unit tests (pure logic only)
├── scripts/
│   └── validate-plugin.ts # Pre-release validation
├── build.ts               # Bun bundler config
├── version-bump.ts        # Syncs version across manifests
├── manifest.json          # Obsidian plugin manifest
├── versions.json          # Obsidian version compatibility map
├── styles.css             # Plugin styles
├── biome.json             # Linter/formatter config
└── tsconfig.json          # TypeScript config
HEREDOC
```

```output
.
├── src/
│   ├── main.ts            # Plugin source (all classes)
│   └── main.test.ts       # Unit tests (pure logic only)
├── scripts/
│   └── validate-plugin.ts # Pre-release validation
├── build.ts               # Bun bundler config
├── version-bump.ts        # Syncs version across manifests
├── manifest.json          # Obsidian plugin manifest
├── versions.json          # Obsidian version compatibility map
├── styles.css             # Plugin styles
├── biome.json             # Linter/formatter config
└── tsconfig.json          # TypeScript config
```

### Module boundaries

Everything lives in `src/main.ts`. The file contains six classes/constructs:

1. **`VaultReviewPlugin`** — Main plugin class (extends `Plugin`). Manages lifecycle, commands, settings, and snapshot state.
2. **`StatusBar`** — Renders review status in Obsidian's status bar with a click menu.
3. **`VaultReviewSettingTab`** — Settings UI (extends `PluginSettingTab`). Snapshot management and status bar toggle.
4. **`ConfirmSnapshotDeleteModal`** — Confirmation dialog for destructive snapshot deletion.
5. **`FileStatusControllerModal`** — Suggest modal (command palette style) for quick review actions.
6. **`ACTIONS`** — Const object mapping action keys to display names.

### Data flow

1. User creates a snapshot → all `.md` files are captured with `to_review` status
2. Plugin persists snapshot to Obsidian's `data.json` via `saveData()`
3. User opens random files, marks them reviewed → status updates in-place
4. File renames/deletes are tracked via vault events
5. Status bar and settings panel read snapshot state to render progress

## Core Walkthrough

### Types and branding

The plugin uses a branded type pattern to distinguish plugin `File` objects from Obsidian's `TFile`. A `Brand<K, T>` intersection type adds a phantom `__brand` field that exists only at the type level.

```bash
sed -n '16,49p' src/main.ts
```

```output
type Brand<K, T> = K & { __brand: T };

type VaultReviewSettings = {
  snapshot?: Snapshot;
  settings: Settings;
};

type Snapshot = {
  files: File[];
  createdAt: Date;
};

type File = Brand<
  {
    path: string;
    status: SnapshotFileStatus;
  },
  "File"
>;

type FileStatus = "new" | "to_review" | "reviewed" | "deleted";

type SnapshotFileStatus = Exclude<FileStatus, "new">;

const toFile = (file: File | TFile, status: SnapshotFileStatus): File => {
  return {
    path: file.path,
    status: status,
  } as File;
};

type Settings = {
  showStatusBar: boolean;
};
```

Key points:

- `VaultReviewSettings` is the top-level persisted shape — an optional `Snapshot` plus a `Settings` object.
- `FileStatus` has four states: `new` (not in snapshot), `to_review`, `reviewed`, `deleted`.
- `SnapshotFileStatus` excludes `new` since snapshot files always have an explicit status.
- `toFile()` is a factory that casts a plain object to the branded `File` type.

### Plugin lifecycle — `onload`

The plugin registers everything in `onload`: ribbon icon, status bar, four commands, settings tab, and vault event handlers.

```bash
sed -n '62,124p' src/main.ts
```

```output
  onload = async () => {
    await this.loadSettings();

    // Ribbon
    this.addRibbonIcon("scan-eye", "Open vault review", () => {
      this.openFileStatusController();
    });

    // Status bar
    this.statusBar = new StatusBar(this.addStatusBarItem(), this);

    // Commands
    this.addCommand({
      id: "open-random-file",
      name: "Open random not reviewed file",
      callback: () => {
        this.openRandomFile();
      },
    });
    this.addCommand({
      id: "complete-review",
      name: "Review file",
      checkCallback: (checking) => {
        if (checking) {
          return this.getActiveFileStatus() === "to_review";
        }

        this.completeReview();
      },
    });
    this.addCommand({
      id: "complete-review-and-open-next",
      name: "Review file and open next random file",
      checkCallback: (checking) => {
        if (checking) {
          return this.getActiveFileStatus() === "to_review";
        }

        this.completeReview({ openNext: true });
      },
    });
    this.addCommand({
      id: "unreview-file",
      name: "Unreview file",
      checkCallback: (checking) => {
        if (checking) {
          return this.getActiveFileStatus() === "reviewed";
        }

        this.unreviewFile();
      },
    });

    // Settings
    this.addSettingTab(new VaultReviewSettingTab(this.app, this));

    // Events
    this.registerEvent(this.app.vault.on("rename", this.handleFileRename));
    this.registerEvent(this.app.vault.on("delete", this.handleFileDelete));
    this.registerEvent(
      this.app.workspace.on("file-open", this.statusBar.update),
    );
  };
```

The four commands use Obsidian's `checkCallback` pattern: when `checking` is `true`, return whether the command should be available. This hides "Review file" when the active file isn't `to_review`, and "Unreview file" when it isn't `reviewed`.

Three vault events are registered: `rename` updates snapshot paths, `delete` marks files as deleted, and `file-open` refreshes the status bar.

### Settings persistence

Settings are loaded with a shallow merge pattern, then the `createdAt` date is rehydrated from its JSON string form.

```bash
sed -n '126,148p' src/main.ts
```

```output
  loadSettings = async () => {
    const data = await this.loadData();
    this.settings = {
      ...DEFAULT_SETTINGS,
      ...data,
      settings: { ...DEFAULT_SETTINGS.settings, ...data?.settings },
    };

    if (typeof this.settings.snapshot?.createdAt === "string") {
      this.settings.snapshot.createdAt = new Date(
        this.settings.snapshot.createdAt,
      );
    }
  };

  saveSettings = async () => {
    await this.saveData(this.settings);
  };

  onExternalSettingsChange = async () => {
    await this.loadSettings();
    this.statusBar.update();
  };
```

The two-level spread (`...data` then `...data?.settings`) ensures new settings fields get defaults without clobbering existing nested values. `onExternalSettingsChange` handles Obsidian syncing settings from another device.

### Core operations — review, unreview, random file

The main workflow methods modify snapshot file statuses and persist.

```bash
sed -n '193,254p' src/main.ts
```

```output
  openRandomFile = () => {
    if (!this.settings.snapshot) {
      new Notice("Vault review snapshot is not created");
      return;
    }

    const files = this.getToReviewFiles();
    if (!files.length) {
      new Notice("All files are reviewed");
      return;
    }

    const randomFile = files[Math.floor(Math.random() * files.length)];
    this.focusFile(randomFile, false);
  };

  private focusFile = async (file: File, newLeaf: boolean | PaneType) => {
    const targetFile = this.app.vault.getFileByPath(file.path);

    if (targetFile) {
      const leaf = this.app.workspace.getLeaf(newLeaf);
      leaf.openFile(targetFile);
    } else {
      new Notice(`Cannot find a file ${file.path}`);
      if (this.settings.snapshot) {
        this.settings.snapshot.files = this.settings.snapshot.files.filter(
          (fp) => fp.path !== file.path,
        );
        this.statusBar.update();
        await this.saveSettings();
      }
    }
  };

  completeReview = async ({
    file,
    openNext = false,
  }: {
    file?: File;
    openNext?: boolean;
  } = {}) => {
    const activeFile = file ?? this.getActiveFile();
    if (!activeFile) {
      return;
    }

    const snapshotFile = this.getSnapshotFile(activeFile.path);

    if (!snapshotFile) {
      new Notice("File was added to snapshot and marked as reviewed");
      this.settings.snapshot?.files.push(toFile(activeFile, "reviewed"));
    } else {
      snapshotFile.status = "reviewed";
    }

    if (openNext) {
      this.openRandomFile();
    }

    this.statusBar.update();
    await this.saveSettings();
  };
```

`openRandomFile` picks a random `to_review` file using `Math.random()`. `focusFile` opens it in the active leaf, or removes it from the snapshot if the file no longer exists on disk.

`completeReview` handles two cases: if the file is already in the snapshot, it flips its status to `reviewed`; if it's new (not in snapshot), it adds it as `reviewed`. The optional `openNext` flag chains into `openRandomFile` for a smooth workflow.

### Snapshot deletion with Promise.withResolvers

Snapshot deletion uses a confirmation modal with a promise-based flow.

```bash
sed -n '275,312p' src/main.ts
```

```output
  public deleteSnapshot = async ({
    askForConfirmation = true,
  }: {
    askForConfirmation?: boolean;
  } = {}) => {
    const { promise, resolve } = Promise.withResolvers<DeleteSnapshotResult>();

    let settled = false;

    const onDelete = async () => {
      if (settled) return;
      settled = true;
      this.settings.snapshot = undefined;
      this.statusBar.update();
      await this.saveSettings();
      resolve("deleted");
    };

    const onCancel = () => {
      if (settled) return;
      settled = true;
      resolve("cancelled");
    };

    if (askForConfirmation) {
      const modal = new ConfirmSnapshotDeleteModal(
        this.app,
        onDelete,
        onCancel,
      );
      modal.onClose = onCancel;
      modal.open();
    } else {
      await onDelete();
    }

    return promise;
  };
```

The `settled` guard prevents double-resolution — `modal.onClose` fires both when the user clicks Cancel *and* when the modal closes after Delete. Without it, `resolve()` would be called twice. `Promise.withResolvers` is a relatively new API (ES2024); see the concerns section for compatibility notes.

### Vault event handlers

File renames update the snapshot path in-place. File deletions mark the snapshot entry as `deleted` rather than removing it, preserving the count for statistics.

```bash
sed -n '314,337p' src/main.ts
```

```output
  private handleFileRename = async (file: TAbstractFile, oldPath: string) => {
    if (file instanceof TFolder) {
      return;
    }

    const snapshotFile = this.getSnapshotFile(oldPath);
    if (snapshotFile) {
      snapshotFile.path = file.path;
      await this.saveSettings();
    }
  };

  private handleFileDelete = async (file: TAbstractFile) => {
    if (file instanceof TFolder || !this.settings.snapshot) {
      return;
    }

    const snapshotFile = this.getSnapshotFile(file.path);
    if (snapshotFile) {
      snapshotFile.status = "deleted";
      this.statusBar.update();
      await this.saveSettings();
    }
  };
```

Both handlers guard against `TFolder` events — only files matter. The rename handler looks up by `oldPath` (not the new path), which is correct since the snapshot still holds the old path at that point.

### StatusBar

The `StatusBar` class manages a clickable status bar element that shows the review state of the active file.

```bash
sed -n '340,407p' src/main.ts
```

```output
class StatusBar {
  element: HTMLElement;
  plugin: VaultReviewPlugin;

  isReviewed = false;

  constructor(element: HTMLElement, plugin: VaultReviewPlugin) {
    this.element = element;
    this.plugin = plugin;

    element.createSpan("status").setText("Not reviewed");
    element.addClass("mod-clickable");
    element.addEventListener("click", this.onClick);

    this.update();
  }

  update = () => {
    if (!this.plugin.settings.snapshot) {
      this.setIsVisible(false);
      return;
    }

    const activeFileStatus = this.plugin.getActiveFileStatus();
    if (!activeFileStatus || activeFileStatus === "deleted") {
      this.setIsVisible(false);
      return;
    }

    this.setIsVisible(this.plugin.settings.settings.showStatusBar);
    this.isReviewed = activeFileStatus === "reviewed";

    if (activeFileStatus === "new") {
      this.setText("New file");
    } else if (activeFileStatus === "to_review") {
      this.setText("Not reviewed");
    } else if (activeFileStatus === "reviewed") {
      this.setText("Reviewed");
    } else {
      this.setText("Unknown status");
    }
  };

  private onClick = (event: MouseEvent) => {
    const menu = new Menu();

    menu.addItem((item) => {
      item.setTitle("Reviewed");
      item.setChecked(this.isReviewed);
      item.onClick(() => this.plugin.completeReview());
    });
    menu.addItem((item) => {
      item.setTitle("Not reviewed");
      item.setChecked(!this.isReviewed);
      item.onClick(() => this.plugin.unreviewFile());
    });

    menu.showAtMouseEvent(event);
  };

  private setText = (text: string) => {
    this.element.getElementsByClassName("status")[0].setText(text);
  };

  private setIsVisible = (isVisible: boolean) => {
    this.element.toggleClass("hidden", !isVisible);
  };
}
```

The status bar hides itself when there's no snapshot, no active file, or the file is deleted. Clicking it opens a context menu to toggle review status. The `mod-clickable` class gives it Obsidian's standard hover cursor.

### FileStatusControllerModal

The suggest modal provides a command-palette-style interface for quick actions. It adapts its suggestions based on the active file's status.

```bash
sed -n '571,650p' src/main.ts
```

```output
const ACTIONS = {
  open_random: {
    name: "Open random not reviewed file",
  },
  review: {
    name: "Review file",
  },
  review_and_next: {
    name: "Review file and open next random file",
  },
  unreview: {
    name: "Unreview file",
  },
} as const;

type Action = keyof typeof ACTIONS;

class FileStatusControllerModal extends SuggestModal<Action> {
  plugin: VaultReviewPlugin;

  constructor(app: App, plugin: VaultReviewPlugin) {
    super(app);
    this.plugin = plugin;

    const fileStatus = this.plugin.getActiveFileStatus();
    this.setPlaceholder(
      !fileStatus
        ? ""
        : fileStatus === "new"
          ? "This file is not in snapshot"
          : fileStatus === "to_review"
            ? "This file is not reviewed"
            : "This file is reviewed",
    );
  }

  getSuggestions = (query: string): Action[] => {
    const activeFile = this.plugin.getActiveFile();
    let actions: Action[] = [];

    if (!activeFile) {
      actions = ["open_random"];
    } else {
      const isReviewed =
        this.plugin.getSnapshotFile(activeFile.path)?.status === "reviewed";

      if (isReviewed) {
        actions = ["open_random", "unreview"];
      } else {
        actions = ["review_and_next", "review", "open_random"];
      }
    }

    return actions.filter((a) =>
      ACTIONS[a].name.toLowerCase().includes(query.toLowerCase()),
    );
  };

  renderSuggestion = (action: Action, el: HTMLElement) => {
    el.createEl("div", { text: ACTIONS[action].name });
  };

  onChooseSuggestion = (action: Action, _evt: MouseEvent | KeyboardEvent) => {
    if (action === "open_random") {
      this.plugin.openRandomFile();
    }

    if (action === "review") {
      this.plugin.completeReview();
    }

    if (action === "review_and_next") {
      this.plugin.completeReview({ openNext: true });
    }

    if (action === "unreview") {
      this.plugin.unreviewFile();
    }
  };
}
```

The modal's `getSuggestions` tailors available actions: reviewed files get "unreview" and "open random"; unreviewed files get "review", "review and next", and "open random". The query filter enables fuzzy search by action name.

### Settings tab — snapshot management and statistics

The settings tab is the primary UI for creating/deleting snapshots and viewing progress.

```bash
sed -n '417,533p' src/main.ts
```

```output
  display(): void {
    const { containerEl } = this;
    containerEl.empty();

    const snapshot = this.plugin.settings.snapshot;

    // Main action
    const settingEl = new Setting(containerEl)
      .setName("Snapshot")
      .setDesc(
        snapshot?.createdAt
          ? `Snapshot created on ${snapshot?.createdAt.toLocaleDateString()}.`
          : "Create a snapshot of the vault.",
      );
    if (snapshot) {
      settingEl.addButton((btn) => {
        btn.setIcon("trash");
        btn.setWarning();
        btn.onClick(async () => {
          await this.plugin.deleteSnapshot();
          this.display();
        });
      });
      settingEl.addButton((btn) => {
        btn.setButtonText("Add all new files to snapshot").onClick(async () => {
          const vaultFiles = this.plugin.app.vault
            .getMarkdownFiles()
            .filter(
              (file) =>
                !this.plugin.settings.snapshot?.files.some(
                  (f) => f.path === file.path,
                ),
            )
            .map((file) => toFile(file, "to_review"));
          this.plugin.settings.snapshot?.files.push(...vaultFiles);
          this.plugin.statusBar.update();
          this.display();
          await this.plugin.saveSettings();
        });
      });
    } else {
      settingEl.addButton((btn) => {
        btn.setButtonText("Create snapshot");
        btn.setCta();
        btn.onClick(async () => {
          const files = this.plugin.app.vault
            .getMarkdownFiles()
            .map((file) => toFile(file, "to_review"));
          this.plugin.settings.snapshot = {
            files,
            createdAt: new Date(),
          };
          this.plugin.statusBar.update();
          this.display();
          await this.plugin.saveSettings();
        });
      });
    }

    // Snapshot info
    if (snapshot) {
      containerEl.createDiv("snapshot-info", (div) => {
        const allFilesLength = this.plugin.app.vault.getMarkdownFiles().length;
        const snapshotFilesLength = snapshot.files.length;
        const deletedFilesLength = snapshot.files.filter(
          (file) => file.status === "deleted",
        ).length;
        const notInSnapshotLength =
          allFilesLength - snapshotFilesLength + deletedFilesLength;
        const reviewedFilesLength = snapshot.files.filter(
          (file) => file.status === "reviewed",
        ).length;
        const toReviewFilesLength =
          snapshotFilesLength - reviewedFilesLength - deletedFilesLength;

        const activeFilesLength = snapshotFilesLength - deletedFilesLength;
        const percentSnapshotCompleted = activeFilesLength
          ? Math.round((reviewedFilesLength / activeFilesLength) * 100)
          : 0;
        const percentSnapshotDeleted = snapshotFilesLength
          ? Math.round((deletedFilesLength / snapshotFilesLength) * 100)
          : 0;

        div.createEl("p").setText(`Markdown files in vault: ${allFilesLength}`);

        const inSnapshotEl = div.createEl("p", "in-snapshot");
        inSnapshotEl
          .createSpan()
          .setText(`In snapshot: ${snapshotFilesLength}`);
        inSnapshotEl.createSpan().setText(`To review: ${toReviewFilesLength}`);
        inSnapshotEl
          .createSpan()
          .setText(
            `Reviewed: ${reviewedFilesLength} (${percentSnapshotCompleted}%)`,
          );
        inSnapshotEl
          .createSpan()
          .setText(
            `Deleted: ${deletedFilesLength} (${percentSnapshotDeleted}%)`,
          );

        div.createEl("p").setText(`Not in snapshot: ${notInSnapshotLength}`);
      });
    }

    new Setting(containerEl)
      .setName("Status bar")
      .setDesc("Show file review status in the status bar.")
      .addToggle((toggle) => {
        toggle.setValue(this.plugin.settings.settings.showStatusBar);
        toggle.onChange(async (value) => {
          this.plugin.settings.settings.showStatusBar = value;
          this.plugin.statusBar.update();
          await this.plugin.saveSettings();
        });
      });
  }
```

The settings tab has two states: no snapshot (shows "Create snapshot" CTA) and active snapshot (shows trash, "Add new files", and statistics). The "Add all new files" button filters vault files against the existing snapshot to find additions since the snapshot was created.

The statistics section computes review progress: total, reviewed, to-review, deleted, and not-in-snapshot counts with percentages. The `notInSnapshotLength` calculation accounts for deleted files that are still in the snapshot array.

### Build system

```bash
sed -n '1,19p' build.ts
```

```output
const watch = process.argv.includes("--watch");

const result = await Bun.build({
  entrypoints: ["src/main.ts"],
  outdir: ".",
  format: "cjs",
  external: ["obsidian", "electron"],
  minify: !watch,
});

if (!result.success) {
  console.error("Build failed");
  for (const message of result.logs) console.error(message);
  process.exit(1);
}

if (watch) console.log("Watching for changes...");

export {};
```

The build uses Bun's native bundler. Key choices:

- **CJS format** — required by Obsidian's plugin loader.
- **External `obsidian` and `electron`** — provided by the host app at runtime.
- **Minified in production**, unminified in watch mode for debugging.
- **No source maps** — see concerns section.

### Tests

```bash
grep -c 'test(' src/main.test.ts
```

```output
16
```

```bash
grep 'describe\|test(' src/main.test.ts
```

```output
import { describe, expect, test } from "bun:test";
describe("toFile", () => {
  test("creates a File with to_review status", () => {
  test("creates a File with reviewed status", () => {
  test("creates a File with deleted status", () => {
describe("getToReviewFiles", () => {
  test("filters to only to_review files", () => {
  test("returns empty array when no files to review", () => {
  test("returns empty for empty snapshot", () => {
describe("getSnapshotFile", () => {
  test("finds file by path", () => {
  test("returns undefined for missing path", () => {
describe("snapshot statistics", () => {
  test("computes correct stats for mixed snapshot", () => {
  test("handles empty snapshot", () => {
  test("handles fully reviewed snapshot", () => {
  test("handles all deleted snapshot", () => {
describe("file rename logic", () => {
  test("updates path of snapshot file", () => {
describe("file delete logic", () => {
  test("marks file as deleted", () => {
describe("DEFAULT_SETTINGS", () => {
  test("has correct defaults", () => {
  test("no snapshot by default", () => {
```

16 tests across 7 describe blocks covering the pure logic: `toFile`, filtering, lookup, statistics, rename, delete, and defaults. Tests **duplicate type definitions and helper functions** from `main.ts` rather than importing them — this is because the Obsidian module can't be imported in a test environment. See concerns section for trade-offs.

### Styles

```bash
cat styles.css
```

```output
.status-bar-item.plugin-vault-review {
  padding: 0 var(--size-2-2);
  gap: var(--size-2-2);
}

.in-snapshot {
  display: flex;
  flex-direction: column;
  gap: var(--size-2-2);
}

.hidden {
  display: none;
}
```

Minimal CSS: status bar spacing, snapshot stats layout, and a `.hidden` utility class. Uses Obsidian CSS variables (`--size-2-2`) for consistency with the host theme.

## Concerns

### Code quality

1. **Single-file monolith** (`src/main.ts`, 651 lines) — All six classes live in one file. This is manageable at current size but will become harder to navigate as features grow. Consider splitting into `plugin.ts`, `statusbar.ts`, `modals.ts`, `settings.ts`, and a shared `types.ts`.

2. **Duplicated types in tests** — `main.test.ts` re-declares `Brand`, `File`, `SnapshotFileStatus`, and `toFile` instead of importing from a shared module. If the source types change, the tests won't catch the drift. Extracting pure types and functions into a separate file (importable by both source and tests) would fix this.

3. **Sequential `if` in `onChooseSuggestion`** (line 633–649) — Four independent `if` blocks execute for every action, but only one can match. Should be `if/else if` or a `switch` to express mutual exclusivity and avoid unnecessary checks. ([GitHub issue #17](https://github.com/philoserf/obsidian-vault-review/issues/17))

4. **Nested ternary in `FileStatusControllerModal` constructor** (line 596–604) — Three levels of ternary for the placeholder text. An `if/else` chain or a lookup object would be clearer. ([GitHub issue #18](https://github.com/philoserf/obsidian-vault-review/issues/18))

5. **Linear file lookup** — `getSnapshotFile` uses `Array.find()` which is O(n) per call. For vaults with thousands of notes, consider a `Map<string, File>` index. Not urgent but worth noting for large vaults.

6. **`getActiveFile` only accepts `.md`** — Filters by `extension !== "md"`, which means non-markdown files are silently ignored. This is intentional for the review use case but isn't documented.

### Community standards and compatibility

7. **`Promise.withResolvers` compatibility** (line 280) — This is an ES2024 feature. Older Obsidian versions running on earlier Electron/V8 may not support it. A manual polyfill (`new Promise((resolve, reject) => ...)`) would be safer. ([GitHub issue #16](https://github.com/philoserf/obsidian-vault-review/issues/16))

8. **No source maps for development** — `build.ts` doesn't generate source maps in either mode. Stack traces from the minified `main.js` are unreadable. Adding `sourcemap: "external"` for dev builds would help debugging. ([GitHub issue #19](https://github.com/philoserf/obsidian-vault-review/issues/19))

9. **No `schemaVersion` in settings** — If the settings shape changes in a future version, there's no migration path. Adding a version field now would make future migrations straightforward. ([GitHub issue #15](https://github.com/philoserf/obsidian-vault-review/issues/15))

10. **Arrow functions as class methods** — The plugin uses arrow function properties (`onload = async () => {}`) throughout. While this avoids `this`-binding issues, it's unconventional for Obsidian plugins and means methods aren't on the prototype (slightly higher memory per instance, though irrelevant for a singleton plugin).

11. **No `onunload`** — The plugin doesn't implement `onunload`. Obsidian's `Plugin` base class handles cleanup of registered events and commands, so this is safe — but an explicit empty `onunload` would signal intentionality.

