# Hybrid Setup

## 🔗 Hybrid Setup: Obsidian + VS Code + GitHub

### 🗂️ 1. **Create a Shared Vault Folder**

Pick a folder that both Obsidian and VS Code will use:

- Example: `C:\Documents\Notes\Thitar Vault`

This folder will be your **Obsidian vault** and your **VS Code workspace**.

---

### 🧠 2. **Open the Folder in Obsidian**

- Open Obsidian → **Open folder as vault**
- Choose `Thitar Vault`
- Start writing notes in Markdown (`.md` files)

---

### 💻 3. **Open the Same Folder in VS Code**

- Open VS Code → File → Open Folder → `Thitar Vault`
- You’ll see all your Obsidian notes
- Use Markdown preview (`Ctrl+Shift+V`) and Git features

---

### 🔄 4. **Initialize Git (if not done yet)**

In VS Code terminal:

```bash
git init
git remote add origin https://github.com/YOUR_USERNAME/obsidian-vault.git
git branch -m master main
git add .
git commit -m "Initial hybrid setup"
git push -u origin main
```

---

### 🔁 5. **Enable Git Sync in Obsidian**

Install the **Obsidian Git plugin**:

- Settings → Community Plugins → Browse → Search "Obsidian Git"
- Install and enable

Configure:

- Auto commit + push intervals
- Commit message template: `Update: {{date}}`
- Enable auto pull before push

---

### 🧼 6. **Add a `.gitignore` File**

To avoid syncing Obsidian system files: Create a `.gitignore` file in your vault folder:

```gitignore
.obsidian/
.DS_Store
Thumbs.db
```

---

### 📱 7. **Access on Mobile**

Install **Obsidian Mobile**:

- Log in with your Obsidian sync account (if using Obsidian Sync)
- Or use GitHub + a mobile Git client (like Working Copy on iOS or Termux on Android)

Alternative: Use **GitHub Mobile** to view/edit notes directly.

---

## 🧩 Optional Enhancements

- Use **VS Code extensions** like GitLens, Markdown All in One
- Use **Obsidian plugins** like Daily Notes, Templates, or Dataview
- Set up **GitHub Actions** to auto-backup or publish notes

---
