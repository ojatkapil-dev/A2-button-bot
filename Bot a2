import os
import json
import time
import threading
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.request import Request, urlopen
from urllib.parse import urlencode

TOKEN = os.environ["BOT_TOKEN"]
ADMIN_ID = int(os.environ["ADMIN_ID"])

API = f"https://api.telegram.org/bot{TOKEN}"

state = {}


def api(method, data=None):
    if data is None:
        data = {}

    encoded = urlencode(data).encode()
    req = Request(
        f"{API}/{method}",
        data=encoded,
        headers={"Content-Type": "application/x-www-form-urlencoded"}
    )

    with urlopen(req, timeout=60) as response:
        return json.loads(response.read().decode())


def send(chat_id, text, buttons=None):
    data = {
        "chat_id": chat_id,
        "text": text
    }

    if buttons:
        data["reply_markup"] = json.dumps({
            "inline_keyboard": buttons
        })

    return api("sendMessage", data)


def edit(chat_id, message_id, text, buttons=None):
    data = {
        "chat_id": chat_id,
        "message_id": message_id,
        "text": text
    }

    if buttons:
        data["reply_markup"] = json.dumps({
            "inline_keyboard": buttons
        })

    return api("editMessageText", data)


def handle_message(message):
    user_id = message["from"]["id"]

    if user_id != ADMIN_ID:
        return

    chat_id = message["chat"]["id"]
    text = message.get("text", "").strip()

    if text == "/start":
        send(
            chat_id,
            "🤖 A2 BUTTON BOT\n\n"
            "PostBot जैसा अपना custom button system.\n\n"
            "📝 /post - नया post बनाएं\n"
            "🆔 /id - अपना Telegram ID देखें"
        )

    elif text == "/id":
        send(chat_id, f"🆔 Your Telegram ID:\n`{user_id}`")

    elif text == "/post":
        state[user_id] = {
            "step": "message",
            "text": "",
            "buttons": []
        }

        send(
            chat_id,
            "📝 अपना post message भेजें.\n\n"
            "Example:\n"
            "🔥 VIP SESSION STARTING SOON\n"
            "Join now!"
        )

    elif user_id in state and state[user_id]["step"] == "message":
        state[user_id]["text"] = text
        state[user_id]["step"] = "button_name"

        send(
            chat_id,
            "🔘 Button का नाम भेजें.\n\n"
            "Example: JOIN VIP\n\n"
            "अगर button नहीं चाहिए तो /done भेजें."
        )

    elif user_id in state and state[user_id]["step"] == "button_name":
        if text == "/done":
            show_preview(user_id, chat_id)
            return

        state[user_id]["button_name"] = text
        state[user_id]["step"] = "button_url"

        send(
            chat_id,
            "🔗 अब उस button का URL भेजें.\n\n"
            "Example:\n"
            "https://example.com"
        )

    elif user_id in state and state[user_id]["step"] == "button_url":
        state[user_id]["buttons"].append([
            {
                "text": state[user_id]["button_name"],
                "url": text
            }
        ])

        state[user_id]["step"] = "button_name"

        send(
            chat_id,
            "✅ Button added!\n\n"
            "दूसरा button का नाम भेजें.\n"
            "या पूरा होने पर /done भेजें."
        )


def show_preview(user_id, chat_id):
    data = state[user_id]

    buttons = data["buttons"]

    buttons.append([
        {
            "text": "📢 SEND PUBLIC",
            "callback_data": "send_public"
        }
    ])

    buttons.append([
        {
            "text": "🔒 SEND PRIVATE",
            "callback_data": "send_private"
        }
    ])

    buttons.append([
        {
            "text": "❌ CANCEL",
            "callback_data": "cancel"
        }
    ])

    send(
        chat_id,
        "👁 POST PREVIEW\n\n" + data["text"],
        buttons
    )


def handle_callback(callback):
    user_id = callback["from"]["id"]

    if user_id != ADMIN_ID:
        return

    data = callback["data"]
    chat_id = callback["message"]["chat"]["id"]
    message_id = callback["message"]["message_id"]

    api("answerCallbackQuery", {
        "callback_query_id": callback["id"]
    })

    if data == "cancel":
        state.pop(user_id, None)
        edit(chat_id, message_id, "❌ Post cancelled.")
        return

    if data == "send_public":
        channel = os.environ.get("PUBLIC_CHANNEL")

        if not channel:
            edit(
                chat_id,
                message_id,
                "⚠️ PUBLIC_CHANNEL अभी set नहीं है."
            )
            return

        post = state[user_id]

        api(
            "sendMessage",
            {
                "chat_id": channel,
                "text": post["text"],
                "reply_markup": json.dumps({
                    "inline_keyboard": post["buttons"]
                })
            }
        )

        edit(chat_id, message_id, "✅ Public Channel में post भेज दिया गया!")
        state.pop(user_id, None)

    elif data == "send_private":
        channel = os.environ.get("PRIVATE_CHANNEL")

        if not channel:
            edit(
                chat_id,
                message_id,
                "⚠️ PRIVATE_CHANNEL अभी set नहीं है."
            )
            return

        post = state[user_id]

        api(
            "sendMessage",
            {
                "chat_id": channel,
                "text": post["text"],
                "reply_markup": json.dumps({
                    "inline_keyboard": post["buttons"]
                })
            }
        )

        edit(chat_id, message_id, "✅ Private Channel में post भेज दिया गया!")
        state.pop(user_id, None)


def polling():
    offset = 0

    while True:
        try:
            result = api(
                "getUpdates",
                {
                    "offset": offset,
                    "timeout": 50
                }
            )

            for update in result.get("result", []):
                offset = update["update_id"] + 1

                if "message" in update:
                    handle_message(update["message"])

                elif "callback_query" in update:
                    handle_callback(update["callback_query"])

        except Exception as e:
            print("Polling error:", e)
            time.sleep(3)


class HealthHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"A2 Button Bot is running!")

    def log_message(self, format, *args):
        return


def web_server():
    port = int(os.environ.get("PORT", 10000))
    server = HTTPServer(("0.0.0.0", port), HealthHandler)
    server.serve_forever()


if __name__ == "__main__":
    threading.Thread(target=web_server, daemon=True).start()
    polling()
