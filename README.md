AI assistant as-is for monitoring admission applications (based on NCES),... based on dummy NCES data file. 
Results like this one...
--- Trend Analysis ---
Total Application Change (2022 -> 2023): 1500 (3.80%)
Trends by major:
Computer Science: Change 500 (5.26%)
Economics: Change 300 (3.57%)
Biology: Change 200 (2.63%)
Trends by state:
Illinois: Change 500 (4.35%)
California: Change 300 (3.37%)
New York: Change 200 (2.63%).
*** Ukrainian prototype version.
# -*- coding: utf-8 -*-
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from datetime import datetime
import os
import json

# --- Конфігурація ---
# Назва файлу для збереження агрегованих даних (історії)
AGGREGATED_DATA_FILE = 'aggregated_applicant_data.csv'
# Назва папки для збереження діаграм
PLOTS_DIR = 'applicant_plots_nces'
# Назва файлу з даними NCES, який ви завантажите
# ЗМІНІТЬ ЦЕ, ЯКЩО ВАШ ФАЙЛ МАЄ ІНШУ НАЗВУ!
NCES_DATA_FILE = 'nces_admissions_data.csv'
# Назва університету, який ми будемо аналізувати
TARGET_UNIVERSITY = 'University of Chicago'

# --- Функції для отримання та парсингу даних NCES ---

def create_mock_nces_data(filename=NCES_DATA_FILE):
    """
    Створює фіктивний CSV-файл, що імітує дані NCES IPEDS.
    У реальному сценарії ви завантажите цей файл з веб-сайту NCES.
    """
    if os.path.exists(filename):
        print(f"Фіктивний файл '{filename}' вже існує. Пропускаємо створення.")
        return

    print(f"Створення фіктивного файлу NCES даних: {filename}")
    data = {
        'UNITID': [144050, 144050, 144050, 144050, 144050],
        'INSTNM': ['University of Chicago', 'University of Chicago', 'University of Chicago', 'University of Chicago', 'University of Chicago'],
        'APPLCNT': [35000, 36500, 38000, 39500, 41000], # Загальна кількість заявок
        'ADMSSN': [3000, 3100, 3200, 3300, 3400], # Прийняті
        'ENRLT': [2000, 2100, 2200, 2300, 2400], # Зараховані
        'MAJOR_CS_APPL': [8000, 8500, 9000, 9500, 10000], # Фіктивні заявки на Комп'ютерні науки
        'MAJOR_ECON_APPL': [7500, 7800, 8100, 8400, 8700], # Фіктивні заявки на Економіку
        'MAJOR_BIO_APPL': [7000, 7200, 7400, 7600, 7800], # Фіктивні заявки на Біологію
        'STATE_IL_APPL': [10000, 10500, 11000, 11500, 12000], # Фіктивні заявки з Іллінойсу
        'STATE_CA_APPL': [8000, 8300, 8600, 8900, 9200], # Фіктивні заявки з Каліфорнії
        'STATE_NY_APPL': [7000, 7200, 7400, 7600, 7800], # Фіктивні заявки з Нью-Йорка
        'YEAR': [2019, 2020, 2021, 2022, 2023] # Рік даних
    }
    df = pd.DataFrame(data)
    df.to_csv(filename, index=False)
    print(f"Фіктивний файл '{filename}' успішно створено.")

def fetch_and_parse_nces_data(nces_filename=NCES_DATA_FILE, target_university=TARGET_UNIVERSITY):
    """
    Завантажує дані з локального файлу NCES CSV та парсить їх.
    ЦЯ ФУНКЦІЯ ПОВИННА БУТИ АДАПТОВАНА ПІД РЕАЛЬНІ НАЗВИ СТОВПЦІВ У ВАШОМУ ФАЙЛІ NCES!
    """
    if not os.path.exists(nces_filename):
        print(f"Помилка: Файл NCES даних '{nces_filename}' не знайдено.")
        print("Будь ласка, завантажте відповідний CSV-файл з NCES IPEDS та розмістіть його у тій же папці, що й скрипт.")
        print("Або запустіть скрипт, щоб створити фіктивні дані для демонстрації.")
        return None

    print(f"Читання даних з файлу NCES: {nces_filename}")
    try:
        df_nces = pd.read_csv(nces_filename)
    except Exception as e:
        print(f"Помилка при читанні файлу NCES: {e}")
        return None

    # Фільтруємо дані для цільового університету
    df_university = df_nces[df_nces['INSTNM'] == target_university].copy()

    if df_university.empty:
        print(f"Не знайдено даних для '{target_university}' у файлі NCES. Перевірте назву університету або дані.")
        return None

    # Стовпці, які потрібно витягти.
    # ЗМІНІТЬ ЦІ НАЗВИ СТОВПЦІВ ВІДПОВІДНО ДО ВАШОГО РЕАЛЬНОГО ФАЙЛУ NCES!
    total_applicants_col = 'APPLCNT'
    year_col = 'YEAR'

    # Приклад: витягуємо дані про спеціальності та штати з фіктивних стовпців
    # У РЕАЛЬНИХ ДАНИХ NCES ЦІ СТОВПЦІ МОЖУТЬ МАТИ ІНШІ НАЗВИ АБО БУТИ ВІДСУТНІМИ ДЛЯ ЗАЯВОК!
    # Можливо, вам доведеться агрегувати дані з різних таблиць IPEDS.
    majors_data_cols = {
        'Комп\'ютерні науки': 'MAJOR_CS_APPL',
        'Економіка': 'MAJOR_ECON_APPL',
        'Біологія': 'MAJOR_BIO_APPL'
    }
    states_data_cols = {
        'Іллінойс': 'STATE_IL_APPL',
        'Каліфорнія': 'STATE_CA_APPL',
        'Нью-Йорк': 'STATE_NY_APPL'
    }

    parsed_data_list = []
    for index, row in df_university.iterrows():
        year = row[year_col]
        total_applicants = row.get(total_applicants_col, 0) # Використовуємо .get для безпеки

        majors_dist = {name: row.get(col, 0) for name, col in majors_data_cols.items()}
        states_dist = {name: row.get(col, 0) for name, col in states_data_cols.items()}

        parsed_data_list.append({
            'timestamp': datetime(year, 1, 1).isoformat(), # Використовуємо 1 січня для року
            'year': year,
            'total_applicants': total_applicants,
            'majors': majors_dist,
            'states': states_dist
        })
    return parsed_data_list

# --- Функції для збереження та завантаження агрегованих даних ---

def save_aggregated_data(new_data_list, filename=AGGREGATED_DATA_FILE):
    """
    Зберігає нові агреговані дані у CSV-файл. Якщо файл існує, додає нові рядки.
    Складні словники (majors, states) зберігаються як JSON-рядки.
    """
    if not new_data_list:
        print("Немає нових агрегованих даних для збереження.")
        return

    # Перетворюємо словники на JSON-рядки для збереження в CSV
    data_to_save = []
    for item in new_data_list:
        data_to_save.append({
            'timestamp': item['timestamp'],
            'year': item['year'],
            'total_applicants': item['total_applicants'],
            'majors': json.dumps(item['majors'], ensure_ascii=False),
            'states': json.dumps(item['states'], ensure_ascii=False)
        })

    df_new = pd.DataFrame(data_to_save)

    if os.path.exists(filename):
        df_existing = pd.read_csv(filename)
        # Об'єднуємо, уникаючи дублікатів за роком/timestamp
        df_combined = pd.concat([df_existing, df_new]).drop_duplicates(subset=['year']).reset_index(drop=True)
    else:
        df_combined = df_new

    df_combined.to_csv(filename, index=False)
    print(f"Агреговані дані успішно збережено у {filename}")

def load_aggregated_data(filename=AGGREGATED_DATA_FILE):
    """
    Завантажує всі збережені агреговані дані з CSV-файлу.
    JSON-рядки перетворюються назад на словники.
    """
    if not os.path.exists(filename):
        print(f"Файл агрегованих даних '{filename}' не знайдено. Починаємо з чистого аркуша.")
        return pd.DataFrame()

    df = pd.read_csv(filename)
    # Перетворюємо JSON-рядки назад на словники
    if 'majors' in df.columns:
        df['majors'] = df['majors'].apply(lambda x: json.loads(x) if pd.notna(x) else {})
    if 'states' in df.columns:
        df['states'] = df['states'].apply(lambda x: json.loads(x) if pd.notna(x) else {})
    df['timestamp'] = pd.to_datetime(df['timestamp'])
    return df

# --- Функції для аналізу та візуалізації ---

def analyze_trends(df):
    """
    Аналізує динаміку змін та виявляє тенденції.
    """
    if df.empty:
        print("Недостатньо даних для аналізу тенденцій.")
        return

    print("\n--- Аналіз Тенденцій ---")
    df_sorted = df.sort_values(by='year').reset_index(drop=True)

    if len(df_sorted) > 1:
        latest_data = df_sorted.iloc[-1]
        previous_data = df_sorted.iloc[-2]

        # Динаміка загальної кількості
        total_change = latest_data['total_applicants'] - previous_data['total_applicants']
        total_percent_change = (total_change / previous_data['total_applicants']) * 100 if previous_data['total_applicants'] != 0 else 0
        print(f"Зміна загальної кількості заявок ({previous_data['year']} -> {latest_data['year']}): {total_change} ({total_percent_change:.2f}%)")

        # Тенденції за напрямками (показуємо топ-3 зміни)
        print("\nТенденції за напрямками:")
        majors_prev = previous_data['majors']
        majors_curr = latest_data['majors']
        major_changes = {}
        for major, count_curr in majors_curr.items():
            count_prev = majors_prev.get(major, 0)
            change = count_curr - count_prev
            percent_change = (change / count_prev) * 100 if count_prev != 0 else (100 if change > 0 else 0)
            major_changes[major] = {'change': change, 'percent_change': percent_change}

        sorted_major_changes = sorted(major_changes.items(), key=lambda item: abs(item[1]['change']), reverse=True)
        for major, data in sorted_major_changes[:3]:
            print(f"  {major}: Зміна {data['change']} ({data['percent_change']:.2f}%)")

        # Тенденції за штатами (показуємо топ-3 зміни)
        print("\nТенденції за штатами:")
        states_prev = previous_data['states']
        states_curr = latest_data['states']
        state_changes = {}
        for state, count_curr in states_curr.items():
            count_prev = states_prev.get(state, 0)
            change = count_curr - count_prev
            percent_change = (change / count_prev) * 100 if count_prev != 0 else (100 if change > 0 else 0)
            state_changes[state] = {'change': change, 'percent_change': percent_change}

        sorted_state_changes = sorted(state_changes.items(), key=lambda item: abs(item[1]['change']), reverse=True)
        for state, data in sorted_state_changes[:3]:
            print(f"  {state}: Зміна {data['change']} ({data['percent_change']:.2f}%)")
    else:
        print("Потрібно більше даних (мінімум 2 записи) для розрахунку тенденцій.")

def visualize_data(df, plots_dir=PLOTS_DIR):
    """
    Будує діаграми на основі зібраних даних.
    """
    if df.empty:
        print("Немає даних для візуалізації.")
        return

    os.makedirs(plots_dir, exist_ok=True)
    print(f"\nЗбереження діаграм у папку: {plots_dir}")

    # 1. Динаміка загальної кількості заявок
    plt.figure(figsize=(12, 6))
    sns.lineplot(x='year', y='total_applicants', data=df, marker='o')
    plt.title('Динаміка Загальної Кількість Заявок (за даними NCES)', fontsize=16)
    plt.xlabel('Рік', fontsize=12)
    plt.ylabel('Кількість Заявок', fontsize=12)
    plt.grid(True)
    plt.xticks(rotation=45)
    plt.tight_layout()
    plt.savefig(os.path.join(plots_dir, 'total_applicants_dynamics_nces.png'))
    plt.close()
    print("  - Збережено: total_applicants_dynamics_nces.png")

    # 2. Розподіл за напрямками навчання (для останнього року)
    latest_majors = df.iloc[-1]['majors']
    if latest_majors:
        majors_df = pd.DataFrame(list(latest_majors.items()), columns=['Напрямок', 'Кількість'])
        plt.figure(figsize=(12, 7))
        sns.barplot(x='Кількість', y='Напрямок', data=majors_df.sort_values(by='Кількість', ascending=False), palette='viridis')
        plt.title(f"Розподіл Заявок за Напрямками Навчання ({df.iloc[-1]['year']} рік)", fontsize=16)
        plt.xlabel('Кількість Заявок', fontsize=12)
        plt.ylabel('Напрямок', fontsize=12)
        plt.tight_layout()
        plt.savefig(os.path.join(plots_dir, 'majors_distribution_nces.png'))
        plt.close()
        print("  - Збережено: majors_distribution_nces.png")
    else:
        print("  - Немає даних для розподілу за напрямками.")

    # 3. Розподіл за штатами (для останнього року)
    latest_states = df.iloc[-1]['states']
    if latest_states:
        states_df = pd.DataFrame(list(latest_states.items()), columns=['Штат', 'Кількість'])
        plt.figure(figsize=(12, 7))
        sns.barplot(x='Кількість', y='Штат', data=states_df.sort_values(by='Кількість', ascending=False), palette='magma')
        plt.title(f"Розподіл Абітурієнтів за Штатами ({df.iloc[-1]['year']} рік)", fontsize=16)
        plt.xlabel('Кількість Абітурієнтів', fontsize=12)
        plt.ylabel('Штат', fontsize=12)
        plt.tight_layout()
        plt.savefig(os.path.join(plots_dir, 'states_distribution_nces.png'))
        plt.close()
        print("  - Збережено: states_distribution_nces.png")
    else:
        print("  - Немає даних для розподілу за штатами.")

# --- Основна логіка скрипта ---

def main():
    """
    Основна функція, яка виконує весь процес:
    1. Створює фіктивний файл NCES (якщо його немає).
    2. Отримує та парсить дані з файлу NCES.
    3. Зберігає агреговані дані.
    4. Завантажує всі агреговані дані.
    5. Аналізує тенденції.
    6. Візуалізує дані.
    """
    print("Початок роботи ШІ-асистента для моніторингу заявок на вступ (на основі NCES).")

    # 1. Створення фіктивного файлу NCES для демонстрації
    create_mock_nces_data()

    # 2. Отримання та парсинг даних з файлу NCES
    # Цей крок поверне список словників, по одному для кожного року, знайденого в NCES файлі
    parsed_nces_data_list = fetch_and_parse_nces_data()
    if not parsed_nces_data_list:
        print("Не вдалося отримати або розпарсити дані NCES. Завершення роботи.")
        return

    # 3. Збереження агрегованих даних (історії)
    save_aggregated_data(parsed_nces_data_list)

    # 4. Завантаження всіх історичних даних
    all_data_df = load_aggregated_data()

    # 5. Аналіз тенденцій
    analyze_trends(all_data_df)

    # 6. Візуалізація даних
    visualize_data(all_data_df)

    print("\nРобота ШІ-асистента завершена. Перевірте файли .csv та .png у відповідних папках.")

if __name__ == "__main__":
    main()
