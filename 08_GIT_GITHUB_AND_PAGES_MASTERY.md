# Tutorial 8: Git, GitHub & GitHub Pages Mastery
### *Version Control, Cloud Portability, and Reading Your Tutorials on Any Device*

---

## 1. Git vs. GitHub: The Mental Model

Beginners often confuse Git and GitHub. Here is the easiest way to understand the difference:

```
┌─────────────────────────────────────────────────────────────┐
│ 📷 GIT (The Camera on your Laptop)                          │
│ • Runs 100% locally on your machine without internet.       │
│ • Takes exact snapshots ("Commits") of your code over time. │
│ • Lets you travel back in time if you break something.      │
└──────────────────────────────┬──────────────────────────────┘
                               │
                       git push│ (Syncing across internet)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ ☁️ GITHUB (The Cloud Album & Collaboration Hub)              │
│ • A website (cloud service) owned by Microsoft.             │
│ • Stores your Git snapshots securely online.                │
│ • Lets you access your code/tutorials from any device.      │
│ • Allows multiple developers to collaborate on one project. │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. The Entire Use Case of GitHub (Why Every Company Uses It)

1. **Cloud Backup & Zero Data Loss**: If your laptop breaks, your code and tutorials are safe in the cloud.
2. **Access from Any Device**: Read your markdown notes, review code, and manage tickets from your phone, iPad, or any office computer.
3. **Collaboration & Code Reviews (Pull Requests)**: Multiple engineers can work on SimplyAdmission simultaneously without overwriting each other's code.
4. **Time Machine (Version History)**: You can see the exact line of code that changed 6 months ago, who changed it, and why.
5. **Automation (CI/CD via GitHub Actions)**: Automatically runs your JUnit tests whenever you upload code. If tests fail, it prevents bad code from going to production.
6. **Free Website Hosting (GitHub Pages)**: Converts your Markdown documentation into a live website accessible via a public link!

---

## 3. Step-by-Step: Storing Your Project & Tutorials on GitHub

### Step A: Create a Free GitHub Account & Repository
1. Go to [github.com](https://github.com) and create an account (if you don't have one).
2. On the top-right corner, click the **`+`** icon ➔ select **New repository**.
3. Fill in the details:
   * **Repository name**: `simplyadmission-core` (or `java-backend-learning`).
   * **Visibility**: Choose **Public** (required for free GitHub Pages) or **Private**.
   * Leave *"Add a README"*, *".gitignore"*, and *"License"* **unchecked** (we already created them locally!).
4. Click **Create repository**.
5. Keep the page open! You will see a URL like:
   `https://github.com/YOUR_USERNAME/simplyadmission-core.git`

---

### Step B: Connect Your Local Project to GitHub (Via Terminal or IntelliJ)

#### Method 1: Using IntelliJ IDEA (Zero Terminal Typing)
1. Open your project in **IntelliJ IDEA**.
2. In the top menu, go to **VCS** (or **Git**) ➔ **Enable Version Control Integration...** ➔ select **Git** ➔ click **OK**.
3. In the top menu, go to **Git** ➔ **GitHub** ➔ **Share Project on GitHub**.
4. Log into your GitHub account when prompted.
5. Click **Share**. IntelliJ will automatically create the repository and upload all your tutorials and code!

#### Method 2: Using the Command Line (Standard Git Commands)
Open PowerShell inside your project folder (`D:\Alok\.project\java`) and run:
```bash
# 1. Initialize local Git tracking
git init

# 2. Stage all files (tutorials, Java code, configs)
git add .

# 3. Create your first snapshot (Commit)
git commit -m "Initial commit: SimplyAdmission core architecture, tests & tutorials"

# 4. Set main branch
git branch -M main

# 5. Link local project to your GitHub cloud repository
git remote add origin https://github.com/YOUR_USERNAME/simplyadmission-core.git

# 6. Push all files to the cloud!
git push -u origin main
```

---

## 4. How to Read Your Tutorials on ANY Device (GitHub Pages Setup)

You asked: *"how i store my tutorial and open on any device when required"*. 

GitHub has a built-in superpower called **GitHub Pages**. It turns your Markdown tutorials into a live, mobile-friendly website that you can open on your **phone, tablet, or work laptop** via a clean URL!

### Step-by-Step Setup:
1. Open your repository on **GitHub.com**.
2. Click the **Settings** tab (gear icon on the top right).
3. In the left sidebar, click **Pages** (under the "Code and automation" section).
4. Under **Build and deployment**:
   * **Source**: Select `Deploy from a branch`.
   * **Branch**: Select `main` and folder `/ (root)` (or `/docs`).
   * Click **Save**.
5. Wait 60 seconds! GitHub will generate your live URL:
   👉 **`https://YOUR_USERNAME.github.io/simplyadmission-core/`**

---

### 📱 How to Access on Your Mobile Phone:
1. Open your phone browser (Safari, Chrome).
2. Visit your GitHub Pages URL or your GitHub repository page:
   * **Option 1 (GitHub Pages Website)**: `https://YOUR_USERNAME.github.io/simplyadmission-core/tutorials/01_THE_BIG_PICTURE_AND_PROCESS`
   * **Option 2 (GitHub Mobile App)**: Download the official **GitHub Mobile App** (iOS / Android), log in, and all your `.md` tutorials are formatted with dark mode, zoom, and instant bookmarking!

---

## 5. How to Manage Versions (The Day-to-Day Git Lifecycle)

Every time you write new code or add a tutorial, follow the **3-Step Rhythm**:

```
 ┌──────────────┐       git add       ┌──────────────────┐      git commit      ┌────────────────┐
 │ Working File │ ──────────────────► │  Staging Area    │ ───────────────────► │ Local History  │
 │  (Your IDE)  │                     │ (Box being packed)│                     │  (Permanent)   │
 └──────────────┘                     └──────────────────┘                      └───────┬────────┘
                                                                                        │
                                                                                git push│
                                                                                        ▼
                                                                                ┌────────────────┐
                                                                                │ GitHub (Cloud) │
                                                                                └────────────────┘
```

### The 3 Core Commands:
1. **`git status`**: Shows what files you changed or created.
2. **`git add .`**: Stages your changes (prepares them for snapshot).
3. **`git commit -m "Detailed message explaining what changed"`**: Takes the permanent snapshot.
4. **`git push`**: Sends the snapshots to GitHub.

---

## 6. Real-World Version Control: Time Travel & Branching

### A. How to Time Travel (Undo Mistakes)
Imagine you wrote some code that broke your web app, and you want to restore the working version from yesterday:
```bash
# View all past snapshots with their unique ID (Commit Hash)
git log --oneline

# Undo your last commit safely without losing your files
git revert <commit-hash>

# Or discard all uncommitted mistakes and return to the last clean state
git restore .
```

### B. Feature Branches: The Enterprise Workflow
At SimplyAdmission, developers **never** commit directly to the `main` branch. 
Instead, they use **Branches** (parallel universes):

```
main branch:           ●─────────────────────────●─────────────────● (Live Production Code)
                        \                       /
feature/lead-webhook:    ●────────●────────────● (Pull Request: Tested & Reviewed)
```

1. **Create a new branch for a feature**:
   ```bash
   git checkout -b feature/round-robin-allocator
   ```
2. Write and test your code on this branch. The `main` production code remains 100% untouched and stable!
3. **Push the branch to GitHub**:
   ```bash
   git push origin feature/round-robin-allocator
   ```
4. **Open a Pull Request (PR)** on GitHub.
   * A senior engineer reviews your code.
   * Automated tests run via CI/CD.
   * Once approved, the branch is **merged** into `main`!

---

## 🎯 Summary Checklist for Today
* [x] Understand Git (local tool) vs GitHub (cloud platform).
* [x] Open project in IntelliJ and connect it to a GitHub repository.
* [x] Enable **GitHub Pages** in Repository Settings to turn your tutorials into a mobile-readable website.
* [x] Learn the 3-step loop: `git add .` ➔ `git commit -m "..."` ➔ `git push`.
