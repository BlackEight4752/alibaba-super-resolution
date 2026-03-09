---
name: alibaba-super-resolution
description: "Enhance video resolution using Alibaba Cloud Super Resolution API. Use when the user wants to: (1) upscale low-res videos to higher resolution, (2) improve video quality before publishing, or (3) convert 480p videos to 1080p."
version: 1.0.0
category: media-processing
argument-hint: "[input video] [output video]"
---

# Alibaba Cloud Super Resolution (阿里云视频超分辨率)

Enhance video resolution using Alibaba Cloud's video super resolution API, converting low-resolution videos to higher resolution (e.g., 480p → 1080p).

## Prerequisites

Set the following environment variables for authentication:

```bash
export ALIBABA_CLOUD_ACCESS_KEY_ID="your-access-key-id"
export ALIBABA_CLOUD_ACCESS_KEY_SECRET="your-access-key-secret"
```

### Optional OSS Configuration (for large files)

For files larger than 2GB or when using OSS directly:

```bash
export ALIYUN_OSS_BUCKET="your-bucket-name"
export ALIYUN_OSS_ENDPOINT="oss-cn-shanghai.aliyuncs.com"
```

## Execution (Python CLI Tool)

A Python CLI tool is provided at `~/.openclaw/workspace/skills/alibaba-super-resolution/alibaba_super_resolve.py`.

### Quick Examples

```bash
# Basic usage: local file → local HD file
python3 ~/.openclaw/workspace/skills/alibaba-super-resolution/alibaba_super_resolve.py \
  --input videos/input-480p.mp4 \
  --output videos/output-1080p.mp4

# Custom bit rate (higher = better quality, larger file)
python3 ~/.openclaw/workspace/skills/alibaba-super-resolution/alibaba_super_resolve.py \
  --input videos/input-480p.mp4 \
  --output videos/output-1080p.mp4 \
  --bit-rate 8

# Do not wait for completion (return job ID immediately)
python3 ~/.openclaw/workspace/skills/alibaba-super-resolution/alibaba_super_resolve.py \
  --input videos/input-480p.mp4 \
  --no-wait

# Check status of an existing job
python3 ~/.openclaw/workspace/skills/alibaba-super-resolution/alibaba_super_resolve.py \
  --status <JOB_ID>

# Wait for an existing job and download result
python3 ~/.openclaw/workspace/skills/alibaba-super-resolution/alibaba_super_resolve.py \
  --wait <JOB_ID> \
  --output videos/output-1080p.mp4
```

## Workflow Integration

### Full Video Enhancement Workflow

```bash
cd /vol2/1000/nvme/OpenClaw/double-dog-radio

# 1. Generate low-res video (480p) using Seedance
python3 scripts/generate-video.py batch-001

# 2. Merge video segments
python3 scripts/merge-videos-with-cover.py \
  storage/videos/segment1.mp4 \
  storage/videos/segment2.mp4 \
  storage/videos/segment3.mp4 \
  -c videos/batch-001/cover.png \
  -o videos/batch-001/video-raw.mp4

# 3. Super resolution upscale (480p → 1080p)
python3 ~/.openclaw/workspace/skills/alibaba-super-resolution/alibaba_super_resolve.py \
  --input videos/batch-001/video-raw.mp4 \
  --output videos/batch-001/video-hd.mp4 \
  --bit-rate 8

# 4. Add subtitles
ffmpeg -i videos/batch-001/video-hd.mp4 \
  -vf "subtitles=storage/subtitles/final.srt:force_style='FontName=Arial,FontSize=16,PrimaryColour=&HFFFFFF,OutlineColour=&H000000,Outline=2'" \
  -c:a copy videos/batch-001/video-final.mp4
```

## Input Requirements

### Video Files

- **Formats**: MP4, MOV, AVI, MKV
- **Max Size**: 2GB (direct upload) / No limit (OSS URL)
- **Max Duration**: 30 minutes
- **Input Resolutions**: 480p, 720p

### Output Resolutions

| Input | Output |
|-------|--------|
| 480p | 1080p (4x upscale) |
| 720p | 2K / 4K (if supported) |

## Bit Rate Settings

| Bit Rate | Quality | File Size | Processing Time | Use Case |
|----------|---------|-----------|-----------------|----------|
| 1-3 | Low | Small | Fast | Preview/Testing |
| 4-6 | Medium | Medium | Medium | Social Media |
| 7-10 | High | Large | Slow | HD Publishing |

## Cost Estimation

| Service | Cost | Note |
|---------|------|------|
| Super Resolution | ~¥0.2/minute | 480p → 1080p |
| OSS Storage | ~¥0.12/GB/month | Standard storage |
| OSS Traffic | ¥0.5/GB | Internet egress |

**Example**: 30 second video 480p→1080p ≈ ¥0.1

## API Reference

### Submit Task (Local File Upload)

```python
from alibabacloud_videoenhan20200320.client import Client
from alibabacloud_videoenhan20200320.models import SuperResolveVideoAdvanceRequest

config = Config(
    access_key_id="YOUR_ACCESS_KEY",
    access_key_secret="YOUR_ACCESS_SECRET",
    endpoint="videoenhan.cn-shanghai.aliyuncs.com",
    region_id="cn-shanghai"
)

client = Client(config)
request = SuperResolveVideoAdvanceRequest()
request.video_url_object = open('input.mp4', 'rb')
request.bit_rate = 5

response = client.super_resolve_video_advance(request)
job_id = response.body.request_id
```

### Query Task Status

```python
from alibabacloud_videoenhan20200320.models import GetAsyncJobResultRequest

request = GetAsyncJobResultRequest()
request.job_id = job_id

response = client.get_async_job_result(request)
status = response.body.data.status
output_url = response.body.data.output_url
```

## Rules

1. **Always check** that credentials are configured before making API calls.
2. **Files over 2GB** must use OSS URL instead of direct upload.
3. **Default bit rate**: 5 (balanced quality/size).
4. **Poll interval**: 5 seconds between status checks.
5. **Default timeout**: 1200 seconds (20 minutes).
6. **Download immediately** - output URLs expire after 24 hours.
7. **Handle errors gracefully** - display clear error messages.

## Troubleshooting

### InvalidOSSUrl

**Error**: `InvalidOSSUrl format`

**Fix**: Check OSS URL format: `oss://bucket/path/to/file.mp4`

### OSSAccessDenied

**Error**: `OSSAccessDenied`

**Fix**: Verify RAM permissions for the access key, ensure it has OSS read/write access.

### VideoTooLarge

**Error**: `File exceeds 2GB limit`

**Fix**: 
- Compress video before uploading
- Use OSS URL instead of direct upload
- Split video into segments

### Timeout

**Error**: `Task timed out`

**Fix**: Increase timeout parameter: `--timeout 1800` (30 minutes)

## Performance Tips

1. **Batch processing**: Process multiple videos in parallel to save time
2. **Test with short clips**: Verify settings before processing long videos
3. **Use draft mode**: Generate low-quality preview first to check quality
4. **Optimize bit rate**: Use lower bit rate for social media, higher for high-quality publishing

## Related Documents

- [SUPER-RESOLVE-GUIDE.md](../../double-dog-radio/SUPER-RESOLVE-GUIDE.md) - Full workflow guide
- [Alibaba Cloud Super Resolution Docs](https://help.aliyun.com/document_detail/378659.html) - Official documentation
- [OSS Configuration Guide](https://help.aliyun.com/document_detail/31883.html) - OSS setup instructions
