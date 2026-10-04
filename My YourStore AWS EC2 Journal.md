# My YourStore AWS EC2 Journal

How I took my YourStore app, which already ran on Render, and got it running on my own AWS server. I wrote this as a story, in the order things happened, including the mistakes. Commands are included so I can repeat the whole thing later.

---

## Where I started

My backend was already running on an EC2 instance. My frontend, a Next.js app, was only deployed on Render. I wanted the same frontend running on AWS too, talking to my AWS backend.

The one thing I had to understand first: Next.js takes the `NEXT_PUBLIC_API_URL` value **while it builds** and writes it into the JavaScript files. After that, changing an env variable on the server does nothing for the browser. If the backend address changes, I have to build the app again.

So one build can't serve both Render and AWS. I needed two builds from the same code, each with its own backend address.

---

## Entry 1: Two builds from one workflow

I created two GitHub Environments, `render` and `aws`. In each one I saved the same secret names with different values:

- `NEXT_PUBLIC_API_URL`
- `INTERNAL_API_URL`

The Render deploy hook went into the `render` environment too.

Then I changed my CI/CD to use a **matrix**. GitHub runs the build job twice, once per environment, and each run picks up its own secrets. Each run pushes a Docker image with its own tag. Here is the final workflow:

```yaml
name: frontend ci/cd

on:
  push:
    branches: [staging, main]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - name: get code
        uses: actions/checkout@v4
      - name: setup node
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - name: install dependencies
        run: npm ci
      - name: lint check
        run: npm run lint
      - name: test
        run: npm run test

  build:
    needs: verify
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include:
          - target: render
            tag_suffix: "-render"
          - target: aws
            tag_suffix: "-aws"
    environment: ${{ matrix.target }}
    steps:
      - name: get code
        uses: actions/checkout@v4
      - name: setup node
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - name: install dependencies
        run: npm ci
      - name: cache next.js build
        uses: actions/cache@v4
        with:
          path: .next/cache
          key: nextjs-${{ matrix.target }}-${{ hashFiles('package-lock.json') }}-${{ hashFiles('**/*.js', '**/*.jsx', '**/*.ts', '**/*.tsx') }}
          restore-keys: |
            nextjs-${{ matrix.target }}-${{ hashFiles('package-lock.json') }}-
      - name: build
        env:
          NEXT_PUBLIC_API_URL: ${{ secrets.NEXT_PUBLIC_API_URL }}
          INTERNAL_API_URL: ${{ secrets.INTERNAL_API_URL }}
        run: npm run build
      - name: login to dockerhub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: build docker image and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ak0384221/yourstore-frontend-public:${{ github.ref_name }}${{ matrix.tag_suffix }}

  deploy-to-render:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: render
    steps:
      - name: deploy to render
        run: curl -X POST "${{ secrets.RENDER_DEPLOY_HOOK }}"
```

The tags look like this:

| Branch | Render image | AWS image |
| --- | --- | --- |
| main | `...frontend-public:main-render` | `...frontend-public:main-aws` |
| staging | `...frontend-public:staging-render` | `...frontend-public:staging-aws` |

Two things to remember: Render's image URL has to use the `-render` tag, and the tag name follows the branch that actually ran.

I had also put `NEXT_PUBLIC_API_URL` in as a secret by mistake. It still works. The only effect is that GitHub hides it in logs. The URL ends up in the browser's JavaScript anyway, so it was never really secret.

---

## Entry 2: A fresh EC2 server

I deleted my old instance and started again, so this time I could set it up properly from the first minute.

When launching, I opened **Advanced details → User data** and pasted this. It installs Docker, turns it on, and lets `ec2-user` use it without `sudo`:

```bash
#!/bin/bash
dnf update -y
dnf install -y docker
systemctl enable --now docker
usermod -aG docker ec2-user
```

It runs only once, on first boot. I had to log in again over SSH before the Docker permission worked.

For the security group I opened these ports:

| Port | Who | What for |
| --- | --- | --- |
| 22 | Only my IP | SSH |
| 80 | Anyone | Backend (browsers call it directly) |
| 3000 | Anyone | Frontend |

I also learned that a new instance gets a new public IP, and a stopped-then-started one does too. Since the backend address is baked into the frontend, a changing IP means rebuilding every time. An **Elastic IP** fixes that.

---

## Entry 3: Starting both containers

I created a shared Docker network so the two containers can find each other:

```bash
docker network create yourstore-net
```

**Backend.** I made `~/yourstore-backend.env`. No quotes, no spaces around `=`, and no slash at the end of the URL:

```
FRONTEND_URL=http://<ELASTIC_IP>:3000
# plus my database URL and other settings
```

```bash
chmod 600 ~/yourstore-backend.env
docker run -d --name yourstore-backend --restart unless-stopped \
  --network yourstore-net -p 80:5000 \
  --env-file ~/yourstore-backend.env \
  ak0384221/yourstore-backend-public:main
```

**Frontend.** I made `~/yourstore-frontend.env`:

```
INTERNAL_API_URL=http://<ELASTIC_IP>
```

```bash
docker run -d --name yourstore-frontend --restart unless-stopped \
  --network yourstore-net -p 3000:3000 \
  --env-file ~/yourstore-frontend.env \
  ak0384221/yourstore-frontend-public:staging-aws
```

The GitHub `aws` secrets both had to be `http://<ELASTIC_IP>` with no port, because the backend is published on port 80 (it listens on 5000 inside the container).

---

## Entry 4: Everything that went wrong

This is the useful part. Each problem looked scary and had a simple cause.

**"manifest not found" when pulling the image.** I tried to pull `main-aws`, but my workflow had run on `staging`, so the image was called `staging-aws`. Lesson: look at the Tags tab on Docker Hub and pull a tag that really exists.

**The browser kept calling a wrong IP.** I had mistyped the IP in the secret. I fixed the secret, but the running image still had the old value baked in. Fixing a secret does nothing until I rebuild, pull the new image, and recreate the container.

**The rebuild still had the old value.** My Next.js build cache doesn't change when only a secret changes, so it reused old output. I deleted the `nextjs-aws-...` entries under Actions → Caches and re-ran the workflow. That cleared it.

**"Failed to fetch" with a correct URL.** The request showed "provisional headers" and no response at all. The backend logs told the truth: `Not allowed by CORS: http://<IP>:3000`. The backend did receive the request and refused it, because the `FRONTEND_URL` inside the running container didn't exactly match the address in the browser. The causes can be an old value (a container only reads its env file when it's created, and `docker restart` doesn't re-read it) or a stray slash, quote, or hidden character. I checked with:

```bash
docker exec yourstore-backend env | grep FRONTEND_URL | cat -A
```

The `cat -A` makes hidden characters visible. I fixed the file, removed the container, and ran it again. Then everything worked.

**A separate bug I found on the way.** The logs also showed `ERR_HTTP_HEADERS_SENT` in `getCartItems`. My code sends a response twice somewhere. It isn't what broke the site, but I still need to fix it by adding `return` in front of each `res.json(...)` there.

My debugging order for next time:

1. Browser Network tab: what URL, what status?
2. Backend `docker logs`: did the request arrive?
3. `curl -i -H "Origin: ..."` from my laptop.
4. `docker exec ... env` to see what the container really has.

---

## Cheat sheet: what do I do after a change?

| I changed... | I need to... |
| --- | --- |
| Frontend code, or a build-time secret | Re-run the workflow, then pull, remove and run the frontend container again |
| `INTERNAL_API_URL` in the runtime env file | Recreate the frontend container, no rebuild |
| Anything in the backend env file | Remove and run the backend container again |
| Backend image | Pull, then recreate the container |

Two reminders: `docker restart` doesn't re-read env files, and `docker run` won't download a newer image by itself.

To trigger a rebuild without changing code: `git commit --allow-empty -m "rebuild" && git push`, or Actions → latest run → Re-run all jobs.

---

## Entry 5: Money and housekeeping

- A stopped instance costs no compute. The disk (EBS) and any Elastic IP still cost a little, and a public IPv4 address costs while attached to a running instance.
- I should check Billing → Free Tier and Bills, and set a Budget alert at around $1.
- Stop the instance when I'm not using it. **Never Terminate** unless I'm done for good, because that deletes everything.
- My containers come back by themselves after a restart, thanks to `--restart unless-stopped` and Docker being enabled at boot.

---

## What's left to do

1. Fix the double response bug in `getCartItems` (`controller/productController.js`, around line 105).
2. Make sure SSH (port 22) is only open to my IP.
3. Get a domain and HTTPS with Caddy or Nginx. After that, update `NEXT_PUBLIC_API_URL` and the backend `FRONTEND_URL`, and rebuild the frontend.
4. Maybe add an automatic deploy-to-AWS step so I don't have to pull by hand.
5. Maybe add a hash of the API URL to the cache key so a changed URL can never reuse old cache.

**Final result:** frontend on port 3000, backend on port 80, both running in Docker on one EC2 server, and the cart, login and API calls all working.