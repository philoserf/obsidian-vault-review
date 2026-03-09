# Obsidian Vault Review — Code Walkthrough

*2026-03-09T04:29:53Z by Showboat 0.6.1*
<!-- showboat-id: 348a91ea-296f-4782-909a-4ff0c2ee2fe3 -->

## What This Plugin Does

Vault Review is an Obsidian plugin that helps you systematically review every note in your vault. It:

1. Takes a **snapshot** of all markdown files at a point in time
2. Lets you **randomly open** unreviewed files one by one
3. **Tracks progress** — each file is `to_review`, `reviewed`, `deleted`, or `new`
4. Responds to vault events (renames, deletes) to keep the snapshot consistent
5. Shows review status in the **status bar** and a **settings tab** with statistics

The entire plugin lives in a single source file (`src/main.ts`, ~640 lines) plus a test file, a build script, and a version-bump utility.

---

## Project Structure

```bash
find . -type f \( -name "*.ts" -o -name "*.json" -o -name "*.css" -o -name "*.yml" -o -name "*.yaml" \) \! -path "*/node_modules/*" \! -path "*/bun.lock*" | sort
```

```output
./.github/dependabot.yml
./.github/workflows/main.yml
./.github/workflows/release.yml
./biome.json
./build.ts
./manifest.json
./package.json
./scripts/validate-plugin.ts
./src/main.test.ts
./src/main.ts
./styles.css
./tsconfig.json
./version-bump.ts
./versions.json
```

Key files:

| File | Purpose |
|------|---------|
| `src/main.ts` | All plugin logic — types, plugin class, UI classes |
| `src/main.test.ts` | Unit tests (Bun test runner) |
| `build.ts` | Bun bundler config |
| `version-bump.ts` | Syncs version across package.json, manifest.json, versions.json |
| `scripts/validate-plugin.ts` | Pre-release validation |
| `manifest.json` | Obsidian plugin metadata |
| `styles.css` | Plugin styles |
| `biome.json` | Linter/formatter config |

---

## Data Model

Everything starts with the type system. The plugin uses TypeScript branded types to distinguish a `File` object (in the snapshot) from a plain object with `path` and `status` fields.

```bash
sed -n '16,55p' src/main.ts
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

const DEFAULT_SETTINGS: VaultReviewSettings = {
  settings: {
    showStatusBar: true,
  },
};
```

Walking through the types:

- **`Brand<K, T>`** — A branded type pattern. Adds a phantom `__brand` field so TypeScript treats `File` as distinct from `{ path: string; status: string }`. The brand exists only at the type level — no runtime cost.
- **`VaultReviewSettings`** — The top-level persisted state. Has an optional `snapshot` (null when no snapshot exists) and a `settings` object.
- **`Snapshot`** — An array of `File` entries plus the creation date.
- **`File`** — A branded type with `path` (vault-relative) and `status`.
- **`FileStatus`** — Four states: `new` (not in snapshot), `to_review`, `reviewed`, `deleted`. Note that `new` only exists at runtime for display — it's never stored in the snapshot.
- **`SnapshotFileStatus`** — `FileStatus` minus `new`. This is what gets persisted.
- **`toFile`** — Factory function. Accepts either a branded `File` or an Obsidian `TFile` and casts the result to the branded type.
- **`DEFAULT_SETTINGS`** — No snapshot, status bar enabled.

> **Concern:** `toFile` uses `as File` to cast, which bypasses the brand. The brand pattern provides type safety at call sites but the factory itself is unchecked. This is a common trade-off with branded types in TypeScript.

---

## Plugin Lifecycle

The plugin class extends Obsidian's `Plugin` base class. `onload` is the entry point — called when Obsidian activates the plugin.

```bash
sed -n '57,123p' src/main.ts
```

```output
export default class VaultReviewPlugin extends Plugin {
  settings!: VaultReviewSettings;

  statusBar!: StatusBar;

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
```

`onload` does five things in order:

1. **Loads settings** from `data.json` via Obsidian's `loadData()` API
2. **Adds a ribbon icon** ("scan-eye") that opens the `FileStatusControllerModal`
3. **Creates the status bar** widget
4. **Registers four commands** — the core review workflow:
   - `open-random-file` — always available
   - `complete-review` — only available when active file is `to_review` (uses `checkCallback`)
   - `complete-review-and-open-next` — same check, but chains to the next random file
   - `unreview-file` — only available when active file is `reviewed`
5. **Registers three event handlers** — `rename`, `delete` on the vault, and `file-open` on the workspace (to update the status bar)

> **Note:** All methods are arrow functions (`onload = async () => {}`), which binds `this` lexically. This avoids the classic JavaScript `this`-binding pitfall when passing methods as callbacks. It also means these methods live on the instance, not the prototype — a minor memory trade-off that's fine for a singleton plugin.

> **Concern:** There is no `onunload` method. Obsidian's `registerEvent` handles cleanup for the three event subscriptions, and `addCommand`/`addRibbonIcon`/`addStatusBarItem` are also managed by the base `Plugin` class. However, `StatusBar` adds a raw `addEventListener("click", ...)` that isn't cleaned up through Obsidian's API. In practice this is fine because the status bar element is destroyed on unload, but it's not idiomatic.

---

## Settings Persistence

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

`loadSettings` does a shallow merge of defaults with persisted data, then a nested merge for the `settings` sub-object. This ensures new settings fields get their defaults even if the user has old persisted data.

The `createdAt` date requires special handling: `JSON.stringify` serializes `Date` as an ISO string, so on load it must be parsed back. The `typeof === "string"` check handles this roundtrip.

`onExternalSettingsChange` is an Obsidian lifecycle hook called when settings are modified outside the plugin (e.g. sync). It reloads and refreshes the UI.

> **Concern:** There's no `schemaVersion` field. If the data model changes in a future release, migration logic would have to rely on heuristics to detect the old format. Adding `schemaVersion: 1` now would make future migrations safe (see issue #15).

---

## Core Query Methods

These methods read state but don't modify it. They're used throughout the plugin.

```bash
sed -n '150,182p' src/main.ts
```

```output
  getActiveFile = (): TFile | null => {
    const activeFile = this.app.workspace.getActiveFile();
    if (activeFile?.extension !== "md") {
      return null;
    }
    return activeFile;
  };

  getSnapshotFile = (path?: string) => {
    path = path ?? this.getActiveFile()?.path;
    if (!path) {
      return;
    }

    return this.settings.snapshot?.files.find((f) => f.path === path);
  };

  getActiveFileStatus = (): FileStatus | undefined => {
    const activeFile = this.getActiveFile();
    if (!activeFile) {
      return;
    }

    return this.getSnapshotFile(activeFile.path)?.status ?? "new";
  };

  getToReviewFiles = () => {
    return (
      this.settings.snapshot?.files.filter(
        (file) => file.status === "to_review",
      ) ?? []
    );
  };
```

- **`getActiveFile`** — Returns the currently open file, but only if it's markdown. Non-markdown files (images, PDFs, canvas) are ignored.
- **`getSnapshotFile`** — Finds a file in the snapshot by path. Accepts an explicit path or falls back to the active file. Returns `undefined` if not found (i.e. file is "new").
- **`getActiveFileStatus`** — Combines the above: returns the status of the active file, defaulting to `"new"` if it's not in the snapshot. Returns `undefined` if no markdown file is open.
- **`getToReviewFiles`** — Returns all files with `to_review` status. Returns `[]` if no snapshot exists.

> **Performance note:** `getSnapshotFile` does a linear scan (`Array.find`). For a vault with thousands of files, this runs on every file-open event (via the status bar update). A `Map<string, File>` index would be O(1) but adds complexity. Linear scan is likely fast enough for typical vault sizes.

---

## Review Workflow

These are the action methods that modify state.

```bash
sed -n '184,225p' src/main.ts
```

```output
  openFileStatusController = () => {
    if (!this.settings.snapshot) {
      new Notice("Vault review snapshot is not created");
      return;
    }

    new FileStatusControllerModal(this.app, this).open();
  };

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
```

**`openRandomFile`** picks a random file from the `to_review` list and opens it via `focusFile`. Uses `Math.random()` — not cryptographically secure, but fine for shuffling notes.

**`focusFile`** is where a significant concern lives (issue #6):

- If `getFileByPath` finds the file, it opens it in a leaf (tab).
- If the file **can't be found**, it shows a notice — then **silently removes the file from the snapshot** and saves. The user loses their review tracking for that file with no confirmation. This is data loss. The file might have been moved (not renamed through Obsidian), or the vault might not be fully indexed yet.

A safer approach: mark the file as `deleted` instead of removing it, or prompt the user.

---

## Marking Files Reviewed / Unreviewed

```bash
sed -n '227,273p' src/main.ts
```

```output
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

  unreviewFile = async (file?: File) => {
    const activeFile = file ?? this.getActiveFile();
    if (!activeFile) {
      return;
    }

    const snapshotFile = this.getSnapshotFile(activeFile.path);

    if (!snapshotFile) {
      new Notice("File was added to snapshot and marked as not reviewed");
      this.settings.snapshot?.files.push(toFile(activeFile, "to_review"));
    } else {
      snapshotFile.status = "to_review";
    }

    this.statusBar.update();
    await this.saveSettings();
  };
```

Both methods follow the same pattern:

1. Get the file (from argument or active file)
2. Look it up in the snapshot
3. If found → mutate `status` in place
4. If not found → add it to the snapshot (auto-enroll)
5. Update UI and persist

The auto-enroll behavior (lines 241-243, 264-266) is interesting: if you review a file that wasn't in the original snapshot, it gets added automatically. This handles files created after the snapshot.

**`completeReview`** also accepts `{ openNext: true }` to chain into `openRandomFile` — this powers the "review and next" workflow.

> **Concern:** Both methods mutate `snapshotFile.status` directly. While functional, this bypasses the `toFile` factory and the brand type guarantee. The object is already branded from initial creation, so the brand persists, but it's a pattern that could cause confusion if the `File` type gains additional required fields.

---

## Snapshot Deletion

```bash
sed -n '275,304p' src/main.ts
```

```output
  public deleteSnapshot = async ({
    askForConfirmation = true,
  }: {
    askForConfirmation?: boolean;
  } = {}) => {
    const { promise, resolve } = Promise.withResolvers<DeleteSnapshotResult>();

    const onDelete = async () => {
      this.settings.snapshot = undefined;
      this.statusBar.update();
      await this.saveSettings();
      resolve("deleted");
    };

    const onCancel = () => resolve("cancelled");

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

`deleteSnapshot` uses `Promise.withResolvers` (ES2024) to create a promise that resolves when the user confirms or cancels.

Two concerns here:

1. **Double-resolve (issue #5):** `onCancel` is assigned as both the cancel button callback AND `modal.onClose`. When the user clicks Delete, `onDelete` resolves with `"deleted"`, then `modal.close()` fires `onClose` → `onCancel` → resolves again with `"cancelled"`. The second resolve is ignored by the Promise spec, but it's sloppy. The fix is to guard against double-calls (e.g., a `resolved` flag).

2. **`Promise.withResolvers` compatibility (issue #16):** This is ES2024. Obsidian's `manifest.json` declares `minAppVersion: "1.0.0"`. If older Obsidian versions ship an Electron that doesn't support this API, the plugin will crash on load.

---

## Event Handlers

```bash
sed -n '306,329p' src/main.ts
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

**`handleFileRename`** — When a file is renamed (or moved), updates the path in the snapshot. Simple and correct for individual files.

> **Concern (issue #8):** When a **folder** is renamed, Obsidian fires a single `rename` event for the folder, not for each child file. The handler returns early for `TFolder`, so all child file paths in the snapshot become stale. Fix: when a folder is renamed, iterate the snapshot and update all paths that start with `oldPath + "/"`.

**`handleFileDelete`** — Marks the file as `deleted` in the snapshot rather than removing it. This is the right approach — it preserves the record and keeps statistics accurate. Note the contrast with `focusFile`, which removes files entirely (the inconsistency flagged in issue #6).

This concludes the `VaultReviewPlugin` class (line 330). The rest of the file defines UI classes.

---

## StatusBar

```bash
sed -n '332,399p' src/main.ts
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

`StatusBar` manages a single status bar element with three behaviors:

1. **`update`** — Called on every file-open event. Hides the bar if there's no snapshot, no active file, or the file is deleted. Otherwise shows the review status text.
2. **`onClick`** — Shows a context menu with "Reviewed" / "Not reviewed" toggle items. Uses Obsidian's `Menu` API positioned at the mouse event.
3. **Visibility** — Uses `toggleClass("hidden", ...)` to show/hide.

> **Concern (issue #7):** The `.hidden` CSS class (`styles.css:12`) is `display: none` with no namespace prefix. Any other plugin or Obsidian theme that defines `.hidden` could conflict. Should be `.vault-review-hidden`.

---

## Settings Tab

```bash
sed -n '401,526p' src/main.ts
```

```output
class VaultReviewSettingTab extends PluginSettingTab {
  plugin: VaultReviewPlugin;

  constructor(app: App, plugin: VaultReviewPlugin) {
    super(app, plugin);
    this.plugin = plugin;
  }

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
}
```

The settings tab has two states:

**No snapshot:** Shows a "Create snapshot" button that calls `vault.getMarkdownFiles()`, wraps each in a `File` with `to_review` status, and saves.

**Snapshot exists:** Shows three controls:
- A trash button to delete the snapshot (with confirmation modal)
- An "Add all new files" button that finds markdown files not already in the snapshot and adds them as `to_review`
- A status bar visibility toggle

Below the controls, it displays **snapshot statistics**: total vault files, files in/not-in snapshot, reviewed/to_review/deleted counts with percentages.

> **Concern (issue #14):** The statistics computation (lines 471-490) is duplicated in the test file as `computeStats()`. It should be extracted into a shared function that both the settings tab and tests use. This would eliminate the duplication and ensure tests verify the real logic.

> **Note:** `getMarkdownFiles()` is called multiple times within `display()` (lines 434, 454, 471). Each call iterates the entire vault. For very large vaults this could be noticeable, though Obsidian likely caches the file list internally.

---

## Modals

```bash
sed -n '528,642p' src/main.ts
```

```output
type DeleteSnapshotResult = "deleted" | "cancelled";

class ConfirmSnapshotDeleteModal extends Modal {
  constructor(
    app: App,
    onDelete: () => Promise<void> | void,
    onCancel: () => Promise<void> | void,
  ) {
    super(app);

    this.setTitle("Delete snapshot?");

    new Setting(this.contentEl)
      .setName("This action cannot be undone")
      .setDesc(
        "You will lose all progress and will need to create a new snapshot.",
      )
      .addButton((btn) => {
        btn.setButtonText("Cancel");
        btn.onClick(async () => {
          await onCancel();
          this.close();
        });
      })
      .addButton((btn) => {
        btn.setButtonText("Delete");
        btn.setWarning();
        btn.onClick(async () => {
          await onDelete();
          this.close();
        });
      });
  }
}

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

### ConfirmSnapshotDeleteModal

A simple confirmation dialog with Cancel and Delete buttons. Uses Obsidian's `Setting` component for layout, which is a common pattern in the plugin ecosystem.

### FileStatusControllerModal

A `SuggestModal` — Obsidian's typeahead/command palette component. It shows context-aware actions:

- **No active file:** Only "Open random"
- **Reviewed file:** "Open random" and "Unreview"
- **Unreviewed file:** "Review and next", "Review", "Open random"

The actions are filtered by the user's search query. `ACTIONS` is defined as `const` with `as const`, so `Action` is a string literal union type derived from the keys.

> **Minor concern (issue #17):** `onChooseSuggestion` uses sequential `if` statements. Since only one action fires per call, `if/else if` or `switch` would communicate mutual exclusivity. Not a bug, just clarity.

> **Minor concern (issue #18):** The nested ternary in the constructor (lines 588-596) sets the placeholder. A lookup map would be more readable.

---

## Build System

```bash
cat build.ts
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

The build uses Bun's native bundler:

- **Entry:** `src/main.ts`
- **Format:** CommonJS — required by Obsidian's plugin loader
- **Externals:** `obsidian` and `electron` are provided by Obsidian at runtime
- **Minification:** Enabled for production, disabled in watch mode
- **Output:** `./main.js` (the file Obsidian loads)

> **Concern (issue #19):** No source maps are generated. In watch/dev mode, adding `sourcemap: "linked"` would help debugging.

> **Note:** The `export {}` at the end makes this file a module (required for top-level `await`).

---

## Version Management

```bash
cat version-bump.ts
```

```output
import { readFileSync, writeFileSync } from "node:fs";

const targetVersion = process.env.npm_package_version;
if (!targetVersion) {
  throw new Error("No version found in package.json");
}

// Update manifest.json
const manifest = JSON.parse(readFileSync("manifest.json", "utf8"));
const { minAppVersion } = manifest;
manifest.version = targetVersion;
writeFileSync("manifest.json", `${JSON.stringify(manifest, null, 2)}\n`);

// Update versions.json
const versions = JSON.parse(readFileSync("versions.json", "utf8"));
versions[targetVersion] = minAppVersion;
writeFileSync("versions.json", `${JSON.stringify(versions, null, 2)}\n`);

console.log(`Updated to version ${targetVersion}`);
```

`version-bump.ts` keeps three files in sync:

1. `package.json` — the source of truth (version updated manually by the developer)
2. `manifest.json` — Obsidian reads this to display plugin version
3. `versions.json` — Maps plugin versions to minimum Obsidian versions

Run via `bun run version` (which sets `npm_package_version` from `package.json`).

---

## CI/CD

```bash
cat .github/workflows/main.yml
```

```output
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: latest
      - run: bun install
      - run: bun run check
```

```bash
cat .github/workflows/release.yml
```

```output
name: Release

on:
  push:
    tags:
      - "*"

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: latest

      - run: |
          bun install
          bun run build

      - name: Create release
        uses: softprops/action-gh-release@v2
        with:
          files: |
            main.js
            manifest.json
          fail_on_unmatched_files: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### CI (`main.yml`)

Runs on push to main and pull requests. Executes `bun run check` which is `tsc --noEmit && biome check .` — type checking and linting.

> **Concern (issue #13):** Tests are not run in CI. `bun test` is missing from the pipeline. The 16 unit tests provide no safety net in CI.

### Release (`release.yml`)

Triggered by any tag push. Builds the plugin and creates a GitHub release with `main.js` and `manifest.json` as assets.

> **Concern (issue #4):** The tag pattern `"*"` matches **any** tag, not just version tags. A tag like `experiment` or `v2-beta` would trigger a release. Should use a semver pattern like `"[0-9]+.[0-9]+.[0-9]+"`.

---

## Tests

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

16 tests across 7 describe blocks. Coverage of pure logic is decent — the test suite covers:

- File creation (`toFile`)
- Filtering (`getToReviewFiles`)
- Lookup (`getSnapshotFile`)
- Statistics computation (4 edge cases)
- Rename and delete mutations
- Default settings

### Critical testing concern (issue #12)

The test file **re-declares** `Brand`, `File`, `SnapshotFileStatus`, and `toFile` locally (lines 2-20) instead of importing from `src/main.ts`. This means:

- Tests verify **copies** of the functions, not the production code
- If the production implementation diverges, tests still pass
- Test results give false confidence

The fix: export pure functions from `main.ts` and import them in tests.

### What's not tested

- Plugin lifecycle (`onload`, `loadSettings`)
- `focusFile` (including the silent-removal bug)
- `deleteSnapshot` (including the double-resolve bug)
- `completeReview` / `unreviewFile` auto-enroll path
- StatusBar, modals, settings tab
- Event handlers (`handleFileRename`, `handleFileDelete`)

---

## Styles

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

Three CSS rules:

1. **`.status-bar-item.plugin-vault-review`** — Properly scoped with Obsidian's plugin class convention. Good.
2. **`.in-snapshot`** — Used for the statistics display in settings. Reasonable.
3. **`.hidden`** — Generic name, `display: none`. This is the collision risk flagged in issue #7. Should be namespaced.

Uses Obsidian's CSS custom properties (`--size-2-2`) which ensures visual consistency with the theme.

---

## Summary of Concerns

All open issues are tracked on GitHub. Here's the consolidated view:

| # | Severity | Issue |
|---|----------|-------|
| 6 | CRITICAL | `focusFile` silently removes files from snapshot when not found |
| 12 | CRITICAL | Tests re-declare types instead of importing production code |
| 8 | HIGH | Folder rename doesn't update child file paths |
| 13 | HIGH | CI doesn't run tests |
| 4 | HIGH | Release workflow triggers on any tag |
| 5 | MEDIUM | `deleteSnapshot` promise double-resolves |
| 7 | MEDIUM | Generic `.hidden` CSS class risks collision |
| 14 | MEDIUM | `computeStats` duplicated between settings tab and tests |
| 15 | MEDIUM | No `schemaVersion` for future migrations |
| 16 | MEDIUM | `Promise.withResolvers` compatibility with older Obsidian |
| 17 | LOW | Sequential `if` in `onChooseSuggestion` |
| 18 | LOW | Nested ternary in modal constructor |
| 19 | LOW | No source maps in dev builds |

### What the code does well

- Clean TypeScript with strict mode enabled
- Good use of Obsidian's API patterns (`registerEvent`, `checkCallback`, `SuggestModal`)
- Zero runtime dependencies
- Branded types for domain modeling
- Arrow functions prevent `this`-binding bugs
- File deletion marks as `deleted` rather than removing (preserves history)
- Settings merge handles forward compatibility for new settings fields
- Biome enforces consistent formatting

