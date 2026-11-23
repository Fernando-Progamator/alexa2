import gc
import os
from machine import Pin
# custom lib #
from wifi import connect
from speaker import Speaker as _Speaker, initSpeaker
from microfone import Microphone as _Microphone, WAVer as _WAVer, startSTT, chunkSTT, endSTT
from config import API_KEYS, FISH_AUDIO_VOICESID, SPEAKER_DATA
from tests import runTest
from storage import mountSDCard
# misc #
from utime import sleep
# my api #


def ensure_dir(path):
    try:
        os.listdir(path)
        return
    except OSError:
        pass

    parent = path.rsplit("/", 1)[0] if "/" in path else ""
    if parent and parent != path:
        ensure_dir(parent)

    try:
        os.mkdir(path)
    except OSError:
        pass

ALEXA_MODES = [
    "NORMAL",
    "MUSIC"
]
MODES_INDEX = 0
ALEXA_MODE = ALEXA_MODES[MODES_INDEX % len(ALEXA_MODES)] # {"NORMAL,MUSIC"}

API = "https://api-stt.onrender.com/"
KEY = API_KEYS["my-stt"]
CHUNK_MEMORY_RESERVE = 64 * 1024
MIN_CHUNK_SIZE = 1024

CURRENT_VOICE = "gojo"

LED_RECORDING = Pin("LED", Pin.OUT) # Led debug, 10 or 25
LED_RECORDING.value(1)
sleep(1)
LED_RECORDING.value(0)

BUTTON_RECORD = Pin(11, Pin.IN) # Gravar
BUTTON_CHANGE = Pin(12, Pin.IN) # Mudar voz ou musica (depende do modo)
BUTTON_MODE = Pin(13, Pin.IN) # Mudar modo (NORMAL ou MUSIC)

Speaker =  _Speaker(SPEAKER_DATA["Sample"])
Microphone = _Microphone()

gc.collect()
print(mountSDCard())
print(os.listdir("/sd"))
LED_RECORDING.value(1)
sleep(1)
LED_RECORDING.value(0)
#
if not connect(): # Connect to WIFIs
    raise OSError
if ALEXA_MODE == "NORMAL":
    while True:
        WAVer = _WAVer(Microphone)
        LED_RECORDING.value(1)
        WAVer.record(5)
        LED_RECORDING.value(0)

        startSTT()
        ensure_dir("/sd/thinking_voices")
        thinking_audio_path = f"/sd/thinking_voices/{CURRENT_VOICE}_pensando.wav"
        with open(thinking_audio_path, "rb") as f:
            f.read(44)
            while True:
                chunk = f.read(4096)
                if not chunk:
                    break
                Speaker.write(chunk)

        with open("/sd/audio.wav", "rb") as wav:
            while True:
                gc.collect()
                chunk_size = max(MIN_CHUNK_SIZE, gc.mem_free() - CHUNK_MEMORY_RESERVE)
                chunk_position = wav.tell()
                while True:
                    try:
                        chunk = wav.read(chunk_size)
                        break
                    except MemoryError:
                        wav.seek(chunk_position)
                        if chunk_size <= MIN_CHUNK_SIZE:
                            raise
                        chunk_size = max(MIN_CHUNK_SIZE, chunk_size // 2)
                if not chunk:
                    break
                print(f"Chunking {chunk_size} bytes")
                chunkSTT(chunk)
        endSTT(CURRENT_VOICE, speaker=Speaker)
else:
    MUSICS = "/sd/musics"
    ensure_dir(MUSICS)
    DIR_MUSICS = [
        f"{MUSICS}/Music1.wav", # GRIEF
        f"{MUSICS}/Music2.wav", # TSII
        f"{MUSICS}/Music3.wav", # AMETHYST
        f"{MUSICS}/Music4.wav", # TIDAL WAVE
        f"{MUSICS}/Music5.wav", # ANTARCTIC LIGHTS
        f"{MUSICS}/TPoV.wav",   # Through the patches of violet, not include ig
        f"{MUSICS}/SmoothCriminal.wav", # GOAT
        f"{MUSICS}/TROLL.wav",  # trollgaem cara, mui pelicugoso
    ]
    MUSIC_INDEX = 5
    DIR_INDEX = MUSIC_INDEX % len(DIR_MUSICS)
    SELECTED = DIR_MUSICS[DIR_INDEX]

    def play():
        with open(SELECTED, "rb") as f:
            while True:
                chunk = f.read(30000)
                if not chunk:
                    break
                Speaker.write(chunk)

    play()
