import asyncio
import hashlib
import logging
import os
import sys
import tempfile
from pathlib import Path

from fishaudio import FishAudio
from telegram import Update
from telegram.constants import ChatAction
from telegram.ext import (
    Application,
    CommandHandler,
    ContextTypes,
    MessageHandler,
    filters,
)

logging.basicConfig(
    format="%(asctime)s %(levelname)s %(name)s: %(message)s",
    level=logging.INFO,
)
# Queste librerie scrivono indirizzi che contengono il token: le silenziamo.
for noisy in ("httpx", "httpcore", "tornado.access"):
    logging.getLogger(noisy).setLevel(logging.WARNING)

logger = logging.getLogger("vocali")

MAX_CHARS = 500
TTS_TIMEOUT = 120


def read_int(name):
    raw = os.environ.get(name, "").strip()
    if raw.lstrip("-").isdigit():
        return int(raw)
    return None


OWNER_ID = read_int("OWNER_ID")
FISH_API_KEY = os.environ.get("FISH_API_KEY", "").strip()
VOICE_MODEL_ID = os.environ.get("VOICE_MODEL_ID", "").strip()

fish_client = FishAudio(api_key=FISH_API_KEY) if FISH_API_KEY else None


def is_owner(update: Update) -> bool:
    user = update.effective_user
    return user is not None and OWNER_ID is not None and user.id == OWNER_ID


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    message = update.effective_message
    if not is_owner(update):
        await message.reply_text("Questo bot è privato.")
        return
    await message.reply_text(
        "Scrivimi un testo (massimo 500 caratteri) e te lo mando come messaggio vocale."
    )


async def show_id(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    user = update.effective_user
    if user is not None:
        await update.effective_message.reply_text(f"Il tuo ID utente: {user.id}")


async def handle_text(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    message = update.effective_message
    if not is_owner(update):
        await message.reply_text("Questo bot è privato.")
        return

    text = (message.text or "").strip()
    if not text:
        return
    if len(text) > MAX_CHARS:
        await message.reply_text(
            f"Il testo è troppo lungo: il massimo è {MAX_CHARS} caratteri."
        )
        return
    if fish_client is None or not VOICE_MODEL_ID:
        await message.reply_text("Il bot non è ancora configurato (voce o chiave mancanti).")
        return

    status = await message.reply_text("Sto generando il vocale...")
    try:
        audio = await asyncio.wait_for(
            asyncio.to_thread(
                fish_client.tts.convert, text=text, reference_id=VOICE_MODEL_ID
            ),
            timeout=TTS_TIMEOUT,
        )
        data = audio if isinstance(audio, (bytes, bytearray)) else b"".join(audio)
        if not data:
            raise RuntimeError("audio vuoto")

        with tempfile.TemporaryDirectory() as tmp:
            path = Path(tmp) / "vocale.mp3"
            path.write_bytes(data)
            await context.bot.send_chat_action(
                update.effective_chat.id, ChatAction.RECORD_VOICE
            )
            with path.open("rb") as audio_file:
                await message.reply_voice(voice=audio_file)
    except asyncio.TimeoutError:
        await message.reply_text("La generazione ha impiegato troppo tempo. Riprova.")
    except Exception as error:
        logger.error("Errore durante la generazione: %s", type(error).__name__)
        await message.reply_text(
            "Non sono riuscito a creare il vocale. Riprova tra poco."
        )
    finally:
        try:
            await status.delete()
        except Exception:
            pass


async def error_handler(update: object, context: ContextTypes.DEFAULT_TYPE) -> None:
    logger.error("Errore non gestito: %s", type(context.error).__name__)


def main() -> None:
    token = os.environ.get("TELEGRAM_TOKEN", "").strip()
    if not token:
        print("ERRORE: manca la variabile TELEGRAM_TOKEN.")
        sys.exit(1)

    base_url = (
        os.environ.get("WEBHOOK_BASE_URL") or os.environ.get("RENDER_EXTERNAL_URL") or ""
    ).rstrip("/")
    if not base_url:
        print("ERRORE: manca l'indirizzo pubblico del servizio (RENDER_EXTERNAL_URL).")
        sys.exit(1)

    port = int(os.environ.get("PORT", "10000"))

    # Percorso e codice segreto derivati dal token: il token non compare negli indirizzi.
    digest = hashlib.sha256(token.encode()).hexdigest()
    url_path = digest[:40]
    secret_token = digest[40:]

    application = (
        Application.builder()
        .token(token)
        .connect_timeout(30)
        .read_timeout(60)
        .write_timeout(60)
        .build()
    )
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("id", show_id))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_text))
    application.add_error_handler(error_handler)

    logger.info("Avvio del bot in modalità webhook sulla porta %s", port)
    application.run_webhook(
        listen="0.0.0.0",
        port=port,
        url_path=url_path,
        webhook_url=f"{base_url}/{url_path}",
        secret_token=secret_token,
        allowed_updates=["message"],
    )


if __name__ == "__main__":
    main()
