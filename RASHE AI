import os
import time
import hmac
import hashlib

import streamlit as st
from google import genai
from google.genai import types
from PIL import Image, ImageOps, ImageDraw

st.set_page_config(page_title="RASHE AI", page_icon="🤖", layout="wide")

SYSTEM_INSTRUCTION = """
You are RASHE AI, an advanced multilingual AI assistant developed by Duresa Shumbura (Rashedin).
You are fluent in Afaan Oromoo, Amharic, and English.
Always detect the user's language and respond in the exact same language (Afaan Oromoo, Amharic, or English).
When processing audio input, transcribe the request accurately and respond in the user's spoken language.
"""

LOCKOUT_THRESHOLD = 5
LOCKOUT_BASE_SECONDS = 30
SESSION_TIMEOUT_SECONDS = 12 * 60 * 60

_FAILED_ATTEMPTS_STORE = st.cache_resource(lambda: {})()


def get_secret(key, default=None):
    try:
        if key in st.secrets:
            return st.secrets[key]
    except Exception:
        pass
    return os.environ.get(key, default)


def get_secret_section(section):
    try:
        if section in st.secrets:
            return dict(st.secrets[section])
    except Exception:
        pass
    return {}


def load_access_config():
    section = get_secret_section("access")
    salt = section.get("salt")
    codes = section.get("codes")
    if salt and codes:
        return salt, dict(codes)
    salt = os.environ.get("ACCESS_SALT")
    raw_hashes = os.environ.get("ACCESS_CODE_HASHES", "")
    codes = {h.strip(): "guest" for h in raw_hashes.split(",") if h.strip()}
    return salt, codes


def hash_code(code, salt):
    return hmac.new(salt.encode("utf-8"), code.encode("utf-8"), hashlib.sha256).hexdigest()


def verify_code(code, salt, codes):
    if not salt or not codes:
        return None
    candidate = hash_code(code, salt)
    matched_label = None
    for stored_hash, label in codes.items():
        if hmac.compare_digest(candidate, stored_hash):
            matched_label = label
    return matched_label


def get_client_key():
    try:
        headers = st.context.headers
        forwarded = headers.get("X-Forwarded-For", "")
        if forwarded:
            return forwarded.split(",")[0].strip()
        remote = headers.get("Remote-Addr")
        if remote:
            return remote
    except Exception:
        pass
    return "global"


@st.cache_resource
def get_client():
    api_key = get_secret("GEMINI_API_KEY") or get_secret("GOOGLE_API_KEY")
    if not api_key:
        raise RuntimeError("missing_api_key")
    os.environ.setdefault("GEMINI_API_KEY", api_key)
    return genai.Client(api_key=api_key)


try:
    client = get_client()
except RuntimeError:
    st.error("GEMINI_API_KEY is not configured. Add it in Streamlit secrets.")
    st.stop()
except Exception as e:
    st.error(f"Could not initialize the Gemini client: {e}")
    st.stop()


if "authenticated" not in st.session_state:
    st.session_state.authenticated = False
if "login_time" not in st.session_state:
    st.session_state.login_time = None
if "access_label" not in st.session_state:
    st.session_state.access_label = None

if st.session_state.authenticated and st.session_state.login_time:
    if time.time() - st.session_state.login_time > SESSION_TIMEOUT_SECONDS:
        st.session_state.authenticated = False
        st.session_state.login_time = None
        st.session_state.access_label = None
        st.warning("Your session expired. Please log in again.")

if not st.session_state.authenticated:
    st.title("🔑 RASHE AI — Access Verification")

    salt, codes = load_access_config()
    if not salt or not codes:
        st.error("Access control isn't configured yet. Add [access] salt + codes in Streamlit secrets.")
        st.stop()

    st.info("Enter your access code below to unlock the application.")
    st.markdown("👉 **Need a code?** Contact the developer to request access.")

    client_key = get_client_key()
    attempts = _FAILED_ATTEMPTS_STORE.get(client_key, {"count": 0, "locked_until": 0})
    now = time.time()

    if now < attempts["locked_until"]:
        remaining = int(attempts["locked_until"] - now)
        st.error(f"Too many failed attempts. Try again in {remaining}s.")
    else:
        user_code = st.text_input("Enter Access Code:", type="password")
        if st.button("Unlock / Seeni"):
            label = verify_code(user_code, salt, codes)
            if label:
                st.session_state.authenticated = True
                st.session_state.login_time = time.time()
                st.session_state.access_label = label
                _FAILED_ATTEMPTS_STORE.pop(client_key, None)
                st.success("Access Granted! Welcome to RASHE AI.")
                st.rerun()
            else:
                attempts["count"] += 1
                if attempts["count"] >= LOCKOUT_THRESHOLD:
                    backoff = LOCKOUT_BASE_SECONDS * (2 ** (attempts["count"] - LOCKOUT_THRESHOLD))
                    attempts["locked_until"] = now + backoff
                _FAILED_ATTEMPTS_STORE[client_key] = attempts
                st.error("Invalid access code.")
    st.stop()


st.sidebar.title("🤖 RASHE AI Profile")
st.sidebar.markdown("**Developer:** Duresa Shumbura (Rashedin)")
if st.session_state.access_label:
    st.sidebar.caption(f"Signed in as: {st.session_state.access_label}")

if st.sidebar.button("Log out"):
    st.session_state.authenticated = False
    st.session_state.login_time = None
    st.session_state.access_label = None
    st.rerun()

photo_style = st.sidebar.selectbox("Photo Frame Style:", ["Circle", "Gold Frame", "Standard Rectangular"])

try:
    img = Image.open("creator.jpg")
    if photo_style == "Circle":
        mask = Image.new('L', img.size, 0)
        draw = ImageDraw.Draw(mask)
        draw.ellipse((0, 0) + img.size, fill=255)
        img = ImageOps.fit(img, mask.size, centering=(0.5, 0.5))
        img.putalpha(mask)
        st.sidebar.image(img, caption="Duresa Shumbura (Rashedin)", use_container_width=True)
    elif photo_style == "Gold Frame":
        bordered_img = ImageOps.expand(img, border=15, fill='gold')
        st.sidebar.image(bordered_img, caption="Duresa Shumbura (Rashedin)", use_container_width=True)
    else:
        st.sidebar.image(img, caption="Duresa Shumbura (Rashedin)", use_container_width=True)
except FileNotFoundError:
    st.sidebar.warning("Creator image 'creator.jpg' not found in working directory.")
except Exception as e:
    st.sidebar.warning(f"Could not load creator image: {e}")

st.sidebar.markdown("""
---
**Supported Languages:**
* 🟢 Afaan Oromoo
* 🟢 Amharic (አማርኛ)
* 🟢 English

**Core Capabilities:**
* 💬 Multilingual Text & Audio Q&A
* 🎨 AI Image Generation
* 🔒 Secure Key Verification
""")

st.title("🤖 RASHE AI")
st.write("Welcome! Ask questions in **Afaan Oromoo**, **Amharic**, or **English** using text or audio.")

tab1, tab2 = st.tabs(["💬 Chat (Text & Voice)", "🎨 Image Generation"])

with tab1:
    if "messages" not in st.session_state:
        st.session_state.messages = []

    for msg in st.session_state.messages:
        with st.chat_message(msg["role"]):
            st.write(msg["content"])

    user_prompt = st.chat_input("Write in Afaan Oromoo, Amharic, or English...")
    audio_file = st.file_uploader("Or upload an audio query (MP3/WAV/M4A):", type=["mp3", "wav", "m4a"])

    if user_prompt or audio_file:
        bot_reply = None
        try:
            if user_prompt:
                st.session_state.messages.append({"role": "user", "content": user_prompt})
                with st.chat_message("user"):
                    st.write(user_prompt)
                response = client.models.generate_content(
                    model="gemini-2.5-flash",
                    contents=user_prompt,
                    config=types.GenerateContentConfig(system_instruction=SYSTEM_INSTRUCTION)
                )
                bot_reply = response.text
            elif audio_file:
                st.audio(audio_file)
                audio_bytes = audio_file.read()
                mime_type = audio_file.type
                response = client.models.generate_content(
                    model="gemini-2.5-flash",
                    contents=[
                        types.Part.from_bytes(data=audio_bytes, mime_type=mime_type),
                        "Listen to this audio file and respond directly in the user's spoken language."
                    ],
                    config=types.GenerateContentConfig(system_instruction=SYSTEM_INSTRUCTION)
                )
                bot_reply = response.text
        except Exception as e:
            st.error(f"Something went wrong talking to the model: {e}")

        if bot_reply:
            st.session_state.messages.append({"role": "assistant", "content": bot_reply})
            with st.chat_message("assistant"):
                st.write(bot_reply)

with tab2:
    st.header("Generate Custom Images")
    img_prompt = st.text_input("Describe the image you want to create:")

    if st.button("Generate Image"):
        if img_prompt:
            with st.spinner("Generating image..."):
                try:
                    translated = client.models.generate_content(
                        model="gemini-2.5-flash",
                        contents=f"Translate this text into a clear English image prompt: {img_prompt}"
                    ).text
                    result = client.models.generate_images(
                        model='imagen-3.0-generate-002',
                        prompt=translated,
                        config=types.GenerateImagesConfig(number_of_images=1, aspect_ratio="1:1")
                    )
                    for gen_img in result.generated_images:
                        st.image(gen_img.image.image_bytes, caption=img_prompt)
                except Exception as e:
                    st.error(f"Image generation error: {e}")
        else:
            st.warning("Please enter a text description first.")
