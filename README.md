import telebot
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton
import sqlite3

# Tənzimləmələr
TOKEN = '8297268773:AAEPluQ5E-ch_qB99qQ48jbgd5w4Q_sUGio'
ADMIN_ID = 5629917631  
CHANNEL_USER = '@isaeyc_bots'
CARD_NUMBER = '4098584495015370'

bot = telebot.TeleBot(8297268773:AAEPluQ5E-ch_qB99qQ48jbgd5w4Q_sUGio)

# Verilənlər Bazasının qurulması
conn = sqlite3.connect('isaevyc.db', check_same_thread=False)
cursor = conn.cursor()
cursor.execute('''CREATE TABLE IF NOT EXISTS users (
                    user_id INTEGER PRIMARY KEY,
                    lang TEXT,
                    profession TEXT,
                    is_premium INTEGER DEFAULT 0
                )''')
conn.commit()

# --- YARDIMÇI FUNKSİYALAR ---
def check_membership(user_id):
    try:
        status = bot.get_chat_member(CHANNEL_USER, user_id).status
        return status in ['member', 'administrator', 'creator']
    except:
        return False

def get_user(user_id):
    cursor.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))
    return cursor.fetchone()

# --- START VƏ DİL SEÇİMİ ---
@bot.message_handler(commands=['start'])
def send_welcome(message):
    user_id = message.chat.id
    if not get_user(user_id):
        cursor.execute("INSERT INTO users (user_id, is_premium) VALUES (?, 0)", (user_id,))
        conn.commit()

    markup = InlineKeyboardMarkup()
    markup.add(
        InlineKeyboardButton("🇦🇿 AZ", callback_data="lang_az"),
        InlineKeyboardButton("🇹🇷 TR", callback_data="lang_tr"),
        InlineKeyboardButton("🇷🇺 RU", callback_data="lang_ru"),
        InlineKeyboardButton("🇬🇧 EN", callback_data="lang_en")
    )
    bot.send_message(user_id, "Salam! Mən **isaevyc ai** 😎\nDanışmağa başlamazdan əvvəl dil seç:", reply_markup=markup, parse_mode="Markdown")

# --- KANAL YOXLANIŞI VƏ PEŞƏ SEÇİMİ ---
@bot.callback_query_handler(func=lambda call: call.data.startswith('lang_'))
def set_language(call):
    lang = call.data.split('_')[1]
    cursor.execute("UPDATE users SET lang = ? WHERE user_id = ?", (lang, call.message.chat.id))
    conn.commit()
    
    bot.edit_message_text(f"Dil seçildi! İndi isə qanunlarımıza tabe ol.", chat_id=call.message.chat.id, message_id=call.message.message_id)
    
    # Kanal yoxlanışı
    if not check_membership(call.message.chat.id):
        markup = InlineKeyboardMarkup()
        markup.add(InlineKeyboardButton("📢 Kanala Qoşul", url=f"https://t.me/{CHANNEL_USER[1:]}"))
        markup.add(InlineKeyboardButton("✅ Qoşuldum", callback_data="check_sub"))
        bot.send_message(call.message.chat.id, f"Kiminlə zarafat edirsən? Əvvəl {CHANNEL_USER} kanalına abunə ol, sonra gəl söhbət edək! 😎", reply_markup=markup)
    else:
        ask_profession(call.message.chat.id)

@bot.callback_query_handler(func=lambda call: call.data == "check_sub")
def verify_sub(call):
    if check_membership(call.message.chat.id):
        bot.delete_message(call.message.chat.id, call.message.message_id)
        bot.send_message(call.message.chat.id, "Halaldır, əsl adamımsan! İndi mənim üçün bir xarakter (peşə) seç:")
        ask_profession(call.message.chat.id)
    else:
        bot.answer_callback_query(call.id, "Yalan danışma, hələ qoşulmamısan! 🤨", show_alert=True)

def ask_profession(chat_id):
    markup = InlineKeyboardMarkup(row_width=2)
    markup.add(
        InlineKeyboardButton("Mafiya/Soxucu 😎", callback_data="prof_mafia"),
        InlineKeyboardButton("Ağıllı Professor 🤓", callback_data="prof_prof"),
        InlineKeyboardButton("Psixoloq 🛋️", callback_data="prof_psy"),
        InlineKeyboardButton("Proqramçı 💻", callback_data="prof_dev")
    )
    bot.send_message(chat_id, "Mən səninlə hansı dildə, hansı xarakterdə danışım?", reply_markup=markup)

@bot.callback_query_handler(func=lambda call: call.data.startswith('prof_'))
def set_profession(call):
    prof = call.data.split('_')[1]
    cursor.execute("UPDATE users SET profession = ? WHERE user_id = ?", (prof, call.message.chat.id))
    conn.commit()
    bot.send_message(call.message.chat.id, "Mükəmməl! Artıq söhbətə hazıram. Mənə istədiyini yaza bilərsən.\n\n_Qeyd: Premium almaq üçün /premium yaz._", parse_mode="Markdown")

# --- PREMİUM SİSTEMİ ---
@bot.message_handler(commands=['premium'])
def buy_premium(message):
    text = (f"💎 **isaevyc ai Premium** 💎\n\n"
            f"Premium olanda mən daha ağıllı, daha sürətli və limitsiz oluram!\n\n"
            f"💳 Kart: `{CARD_NUMBER}`\n"
            f"💰 Qiymət: 2 AZN\n\n"
            f"Ödənişi etdikdən sonra qəbzi (skrini) yaradıcıma, yəni @isaevyc -ya at. O sənin hesabını aktivləşdirəcək.")
    bot.send_message(message.chat.id, text, parse_mode="Markdown")

# --- ADMİN PANelİ ---
@bot.message_handler(commands=['stat', 'premium_ver', 'premium_al', 'yolla'])
def admin_commands(message):
    if message.chat.id != ADMIN_ID:
        bot.send_message(message.chat.id, "Sən admin deyilsən, bura burun soxma! 🛑")
        return

    cmd = message.text.split()[0]
    
    if cmd == '/stat':
        cursor.execute("SELECT COUNT(*) FROM users")
        count = cursor.fetchone()[0]
        cursor.execute("SELECT COUNT(*) FROM users WHERE is_premium = 1")
        prem_count = cursor.fetchone()[0]
        bot.send_message(ADMIN_ID, f"📊 Ümumi istifadəçi: {count}\n💎 Premium istifadəçi: {prem_count}")
        
    elif cmd == '/premium_ver':
        try:
            target_id = int(message.text.split()[1])
            cursor.execute("UPDATE users SET is_premium = 1 WHERE user_id = ?", (target_id,))
            conn.commit()
            bot.send_message(ADMIN_ID, f"✅ {target_id} ID-li istifadəçiyə Premium verildi!")
            bot.send_message(target_id, "Təbriklər! @isaevyc tərəfindən sənə 💎 Premium verildi. Artıq limitsizsən!")
        except:
            bot.send_message(ADMIN_ID, "İstifadə forması: /premium_ver 123456789")

    elif cmd == '/premium_al':
        try:
            target_id = int(message.text.split()[1])
            cursor.execute("UPDATE users SET is_premium = 0 WHERE user_id = ?", (target_id,))
            conn.commit()
            bot.send_message(ADMIN_ID, f"❌ {target_id} ID-li istifadəçinin Premiumu alındı!")
            bot.send_message(target_id, "Təəssüf ki, Premium vaxtın bitdi və ya alındı. Yeniləmək üçün /premium yaz.")
        except:
            bot.send_message(ADMIN_ID, "İstifadə forması: /premium_al 123456789")

    elif cmd == '/yolla':
        text_to_send = message.text.replace('/yolla ', '')
        if text_to_send == '/yolla':
            bot.send_message(ADMIN_ID, "Nə göndərim? Mətn yaz. Məsələn: /yolla Salam millət!")
            return
            
        cursor.execute("SELECT user_id FROM users")
        users = cursor.fetchall()
        success = 0
        for u in users:
            try:
                bot.send_message(u[0], f"📢 **isaevyc-dən Mesaj:**\n\n{text_to_send}", parse_mode="Markdown")
                success += 1
            except:
                pass
        bot.send_message(ADMIN_ID, f"Mesaj {success} nəfərə uğurla çatdırıldı!")

# --- Aİ SÖHBƏT HİSSƏSİ (İSTİFADƏÇİ MESAJ YAZANDA) ---
@bot.message_handler(func=lambda message: True)
def ai_chat(message):
    user_id = message.chat.id
    if not check_membership(user_id):
        bot.send_message(user_id, f"Məni aldatmağa çalışma! Əvvəl {CHANNEL_USER} kanalına qoşul!")
        return

    user = get_user(user_id)
    is_premium = user[3]
    profession = user[2]
    
    # BURA ƏSL SÜNİ İNTELLEKT APİ-Sİ QOŞULA BİLƏR (Məsələn: OpenAI / Gemini)
    # Hələlik zarafatcıl və məntiqli cavablar verən sadə bir sistem yazmışıq:
    
    if not is_premium and len(message.text) > 50:
        bot.send_message(user_id, "Kasıb versiyada bu qədər uzun cümlə yazmaq olmaz! 😂 Qısa yaz və ya /premium al.")
        return

    if profession == "mafia":
        bot.send_message(user_id, "Qulaq as, brat. Mənə belə suallar vermə, yoxsa problemi 'obşakda' həll edərik. 😎 Sözün düzü: " + message.text[::-1] + " (Sənə soxucu cavab verdim!)")
    elif profession == "prof":
        bot.send_message(user_id, "Elmi cəhətdən yanaşsaq, sənin dediyin bu fərziyyə tamamilə yanlışdır, gənc dostum. 🤓")
    else:
        bot.send_message(user_id, f"Sən {message.text} dedin, amma mən isaevyc-nin yaratdığı bir süni intellekt olaraq hələ düşünürəm... 🤔")

bot.polling(none_stop=True)




