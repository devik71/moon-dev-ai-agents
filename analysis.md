# 🧠 Аналіз архітектури та функціональності проєкту MoonDev

## 🔍 Огляд архітектури

### Структура проєкту
Проєкт MoonDev представляє собою модульну систему AI-агентів для алгоритмічного трейдингу з наступною архітектурою:

```
src/
├── agents/           # 48+ спеціалізованих AI-агентів
├── models/           # Уніфікована абстракція LLM (ModelFactory)
├── strategies/       # Користувацькі торгові стратегії
├── data/            # Результати агентів, пам'ять, аналіз
├── config.py        # Глобальна конфігурація
├── main.py          # Головний оркестратор
├── nice_funcs.py    # ~1200 рядків утиліт для трейдингу
├── exchange_manager.py # Уніфікований інтерфейс бірж
└── ezbot.py         # Legacy контролер
```

### Ключові компоненти

**ModelFactory Pattern**: Центральна абстракція для роботи з 6+ LLM провайдерами:
- Anthropic Claude (за замовчуванням)
- OpenAI GPT-4/GPT-5
- DeepSeek (Chat/Reasoner)
- Groq (Mixtral, Llama)
- Google Gemini
- Ollama (локальні моделі)
- xAI Grok

**Exchange Manager**: Уніфікований інтерфейс для торгівлі на:
- Solana (on-chain DEX, тільки лонги)
- HyperLiquid (перпетуали, лонги/шорти)
- Aster (ф'ючерси, лонги/шорти)

**Agent Ecosystem**: 48+ спеціалізованих агентів, згрупованих за функціональністю:
- Trading: `trading_agent`, `strategy_agent`, `risk_agent`, `copybot_agent`
- Market Analysis: `sentiment_agent`, `whale_agent`, `funding_agent`, `liquidation_agent`
- Content: `chat_agent`, `clips_agent`, `tweet_agent`, `video_agent`
- Research: `rbi_agent` (Research-Based Inference)

## 🧠 Логіка роботи RBI Agent

RBI Agent — найскладніший компонент системи, що автоматизує повний цикл розробки торгових стратегій:

### Workflow RBI Agent v3.0

1. **Research Phase** (Дослідження)
   - Приймає YouTube URL, PDF або текстовий опис стратегії
   - Використовує `youtube-transcript-api` для отримання транскриптів
   - Використовує `PyPDF2` для екстракції тексту з PDF
   - AI аналізує контент і генерує структурований опис стратегії

2. **Backtest Phase** (Бек-тестування)
   - Генерує код на основі `backtesting.py` бібліотеки
   - Використовує TA-Lib та pandas-ta для індикаторів
   - Створює повноцінну стратегію з entry/exit логікою

3. **Package Check Phase** (Перевірка пакетів)
   - Замінює заборонені `backtesting.lib` імпорти на TA-Lib
   - Оптимізує використання індикаторів

4. **Debug Loop** (Цикл налагодження)
   - Виконує код через `subprocess` в conda середовищі
   - Автоматично виправляє помилки через Debug AI
   - Максимум 10 ітерацій налагодження

5. **Optimization Loop** (Цикл оптимізації) — **НОВЕ в v3.0**
   - Встановлює цільову прибутковість (TARGET_RETURN)
   - Якщо результат < цілі → Optimization AI покращує стратегію
   - Цикл продовжується до досягнення цілі або максимуму ітерацій
   - Зберігає найкращу версію як `TARGET_HIT_{return}pct.py`

### Унікальні особливості RBI

**Autonomous Execution**: Повністю автономне виконання від ідеї до готової стратегії
**Multi-Model Support**: Використання різних моделей для різних етапів (Research: GPT-5, Debug: Grok-4)
**Error Recovery**: Інтелектуальне виявлення та виправлення помилок
**Performance Tracking**: Автоматичне парсинг результатів і оптимізація під цільову прибутковість

## 🧩 Взаємодія агентів

### Паттерн комунікації
Агенти працюють за принципом **loose coupling** через спільні дані:

1. **Data Flow Pattern**:
   ```
   Config/Input → Agent Init → API Data Fetch → Data Parsing →
   LLM Analysis → Decision Output → Result Storage → Trade Execution
   ```

2. **Shared Resources**:
   - `src/data/` — спільне сховище результатів
   - `config.py` — глобальні налаштування
   - `nice_funcs.py` — спільні утиліти
   - ModelFactory — уніфікований доступ до LLM

3. **Main Loop Orchestration** (`main.py`):
   ```python
   while True:
       if risk_agent: risk_agent.run()      # Спочатку ризик-менеджмент
       if trading_agent: trading_agent.run() # Потім торгові рішення
       if strategy_agent: strategy_agent.run() # Стратегічний аналіз
       sleep(SLEEP_BETWEEN_RUNS_MINUTES)
   ```

### Типи взаємодії

**Sequential Execution**: Агенти запускаються послідовно в main.py
**File-based Communication**: Обмін даними через CSV/JSON файли
**Shared State**: Спільний доступ до портфеля через `nice_funcs.py`
**Event-driven**: Деякі агенти реагують на зміни файлів

## ⚙️ Ключові залежності

### Криптовалютні/Торгові
- **solana==0.35.1** — Solana blockchain взаємодія
- **hyperliquid-python-sdk==0.20.0** — HyperLiquid API
- **eth-account==0.11.0** — Ethereum ключі для HyperLiquid
- **web3==6.20.1** — Web3 взаємодія

### AI/LLM
- **anthropic==0.40.0** — Claude API
- **openai==1.59.5** — GPT/xAI API (спільний клієнт)
- **groq==0.16.0** — Groq API
- **google-generativeai==0.8.3** — Gemini API

### Аналіз даних
- **pandas==2.1.4** — Обробка даних
- **pandas-ta==0.3.14b0** — Технічні індикатори
- **TA-Lib==0.4.32** — Професійні індикатори
- **Backtesting==0.3.3** — Бек-тестування стратегій
- **yfinance==0.2.43** — Фінансові дані

### Медіа обробка
- **opencv-python==4.10.0.84** — Відео обробка
- **whisper==1.1.10** — Транскрипція аудіо
- **elevenlabs==0.2.24** — Синтез мовлення
- **youtube-transcript-api==0.6.2** — YouTube транскрипти

## 🚀 Запуск у Cursor DevContainer

### Налаштування середовища
```bash
# Активація існуючого conda середовища
conda activate tflow

# Встановлення залежностей
pip install -r requirements.txt

# Створення .env файлу з API ключами
cp .env_example .env
# Додати ключі: ANTHROPIC_KEY, OPENAI_KEY, BIRDEYE_API_KEY, etc.
```

### Запуск системи
```bash
# Головний оркестратор (всі агенти)
python src/main.py

# Окремі агенти
python src/agents/rbi_agent.py        # RBI дослідження
python src/agents/trading_agent.py    # Торгівля
python src/agents/risk_agent.py       # Ризик-менеджмент
```

### Конфігурація RBI
```python
# В src/agents/rbi_agent.py
TARGET_RETURN = 50  # Цільова прибутковість 50%
MAX_OPTIMIZATION_ITERATIONS = 10  # Максимум оптимізацій
CONDA_ENV = "tflow"  # Conda середовище
```

## 🧱 Пропозиції покращень

### Архітектурні
1. **Dependency Injection**: Замінити прямі імпорти на DI контейнер
2. **Event Bus**: Впровадити pub/sub для агентської комунікації
3. **Configuration Management**: Винести конфігурацію в YAML/JSON з валідацією
4. **Service Discovery**: Реєстр доступних агентів і їх можливостей

### Аналітичні
1. **Walk-Forward Analysis**: Додати rolling window валідацію для RBI
2. **Monte Carlo Simulation**: Стохастичне тестування стратегій
3. **Risk Metrics**: Розширити метрики (Sharpe, Sortino, Max DD, VaR)
4. **Market Regime Detection**: Адаптивні стратегії під різні ринкові умови

### Інфраструктурні
1. **Docker Containerization**: 
   ```dockerfile
   FROM python:3.10-slim
   RUN conda install -c conda-forge ta-lib
   COPY requirements.txt .
   RUN pip install -r requirements.txt
   ```

2. **CI/CD Pipeline**:
   ```yaml
   # .github/workflows/test.yml
   - name: Run RBI Tests
     run: pytest tests/test_rbi_agent.py
   - name: Backtest Validation
     run: python scripts/validate_strategies.py
   ```

3. **MLflow Integration**:
   ```python
   import mlflow
   mlflow.log_metric("return_pct", strategy_return)
   mlflow.log_artifact("strategy.py")
   ```

### UX покращення
1. **Web Dashboard**: FastAPI + React інтерфейс для моніторингу
2. **Real-time Logging**: Structured logging з ELK stack
3. **Alert System**: Telegram/Discord боти для сповіщень
4. **Strategy Marketplace**: Обмін стратегіями між користувачами

## ⚠️ Ризики і граничні випадки

### Технічні ризики
1. **API Rate Limits**: BirdEye, LLM провайдери можуть обмежувати запити
2. **Model Hallucinations**: AI може генерувати некоректний код
3. **Overfitting**: RBI може переоптимізувати під історичні дані
4. **Memory Leaks**: Довготривалі процеси можуть споживати пам'ять

### Фінансові ризики
1. **Slippage**: Реальне виконання відрізняється від бек-тестів
2. **Liquidity Risk**: Малоліквідні токени можуть не виконуватися
3. **Flash Crashes**: Швидкі обвали можуть обійти стоп-лосси
4. **MEV Attacks**: Максимальне витягування вартості на Solana

### Операційні ризики
1. **Key Exposure**: API ключі в .env файлах
2. **Network Failures**: Втрата з'єднання під час торгів
3. **Exchange Downtime**: Недоступність бірж
4. **Code Injection**: Генерований код може містити вразливості

### Граничні випадки
1. **Zero Trades**: Стратегії можуть не генерувати сигналів
2. **Infinite Loops**: RBI може зациклитися на оптимізації
3. **Resource Exhaustion**: Занадто багато агентів одночасно
4. **Data Corruption**: Пошкодження історичних даних

### Рекомендації з мітигації
1. **Circuit Breakers**: Автоматична зупинка при критичних втратах
2. **Sandbox Execution**: Ізольоване виконання згенерованого коду
3. **Backup Systems**: Резервні API провайдери та біржі
4. **Monitoring**: Real-time моніторинг всіх компонентів

---

**Висновок**: MoonDev представляє собою амбітний проєкт, що демонструє сучасні паттерни AI-агентської архітектури. Основна сила — в модульності та уніфікованому підході до різних LLM/бірж. Головні ризики — в складності системи та залежності від зовнішніх API. Проєкт має великий потенціал для розвитку в напрямку enterprise-рішення з додаванням proper DevOps практик.