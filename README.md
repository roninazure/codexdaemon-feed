# .github/scripts/update_neural_diagnostics.py
import os
import datetime
from pathlib import Path

REPO_ROOT = Path(__file__).resolve().parent.parent
README_PATH = REPO_ROOT / "README.md"
INJECT_START = "<!-- CODEX_NEURAL_DIAGNOSTICS_START -->"
INJECT_END = "<!-- CODEX_NEURAL_DIAGNOSTICS_END -->"

EXCLUDE_DIRS = {".git", ".venv", "__pycache__", "node_modules"}
INCLUDE_EXTS = {".py", ".sh", ".yml", ".yaml", ".md"}


def scan_repo():
    total_lines = 0
    total_files = 0
    top_risk_score = 0
    top_risk_file = ""

    for root, dirs, files in os.walk(REPO_ROOT):
        dirs[:] = [d for d in dirs if d not in EXCLUDE_DIRS]
        for file in files:
            if Path(file).suffix.lower() in INCLUDE_EXTS:
                fpath = Path(root) / file
                try:
                    with open(fpath, "r", encoding="utf-8", errors="ignore") as f:
                        lines = f.readlines()
                    line_count = len(lines)
                    total_files += 1
                    total_lines += line_count
                    score = min(100, (line_count / 5) + (10 if "runner" in file else 0))
                    if score > top_risk_score:
                        top_risk_score = score
                        top_risk_file = fpath.relative_to(REPO_ROOT)
                except Exception:
                    continue

    code_to_doc_ratio = round(total_lines / max(total_files, 1), 2)
    return total_files, total_lines, code_to_doc_ratio, top_risk_score, top_risk_file


def build_html_block(total_files, total_lines, ratio, risk_score, risk_file):
    timestamp = datetime.datetime.utcnow().isoformat(timespec="seconds") + "Z"
    return f"""
{INJECT_START}
<div align="center" style="border:1px solid #6a0dad; border-radius:10px; padding:10px; margin-top:30px; background:#111; box-shadow:0 0 20px #6a0dad; font-family:monospace;">

<h3 style="color:#39ff14;">🧠 CodexDaemon Neural Diagnostics</h3>
<p style="font-size:14px;">
🧬 Files Scanned: <b>{total_files}</b><br>
📏 Total Lines: <b>{total_lines}</b><br>
⚖️ Code-to-Doc Ratio: <b>{ratio}</b><br>
☢️ Max Mutation Risk: <span style="color:#ff4d4d;"><b>{risk_score:.1f}%</b></span> (<code>{risk_file}</code>)
</p>
<p style="font-size:12px; color:#999;">Last diagnostic scan: {timestamp}</p>
</div>
{INJECT_END}
"""


def update_readme():
    total_files, total_lines, ratio, risk_score, risk_file = scan_repo()
    html_block = build_html_block(total_files, total_lines, ratio, risk_score, risk_file)

    with open(README_PATH, "r", encoding="utf-8") as f:
        lines = f.readlines()

    new_lines = []
    in_block = False
    for line in lines:
        if INJECT_START in line:
            in_block = True
            new_lines.append(html_block + "\n")
        elif INJECT_END in line:
            in_block = False
            continue
        elif not in_block:
            new_lines.append(line)

    with open(README_PATH, "w", encoding="utf-8") as f:
        f.writelines(new_lines)


if __name__ == "__main__":
    update_readme()
