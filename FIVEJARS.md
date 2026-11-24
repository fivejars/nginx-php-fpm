# Five Jars fork

This is the Five Jars fork of [dwchiang/nginx-php-fpm](https://github.com/dwchiang/nginx-php-fpm). We keep it so we can ship PHP and nginx security releases without waiting for upstream to publish them.

- Images: `ghcr.io/fivejars/nginx-php-fpm:<php>-fpm-<os>-nginx-<nginx>` (public, no login needed to pull)
- Package page: https://github.com/orgs/fivejars/packages/container/package/nginx-php-fpm
- Platform: `linux/amd64` only
- Images are built and pushed by hand. There is no CI in this fork.

## How this fork differs from upstream

- Upstream generates `Dockerfiles/{OS}/Dockerfile-*` with a toolkit that isn't in this repo. We don't have it, so we add new Dockerfiles by copying the previous one and bumping versions by hand (see below). Only the Dockerfiles we actually use are added.
- `Dockerfile-8.4.28-fpm-alpine3.24-nginx-1.31.9` and newer: `https://nginx.org/keys/nginx_signing.rsa.pub` returned 404 in 2026-11, so the nginx signing key is copied from the official `nginx:<version>-alpine-slim` image (`COPY --from=...`) instead of being downloaded. The pinned `KEY_SHA512` check is unchanged.

## Adding a new PHP / nginx version

The commands below use macOS `sed -i ''`. On Linux, use `sed -i`.

### 0. Set the versions

```bash
PHP_OLD=8.4.28   NGINX_OLD=1.31.9
PHP_NEW=8.4.29   NGINX_NEW=1.31.10      # example values
ALPINE=alpine3.24
OLD=Dockerfile-${PHP_OLD}-fpm-${ALPINE}-nginx-${NGINX_OLD}
NEW=Dockerfile-${PHP_NEW}-fpm-${ALPINE}-nginx-${NGINX_NEW}
TAG=${PHP_NEW}-fpm-${ALPINE}-nginx-${NGINX_NEW}
IMAGE=ghcr.io/fivejars/nginx-php-fpm:${TAG}
```

### 1. Check that the new versions exist

```bash
# The official PHP image tag must exist
curl -s "https://hub.docker.com/v2/repositories/library/php/tags?name=${PHP_NEW}-fpm-${ALPINE}" | grep -o '"name":"[^"]*"'

# The official nginx Dockerfile for the new version: read NJS_VERSION, ACME_VERSION and PKGOSSCHECKSUM
gh api "repos/nginx/docker-nginx/contents/mainline/alpine/Dockerfile?ref=${NGINX_NEW}" --jq .content | base64 -d \
  | grep -E 'ENV (NJS|ACME)_VERSION|PKGOSSCHECKSUM='
```

Write down the new `ACME_VERSION`, `NJS_VERSION` and the 128-character checksum.

### 2. Sync the fork and create a branch

```bash
gh repo sync fivejars/nginx-php-fpm --source dwchiang/nginx-php-fpm   # pull in upstream changes
gh repo clone fivejars/nginx-php-fpm && cd nginx-php-fpm
git switch -c php-${PHP_NEW}-nginx-${NGINX_NEW}
```

If upstream has already published the new Dockerfile, use theirs and skip to step 4. You only need to re-apply the signing-key fix from step 3, and only while nginx.org still returns 404 for the key.

### 3. Create the new Dockerfile from the previous one

```bash
cd Dockerfiles/${ALPINE}
cp $OLD $NEW
sed -i '' \
  -e "s/php:${PHP_OLD}-fpm/php:${PHP_NEW}-fpm/" \
  -e "s/${NGINX_OLD}/${NGINX_NEW}/g" \
  $NEW
grep -nE '^FROM|ENV (NGINX|NJS|ACME)_VERSION|PKGOSSCHECKSUM=|COPY --from' $NEW
```

The second `sed` rule updates every place the nginx version appears: `NGINX_VERSION`, the comment links and the `COPY --from=nginx:...` key line.

Then edit `$NEW` by hand:

- `ACME_VERSION` and `NJS_VERSION`: set them to the values from step 1.
- Both `PKGOSSCHECKSUM=` lines: replace them with the new checksum.

Check whether nginx.org still returns 404 for the key:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://nginx.org/keys/nginx_signing.rsa.pub
```

- If it prints 200, you can restore the original `wget` line and drop the `COPY --from`. Keeping the `COPY --from` is also fine.
- Either way, the `KEY_SHA512` check stays.

### 4. Test build

The build context is `Dockerfiles/`.

```bash
cd ..    # now in Dockerfiles/
docker build --platform linux/amd64 --pull -f ${ALPINE}/${NEW} -t ${IMAGE} .
docker run --rm --platform linux/amd64 --entrypoint sh ${IMAGE} -c 'php -v | head -1; nginx -v'
```

Use `--entrypoint sh` for checks like this. Without it, the image entrypoint starts supervisord and ignores the command.

### 5. Open a PR on the fork, review it, merge it

```bash
cd ..
git add Dockerfiles/${ALPINE}/${NEW}
git commit -m "Add PHP ${PHP_NEW} + nginx ${NGINX_NEW} on ${ALPINE}"
git push -u origin HEAD
gh pr create --repo fivejars/nginx-php-fpm --base master --fill
```

Pass `--repo fivejars/nginx-php-fpm` explicitly. Without it, `gh` may target dwchiang's upstream repo.

### 6. After the merge, build from master and push

```bash
git switch master && git pull
REV=$(git rev-parse HEAD)
cd Dockerfiles
docker build --platform linux/amd64 --pull -f ${ALPINE}/${NEW} \
  --label org.opencontainers.image.source=https://github.com/fivejars/nginx-php-fpm \
  --label org.opencontainers.image.revision=${REV} \
  -t ${IMAGE} .

gh auth status | grep scopes        # must include write:packages
# if it doesn't:  gh auth refresh -h github.com -s write:packages
gh auth token | docker login ghcr.io -u "$(gh api user --jq .login)" --password-stdin
docker push ${IMAGE}
docker logout ghcr.io
```

New tags go into the existing package, which is already public, so there's nothing to change in its settings.

### 7. Check that it can be pulled without login

```bash
docker rmi ${IMAGE} && docker pull --platform linux/amd64 ${IMAGE}
```

### 8. Update the consuming projects

In each project that builds on this image:

```bash
sed -i '' "s#^FROM ghcr.io/fivejars/nginx-php-fpm:.*#FROM ${IMAGE}#" Dockerfile
docker build --platform linux/amd64 -t project-test:local .   # optional full check
docker rmi project-test:local
```

Then open a PR in that project with its usual commit and PR rules.
