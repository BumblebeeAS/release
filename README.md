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

## Build a static site

```bash
zensical build      # outputs to ./site
# or:
mkdocs build
```

## Self-hosted deployment

The site is just a static `./site/` directory after `build`. Push it to your
server with rsync:

```bash
# one-shot: build then sync
zensical build && \
  rsync -avz --delete site/ user@your-server:/var/www/bluerov-docs/
```

Nginx serves `/var/www/bluerov-docs/` as a static site. Suggested location
block:

```nginx
server {
  listen 80;
  server_name docs.bluerov.example.com;
  root /var/www/bluerov-docs;
  index index.html;
  location / { try_files $uri $uri/ =404; }
}
```

## Layout

```
.
├── mkdocs.yml        # site config (sections, theme, extensions)
├── docs/             # markdown source
│   ├── index.md
│   ├── overview/     # architecture, conventions, running the sim
│   ├── packages/     # one page per src/ package
│   └── strategies/   # behaviour-tree mission deep dives
└── site/             # generated output (gitignored)
```

To add a new page: drop a markdown file under `docs/<section>/` and add it
under the matching `nav:` entry in `mkdocs.yml`.
