# README for Telegram Bot with aiogram

## Overview

This repository contains a Telegram bot implemented using the aiogram library. The bot allows users to check the index status of URLs and simulate Googlebot visits to a list of URLs provided in text files.

## Features

- **Start Command**: Provides an inline keyboard with options to check the index status or start Googlebot visits.
- **Check Index Status**: Users can send a text file with URLs to check their Google index status.
- **Start Googlebot Visits**: Users can send a text file with URLs for simulating Googlebot visits.

## Prerequisites

- Python 3.7+
- A Telegram bot token from BotFather

## Installation

1. Clone the repository:
   ```bash
   git clone <repository-url>
   cd <repository-directory>```

Install the required packages:

```bash
Копировать код
pip install -r requirements.txt```

Usage
Run the bot:

```bash
python bot.py```
Start a conversation with your bot on Telegram.

Use the /start command to see the options.

Code Explanation
Importing Libraries
```python
Копировать код
import logging
import aiohttp
from aiogram import Bot, types
from aiogram.dispatcher.router import Router
from aiogram.dispatcher.dispatcher import Dispatcher
from aiogram.client.session.aiohttp import AiohttpSession
from aiogram.filters import Command
import asyncio
from aiogram.fsm.context import FSMContext
import pandas as pd
import os
import aiofiles
import requests
from aiogram.filters.state import State, StatesGroup```

Bot Initialization
```python
Копировать код
API_TOKEN = 'your-telegram-bot-token'
logging.basicConfig(level=logging.DEBUG, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)
bot = Bot(token=API_TOKEN, session=AiohttpSession())
router = Router()
router_check_index = Router()
router_start_index = Router()```

States Definition
```python
Копировать код
class UserState(StatesGroup):
    wait_for_file_after_check_index = State()
    wait_for_file_after_start_index = State()```

Command Handlers
```python
Копировать код
@router.message(Command('start'))
async def send_welcome(message: types.Message):
    keyboard = types.InlineKeyboardMarkup(inline_keyboard=[
        [types.InlineKeyboardButton(text="Check Index Status", callback_data="check_index")],
        [types.InlineKeyboardButton(text="Start Fast Index", callback_data="start_index")]
    ])
    await message.answer("Choose an action:", reply_markup=keyboard)```

Callback Query Handler
```python
Копировать код
@router.callback_query()
async def handle_callback_query(query: types.CallbackQuery, state: FSMContext):
    await query.answer()
    if query.data == "check_index":
        await query.message.answer("Please send a text file(.txt) with URLs to check their index status.")
        await state.set_state(UserState.wait_for_file_after_check_index)
    elif query.data == "start_index":
        await query.message.answer("Please send a text file(.txt) with URLs for Googlebot visits.")
        await state.set_state(UserState.wait_for_file_after_start_index)```

Document Handler
```python
Копировать код
@router.message()
async def handle_document(message: types.Message, state: FSMContext):
    current_state = await state.get_state()
    if current_state == UserState.wait_for_file_after_check_index.state:
        await handle_check_index_document(message)
        await state.set_state(None)
    elif current_state == UserState.wait_for_file_after_start_index.state:
        await handle_start_index_document(message)
        await state.set_state(None)```

Check Index Status Function
```python
Копировать код
async def check_indexation(session, query, headers):
    url = f'https://www.google.com/search?q={query}'
    async with session.get(url, headers=headers) as response:
        text = await response.text()
        return "не знайдено жодного документа" not in text

async def check_indexing(filename):
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36"
    }
    data = {'URL': [], 'Status': []}
    async with aiohttp.ClientSession() as session, aiofiles.open(filename, mode='r', encoding='utf-8') as f:
        urls = await f.readlines()
        for url in urls:
            url = url.strip()
            if not url:
                continue
            indexed = True
            if not await check_indexation(session, f'site:{url}', headers):
                indexed = False
            elif not await check_indexation(session, f'inurl:{url}', headers):
                indexed = False
            elif not await check_indexation(session, url, headers):
                indexed = False
            elif not await check_indexation(session, f'"{url}"', headers):
                indexed = False
            status = "In Index" if indexed else "Not in index"
            data['URL'].append(url)
            data['Status'].append(status)
            await asyncio.sleep(7)
    df = pd.DataFrame(data)
    df.to_excel('_results.xlsx', index=False)
    return '_results.xlsx'```

Googlebot Visit Function
```python
Копировать код
async def visit_as_googlebot(session, url, retries=3):
    headers_googlebot = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36"
    }
    for attempt in range(retries):
        try:
            async with session.get(url, headers=headers_googlebot, timeout=60) as response:
                if response.status == 200:
                    return "Visited"
                else:
                    return f"Access error: HTTP {response.status}"
        except asyncio.TimeoutError:
            if attempt == retries - 1:
                return "Timeout error"
        except Exception as e:
            if attempt == retries - 1:
                return f"Error: {str(e)}"
        await asyncio.sleep(5)

async def handle_googlebot_visits(file_path):
    data = {'URL': [], 'Status': []}
    async with aiohttp.ClientSession() as session, aiofiles.open(file_path, mode='r', encoding='utf-8') as f:
        urls = await f.readlines()
        for url in urls:
            url = url.strip()
            if url:
                visit_status = await visit_as_googlebot(session, url)
                data['URL'].append(url)
                data['Status'].append(visit_status)
                await asyncio.sleep(7)
    df = pd.DataFrame(data)
    result_path = 'googlebot_visits.xlsx'
    df.to_excel(result_path, index=False)
    return result_path```

Main Function
```python
Копировать код
async def main():
    dp = Dispatcher()
    dp.include_router(router)
    dp.include_router(router_check_index)
    dp.include_router(router_start_index)
    await dp.start_polling(bot)

if __name__ == '__main__':
    asyncio.run(main())```

