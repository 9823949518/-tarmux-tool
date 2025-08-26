# -tarmux-tool
#!/usr/bin/env python3
"""
GPT-5 Master FULL — Termux-ready enhanced edition
- Banner generator (Pillow) with text-wrapping & font fallback
- Image->clip via ffmpeg, concat, optional compression
- Music overlay with safe existence checks & fallback beep
- Robust TTS via espeak (with print fallback)
- Interactive CLI: start, chat, app, serve, status, clean, compress, exit
- App skeleton generator
- Background health worker + auto-update stub (safe; no auto-install)
- HTTP server with index for multi-device access
- Simulated touch/animation for phone-like feel
- Extensive error handling & logging
"""
import os, sys, time, threading, subprocess, random, datetime, shutil, logging, textwrap, socket, json
from http.server import SimpleHTTPRequestHandler, HTTPServer
from socketserver import ThreadingMixIn
from PIL import Image, ImageDraw, ImageFont

# ---------- Configuration ----------
HOME = os.path.expanduser("~")
GPT5 = os.path.join(HOME, "GPT5")
BANNERS = os.path.join(GPT5, "Banners")
MUSIC = os.path.join(GPT5, "Music")
FINAL = os.path.join(GPT5, "Final")
APPHOOK = os.path.join(GPT5, "Apps")
TEMP = os.path.join(GPT5, "Temp")
LOGFILE = os.path.join(GPT5, "gpt5_master_full.log")
FONT_CANDIDATES = [
    "/system/fonts/Roboto-Regular.ttf",
    "/system/fonts/NotoSans-Regular.ttf",
    "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf"
]
# optional: set RAW_UPDATE_URL environment variable to enable safe auto-update check (no auto-run)
AUTO_UPDATE_URL = os.environ.get("GPT5_AUTO_UPDATE_URL", "").strip()

for d in (GPT5, BANNERS, MUSIC, FINAL, APPHOOK, TEMP):
    os.makedirs(d, exist_ok=True)

logging.basicConfig(filename=LOGFILE, level=logging.INFO,
                    format="%(asctime)s %(levelname)s: %(message)s")

# ---------- Utilities ----------
def safe_run(cmd, timeout=120):
    """Run subprocess command safely, return (rc, stdout, stderr)"""
    try:
        proc = subprocess.run(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True, timeout=timeout)
        return proc.returncode, proc.stdout.strip(), proc.stderr.strip()
    except subprocess.TimeoutExpired as e:
        logging.exception("safe_run timeout")
        return 124, "", f"Timeout: {e}"
    except Exception as e:
        logging.exception("safe_run failed")
        return 1, "", str(e)

def speak(text):
    """espeak TTS fallback to print"""
    try:
        t = text.replace('"', "'")
        # launch non-blocking so CLI remains responsive
        subprocess.Popen(["espeak", "-s", "150", t])
    except Exception:
        print("[TTS]", text)

def timestamp():
    return datetime.datetime.now().strftime("%Y%m%d_%H%M%S")

def get_local_ip():
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        s.connect(("8.8.8.8", 80))
        ip = s.getsockname()[0]
        s.close()
        return ip
    except Exception:
        return "127.0.0.1"

# ---------- Font helper ----------
def pick_font(size=36):
    for p in FONT_CANDIDATES:
        if os.path.exists(p):
            try:
                return ImageFont.truetype(p, size)
            except Exception:
                continue
    return ImageFont.load_default()

# ---------- Banner (with wrapping & contrast) ----------
def contrast_color(bg):
    # determine readable foreground (white or black)
    r,g,b = bg
    luminance = (0.299*r + 0.587*g + 0.114*b)
    return (0,0,0) if luminance > 186 else (255,255,255)

def safe_wrap(text, font, max_width, draw):
    lines = []
    for paragraph in text.split("\n"):
        wrapped = textwrap.wrap(paragraph, width=40)
        # refine wrap by measuring pixel width
        for w in wrapped:
            # if too long, split roughly
            while draw.textsize(w, font=font)[0] > max_width:
                # cut by characters
                w = w[:-1]
            lines.append(w)
    return lines

def banner(text, width=1050, height=300, emoji=None, out_folder=BANNERS):
    try:
        os.makedirs(out_folder, exist_ok=True)
        filename = os.path.join(out_folder, f"banner_{timestamp()}_{random.randint(1000,9999)}.png")
        bg = (random.randint(20,230), random.randint(20,230), random.randint(20,230))
        img = Image.new("RGB", (width, height), bg)
        draw = ImageDraw.Draw(img)
        font = pick_font(36)
        fg = contrast_color(bg)
        if not emoji:
            emoji = random.choice(['🚀','🌟','🎵','💻','🔥','🎯','✨','🎬','📱'])
        # wrap text nicely
        text_full = f"{text} {emoji}"
        lines = safe_wrap(text_full, font, width-40, draw)
        y = 30
        for line in lines:
            w,h = draw.textsize(line, font=font)
            draw.text(((width-w)//2, y), line, fill=fg, font=font)
            y += h + 8
        img.save(filename)
        logging.info(f"Banner created: {filename}")
        return filename
    except Exception:
        logging.exception("banner creation failed")
        return None

# ---------- FFmpeg helpers with retries ----------
def image_to_clip(image_path, out_path, duration=2, fps=24, max_retries=2):
    for attempt in range(1, max_retries+1):
        try:
            cmd = [
                "ffmpeg", "-y", "-loop", "1", "-i", image_path,
                "-c:v", "libx264", "-t", str(duration), "-pix_fmt", "yuv420p",
                "-vf", f"scale=trunc(iw/2)*2:trunc(ih/2)*2", "-r", str(fps),
                out_path
            ]
            rc, out, err = safe_run(cmd)
            if rc == 0:
                logging.info(f"Created clip: {out_path}")
                return True
            else:
                logging.warning(f"ffmpeg image->clip failed (attempt {attempt}): {err}")
        except Exception:
            logging.exception("image_to_clip exception")
        time.sleep(0.8)
    return False

def concat_clips(clip_paths, output_path):
    try:
        list_file = os.path.join(TEMP, f"clips_{timestamp()}.txt")
        with open(list_file, "w") as f:
            for p in clip_paths:
                f.write(f"file '{p}'\n")
        cmd = ["ffmpeg", "-y", "-f", "concat", "-safe", "0", "-i", list_file, "-c", "copy", output_path]
        rc, out, err = safe_run(cmd)
        if rc == 0:
            logging.info(f"Concatenated final: {output_path}")
            return True
        else:
            logging.error(f"ffmpeg concat failed: {err}")
            return False
    except Exception:
        logging.exception("concat_clips exception")
        return False

def overlay_music(video_in, music_path, output_path):
    try:
        cmd = ["ffmpeg", "-y", "-i", video_in, "-i", music_path, "-c:v", "copy", "-c:a", "aac", "-shortest", output_path]
        rc, out, err = safe_run(cmd)
        if rc == 0:
            logging.info(f"Overlayed music: {output_path}")
            return True
        else:
            logging.error(f"overlay_music failed: {err}")
            return False
    except Exception:
        logging.exception("overlay_music exception")
        return False

def compress_video(input_path, output_path, crf=28):
    try:
        cmd = ["ffmpeg", "-y", "-i", input_path, "-vcodec", "libx264", "-crf", str(crf), output_path]
        rc, out, err = safe_run(cmd, timeout=600)
        if rc == 0:
            logging.info(f"Compressed video: {output_path}")
            return True
        else:
            logging.error(f"compress_video failed: {err}")
            return False
    except Exception:
        logging.exception("compress_video exception")
        return False

# ---------- Feature Engine ----------
DEFAULT_FEATURES = {
    "Power / Capability": 100,
    "Multi-device Integration": 100,
    "Cyber-security Features": 100,
    "Multi-language AI Support": 100,
    "Auto-update & Self-healing": 100,
    "Error Handling / Skipper": 100,
    "Emoji / Banner / UI Effects": 100,
    "Music & Audio Feedback": 100
}

def pick_random_music():
    try:
        files = [os.path.join(MUSIC, f) for f in os.listdir(MUSIC) if f.lower().endswith(('.mp3','.ogg','.wav'))]
        files = [f for f in files if os.path.isfile(f)]
        return random.choice(files) if files else None
    except Exception:
        return None

def fallback_beep(out_path, duration=1):
    """
    Create a tiny silent audio fallback or use ffmpeg to generate a tone if available.
    If ffmpeg cannot generate tone, return None.
    """
    try:
        tone = os.path.join(TEMP, f"tone_{timestamp()}.wav")
        # generate 440Hz sine tone if ffmpeg supports 'sine' source
        rc, o, e = safe_run(["ffmpeg", "-filters"])
        if rc == 0 and "sine" in o:
            cmd = ["ffmpeg", "-y", "-f", "lavfi", "-i", f"sine=frequency=440:duration={duration}", tone]
            rc2, _, err = safe_run(cmd)
            if rc2 == 0:
                return tone
        return None
    except Exception:
        logging.exception("fallback_beep failed")
        return None

def generate_demo(features=None, per_clip_duration=2, add_music=True, compress=False):
    features = features or DEFAULT_FEATURES
    speak("Starting demo generation. Please wait.")
    clip_files = []
    try:
        total = len(features)
        idx = 0
        for name, val in features.items():
            idx += 1
            print(f"[{idx}/{total}] Creating banner for: {name}")
            text = f"{name}: {val}/100"
            img = banner(text)
            if not img:
                logging.warning("Banner not created, skipping feature: " + name)
                continue
            clip = os.path.join(FINAL, f"clip_{timestamp()}_{random.randint(1000,9999)}.mp4")
            ok = image_to_clip(img, clip, duration=per_clip_duration)
            if ok:
                clip_files.append(clip)
            else:
                logging.warning("Clip not created for image: " + img)
        if not clip_files:
            speak("No clips created.")
            return None
        final_video = os.path.join(FINAL, f"GPT5_Feature_Demo_{timestamp()}.mp4")
        ok = concat_clips(clip_files, final_video)
        if not ok:
            speak("Failed to concatenate clips.")
            return None
        if add_music:
            mus = pick_random_music()
            if mus:
                outm = os.path.join(FINAL, f"GPT5_Feature_Demo_MUSIC_{timestamp()}.mp4")
                ok2 = overlay_music(final_video, mus, outm)
                if ok2:
                    final_video = outm
            else:
                # try to generate a tone fallback
                tone = fallback_beep(os.path.join(TEMP, "tone.wav"))
                if tone:
                    outm = os.path.join(FINAL, f"GPT5_Feature_Demo_MUSIC_{timestamp()}.mp4")
                    if overlay_music(final_video, tone, outm):
                        final_video = outm
        if compress:
            comp = os.path.join(FINAL, f"GPT5_Feature_Demo_COMP_{timestamp()}.mp4")
            if compress_video(final_video, comp, crf=28):
                final_video = comp
        speak("Demo video created.")
        logging.info(f"Demo created: {final_video}")
        return final_video
    except Exception:
        logging.exception("generate_demo failed")
        speak("Demo generation failed due to an error.")
        return None
    finally:
        # basic cleanup of temp clips older than 1 day
        try:
            for f in os.listdir(TEMP):
                p = os.path.join(TEMP, f)
                try:
                    if os.path.isfile(p) and (time.time() - os.path.getmtime(p)) > 24*3600:
                        os.remove(p)
                except Exception:
                    pass
        except Exception:
            pass

# ---------- Chat ----------
def chat_loop():
    speak("Chat system activated. Type 'exit' to return.")
    print("💬 GPT-5 Chat (type 'exit' to go back)")
    try:
        while True:
            u = input("You: ").strip()
            if u.lower() in ("exit","quit"):
                speak("Exiting chat.")
                print("✅ Chat closed.")
                break
            if any(k in u.lower() for k in ("hi","hello","hey")):
                reply = "Hello! How can I help you today?"
            elif "demo" in u.lower():
                reply = "Type 'start' from main menu to generate the demo video."
            elif "app" in u.lower():
                reply = "Use 'app' command in main menu to create an app skeleton."
            elif "status" in u.lower():
                reply = "Type 'status' in main menu to view system status."
            else:
                reply = f"I received: {u}"
            # typing effect
            for c in reply:
                sys.stdout.write(c)
                sys.stdout.flush()
                time.sleep(0.01)
            print()
            speak(reply)
    except Exception:
        logging.exception("chat_loop failed")
        print("Chat error, returning to main menu.")
        speak("Chat error, returning to main menu.")

# ---------- App skeleton ----------
def create_app(app_name, features=None):
    try:
        base = os.path.join(APPHOOK, app_name)
        os.makedirs(base, exist_ok=True)
        os.makedirs(os.path.join(base, "Banners"), exist_ok=True)
        os.makedirs(os.path.join(base, "Music"), exist_ok=True)
        os.makedirs(os.path.join(base, "Assets"), exist_ok=True)
        main_py = os.path.join(base, "main.py")
        with open(main_py, "w") as f:
            f.write("# Auto-generated app skeleton (basic)\n")
            f.write("print('This is a simple app skeleton generated by GPT-5 Master Full')\n")
            f.write("features = " + repr(features or DEFAULT_FEATURES) + "\n")
            f.write("print('Features:', features)\n")
        speak(f"App {app_name} created under GPT5/Apps")
        logging.info(f"App skeleton created: {base}")
        return base
    except Exception:
        logging.exception("create_app failed")
        speak("App creation failed.")
        return None

# ---------- HTTP server (threaded) with index ----------
class ThreadedHTTPServer(ThreadingMixIn, HTTPServer):
    daemon_threads = True

class ServeThread(threading.Thread):
    def __init__(self, directory=FINAL, port=8000):
        super().__init__(daemon=True)
        self.directory = directory
        self.port = port
        self.httpd = None

    def run(self):
        try:
            os.chdir(self.directory)
            Handler = SimpleHTTPRequestHandler
            server = ThreadedHTTPServer(('', self.port), Handler)
            self.httpd = server
            print(f"🔗 Serving {self.directory} at http://{get_local_ip()}:{self.port}")
            logging.info(f"Serving {self.directory} at port {self.port}")
            server.serve_forever()
        except Exception:
            logging.exception("ServeThread failed")
            print("HTTP server failed to start.")

    def stop(self):
        if self.httpd:
            try:
                self.httpd.shutdown()
            except Exception:
                pass

# ---------- Background health worker & auto-update stub ----------
bg_state = {"running": True, "last_demo": None, "music_count": 0}
def background_worker():
    speak("Background worker started.")
    while bg_state["running"]:
        try:
            # music folder monitoring
            cnt = len([f for f in os.listdir(MUSIC) if f.lower().endswith(('.mp3','.ogg','.wav'))])
            if cnt != bg_state["music_count"]:
                bg_state["music_count"] = cnt
                logging.info(f"Music folder changed. files={cnt}")
            # ffmpeg health
            rc, out, err = safe_run(["ffmpeg", "-version"])
            if rc != 0:
                logging.warning("ffmpeg not available or failing.")
            # auto-update check (safe: only check if URL provided; do not auto-install)
            if AUTO_UPDATE_URL:
                try:
                    # lightweight HEAD request using curl if available
                    rc2, out2, err2 = safe_run(["curl", "-Is", AUTO_UPDATE_URL], timeout=10)
                    if rc2 == 0 and "200" in out2.splitlines()[0]:
                        logging.info("Auto-update URL reachable (stub).")
                except Exception:
                    pass
            time.sleep(5)
        except Exception:
            logging.exception("background_worker exception")
            time.sleep(5)

# ---------- Simulated touch animation ----------
def simulate_touch(duration=3):
    frames = ["(•_•)","( •_•)>⌐■-■","(⌐■_■)","(•_•)"]
    end = time.time() + duration
    while time.time() < end:
        for f in frames:
            print("\r" + f + "  Touch simulation...", end="", flush=True)
            time.sleep(0.25)
    print("\rTouch simulation ended.           ")

# ---------- Status & cleanup ----------
def show_status():
    print("System status:")
    print(f" GPT5 dir: {GPT5}")
    print(f" Banners: {BANNERS} (files: {len(os.listdir(BANNERS))})")
    print(f" Music: {MUSIC} (files: {len([f for f in os.listdir(MUSIC) if f.lower().endswith(('.mp3','.ogg','.wav'))])})")
    print(f" Final: {FINAL} (files: {len(os.listdir(FINAL))})")
    print(f" Apps: {APPHOOK} (apps: {len(os.listdir(APPHOOK))})")
    print(f" Temp: {TEMP} (files: {len(os.listdir(TEMP))})")
    print(f" Background worker: {'running' if bg_thread.is_alive() else 'stopped'}")
    print(f" HTTP serve thread: {'running' if serve_thread and serve_thread.is_alive() else 'stopped'}")
    speak("Status displayed.")

def clean_temp(older_than_hours=24):
    now = time.time()
    removed = 0
    for fname in os.listdir(TEMP):
        p = os.path.join(TEMP, fname)
        try:
            if os.path.isfile(p) and (now - os.path.getmtime(p)) > older_than_hours*3600:
                os.remove(p)
                removed += 1
        except Exception:
            pass
    print(f"Cleaned {removed} temp files.")

# ---------- Main CLI ----------
serve_thread = None
bg_thread = threading.Thread(target=background_worker, daemon=True)
bg_thread.start()

def print_help():
    print("""Available commands:
  start        → generate demo video now (asks default/custom)
  chat         → open chat system
  app          → create app skeleton (app name prompt)
  serve        → start/stop HTTP server for Final folder (multi-device)
  status       → show system status
  clean        → clean old temp files
  compress     → compress last demo to smaller file (CRF prompt)
  touchsim     → run short touch animation simulation
  help         → this help
  exit         → quit
""")

def main_loop():
    global serve_thread
    print("🎬 Welcome to GPT-5 Master Full (Termux edition)")
    print_help()
    while True:
        try:
            cmd = input("\nEnter command (start/help/exit): ").strip().lower()
            if cmd == "exit":
                speak("Exiting system, goodbye.")
                print("Exiting...")
                if serve_thread and serve_thread.is_alive():
                    serve_thread.stop()
                bg_state["running"] = False
                break
            elif cmd == "help":
                print_help()
            elif cmd == "start":
                use_custom = input("Use default features? (y/n): ").strip().lower()
                if use_custom == "n":
                    feats = {}
                    while True:
                        n = input("Feature name (blank to stop): ").strip()
                        if not n:
                            break
                        v = input("Feature value (0-100): ").strip()
                        try:
                            v = int(v)
                        except:
                            v = 100
                        feats[n] = v
                    features = feats if feats else DEFAULT_FEATURES
                else:
                    features = DEFAULT_FEATURES
                per_dur = input("Per-clip duration seconds (default 2): ").strip()
                try:
                    per_dur = float(per_dur) if          per_dur else 2.0    
                
