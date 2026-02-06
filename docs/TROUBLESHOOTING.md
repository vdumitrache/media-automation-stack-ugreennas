# Troubleshooting

## SABnzbd: Stuck Unpack Loop

**Symptom:** Radarr shows "Downloading" at 100% with 0 B file size. SABnzbd UI is unresponsive or Save fails. Logs show `Unpacked files []` repeatedly.

**Cause:** NZB had obfuscated filenames + par2 files but no RARs. The unpacker finds nothing to extract and retries on every SABnzbd restart, creating a new `_UNPACK_*` directory each time. Each copy is 20-50+ GB — this can silently eat TBs of disk space. The stuck post-processing loop also locks up SABnzbd's API and UI.

**Diagnose:**
```bash
# _UNPACK_ buildup = stuck unpack loop
ls -d /volume1/Media/downloads/_UNPACK_* | wc -l
du -shc /volume1/Media/downloads/_UNPACK_*

# Confirm in SABnzbd logs
docker logs sabnzbd --tail 200 2>&1 | grep "Unpacked files"
# "Unpacked files []" = nothing to unpack, stuck
```

**Fix:**
```bash
# 1. Stop SABnzbd (API will be unresponsive, must use docker stop)
docker stop sabnzbd

# 2. Delete the postproc queue to clear the stuck job
#    (The history API delete is NOT enough — the postproc queue is separate
#    and will re-trigger the loop on every restart)
sudo rm /volume1/@docker/volumes/arr-stack_sabnzbd-config/_data/admin/postproc2.sab

# 3. Delete all failed _UNPACK_ attempts to reclaim disk space
rm -rf /volume1/Media/downloads/_UNPACK_<release_name>*

# 4. Move the actual file (in incomplete/) to the movie folder
mkdir -p "/volume1/Media/movies/MovieName (Year)"
mv "/volume1/Media/downloads/incomplete/<release>/obfuscated.mkv" \
   "/volume1/Media/movies/MovieName (Year)/MovieName (Year).mkv"
rm -rf "/volume1/Media/downloads/incomplete/<release>"

# 5. Start SABnzbd back up
docker start sabnzbd

# 6. Remove from Radarr queue (get queue ID from queue API)
docker exec radarr curl -s -X DELETE \
  "http://localhost:7878/api/v3/queue/ID?removeFromClient=false&blocklist=false&apikey=KEY"

# 7. Tell Radarr to pick up the file
docker exec radarr curl -s -X POST "http://localhost:7878/api/v3/command" \
  -H "Content-Type: application/json" -H "X-Api-Key: KEY" \
  -d '{"name":"RefreshMovie","movieIds":[MOVIE_ID]}'
```

**Prevention:** No SABnzbd setting fully prevents this. Monitor disk usage (Beszel/duc) and investigate if a movie stays at "Downloading 100%" for more than 30 minutes.
