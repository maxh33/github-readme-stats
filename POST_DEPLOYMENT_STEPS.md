# Post-Deployment Steps

## After Successful Deployment

Once your github-readme-stats service is deployed and running at `https://stats.maxhaider.dev`, follow these steps:

### 1. Verify Deployment is Working

Test your endpoints before updating your profile:

```bash
# Test stats card
curl "https://stats.maxhaider.dev/api/?username=maxh33&count_private=true&show_icons=true&theme=apprentice&show=prs_merged,prs_merged_percentage"

# Test top languages card
curl "https://stats.maxhaider.dev/api/top-langs/?username=maxh33&size_weight=1&count_weight=0&theme=apprentice&langs_count=7&hide=html,CSS,scss&layout=donut"
```

Both should return SVG content (not errors).

### 2. Update Your GitHub Profile README

**Repository:** `maxh33/maxh33` (special profile repository)

**Current code (using Vercel):**
```html
<a href="https://github.com/maxh33">
    <img height="228" src="https://github-readme-stats.vercel.app/api?username=maxh33&count_private=true&show_icons=true&theme=apprentice&show=prs_merged,prs_merged_percentage"/>
    <img height="344" src="https://github-readme-stats.vercel.app/api/top-langs/?username=maxh33&size_weight=1&count_weight=0&theme=apprentice&langs_count=7&hide=html,CSS,scss&layout=donut"/>
</a>
```

**New code (using your self-hosted instance):**
```html
<a href="https://github.com/maxh33">
    <img height="228" src="https://stats.maxhaider.dev/api?username=maxh33&count_private=true&show_icons=true&theme=apprentice&show=prs_merged,prs_merged_percentage"/>
    <img height="344" src="https://stats.maxhaider.dev/api/top-langs/?username=maxh33&size_weight=1&count_weight=0&theme=apprentice&langs_count=7&hide=html,CSS,scss&layout=donut"/>
</a>
```

**What changed:**
- `github-readme-stats.vercel.app` → `stats.maxhaider.dev`

### 3. Test Your Profile

After updating:
1. Go to `https://github.com/maxh33`
2. Verify both cards are displaying correctly
3. Check that stats are loading (may take a few seconds first time)
4. Verify the theme and styling match your preferences

### 4. Monitor Performance

After going live, monitor your self-hosted instance:

```bash
# SSH to VPS
ssh your-user@your-vps-ip

# Check container logs
docker logs -f github-readme-stats

# Watch for API requests
docker logs github-readme-stats | grep "GET /api"

# Check rate limit usage
docker logs github-readme-stats | grep -i "rate limit"
```

### 5. Set Up Caching (Optional but Recommended)

GitHub profiles are viewed frequently. Consider adding caching headers to reduce load:

**Add to express.js** (if not already present):
```javascript
// Add cache headers for stats responses
app.use((req, res, next) => {
  if (req.path.startsWith('/api/')) {
    res.set('Cache-Control', 'public, max-age=1800'); // 30 minutes
  }
  next();
});
```

This reduces API calls to GitHub by caching responses for 30 minutes.

## Comparison: Vercel vs Self-Hosted

| Feature | Vercel (Free) | Self-Hosted |
|---------|---------------|-------------|
| **Cost** | Free | ~$5-10/month (VPS) |
| **Rate Limits** | Shared (often hits limits) | Your own tokens (20k/hour with 4 PATs) |
| **Uptime** | ~99.9% | Depends on your VPS |
| **Customization** | Limited | Full control |
| **Latency** | Global CDN | Single location (your VPS) |
| **Privacy** | Third-party service | Your infrastructure |

## Benefits of Self-Hosting

1. **No Rate Limiting Issues:** Vercel's free tier shares rate limits across all users
2. **Full Control:** Customize themes, add features, modify behavior
3. **Privacy:** Your data stays on your infrastructure
4. **Learning:** Understand the full stack deployment
5. **Portfolio Project:** Demonstrates DevOps/deployment skills

## Rollback Plan

If you need to revert to Vercel:

1. Simply change URLs back in your profile README:
   - `stats.maxhaider.dev` → `github-readme-stats.vercel.app`

2. Keep your self-hosted instance running for testing/development

## Troubleshooting

### Cards Not Loading
**Symptoms:** Broken image icons in profile

**Check:**
```bash
# Test direct access
curl -I https://stats.maxhaider.dev/api/?username=maxh33

# Should return 200 OK with Content-Type: image/svg+xml
```

**Common causes:**
- Service not running: `docker ps --filter "name=github-readme-stats"`
- DNS not propagated yet: `dig stats.maxhaider.dev`
- SSL certificate issue: `curl -v https://stats.maxhaider.dev`
- Firewall blocking: Check VPS firewall rules

### Stats Showing Errors
**Symptoms:** SVG displays but shows error message

**Check:**
```bash
# View recent errors
docker logs github-readme-stats --tail 50 | grep -i error

# Common errors:
# - "Bad credentials" → Check PAT tokens in .env
# - "Rate limit exceeded" → Add more PAT tokens
# - "User not found" → Check username parameter
```

### Slow Loading
**Symptoms:** Cards take >3 seconds to load

**Solutions:**
1. Add caching headers (see section 5 above)
2. Add more PAT tokens for load balancing
3. Consider adding Redis for response caching
4. Use a CDN like Cloudflare (optional)

## Advanced: Adding Custom Themes

Your self-hosted instance allows custom theme creation:

1. Edit `src/themes/index.js`
2. Add your theme:
```javascript
export const themes = {
  // ... existing themes
  mycustomtheme: {
    title_color: "58a6ff",
    icon_color: "1f6feb",
    text_color: "c9d1d9",
    bg_color: "0d1117",
    border_color: "30363d",
  },
};
```
3. Commit and push (triggers auto-deploy)
4. Use: `?theme=mycustomtheme`

## Monitoring Checklist

Set up these monitors (optional):

- [ ] Uptime monitoring (UptimeRobot, Pingdom, etc.)
- [ ] SSL certificate expiration alerts (Traefik handles renewal)
- [ ] Container health checks (already configured)
- [ ] Disk space monitoring
- [ ] Rate limit usage tracking
- [ ] Error rate alerts

## Final Checklist

Before updating your profile:

- [ ] Deployment successful (GitHub Actions green)
- [ ] Container running on VPS (`docker ps`)
- [ ] DNS resolving correctly (`dig stats.maxhaider.dev`)
- [ ] HTTPS working (`curl https://stats.maxhaider.dev`)
- [ ] Stats card endpoint works (test URL)
- [ ] Top languages endpoint works (test URL)
- [ ] No errors in logs (`docker logs github-readme-stats`)
- [ ] Response time acceptable (<2 seconds)

## Support

If you encounter issues:
1. Check [DOCKER_SETUP.md](DOCKER_SETUP.md) for Docker issues
2. Check [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md) for deployment issues
3. Review GitHub Actions logs
4. Check container logs: `docker logs github-readme-stats`
5. Test endpoints manually with curl

## Success Criteria

✅ Profile loads within 2 seconds
✅ Both stats cards display correctly
✅ Stats update when refreshed
✅ No error messages in cards
✅ Container stays running (no restarts)
✅ Logs show successful API requests
✅ SSL certificate valid (green padlock)

---

**Ready to switch?** Once all checks pass, update your profile README and enjoy your self-hosted stats! 🚀
