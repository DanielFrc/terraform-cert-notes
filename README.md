# Terraform Certification Notes

An Obsidian-compatible knowledge base for studying **Terraform Associate 004**, with related **AWS** and cloud engineering concepts.

Built for certification preparation, long-term retention, and practical cloud engineering reference — not just exam memorization.

## What Is This Repository?

- Structured study notes covering Terraform fundamentals, workflow, file organization, and configuration.
- Written in Markdown with Obsidian wikilinks (`[[note-name]]`) for navigation between notes.
- Focused on understanding _why_ patterns exist, common mistakes, and real-world engineering tradeoffs.
- Aligned with HashiCorp Terraform Associate 004 objectives, with AWS Solutions Architect Associate tie-ins where relevant.

**Start here:** [00_Index.md](00_Index.md) — progress tracker, weak areas, and external resources.

## Notes

| Note | Topics |
|---|---|
| [01_Fundamentals](01_Fundamentals.md) | IaC concepts, HCL basics, resource referencing, core components, Terraform editions |
| [02_Terraform_Workflow](02_Terraform_Workflow.md) | `init`, `validate`, `fmt`, `plan`, `apply`, `destroy`, dependencies, parallelism |
| [03_Terraform_File_Structure](03_Terraform_File_Structure.md) | File organization, state files, working directory, version control practices |
| [04_Terraform_Configuration](04_Terraform_Configuration.md) | Providers, `required_providers`, aliases, authentication, version constraints |

## Study Progress

Current coverage (see [00_Index.md](00_Index.md) for the live checklist):

- [x] Fundamentals
- [x] Core Workflow
- [x] File Structure
- [x] Providers _(partial)_
- [ ] Modules
- [ ] State Management (remote backend, locking, drift)
- [ ] Terraform Cloud / HCP Terraform
- [ ] Security & Secrets

## Repository Structure

```
terraform-cert-notes/
├── 00_Index.md                  # Entry point — navigation, progress, resources
├── 01_Fundamentals.md
├── 02_Terraform_Workflow.md
├── 03_Terraform_File_Structure.md
├── 04_Terraform_Configuration.md
└── README.md
```

Notes are numbered for study order. Each note is self-contained but cross-linked to related topics.

## Using This Repository in Obsidian

Obsidian is recommended to get the most out of this vault — wikilinks, backlinks, and the graph view connect related concepts automatically.

### 1. Download the Repository

**Option A — Git clone**

```sh
git clone <repository-url>
cd terraform-cert-notes
```

**Option B — Download ZIP**

Download the repository from GitHub, extract the archive, and note the folder path.

### 2. Open as an Obsidian Vault

1. Open **Obsidian**.
2. Click **Open folder as vault** (or **Open another vault** → **Open folder as vault**).
3. Select the `terraform-cert-notes` folder.
4. Obsidian loads all `.md` files in the vault.

### 3. Configure Obsidian (Optional)

These settings improve readability but are not required:

- **Settings → Files and links → New link format** — set to **Relative path to file** (keeps links portable if you move the vault).
- **Settings → Files and links → Use [[Wikilinks]]** — enabled by default; required for `[[note-name]]` navigation.
- **Settings → Editor → Strict line breaks** — disable if you prefer standard Markdown line-break behavior.

### 4. Start Reading

1. Open [00_Index.md](00_Index.md) from the file explorer or quick switcher (`Cmd/Ctrl + O`).
2. Follow wikilinks (e.g., `[[01_Fundamentals]]`) to move between notes.
3. Use the **Graph view** to see how topics connect as the vault grows.

> **Tip:** If you already have an existing Obsidian vault, you can copy or symlink this folder into it instead of opening it as a standalone vault. Wikilinks resolve as long as all notes share the same vault root.

## Resources

- [Udemy — Terraform Associate 004 courses](https://www.udemy.com/courses/search/?q=terraform+associate+004)
- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [Terraform Registry](https://registry.terraform.io/)
- [Terraform Associate 004 Exam Guide](https://developer.hashicorp.com/certifications/infrastructure-automation)

## Contributing

This is a personal study vault, but suggestions and corrections are welcome via issues or pull requests.

## Disclaimer
These are my personal study notes taken while completing the Udemy — Terraform Associate 004 course on Udemy. 
The content consists of my own interpretations, summaries, and self-directed expansions enriched using AI tools for educational purposes. This repository is not affiliated with, endorsed by, or connected to the course instructor or Udemy.

