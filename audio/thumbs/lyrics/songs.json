from fastapi import FastAPI, UploadFile, File, Form, HTTPException, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles
import uuid, json, shutil, os, base64, requests
from pathlib import Path

# ===================== CONFIG =====================

GITHUB_TOKEN = os.getenv("GITHUB_TOKEN")
GITHUB_REPO = os.getenv("GITHUB_REPO")   # e.g. lucaaa620/karaoke-storage

if not GITHUB_TOKEN or not GITHUB_REPO:
    raise Exception("Missing GITHUB_TOKEN or GITHUB_REPO environment variable")

GITHUB_API = f"https://api.github.com/repos/{GITHUB_REPO}"
RAW_BASE = f"https://raw.githubusercontent.com/{GITHUB_REPO}/main"

ADMIN_TOKEN = "myadmin123"

app = FastAPI()

# CORS allow all
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# =====================================================
# Helper: upload file to GitHub
# =====================================================

def upload_to_github(path: str, content: bytes):
    url = f"{GITHUB_API}/contents/{path}"
    encoded = base64.b64encode(content).decode()

    resp = requests.put(
        url,
        headers={
            "Authorization": f"Bearer {GITHUB_TOKEN}",
            "Accept": "application/vnd.github+json",
        },
        json={
            "message": f"Upload {path}",
            "content": encoded
        }
    )
    if resp.status_code not in [200, 201]:
        raise Exception(f"GitHub upload failed: {resp.text}")

# =====================================================
# Helper: Read+Write songs.json remotely
# =====================================================

def get_songs_json():
    url = f"{RAW_BASE}/songs.json"
    return requests.get(url).json()

def update_songs_json(data):
    path = "songs.json"
    url = f"{GITHUB_API}/contents/{path}"

    # get existing SHA (required by GitHub)
    existing = requests.get(url, headers={"Authorization": f"Bearer {GITHUB_TOKEN}"}).json()
    sha = existing.get("sha")

    encoded = base64.b64encode(json.dumps(data, indent=2).encode()).decode()

    resp = requests.put(
        url,
        headers={"Authorization": f"Bearer {GITHUB_TOKEN}"},
        json={
            "message": "Update songs.json",
            "content": encoded,
            "sha": sha
        }
    )

    if resp.status_code not in [200, 201]:
        raise Exception("Failed to update songs.json: " + resp.text)

# =====================================================
# ROUTES
# =====================================================

@app.get("/")
def home():
    return {"status": "Backend running", "github": GITHUB_REPO}

@app.get("/songs")
def get_songs():
    return get_songs_json()


# =====================================================
# 🔥 AUTO UPLOAD SONG (GitHub)
# =====================================================

@app.post("/admin/upload")
async def upload_song(
    token: str = Form(...),
    title: str = Form(...),
    artist: str = Form("Unknown"),
    audio: UploadFile = File(...),
    lyrics: UploadFile = File(None),
    thumb: UploadFile = File(None),
):
    if token != ADMIN_TOKEN:
        raise HTTPException(401, "Invalid token")

    song_id = str(uuid.uuid4())[:8]  # short ID

    # -------- upload audio --------
    audio_bytes = await audio.read()
    audio_ext = Path(audio.filename).suffix
    audio_path = f"audio/{song_id}{audio_ext}"
    upload_to_github(audio_path, audio_bytes)

    # -------- upload thumbnail --------
    thumb_url = ""
    if thumb:
        thumb_bytes = await thumb.read()
        thumb_ext = Path(thumb.filename).suffix
        thumb_path = f"thumbs/{song_id}{thumb_ext}"
        upload_to_github(thumb_path, thumb_bytes)
        thumb_url = f"{RAW_BASE}/{thumb_path}"

    # -------- upload lyrics --------
    lyrics_url = ""
    if lyrics:
        lyrics_bytes = await lyrics.read()
        lyr_path = f"lyrics/{song_id}.lrc"
        upload_to_github(lyr_path, lyrics_bytes)
        lyrics_url = f"{RAW_BASE}/{lyr_path}"

    # -------- update songs.json --------
    db = get_songs_json()

    new_entry = {
        "id": song_id,
        "title": title,
        "artist": artist,
        "audioUrl": f"{RAW_BASE}/{audio_path}",
        "thumbUrl": thumb_url,
        "lyricsLrc": lyrics_url
    }

    db["songs"].append(new_entry)
    update_songs_json(db)

    return {"ok": True, "song": new_entry}
