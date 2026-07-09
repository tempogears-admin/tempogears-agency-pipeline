# Attractive & Charming Agency — TikTok Scheduling Pipeline

This repo manages TikTok post schedules for all agency clients using [TempoGears](https://tempogears.com). When a `posts.csv` file is updated and pushed, GitHub Actions automatically schedules the new posts through TempoGears.

---

## How it works

```
Client uploads video → S3 bucket
Agency updates posts.csv → git push
GitHub Actions detects change → downloads videos from S3
tg bulk-schedule runs → TempoGears queues posts
TempoGears posts to TikTok at scheduled time ✓
```

### The auth picture

There are two separate auth layers — you only set them up once:

**1. TikTok account auth (one-time per client account, done on your laptop)**

Before the pipeline can post on behalf of a client, you need to connect their TikTok account to TempoGears:

```bash
npm install -g tiktok-tg
tg auth api-key <your-tempogears-api-key>
tg auth login   # opens browser — have the client approve, or use the agency test account
```

This stores an encrypted OAuth token in the TempoGears backend. It auto-refreshes — you never need to do it again unless the client revokes access.

**2. Pipeline auth (stored in GitHub Secrets, used in CI)**

The pipeline authenticates to TempoGears using just the API key. It never talks to TikTok directly — TempoGears handles that using the stored tokens from step 1.

---

## Client upload portal

Instead of clients uploading videos directly to S3, you can send them a TempoGears upload link. They drop their video in a browser, and a GitHub PR is automatically opened for your review — no S3 credentials, no GitHub access needed on their end.

### How it works

```
Agency generates link → sends to client
Client visits tempogears.com/upload?... → drops MP4
Video uploads directly to S3 (no size limit, progress bar)
TempoGears opens a PR in this repo with the new CSV row
Client lands on a status page showing their submission
Agency reviews PR → edits caption/time if needed → merges
GitHub Actions runs → post is scheduled ✓
```

### Generating an upload link (Live Demo!)

Build the URL with all four parameters pre-filled — the client only needs to drop their video:

```
https://tempogears.com/upload?client=nova-fitness&account=@novafitness&caption=Monday+motivation+💪+%23fitness&time=2026-07-14+07:00+UTC
```

| Parameter | Description | Example |
|-----------|-------------|---------|
| `client` | Matches the folder name under `clients/` | `nova-fitness` |
| `account` | TikTok handle to post to | `@novafitness` |
| `caption` | Caption and hashtags (URL-encode spaces as `+`) | `Monday+motivation+💪` |
| `time` | Scheduled time in UTC | `2026-07-14+07:00+UTC` |

> **Tip:** Build these links in a spreadsheet — one row per planned post, one column per parameter, one formula to generate the full URL. Share each row's link with the relevant creator when you're ready for their video.

🔧🔧🔧Visit the pre-filled url above to demo a new video to be scheduled!! 🔧🔧🔧

### What the client sees

1. **Upload page** — shows the scheduled account, caption, and time locked in. They just drop the MP4 and hit Submit.
2. **Status page** — after upload they land on `tempogears.com/upload-status` showing a timeline: uploaded → review sent → pending agency approval → goes live.

### What the agency sees

A pull request in this repo, one per submission:

```
PR: "New post for nova-fitness — @novafitness at 2026-07-14 07:00 UTC"

Changes:
  clients/nova-fitness/posts.csv  (+1 row)
```

Review the caption and time in the diff, edit if needed, then merge. The pipeline runs automatically on merge.

### CSV format with upload portal

The portal adds a `video_url` column (a 7-day pre-signed download URL). The pipeline handles this automatically — no S3 credentials needed in GitHub Secrets for portal-submitted videos.

```csv
account,video_file,video_url,caption,time
@novafitness,videos/nova-workout.mp4,"https://s3.amazonaws.com/...","Monday motivation 💪",2026-07-14 07:00 UTC
```

---

## Directory structure

```
clients/
  <client-name>/
    posts.csv          ← edit this to schedule posts
    videos/            ← created at runtime by pipeline (downloaded from S3)
.github/
  workflows/
    schedule.yml       ← the pipeline
```

---

## posts.csv format

| Column | Description | Example |
|--------|-------------|---------|
| `account` | TikTok handle (must be connected via `tg auth login`) | `@luxebrand` |
| `video_file` | Path to video relative to the client folder | `videos/promo.mp4` |
| `caption` | Post caption including hashtags | `New drop 🔥 #fashion` |
| `time` | Scheduled time in UTC | `2026-07-14 10:00 UTC` |

**Example:**
```csv
account,video_file,caption,time
@luxebrand,videos/summer-drop.mp4,"Summer collection just dropped 🔥 #luxury",2026-07-14 10:00 UTC
@luxebrand_vip,videos/summer-drop.mp4,"VIP exclusive preview 👑",2026-07-14 09:00 UTC
```

---

## Client video uploads

Clients upload videos to the agency S3 bucket under their client folder:

```
s3://<S3_VIDEOS_BUCKET>/
  luxe-brand/videos/luxe-summer-drop.mp4
  nova-fitness/videos/nova-workout-monday.mp4
  bloom-beauty/videos/bloom-skincare-routine.mp4
```

**To give a client upload access**, create an IAM user with this policy (replace `<bucket>` and `<client-name>`):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:PutObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::<bucket>/<client-name>/*",
      "arn:aws:s3:::<bucket>"
    ]
  }]
}
```

Give the client their `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` and have them use the AWS CLI or S3 console to upload.

---

## GitHub Secrets

Go to **Settings → Secrets and variables → Actions → New repository secret** and add:

| Secret | What it is | Where to get it |
|--------|-----------|-----------------|
| `TG_API_KEY` | Your TempoGears API key | Emailed to you when you signed up at tempogears.com/pricing. Run `tg auth api-key` to see it. |
| `AWS_ACCESS_KEY_ID` | AWS access key for reading from S3 | AWS Console → IAM → Users → Create user → Attach `AmazonS3ReadOnlyAccess` |
| `AWS_SECRET_ACCESS_KEY` | AWS secret for the above key | Shown once when you create the IAM user |
| `S3_VIDEOS_BUCKET` | Name of your S3 bucket for client videos | The bucket you create for client uploads (e.g. `aca-client-videos`) |

---

## Triggering the pipeline

The pipeline runs automatically whenever a `posts.csv` file is pushed:

```bash
# Edit a client's schedule
nano clients/nova-fitness/posts.csv

# Push — pipeline starts automatically
git add clients/nova-fitness/posts.csv
git commit -m "Schedule Nova Fitness week of Jul 14"
git push
```

Only the clients whose CSVs changed are processed — not all clients on every run.

---

## Adding a new client

1. Connect their TikTok account on your laptop:
   ```bash
   tg auth login   # have the client approve in the browser
   ```

2. Create their folder:
   ```bash
   mkdir clients/<client-name>
   touch clients/<client-name>/posts.csv
   ```

3. Add the CSV header:
   ```
   account,video_file,caption,time
   ```

4. Have them upload videos to `s3://<S3_VIDEOS_BUCKET>/<client-name>/videos/`

5. Fill in `posts.csv` and push.
