# Static Site Preview (SSP)

Deploy static site previews via ssh.

## NOTICE

Please note that this is in no way a secure solution for hosting static site previews.

- You should only use this in repositories you **DO** trust.
- You should only use this on a server
  - you **DO NOT** care about.
  - where you **DO NOT** have sensitive information stored.
  - where you **DO NOT** run other sensitive services.
- Unless you jail the specific ssh user, you are allowing a GitHub Action full ssh access to your server.
- You allow any other repository with access to the same preview server to overwrite each other's previews.

  (Though this should not happen under normal circumstances, as the GitHub Action uses a hash of repository name + pull request number)

## Screenshots

### GitHub Pull Request comment

![GitHub Pull Request comment][github-pull-request-comment-screenshot]

## Usage

### GitHub Action

```yml
name: preview

on: [pull_request]

jobs:
  deploy:
    runs-on: ubuntu-latest

    permissions:
      pull-requests: write # needed for preview pull request comment
      actions: read

    # Deploy to the preview environment
    environment:
      name: preview-${{ github.event.number }}
      url: ${{ steps.deploy-preview.outputs.url }}

    steps:
      - uses: actions/download-artifact@v4
        with:
          path: ./dist

      - name: deploy preview
        id: deploy-preview
        uses: dafnik/ssp@v1
        # with:
        #   source: dist/*
        #   target: /var/www/preview
        #   host: preview.yxz.abc
        #   port: 22
        #   username: ubuntu
        #   key: ${{ secrets.PREVIEW_SSH_PRIVATE_KEY }}
        #   strip_components: 0
        #   delete_threshold_days: 30
```

| Inputs                  | Default value | Required | Description                                                                    |
| ----------------------- | ------------- | -------- | ------------------------------------------------------------------------------ |
| `source`                |               | x        | Path to the files which should be deployed                                     |
| `target`                |               | x        | Preview server target path, must be a directory path.                          |
| `host`                  |               | x        | Preview server domain                                                          |
| `port`                  | `22`          |          | Preview server ssh port                                                        |
| `username`              |               | x        | Preview server ssh username                                                    |
| `key`                   |               | x        | Preview server ssh key content of private key. ex raw content of ~/.ssh/id_rsa |
| `strip_components`      | `0`           |          | remove the specified number of leading path elements                           |
| `delete_threshold_days` | `30`          |          | Number of days after inactive previews are deleted                             |

Furthermore, see [action.yml](action.yml)

### NGINX Configuration

Previews are stored in the `/var/www/preview` directory by default.

```
# /etc/nginx/site-enabled/preview

server {
    listen 80;
    server_name *.preview.xyz.abc;
    return 301 https://$server_name$request_uri;
}


server {
    listen 443 ssl http2;
    server_name ~^(?P<sub>.+)\.preview\.xyz\.abc$;

    ssl_certificate /etc/letsencrypt/live/preview.xyz.abc/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/preview.xyz.abc/privkey.pem;

    # GZIP
    gzip on;
    gzip_disable "msie6";

    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_min_length 256;
    gzip_types text/xml text/javascript font/ttf font/eot font/otf application/rdf+xml application/x-javascript application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

    # Remove X-Powered-By, which is an information leak
    fastcgi_hide_header X-Powered-By;

    # Do not send nginx server header
    server_tokens off;

    add_header Access-Control-Allow-Origin *;
    add_header Strict-Transport-Security  "max-age=31536000; includeSubDomains; preload" always;
    add_header Referrer-Policy            "strict-origin" always;
    add_header X-Frame-Options            "SAMEORIGIN"    always;
    add_header X-XSS-Protection           "1; mode=block" always;
    add_header X-Content-Type-Options     "nosniff"       always;

    access_log on;
    error_log off;

    root /var/www/preview/$sub;

    location / {
        # Check if the root directory exists
        if (!-d $document_root) {
            return 404;
        }

        try_files $uri $uri/ /index.html;

        index index.html;
    }
}
```

### Cleanup cron job

Delete previews with no activity in the last 30 days.

```bash
# /home/ubuntu/cronDeleteUnusedPreviews.sh

# Define the directory to search in. Modify this variable to suit your needs.
SEARCH_DIR="/var/www/preview"

# Find and delete directories not modified in the last 30 days.
find "$SEARCH_DIR" -type d -mtime +30 -exec rm -rf {} +

# Explanation:
# - `find "$SEARCH_DIR"`: Start searching in the specified directory.
# - `-type d`: Only look for directories.
# - `-mtime +30`: Find directories that were last modified more than 30 days ago.
# - `-exec rm -rf {} +`: Delete each directory found ({} is replaced by the found directory name).
```

Crontab example:

```bash
@daily /home/ubuntu/cronDeleteUnusedPreviews.sh
```

## Release instructions

In order to release a new version of this action:

1. Locate the semantic version of the [upcoming release][release-list] (a draft is maintained by the [`draft-release` workflow][draft-release]).

2. Publish the draft release from the `main` branch with semantic version as the tag name, _with_ the checkbox to publish to the GitHub Marketplace checked. :ballot_box_with_check:

3. After publishing the release, the [`release` workflow][release] will automatically run to create/update the corresponding the major version tag such as `v0`.

   ⚠️ Environment approval is required. Check the [Release workflow run list][release-workflow-runs].

## License

The scripts and documentation in this project are released under the [MIT License](LICENSE).

<!-- references -->

[release-list]: https://github.com/dafnik/ssp/releases
[draft-release]: .github/workflows/draft-release.yml
[release]: .github/workflows/release.yml
[release-workflow-runs]: https://github.com/dafnik/ssp/actions/workflows/release.yml
[github-pull-request-comment-screenshot]: .github/ssp/pr-comment-screenshot.png
