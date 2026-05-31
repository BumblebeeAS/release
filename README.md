# BlueROV Workspace — Documentation Site

Source for the BlueROV2 ROS 2 + Gazebo documentation site. Built with
[Zensical](https://zensical.org) (drop-in compatible with Material for MkDocs,
so `mkdocs serve` also works).

## Local development

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Live-reloading dev server on http://localhost:8000
zensical serve
# or, equivalently:
mkdocs serve
```
