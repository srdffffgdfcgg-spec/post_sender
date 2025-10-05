#!/usr/bin/env python3
# -*- coding: utf-8 -*-

"""
Простой Gmail клиент:
- Отправка письма (SMTP)
- Чтение последних N писем (IMAP)
- Сохранение вложений в текущую директорию
Настройки: EMAIL, APP_PASSWORD
"""

import imaplib
import smtplib
import email
import os
import ssl
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from email.mime.application import MIMEApplication
from email.header import decode_header, make_header
from pathlib import Path

# ====== НАСТРОЙКИ: вставь свои данные ======
EMAIL = "мій_емеіл@gmail.com"         # Твой Gmail
APP_PASSWORD = "Отримати після встановлення двохетапноїперевірки в налаштуваннях безпеки"      # 16-значный app password (без пробелов)
# =========================================

SMTP_SERVER = "smtp.gmail.com"
SMTP_PORT = 587
IMAP_SERVER = "imap.gmail.com"
ATTACH_SAVE_DIR = Path.cwd() / "attachments"
ATTACH_SAVE_DIR.mkdir(exist_ok=True)

# ---------- ОТПРАВКА ПИСЬМА ----------
def send_email(to_address: str, subject: str, body: str, attachments: list[str] = None):
    attachments = attachments or []
    msg = MIMEMultipart()
    msg["From"] = EMAIL
    msg["To"] = to_address
    msg["Subject"] = subject

    msg.attach(MIMEText(body, "plain", "utf-8"))

    # добавляем вложения
    for file_path in attachments:
        file_path = Path(file_path)
        if not file_path.exists():
            print(f"⚠️ Вложение не найдено: {file_path}")
            continue
        part = MIMEApplication(file_path.read_bytes(), Name=file_path.name)
        part['Content-Disposition'] = f'attachment; filename="{file_path.name}"'
        msg.attach(part)

    context = ssl.create_default_context()
    with smtplib.SMTP(SMTP_SERVER, SMTP_PORT) as server:
        server.ehlo()
        server.starttls(context=context)
        server.ehlo()
        server.login(EMAIL, APP_PASSWORD)
        server.send_message(msg)
    print("✅ Письмо отправлено на", to_address)


# ---------- ЧТЕНИЕ ВХОДЯЩИХ (IMAP) ----------
def decode_mime_words(s):
    if not s:
        return ""
    return str(make_header(decode_header(s)))

def save_attachment(part, prefix="attachment"):
    filename = part.get_filename()
    if not filename:
        filename = f"{prefix}"
    filename = decode_mime_words(filename)
    safe_name = "".join(c for c in filename if c.isprintable())
    out_path = ATTACH_SAVE_DIR / safe_name
    counter = 1
    while out_path.exists():
        out_path = ATTACH_SAVE_DIR / f"{out_path.stem}_{counter}{out_path.suffix}"
        counter += 1
    with open(out_path, "wb") as f:
        f.write(part.get_payload(decode=True))
    return out_path

def read_inbox(limit=10, unseen_only=False, mark_as_read=False):
    mail = imaplib.IMAP4_SSL(IMAP_SERVER)
    mail.login(EMAIL, APP_PASSWORD)
    mail.select("inbox")

    criterion = 'UNSEEN' if unseen_only else 'ALL'
    typ, data = mail.search(None, criterion)
    if typ != 'OK':
        print("Ошибка поиска писем:", typ)
        return

    mail_ids = data[0].split()
    if not mail_ids:
        print("Входящих писем не найдено.")
        return

    # берём последние limit писем
    selected_ids = mail_ids[-limit:]

    for num in reversed(selected_ids):
        typ, msg_data = mail.fetch(num, '(RFC822)')
        if typ != 'OK':
            print("Ошибка fetch:", typ)
            continue
        raw = msg_data[0][1]
        msg = email.message_from_bytes(raw)

        from_ = decode_mime_words(msg.get("From"))
        subject = decode_mime_words(msg.get("Subject"))
        date_ = msg.get("Date")

        print(f"\n--- Письмо ID {num.decode()} ---")
        print("От:", from_)
        print("Тема:", subject)
        print("Дата:", date_)

        # тело + вложения
        body_text = ""
        attachments_saved = []
        if msg.is_multipart():
            for part in msg.walk():
                ctype = part.get_content_type()
                disp = str(part.get("Content-Disposition"))
                if part.is_multipart():
                    continue
                if ctype == "text/plain" and "attachment" not in disp:
                    payload = part.get_payload(decode=True)
                    if payload:
                        try:
                            body_text += payload.decode(part.get_content_charset() or "utf-8", errors="replace")
                        except Exception:
                            body_text += payload.decode("utf-8", errors="replace")
                elif "attachment" in disp or part.get_filename():
                    saved = save_attachment(part, prefix=f"attach_{num.decode()}")
                    attachments_saved.append(saved)
        else:
            payload = msg.get_payload(decode=True)
            if payload:
                body_text = payload.decode(msg.get_content_charset() or "utf-8", errors="replace")

        print("Текст (первые 800 символов):")
        print(body_text[:800] + ("..." if len(body_text) > 800 else ""))
        if attachments_saved:
            print("📎 Сохранены вложения:")
            for p in attachments_saved:
                print("  -", p)
        else:
            print("Вложений нет.")

        # пометить как прочитанное?
        if mark_as_read:
            mail.store(num, '+FLAGS', '\\Seen')

    mail.logout()

# ---------- ПРИМЕР ВЗАИМОДЕЙСТВИЯ (CLI) ----------
def main():
    print("Gmail client — отправка/чтение писем")
    print("1 - Отправить письмо")
    print("2 - Прочитать входящие (последние N)")
    print("3 - Прочитать только непрочитанные")
    print("q - Выход")
    choice = input("Выбор: ").strip().lower()
    if choice == "1":
        to_addr = input("Кому (email): ").strip()
        subj = input("Тема: ").strip()
        print("Введите текст (заканчивается строкой с одним 'END'):")
        lines = []
        while True:
            ln = input()
            if ln.strip() == "END":
                break
            lines.append(ln)
        body = "\n".join(lines)
        attach_input = input("Файлы для вложения (через запятую), пусто — без вложений: ").strip()
        attachments = [p.strip() for p in attach_input.split(",")] if attach_input else []
        send_email(to_addr, subj, body, attachments)
    elif choice == "2":
        n = input("Сколько последних писем показать? (по умолчанию 10): ").strip()
        n = int(n) if n.isdigit() and int(n) > 0 else 10
        read_inbox(limit=n, unseen_only=False, mark_as_read=False)
    elif choice == "3":
        n = input("Сколько последних непрочитанных показать? (по умолчанию 10): ").strip()
        n = int(n) if n.isdigit() and int(n) > 0 else 10
        read_inbox(limit=n, unseen_only=True, mark_as_read=True)
    else:
        print("Выход.")

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\nОтмена пользователем.")
