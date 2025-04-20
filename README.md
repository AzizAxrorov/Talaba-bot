import math
import requests
import json
import uuid
from datetime import datetime
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes, CallbackQueryHandler
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
import os

# Tokenlar va API kalitlari to‘g‘ridan-to‘g‘ri kodda
TELEGRAM_TOKEN = "8178324958:AAEA-gSrfAnynGVdLb9vrhspcUBWZkvBJzI"  # Replace with your Telegram Bot Token
GEMINI_API_KEY = "AIzaSyDA1bp80MVka2huOYr5fbvWbVDNfzTmGSk"     # Replace with your Gemini API Key

# Gemini API sozlamalari
GEMINI_API_URL = "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent"

# MyMemory API sozlamalari
MYMEMORY_API_URL = "https://api.mymemory.translated.net/get"

# Foydalanuvchilar ma'lumotlari fayli
USERS_FILE = "users_data.json"
users_data = {}

# Bayroqlar
FLAGS = {
    "uz": "🇺🇿", "kk": "🇰🇿", "az": "🇦🇿", "ky": "🇰🇬", "tr": "🇹🇷", "tt": "🇷🇺"  # Tatar uchun Rossiya bayrog‘i
}

# TCP/IP qatlamlari ma'lumotlari
TCPIP_LAYERS = {
    1: {
        "name": "Ilova qatlami (Application Layer)",
        "description": "Foydalanuvchi ilovalari bilan bevosita ishlaydi, ma'lumotlar foydalanuvchiga taqdim etiladi yoki undan qabul qilinadi.",
        "protocols": "HTTP, HTTPS, FTP, SMTP, DNS, POP3, IMAP",
        "devices": "Veb-serverlar, email mijozlari",
        "example": "Veb-sayt ochishda HTTP so‘rovi yuboriladi."
    },
    2: {
        "name": "Transport qatlami (Transport Layer)",
        "description": "Ma'lumotlarni ishonchli (TCP) yoki tez (UDP) yetkazib beradi, xatolarni tekshiradi va portlar orqali aloqa qiladi.",
        "protocols": "TCP, UDP",
        "devices": "Firewall, load balancer",
        "example": "Video oqimida UDP protokoli ishlatiladi."
    },
    3: {
        "name": "Internet qatlami (Internet Layer)",
        "description": "Ma'lumotlarni marshrutlashtiradi, IP manzillarni boshqaradi va tarmoqlar orasida aloqa ta'minlaydi.",
        "protocols": "IP (IPv4, IPv6), ICMP, ARP",
        "devices": "Router, Layer 3 switch",
        "example": "Router ma'lumotlarni IP manzil bo‘yicha boshqa tarmoqqa yo‘naltiradi."
    },
    4: {
        "name": "Ulanish qatlami (Link Layer)",
        "description": "Ma'lumotlarni ramka sifatida uzatadi, xatolarni aniqlaydi va MAC manzillar bilan ishlaydi.",
        "protocols": "Ethernet, Wi-Fi (802.11), PPP",
        "devices": "Switch, NIC (tarmoq kartasi)",
        "example": "Switch MAC manzil bo‘yicha ma'lumotlarni qurilmaga yuboradi."
    }
}

# OSI qatlamlari ma'lumotlari
def get_layer_info(choice):
    layers = {
        1: {
            "name": "Application Layer (Ilova qatlami)",
            "description": "Foydalanuvchi ilovalari bilan bevosita ishlaydi, ma'lumotlar foydalanuvchiga taqdim etiladi yoki undan qabul qilinadi.",
            "protocols": "HTTP, HTTPS, FTP, SMTP, DNS, POP3, IMAP",
            "devices": "Veb-brauzerlar, email mijozlari",
            "example": "Siz brauzerda 'www.example.com' saytini ochsangiz, HTTP/HTTPS protokoli ishlaydi."
        },
        2: {
            "name": "Presentation Layer (Taqdimot qatlami)",
            "description": "Ma'lumotlarni formatlash, shifrlash va siqishni amalga oshiradi. Ma'lumotlar tushunarli shaklga keltiriladi.",
            "protocols": "SSL/TLS, JPEG, PNG, MPEG",
            "devices": "Shifrlash serverlari, multimedia kodeklari",
            "example": "Saytga kirganda HTTPS orqali ma'lumotlar shifrlanadi."
        },
        3: {
            "name": "Session Layer (Seans qatlami)",
            "description": "Foydalanuvchi seanslarini boshqaradi, ulanishlarni ochadi, yopadi va sinxronlashtiradi.",
            "protocols": "NetBIOS, RPC, PPTP",
            "devices": "Seans boshqaruv serverlari",
            "example": "Video qo‘ng‘iroq paytida ulanishni boshqarish."
        },
        4: {
            "name": "Transport Layer (Transport qatlami)",
            "description": "Ma'lumotlarni ishonchli (TCP) yoki tez (UDP) yetkazib beradi, xatolarni tekshiradi va portlar orqali aloqa qiladi.",
            "protocols": "TCP, UDP",
            "devices": "Firewall, load balancer",
            "example": "TCP orqali veb-sayt ma'lumotlari bo‘laklarga bo‘linib, ketma-ket yetkaziladi."
        },
        5: {
            "name": "Network Layer (Tarmoq qatlami)",
            "description": "Ma'lumotlarni marshrutlashtiradi, IP manzillarni boshqaradi va tarmoqlar orasida aloqa ta'minlaydi.",
            "protocols": "IP (IPv4, IPv6), ICMP, ARP",
            "devices": "Router, Layer 3 switch",
            "example": "Router ma'lumotlarni IP manzil bo‘yicha boshqa tarmoqqa yo‘naltiradi."
        },
        6: {
            "name": "Data Link Layer (Ma'lumot uzatish qatlami)",
            "description": "Ma'lumotlarni ramka sifatida uzatadi, xatolarni aniqlaydi va MAC manzillar bilan ishlaydi.",
            "protocols": "Ethernet, Wi-Fi (802.11), PPP",
            "devices": "Switch, bridge, NIC (tarmoq kartasi)",
            "example": "Switch MAC manzil bo‘yicha ma'lumotlarni qurilmaga yuboradi."
        },
        7: {
            "name": "Physical Layer (Fizik qatlam)",
            "description": "Ma'lumotlarni bit sifatida uzatadi, elektr signalari, kabellar va ulagichlar bilan ishlaydi.",
            "protocols": "USB, Bluetooth, DSL, ISDN",
            "devices": "Kabellar, hub, optik tolalar, antenna",
            "example": "Ma'lumotlar optik kabel orqali bit sifatida uzatiladi."
        }
    }
    return layers.get(choice, None)

# Transliteratsiya jadvallari
TRANSLIT_TABLES = {
    "uz": {
        "lotin_to_kiril": {
            "a": "а", "b": "б", "d": "д", "e": "э", "f": "ф", "g": "г", "h": "ҳ",
            "i": "и", "j": "ж", "k": "к", "l": "л", "m": "м", "n": "н", "o": "о",
            "p": "п", "q": "қ", "r": "р", "s": "с", "t": "т", "u": "у", "v": "в",
            "x": "х", "y": "й", "z": "з", "o‘": "ў", "g‘": "ғ", "sh": "ш", "ch": "ч",
            "A": "А", "B": "Б", "D": "Д", "E": "Э", "F": "Ф", "G": "Г", "H": "Ҳ",
            "I": "И", "J": "Ж", "K": "К", "L": "Л", "M": "М", "N": "Н", "O": "О",
            "P": "П", "Q": "Қ", "R": "Р", "S": "С", "T": "Т", "U": "У", "V": "В",
            "X": "Х", "Y": "Й", "Z": "З", "O‘": "Ў", "G‘": "Ғ", "Sh": "Ш", "Ch": "Ч"
        },
        "kiril_to_lotin": {
            "а": "a", "б": "b", "д": "d", "э": "e", "ф": "f", "г": "g", "ҳ": "h",
            "и": "i", "ж": "j", "к": "k", "л": "л", "м": "м", "н": "н", "о": "о",
            "п": "п", "қ": "q", "р": "р", "с": "с", "т": "т", "у": "у", "в": "в",
            "х": "х", "й": "й", "з": "з", "ў": "o‘", "ғ": "g‘", "ш": "ш", "ч": "ч",
            "А": "А", "Б": "Б", "Д": "Д", "Э": "Э", "Ф": "Ф", "Г": "Г", "Ҳ": "Ҳ",
            "И": "И", "Ж": "Ж", "К": "К", "Л": "Л", "М": "М", "Н": "Н", "О": "О",
            "П": "П", "Қ": "Қ", "Р": "Р", "С": "С", "Т": "Т", "У": "У", "В": "В",
            "Х": "Х", "Й": "Й", "З": "З", "Ў": "O‘", "Ғ": "G‘", "Ш": "Ш", "Ч": "Ч"
        }
    },
    "kk": {
        "lotin_to_kiril": {
            "a": "а", "ä": "ә", "b": "б", "d": "д", "e": "е", "f": "ф", "g": "г",
            "ğ": "ғ", "h": "х", "ı": "ы", "i": "і", "j": "ж", "k": "к", "q": "қ",
            "l": "л", "m": "м", "n": "н", "ñ": "ң", "o": "о", "ö": "ө", "p": "п",
            "r": "р", "s": "с", "t": "т", "u": "у", "ū": "ұ", "ü": "ү", "v": "в",
            "y": "й", "z": "з", "ş": "ш", "ç": "ч",
            "A": "А", "Ä": "Ә", "B": "Б", "D": "Д", "E": "Е", "F": "Ф", "G": "Г",
            "Ğ": "Ғ", "H": "Х", "I": "Ы", "İ": "І", "J": "Ж", "K": "К", "Q": "Қ",
            "L": "Л", "M": "М", "N": "Н", "Ñ": "Ң", "O": "О", "Ö": "Ө", "P": "П",
            "R": "Р", "S": "С", "T": "Т", "U": "У", "Ū": "Ұ", "Ü": "Ү", "V": "В",
            "Y": "Й", "Z": "З", "Ş": "Ш", "Ç": "Ч"
        },
        "kiril_to_lotin": {
            "а": "a", "ә": "ä", "б": "b", "д": "d", "е": "e", "ф": "f", "г": "g",
            "ғ": "ğ", "х": "h", "ы": "ı", "і": "i", "ж": "j", "к": "k", "қ": "q",
            "л": "l", "м": "m", "н": "n", "ң": "ñ", "о": "o", "ө": "ö", "п": "p",
            "р": "р", "с": "с", "т": "т", "у": "у", "ұ": "ū", "ү": "ü", "в": "в",
            "й": "й", "з": "з", "ш": "ш", "ч": "ч",
            "А": "А", "Ә": "Ә", "Б": "Б", "Д": "Д", "Е": "Е", "Ф": "Ф", "Г": "Г",
            "Ғ": "Ғ", "Х": "Х", "Ы": "Ы", "І": "І", "Ж": "Ж", "К": "К", "Қ": "Қ",
            "Л": "Л", "М": "М", "Н": "Н", "Ң": "Ң", "О": "О", "Ө": "Ө", "П": "П",
            "Р": "Р", "С": "С", "Т": "Т", "У": "У", "Ұ": "Ұ", "Ү": "Ү", "В": "В",
            "Й": "Й", "З": "З", "Ш": "Ш", "Ч": "Ч"
        }
    },
    "az": {
        "lotin_to_kiril": {
            "a": "а", "b": "б", "c": "дж", "ç": "ч", "d": "д", "e": "е", "ə": "ә",
            "f": "ф", "g": "г", "ğ": "ғ", "h": "х", "x": "х", "ı": "ы", "i": "и",
            "j": "ж", "k": "к", "q": "г", "l": "л", "m": "м", "n": "н", "o": "о",
            "ö": "ө", "p": "п", "r": "р", "s": "с", "ş": "ш", "t": "т", "u": "у",
            "ü": "ү", "v": "в", "y": "й", "z": "з",
            "A": "А", "B": "Б", "C": "Дж", "Ç": "Ч", "D": "Д", "E": "Е", "Ə": "Ә",
            "F": "Ф", "G": "Г", "Ğ": "Ғ", "H": "Х", "X": "Х", "I": "Ы", "İ": "И",
            "J": "Ж", "K": "К", "Q": "Г", "L": "Л", "M": "М", "N": "Н", "O": "О",
            "Ö": "Ө", "P": "П", "R": "Р", "S": "С", "Ş": "Ш", "T": "Т", "U": "У",
            "Ü": "Ү", "V": "В", "Y": "Й", "Z": "З"
        },
        "kiril_to_lotin": {
            "а": "a", "б": "b", "дж": "c", "ч": "ç", "д": "d", "е": "e", "ә": "ə",
            "ф": "f", "г": "g", "ғ": "ğ", "х": "h", "ы": "ı", "и": "i", "ж": "j",
            "к": "k", "г": "q", "л": "l", "м": "m", "н": "n", "о": "o", "ө": "ö",
            "п": "p", "р": "р", "с": "с", "ш": "ш", "т": "т", "у": "у", "ү": "ü",
            "в": "в", "й": "й", "з": "з",
            "А": "А", "Б": "Б", "Дж": "C", "Ч": "Ч", "Д": "Д", "Е": "Е", "Ә": "Ə",
            "Ф": "Ф", "Г": "Г", "Ғ": "Ғ", "Х": "Х", "Ы": "Ы", "И": "И", "Ж": "Ж",
            "К": "К", "Г": "Q", "Л": "Л", "М": "М", "Н": "Н", "О": "О", "Ө": "Ө",
            "П": "П", "Р": "Р", "С": "С", "Ш": "Ш", "Т": "Т", "У": "У", "Ү": "Ü",
            "В": "В", "Й": "Й", "З": "З"
        }
    },
    "ky": {
        "lotin_to_kiril": {
            "a": "а", "b": "б", "d": "д", "e": "е", "f": "ф", "g": "г", "h": "х",
            "i": "и", "j": "ж", "k": "к", "l": "л", "m": "м", "n": "н", "ñ": "ң",
            "o": "о", "ö": "ө", "p": "п", "r": "р", "s": "с", "t": "т", "u": "у",
            "ü": "ү", "v": "в", "y": "й", "z": "з",
            "A": "А", "B": "Б", "D": "Д", "E": "Е", "F": "Ф", "G": "Г", "H": "Х",
            "I": "И", "J": "Ж", "K": "К", "L": "Л", "M": "М", "N": "Н", "Ñ": "Ң",
            "O": "О", "Ö": "Ө", "P": "П", "R": "Р", "S": "С", "T": "Т", "U": "У",
            "Ü": "Ү", "V": "В", "Y": "Й", "Z": "З"
        },
        "kiril_to_lotin": {
            "а": "a", "б": "b", "д": "d", "е": "e", "ф": "f", "г": "g", "х": "h",
            "и": "i", "ж": "j", "к": "k", "л": "л", "м": "м", "н": "н", "ң": "ñ",
            "о": "о", "ө": "ө", "п": "п", "р": "р", "с": "с", "т": "т", "у": "у",
            "ү": "ü", "в": "в", "й": "й", "з": "з",
            "А": "А", "Б": "Б", "Д": "Д", "Е": "Е", "Ф": "Ф", "Г": "Г", "Х": "Х",
            "И": "И", "Ж": "Ж", "К": "К", "Л": "Л", "М": "М", "Н": "Н", "Ң": "Ң",
            "О": "О", "Ө": "Ө", "П": "П", "Р": "Р", "С": "С", "Т": "Т", "У": "У",
            "Ү": "Ü", "В": "В", "Й": "Й", "З": "З"
        }
    },
    "tr": {
        "lotin_to_kiril": {},
        "kiril_to_lotin": {}
    },
    "tt": {
        "lotin_to_kiril": {
            "a": "а", "ä": "ә", "b": "б", "d": "д", "e": "е", "f": "ф", "g": "г",
            "h": "х", "ı": "ы", "i": "и", "j": "ж", "k": "к", "q": "к", "l": "л",
            "m": "м", "n": "н", "ñ": "ң", "o": "о", "ö": "ө", "p": "п", "r": "р",
            "s": "с", "t": "т", "u": "у", "ü": "ү", "w": "в", "y": "й", "z": "з",
            "ş": "ш", "ç": "ч",
            "A": "А", "Ä": "Ә", "B": "Б", "D": "Д", "E": "Е", "F": "Ф", "G": "Г",
            "H": "Х", "I": "Ы", "İ": "И", "J": "Ж", "K": "К", "Q": "К", "L": "Л",
            "M": "М", "N": "Н", "Ñ": "Ң", "O": "О", "Ö": "Ө", "P": "П", "R": "Р",
            "S": "С", "T": "Т", "U": "У", "Ü": "Ү", "W": "В", "Y": "Й", "Z": "З",
            "Ş": "Ш", "Ç": "Ч"
        },
        "kiril_to_lotin": {
            "а": "a", "ә": "ä", "б": "b", "д": "d", "е": "e", "ф": "f", "г": "g",
            "х": "h", "ы": "ı", "и": "i", "ж": "j", "к": "k", "л": "л", "м": "м",
            "н": "н", "ң": "ñ", "о": "о", "ө": "ө", "п": "п", "р": "р", "с": "с",
            "т": "т", "у": "у", "ү": "ü", "в": "в", "й": "й", "з": "з", "ш": "ш",
            "ч": "ч",
            "А": "А", "Ә": "Ә", "Б": "Б", "Д": "Д", "Е": "Е", "Ф": "Ф", "Г": "Г",
            "Х": "Х", "Ы": "Ы", "И": "И", "Ж": "Ж", "К": "К", "Л": "Л", "М": "М",
            "Н": "Н", "Ң": "Ң", "О": "О", "Ө": "Ө", "П": "П", "Р": "Р", "С": "С",
            "Т": "Т", "У": "У", "Ү": "Ü", "В": "В", "Й": "Й", "З": "З", "Ш": "Ш",
            "Ч": "Ч"
        }
    }
}

# Alifbo aniqlash funksiyasi
def detect_script(text):
    kiril_chars = set("абвгдеёжзийклмнопрстуфхцчшщъыьэюяўғқҳңөүұә")
    lotin_chars = set("o‘g‘şçäñöüūə")
    kiril_count = sum(1 for char in text if char.lower() in kiril_chars)
    lotin_count = sum(1 for char in text if char.lower() in lotin_chars)
    return "kiril" if kiril_count > lotin_count else "lotin"

# Transliteratsiya funksiyasi
def transliterate(text, source_lang, target_lang, source_script=None, target_script=None):
    if source_lang == "tr" and target_lang == "tr":
        return text
    if source_lang not in TRANSLIT_TABLES and source_lang not in ["lotin", "kiril"]:
        return f"Xato: Manba til ({source_lang}) qo‘llab-quvvatlanmaydi."
    if target_lang not in TRANSLIT_TABLES and target_lang not in ["lotin", "kiril"]:
        return f"Xato: Maqsad til ({target_lang}) qo‘llab-quvvatlanmaydi."
    if source_script is None:
        source_script = detect_script(text)
    if target_script is None:
        target_script = "lotin" if target_lang == "tr" else source_script
    if source_lang == target_lang and source_lang in TRANSLIT_TABLES:
        direction = f"{source_script}_to_{target_script}"
        if direction not in ["lotin_to_kiril", "kiril_to_lotin"]:
            return f"Xato: Noto‘g‘ri skript yo‘nalishi ({direction})."
        table = TRANSLIT_TABLES[source_lang][direction]
        for src, dest in sorted(table.items(), key=lambda x: len(x[0]), reverse=True):
            text = text.replace(src, dest)
        return text
    if source_lang in TRANSLIT_TABLES and source_script == "kiril":
        table = TRANSLIT_TABLES[source_lang]["kiril_to_lotin"]
        for src, dest in sorted(table.items(), key=lambda x: len(x[0]), reverse=True):
            text = text.replace(src, dest)
    if target_lang in TRANSLIT_TABLES and target_script == "kiril":
        table = TRANSLIT_TABLES[target_lang]["lotin_to_kiril"]
        for src, dest in sorted(table.items(), key=lambda x: len(x[0]), reverse=True):
            text = text.replace(src, dest)
    return text

# Foydalanuvchilar ma'lumotlari fayli
def load_users_data():
    global users_data
    try:
        with open(USERS_FILE, 'r') as file:
            users_data = json.load(file)
    except (FileNotFoundError, json.JSONDecodeError):
        users_data = {}
        with open(USERS_FILE, 'w') as file:
            json.dump(users_data, file, indent=4)

def save_users_data():
    with open(USERS_FILE, 'w') as file:
        json.dump(users_data, file, indent=4)

def update_user_data(user_id, username, first_name, message=None):
    user_id_str = str(user_id)
    if user_id_str not in users_data:
        users_data[user_id_str] = {
            "username": username or "None",
            "first_name": first_name,
            "messages": []
        }
    if message:
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        users_data[user_id_str]["messages"].append({
            "text": message,
            "timestamp": timestamp
        })
    save_users_data()

# Gemini API so‘rovi
def query_gemini(text):
    try:
        headers = {
            "Content-Type": "application/json",
            "x-goog-api-key": GEMINI_API_KEY
        }
        payload = {
            "contents": [{
                "parts": [{"text": text}]
            }]
        }
        response = requests.post(GEMINI_API_URL, headers=headers, json=payload, timeout=10)
        response.raise_for_status()
        result = response.json()
        if "candidates" in result and result["candidates"]:
            return result["candidates"][0]["content"]["parts"][0]["text"]
        return "Javob topilmadi: Gemini API’dan kutilgan formatda ma’lumot kelmadi."
    except requests.exceptions.HTTPError as http_err:
        return f"Gemini xatosi: HTTP {http_err}"
    except requests.exceptions.ConnectionError:
        return "Gemini xatosi: Internet aloqangizni tekshiring."
    except requests.exceptions.Timeout:
        return "Gemini xatosi: Server sekin javob qaytarmoqda."
    except requests.exceptions.RequestException as e:
        return f"Gemini xatosi: {str(e)}"

# MyMemory API so‘rovi
def mymemory_translate(text, source_lang, target_lang):
    try:
        params = {
            "q": text,
            "langpair": f"{source_lang}|{target_lang}"
        }
        response = requests.get(MYMEMORY_API_URL, params=params, timeout=10)
        response.raise_for_status()
        result = response.json()
        if result["responseStatus"] == 200 and "translatedText" in result["responseData"]:
            return result["responseData"]["translatedText"]
        return "Tarjima xatosi: API’dan kutilgan formatda ma’lumot kelmadi."
    except requests.exceptions.HTTPError as http_err:
        return f"Tarjima xatosi: HTTP {http_err}"
    except requests.exceptions.ConnectionError:
        return "Tarjima xatosi: Internet aloqangizni tekshiring."
    except requests.exceptions.Timeout:
        return "Tarjima xatosi: MyMemory serveri sekin javob qaytarmoqda."
    except requests.exceptions.RequestException as e:
        return f"Tarjima xatosi: {str(e)}"

# RSA funksiyalari
def is_coprime(a, b):
    return math.gcd(a, b) == 1

def is_prime(n):
    if n <= 1:
        return False
    if n <= 3:
        return True
    if n % 2 == 0 or n % 3 == 0:
        return False
    i = 5
    while i * i <= n:
        if n % i == 0 or n % (i + 2) == 0:
            return False
        i += 6
    return True

def mod_inverse(e, phi):
    for d in range(3, phi):
        if (d * e) % phi == 1:
            return d
    raise ValueError("mod_inverse topilmadi")

def generate_keys(p, q):
    if not is_prime(p) or not is_prime(q):
        raise ValueError("p va q tub sonlar bo'lishi kerak!")
    if p == q:
        raise ValueError("p va q bir xil bo'lmasligi kerak!")
    n = p * q
    phi = (p - 1) * (q - 1)
    e = 65537
    if not is_coprime(e, phi):
        e = 3
        while e < phi:
            if is_coprime(e, phi):
                break
            e += 2
    d = mod_inverse(e, phi)
    return (e, n), (d, n)

def encrypt(public_key, plaintext):
    e, n = public_key
    cipher = [pow(ord(char), e, n) for char in plaintext]
    return cipher

def decrypt(private_key, ciphertext):
    d, n = private_key
    cipher_list = list(map(int, ciphertext.split(',')))
    plain = [chr(pow(char, d, n)) for char in cipher_list]
    return ''.join(plain)

# Inline keyboard menyulari
def main_menu():
    keyboard = [
        [
            InlineKeyboardButton("🤖 Yordamchi Chat", callback_data='chat'),
            InlineKeyboardButton("📞 Aloqa", callback_data='aloqa'),
            InlineKeyboardButton("📚 OSI Modeli", callback_data='osi')
        ],
        [
            InlineKeyboardButton("📚 TCP/IP Modeli", callback_data='tcpip'),
            InlineKeyboardButton("🔒 RSA Shifrlash", callback_data='rsa'),
            InlineKeyboardButton("🌐 Tarjimon", callback_data='tarjima')
        ],
        [
            InlineKeyboardButton("🔤 Transliteratsiya", callback_data='translit')
        ]
    ]
    return InlineKeyboardMarkup(keyboard)

def osi_menu():
    keyboard = [
        [
            InlineKeyboardButton("1. Application", callback_data='osi_1'),
            InlineKeyboardButton("2. Presentation", callback_data='osi_2'),
            InlineKeyboardButton("3. Session", callback_data='osi_3')
        ],
        [
            InlineKeyboardButton("4. Transport", callback_data='osi_4'),
            InlineKeyboardButton("5. Network", callback_data='osi_5'),
            InlineKeyboardButton("6. Data Link", callback_data='osi_6')
        ],
        [
            InlineKeyboardButton("7. Physical", callback_data='osi_7'),
            InlineKeyboardButton("Orqaga", callback_data='main')
        ]
    ]
    return InlineKeyboardMarkup(keyboard)

def tcpip_menu():
    keyboard = [
        [
            InlineKeyboardButton("1. Ilova", callback_data='tcpip_1'),
            InlineKeyboardButton("2. Transport", callback_data='tcpip_2')
        ],
        [
            InlineKeyboardButton("3. Internet", callback_data='tcpip_3'),
            InlineKeyboardButton("4. Ulanish", callback_data='tcpip_4')
        ],
        [InlineKeyboardButton("Orqaga", callback_data='main')]
    ]
    return InlineKeyboardMarkup(keyboard)

def tarjima_menu():
    keyboard = [
        [
            InlineKeyboardButton(f"{FLAGS['uz']} O‘zbek → 🇷🇺 Rus", callback_data='tarjima_uz_ru'),
            InlineKeyboardButton(f"🇷🇺 Rus → {FLAGS['uz']} O‘zbek", callback_data='tarjima_ru_uz')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['uz']} O‘zbek → 🇸🇦 Arab", callback_data='tarjima_uz_ar'),
            InlineKeyboardButton(f"🇸🇦 Arab → {FLAGS['uz']} O‘zbek", callback_data='tarjima_ar_uz')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['uz']} O‘zbek → 🇬🇧 Ingliz", callback_data='tarjima_uz_en'),
            InlineKeyboardButton(f"🇬🇧 Ingliz → {FLAGS['uz']} O‘zbek", callback_data='tarjima_en_uz')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['uz']} O‘zbek → {FLAGS['tr']} Turk", callback_data='tarjima_uz_tr'),
            InlineKeyboardButton(f"{FLAGS['tr']} Turk → {FLAGS['uz']} O‘zbek", callback_data='tarjima_tr_uz')
        ],
        [InlineKeyboardButton("🔙 Orqaga", callback_data='main')]
    ]
    return InlineKeyboardMarkup(keyboard)

def translit_menu():
    keyboard = [
        [
            InlineKeyboardButton(f"{FLAGS['uz']} Uzbek Lotin → Kiril", callback_data='translit_uz_lotin_kiril'),
            InlineKeyboardButton(f"{FLAGS['uz']} Uzbek Kiril → Lotin", callback_data='translit_uz_kiril_lotin')
        ],
        [
            InlineKeyboardButton(f"Lotin → {FLAGS['kk']} Qozoq", callback_data='translit_lotin_kk'),
            InlineKeyboardButton(f"{FLAGS['kk']} Qozoq → Lotin", callback_data='translit_kk_lotin')
        ],
        [
            InlineKeyboardButton(f"Kiril → {FLAGS['kk']} Qozoq", callback_data='translit_kiril_kk'),
            InlineKeyboardButton(f"{FLAGS['kk']} Qozoq → Kiril", callback_data='translit_kk_kiril')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['az']} Ozarbayjon → Lotin", callback_data='translit_az_lotin'),
            InlineKeyboardButton(f"Lotin → {FLAGS['az']} Ozarbayjon", callback_data='translit_lotin_az')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['az']} Ozarbayjon → Kiril", callback_data='translit_az_kiril'),
            InlineKeyboardButton(f"{FLAGS['kk']} Qozoq → {FLAGS['az']} Ozarbayjon", callback_data='translit_kk_az')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['az']} Ozarbayjon → {FLAGS['kk']} Qozoq", callback_data='translit_az_kk'),
            InlineKeyboardButton(f"{FLAGS['ky']} Qirg‘iz → Lotin", callback_data='translit_ky_lotin')
        ],
        [
            InlineKeyboardButton(f"Lotin → {FLAGS['ky']} Qirg‘iz", callback_data='translit_lotin_ky'),
            InlineKeyboardButton(f"{FLAGS['ky']} Qirg‘iz → Kiril", callback_data='translit_ky_kiril')
        ],
        [
            InlineKeyboardButton(f"Kiril → {FLAGS['ky']} Qirg‘iz", callback_data='translit_kiril_ky'),
            InlineKeyboardButton(f"{FLAGS['ky']} Qirg‘iz → {FLAGS['kk']} Qozoq", callback_data='translit_ky_kk')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['kk']} Qozoq → {FLAGS['ky']} Qirg‘iz", callback_data='translit_kk_ky'),
            InlineKeyboardButton(f"{FLAGS['ky']} Qirg‘iz → {FLAGS['az']} Ozarbayjon", callback_data='translit_ky_az')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['az']} Ozarbayjon → {FLAGS['ky']} Qirg‘iz", callback_data='translit_az_ky'),
            InlineKeyboardButton(f"Lotin → {FLAGS['tr']} Turk", callback_data='translit_lotin_tr')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['tr']} Turk → Lotin", callback_data='translit_tr_lotin'),
            InlineKeyboardButton(f"{FLAGS['tr']} Turk → {FLAGS['az']} Ozarbayjon", callback_data='translit_tr_az')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['az']} Ozarbayjon → {FLAGS['tr']} Turk", callback_data='translit_az_tr'),
            InlineKeyboardButton(f"{FLAGS['tr']} Turk → {FLAGS['uz']} Uzbek", callback_data='translit_tr_uz')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['uz']} Uzbek → {FLAGS['tr']} Turk", callback_data='translit_uz_tr'),
            InlineKeyboardButton(f"{FLAGS['tr']} Turk → {FLAGS['kk']} Qozoq", callback_data='translit_tr_kk')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['kk']} Qozoq → {FLAGS['tr']} Turk", callback_data='translit_kk_tr'),
            InlineKeyboardButton(f"{FLAGS['tr']} Turk → {FLAGS['ky']} Qirg‘iz", callback_data='translit_tr_ky')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['ky']} Qirg‘iz → {FLAGS['tr']} Turk", callback_data='translit_ky_tr'),
            InlineKeyboardButton(f"Lotin → {FLAGS['tt']} Tatar", callback_data='translit_lotin_tt')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['tt']} Tatar → Lotin", callback_data='translit_tt_lotin'),
            InlineKeyboardButton(f"{FLAGS['tt']} Tatar → Kiril", callback_data='translit_tt_kiril')
        ],
        [
            InlineKeyboardButton(f"Kiril → {FLAGS['tt']} Tatar", callback_data='translit_kiril_tt'),
            InlineKeyboardButton(f"{FLAGS['tt']} Tatar → {FLAGS['kk']} Qozoq", callback_data='translit_tt_kk')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['kk']} Qozoq → {FLAGS['tt']} Tatar", callback_data='translit_kk_tt'),
            InlineKeyboardButton(f"{FLAGS['tt']} Tatar → {FLAGS['uz']} Uzbek", callback_data='translit_tt_uz')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['uz']} Uzbek → {FLAGS['tt']} Tatar", callback_data='translit_uz_tt'),
            InlineKeyboardButton(f"{FLAGS['tt']} Tatar → {FLAGS['ky']} Qirg‘iz", callback_data='translit_tt_ky')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['ky']} Qirg‘iz → {FLAGS['tt']} Tatar", callback_data='translit_ky_tt'),
            InlineKeyboardButton(f"{FLAGS['tt']} Tatar → {FLAGS['az']} Ozarbayjon", callback_data='translit_tt_az')
        ],
        [
            InlineKeyboardButton(f"{FLAGS['az']} Ozarbayjon → {FLAGS['tt']} Tatar", callback_data='translit_az_tt'),
            InlineKeyboardButton("🔙 Orqaga", callback_data='main')
        ]
    ]
    return InlineKeyboardMarkup(keyboard)

def admin_menu():
    keyboard = [
        [
            InlineKeyboardButton("Foydalanuvchilar ro‘yxati", callback_data='admin_users'),
            InlineKeyboardButton("Ko‘rsatmalar", callback_data='admin_help')
        ],
        [InlineKeyboardButton("Orqaga", callback_data='main')]
    ]
    return InlineKeyboardMarkup(keyboard)

# Telegram bot buyrug‘lari
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    update_user_data(user.id, user.username, user.first_name)
    await update.message.reply_text(
        "Salom! Aziz yordamchi botiga xush kelibsiz.\nQuyidagi bo‘limlardan birini tanlang:",
        reply_markup=main_menu()
    )
    context.user_data['state'] = None

async def aziz(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Admin panel uchun parolni kiriting:")
    context.user_data['state'] = 'admin_parol_kutish'

async def tcpip(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "📚 TCP/IP Modeli Bo‘limi\nQuyidagi qatlamlardan birini tanlang:",
        reply_markup=tcpip_menu()
    )

async def button_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    user = update.effective_user
    update_user_data(user.id, user.username, user.first_name)

    if query.data == 'chat':
        await query.message.reply_text(
            "Assalomu alekum! Aziz yordamchi chatiga xush kelibsiz!\nSavol yoki suhbat uchun xabar yuboring (masalan, 'Salom, sen kimsan?').",
            reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("Orqaga", callback_data='main')]])
        )
        context.user_data['state'] = 'chat'

    elif query.data == 'aloqa':
        keyboard = InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton("📢 Telegram Kanalimiz", url='https://t.me/aziza_music_uz')],
            [InlineKeyboardButton("🔙 Orqaga", callback_data='main')]
        ])
        await query.message.reply_text(
            "📞 Aloqa uchun:\nTelefon: [+998907147656](tel:+998907147656)\nTelegram: @Azizamusiccontactbot\nKanalimizga qo‘shiling!",
            reply_markup=keyboard,
            parse_mode="Markdown"
        )

    elif query.data.startswith('osi_'):
        choice = int(query.data.split('_')[1])
        layer = get_layer_info(choice)
        response = (
            f"Qatlam: {layer['name']}\n"
            f"Tavsif: {layer['description']}\n"
            f"Protokollar: {layer['protocols']}\n"
            f"Qurilmalar: {layer['devices']}\n"
            f"Misol: {layer['example']}"
        )
        await query.message.reply_text(response, reply_markup=osi_menu())

    elif query.data == 'osi':
        await query.message.reply_text("📚 OSI Modeli Bo‘limi\nQuyidagi qatlamlardan birini tanlang:", reply_markup=osi_menu())

    elif query.data.startswith('tcpip_'):
        choice = int(query.data.split('_')[1])
        layer = TCPIP_LAYERS.get(choice)
        response = (
            f"Qatlam: {layer['name']}\n"
            f"Tavsif: {layer['description']}\n"
            f"Protokollar: {layer['protocols']}\n"
            f"Qurilmalar: {layer['devices']}\n"
            f"Misol: {layer['example']}"
        )
        await query.message.reply_text(response, reply_markup=tcpip_menu())

    elif query.data == 'tcpip':
        await query.message.reply_text(
            "📚 TCP/IP Modeli Bo‘limi\nQuyidagi qatlamlardan birini tanlang:",
            reply_markup=tcpip_menu()
        )

    elif query.data == 'rsa':
        await query.message.reply_text(
            "🔒 RSA Shifrlash Bo‘limi\n"
            "Avval shifrlamoqchi bo'lgan matningizni yuboring (lotin harflari, masalan, SALOM).",
            reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("Orqaga", callback_data='main')]])
        )
        context.user_data['state'] = 'rsa_matn_kutish'

    elif query.data == 'tarjima':
        await query.message.reply_text("🌐 Tarjimon Bo‘limi\nTarjima yo‘nalishini tanlang:", reply_markup=tarjima_menu())

    elif query.data == 'translit':
        await query.message.reply_text("🔤 Transliteratsiya Bo‘limi\nYo‘nalishni tanlang:", reply_markup=translit_menu())

    elif query.data.startswith('tarjima_'):
        state_map = {
            'tarjima_uz_ru': ('uz', 'ru'), 'tarjima_ru_uz': ('ru', 'uz'),
            'tarjima_uz_ar': ('uz', 'ar'), 'tarjima_ar_uz': ('ar', 'uz'),
            'tarjima_uz_en': ('uz', 'en'), 'tarjima_en_uz': ('en', 'uz'),
            'tarjima_uz_tr': ('uz', 'tr'), 'tarjima_tr_uz': ('tr', 'uz')
        }
        source_lang, target_lang = state_map[query.data]
        await query.message.reply_text(
            f"Matnni yuboring ({source_lang.upper()} → {target_lang.upper()}):\n"
            "Har bir yuborilgan matn avtomatik tarjima qilinadi.",
            reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 Orqaga", callback_data='tarjima')]])
        )
        context.user_data['state'] = query.data

    elif query.data.startswith('translit_'):
        parts = query.data.split('_')
        source = parts[1]
        target = parts[2]
        source_script = None
        target_script = None
        if source == "uz":
            source_script = "lotin" if target == "kiril" else "kiril" if target == "lotin" else None
            target_script = "kiril" if target == "kiril" else "lotin" if target == "lotin" else None
        elif source == "lotin":
            source_script = "lotin"
        elif source == "kiril":
            source_script = "kiril"
        elif target == "lotin":
            target_script = "lotin"
        elif target == "kiril":
            target_script = "kiril"
        else:
            source_script = detect_script(query.message.text) if query.message.text else "lotin"
            target_script = "lotin" if target == "tr" else source_script
        context.user_data['translit_source'] = source
        context.user_data['translit_target'] = target
        context.user_data['translit_source_script'] = source_script
        context.user_data['translit_target_script'] = target_script
        source_flag = FLAGS.get(source, "") if source in FLAGS else ""
        target_flag = FLAGS.get(target, "") if target in FLAGS else ""
        await query.message.reply_text(
            f"Matnni yuboring ({source_flag} {source.upper()} → {target_flag} {target.upper()}):\n"
            "Masalan, o‘zbek tilida: 'o‘zbek' yoki 'ўзбек'.",
            reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 Orqaga", callback_data='translit')]])
        )
        context.user_data['state'] = f'translit_{source}_{target}'

    elif query.data == 'main':
        await query.message.reply_text("Aziz yordamchi botiga qaytdingiz:", reply_markup=main_menu())
        context.user_data['state'] = None

    elif query.data == 'admin_users':
        load_users_data()
        if not users_data:
            await query.message.reply_text("Foydalanuvchilar ro'yxati bo'sh.", reply_markup=admin_menu())
        else:
            result = "Foydalanuvchilar ro'yxati:\n\n"
            for user_id, user_info in users_data.items():
                result += f"ID: {user_id}\n"
                result += f"Username: @{user_info['username']}\n"
                result += f"Ismi: {user_info['first_name']}\n"
                result += f"Xabarlar soni: {len(user_info['messages'])}\n"
                if user_info['messages']:
                    result += "Oxirgi xabarlar:\n"
                    for msg in user_info['messages'][-5:]:
                        result += f"- {msg['timestamp']}: {msg['text']}\n"
                result += "\n" + "-" * 30 + "\n"
            if len(result) > 4000:
                chunks = [result[i:i + 4000] for i in range(0, len(result), 4000)]
                for chunk in chunks:
                    await query.message.reply_text(chunk, reply_markup=admin_menu())
            else:
                await query.message.reply_text(result, reply_markup=admin_menu())

    elif query.data == 'admin_help':
        await query.message.reply_text(
            "Admin Ko‘rsatmalari:\n"
            "- Foydalanuvchilar ro‘yxati: Barcha foydalanuvchilar va ularning xabarlarini ko‘rish.\n"
            "- Parol: Admin paneliga kirish uchun parol (standart: 12345678).\n"
            "- Ma’lumotlar saqlash: Foydalanuvchi xabarlari 'users_data.json' faylida saqlanadi.\n"
            "- Bo‘limlar: Yordamchi Chat, Aloqa, OSI Modeli, TCP/IP Modeli, RSA Shifrlash, Tarjimon, Transliteratsiya.\n"
            "- Aloqa: Telefon: +998907147656, Telegram: @Azizamusiccontactbot.",
            reply_markup=admin_menu()
        )

async def shifrlash(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if 'message' not in context.user_data:
        await update.message.reply_text("Avval matn yuboring! Matn kiritish uchun /rsa bo‘limiga o‘ting.")
        return
    await update.message.reply_text("p tub sonini kiriting (masalan, 1009):")
    context.user_data['state'] = 'rsa_p_kutish'

async def deshifrlash(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Shifrlangan matnni yuboring (raqamlar vergul bilan ajratilgan, masalan: 1234,4567,2345):"
    )
    context.user_data['state'] = 'rsa_shifrlangan_matn_kutish'

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    state = context.user_data.get('state')
    message = update.message.text
    user = update.effective_user
    update_user_data(user.id, user.username, user.first_name, message)

    if state == 'admin_parol_kutish':
        if message == "12345678":
            await update.message.reply_text("Admin panel ochildi. Amalni tanlang:", reply_markup=admin_menu())
            context.user_data['state'] = None
        else:
            await update.message.reply_text("Parol noto'g'ri! Admin panel yopildi.")
            context.user_data['state'] = None
        return

    if state == 'chat':
        answer = query_gemini(message)
        if "Gemini xatosi" in answer:
            await update.message.reply_text(
                f"Xato: {answer}\nIltimos, internet aloqangizni tekshiring yoki qayta urinib ko‘ring.",
                reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("Orqaga", callback_data='main')]])
            )
            return
        await update.message.reply_text(
            f"🌟 Javob: {answer}\n\nYana savol yuboring yoki orqaga qayting:",
            reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("Orqaga", callback_data='main')]])
        )

    elif state == 'rsa_matn_kutish':
        context.user_data['message'] = message.upper()
        await update.message.reply_text(
            f"Matn qabul qilindi: {message.upper()}\n"
            "Endi /shifrlash buyrug‘ini yuboring.",
            reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("Orqaga", callback_data='main')]])
        )
        context.user_data['state'] = None

    elif state == 'rsa_p_kutish':
        try:
            p = int(message)
            if not is_prime(p):
                await update.message.reply_text(f"{p} tub son emas! Iltimos, tub son kiriting.")
                return
            context.user_data['p'] = p
            await update.message.reply_text(
                f"{p} tub son. Endi q tub sonini kiriting (masalan, 1013):",
                reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("Orqaga", callback_data='rsa')]])
            )
            context.user_data['state'] = 'rsa_q_kutish'
        except ValueError:
            await update.message.reply_text("Xatolik: Son kiriting!")

    elif state == 'rsa_q_kutish':
        try:
            q = int(message)
            if not is_prime(q):
                await update.message.reply_text(f"{q} tub son emas! Iltimos, tub son kiriting.")
                return
            p = context.user_data['p']
            if p == q:
                await update.message.reply_text("p va q bir xil bo‘lmasligi kerak! Boshqa q kiriting.")
                return
            public_key, private_key = generate_keys(p, q)
            plaintext = context.user_data['message']
            encrypted_msg = encrypt(public_key, plaintext)
            response = (
                f"p={p}, q={q}\n"
                f"Ochiq kalit (e, n): {public_key}\n"
                f"Maxfiy kalit (d, n): {private_key}\n"
                f"Asl xabar: {plaintext}\n"
                f"Shifrlangan xabar: {', '.join(map(str, encrypted_msg))}\n"
                "Shifrdan ochish uchun /deshifrlash buyrug‘ini ishlating."
            )
            context.user_data['state'] = None
            context.user_data.pop('message', None)
            context.user_data.pop('p', None)
            await update.message.reply_text(response, reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("Orqaga", callback_data='main')]]))
        except ValueError as e:
            await update.message.reply_text(f"Xatolik: {str(e)}")

    elif state == 'rsa_shifrlangan_matn_kutish':
        try:
            context.user_data['cipher_text'] = message
            await update.message.reply_text(
                "Maxfiy kalitni kiriting (d, n formatida, masalan: 3473,5963):",
                reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("Orqaga", callback_data='rsa')]])
            )
            context.user_data['state'] = 'rsa_kalit_kutish'
        except Exception as e:
            await update.message.reply_text(f"Xatolik: {str(e)}")

    elif state == 'rsa_kalit_kutish':
        try:
            d, n = map(int, message.split(','))
            private_key = (d, n)
            cipher_text = context.user_data['cipher_text']
            decrypted_msg = decrypt(private_key, cipher_text)
            response = f"Shifrdan ochilgan xabar: {decrypted_msg}"
            context.user_data['state'] = None
            context.user_data.pop('cipher_text', None)
            await update.message.reply_text(response, reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("Orqaga", callback_data='main')]]))
        except Exception as e:
            await update.message.reply_text(f"Xatolik: {str(e)}")

    elif state.startswith('tarjima_'):
        state_map = {
            'tarjima_uz_ru': ('uz', 'ru'), 'tarjima_ru_uz': ('ru', 'uz'),
            'tarjima_uz_ar': ('uz', 'ar'), 'tarjima_ar_uz': ('ar', 'uz'),
            'tarjima_uz_en': ('uz', 'en'), 'tarjima_en_uz': ('en', 'uz'),
            'tarjima_uz_tr': ('uz', 'tr'), 'tarjima_tr_uz': ('tr', 'uz')
        }
        source_lang, target_lang = state_map[state]
        translated = mymemory_translate(message, source_lang, target_lang)
        source_flag = FLAGS.get(source_lang, "") if source_lang in FLAGS else ""
        target_flag = FLAGS.get(target_lang, "") if target_lang in FLAGS else ""
        await update.message.reply_text(
            f"{source_flag} ➡️ {target_flag} Tarjima: {translated}\n\nYana matn yuboring yoki orqaga qayting:",
            reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 Orqaga", callback_data='tarjima')]])
        )

    elif state.startswith('translit_'):
        source = context.user_data.get('translit_source')
        target = context.user_data.get('translit_target')
        source_script = context.user_data.get('translit_source_script')
        target_script = context.user_data.get('translit_target_script')
        result = transliterate(message, source, target, source_script, target_script)
        source_flag = FLAGS.get(source, "") if source in FLAGS else ""
        target_flag = FLAGS.get(target, "") if target in FLAGS else ""
        await update.message.reply_text(
            f"{source_flag} ➡️ {target_flag} Natija: {result}\n\nYana matn yuboring yoki orqaga qayting:",
            reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 Orqaga", callback_data='translit')]])
        )

def main():
    load_users_data()
    application = Application.builder().token(TELEGRAM_TOKEN).build()

    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("Aziz", aziz))
    application.add_handler(CommandHandler("tcpip", tcpip))
    application.add_handler(CommandHandler("shifrlash", shifrlash))
    application.add_handler(CommandHandler("deshifrlash", deshifrlash))
    application.add_handler(CallbackQueryHandler(button_callback))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

    print("Bot ishga tushdi...")
    application.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()
