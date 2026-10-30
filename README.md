import asyncio
import random
from aiogram import Bot, Dispatcher, F
from aiogram.filters import Command
from aiogram.types import InlineKeyboardButton, InlineKeyboardMarkup, CallbackQuery, Message
from aiogram.utils.keyboard import InlineKeyboardBuilder

BOT_TOKEN = "ВАШ_ТОКЕН_ОТ_BOTFATHER"  # <-- вставьте сюда токен от @BotFather

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()

# Хранилище активных игр: user_id -> {board, opened, lost, won, score, mines_count}
games = {}

def generate_board(rows=5, cols=5, mine_count=3):
    total_cells = rows * cols
    mine_positions = set(random.sample(range(total_cells), mine_count))
    board = [[False for _ in range(cols)] for _ in range(rows)]
    for pos in mine_positions:
        r, c = divmod(pos, cols)
        board[r][c] = True
    return board

def create_keyboard(user_id, show_all=False):
    game = games.get(user_id)
    if not game:
        return None
    
    board = game["board"]
    opened = game["opened"]
    rows, cols = len(board), len(board)
    builder = InlineKeyboardBuilder()
    
    for r in range(rows):
        for c in range(cols):
            if opened[r][c] or show_all:
                if board[r][c]:
                    text = "💣"
                    callback = "ignore"
                else:
                    text = "💎"
                    callback = "ignore"
            else:
                text = "❓"
                callback = f"cell_{r}_{c}"
            builder.button(text=text, callback_data=callback)
        builder.adjust(cols)
    
    if game["won"]:
        builder.button(text="✨ Забрать очки", callback_data="collect")
    elif game["lost"]:
        builder.button(text="🔙 Главное меню", callback_data="menu")
    else:
        builder.button(text="🔙 Главное меню", callback_data="menu")
    
    return builder.as_markup()

@dp.message(Command("start"))
async def cmd_start(message: Message):
    builder = InlineKeyboardBuilder()
    builder.button(text="🎮 Играть", callback_data="play")
    builder.button(text="ℹ️ Правила", callback_data="rules")
    await message.answer(
        "Добро пожаловать в мини‑игру «Mines»!\n\n"
        "Это бесплатная версия — без реальных ставок. "
        "Открывайте клетки, избегайте мин и набирайте очки!",
        reply_markup=builder.as_markup()
    )

@dp.callback_query(F.data == "play")
async def cb_play(callback: CallbackQuery):
    builder = InlineKeyboardBuilder()
    for mines in [2, 3, 5, 7]:
        builder.button(text=f"💥 {mines} мин", callback_data=f"mines_{mines}")
    builder.adjust(2)
    await callback.message.edit_text(
        "Выберите количество мин на поле (чем больше — тем выше риск и награда):",
        reply_markup=builder.as_markup()
    )

@dp.callback_query(F.data.startswith("mines_"))
async def cb_mines(callback: CallbackQuery):
    mines_count = int(callback.data.split("_")) [1](https://github.com/Git-CmdServ/Mines)
    user_id = callback.from_user.id
    rows, cols = 5, 5
    
    game = {
        "board": generate_board(rows, cols, mines_count),
        "opened": [[False for _ in range(cols)] for _ in range(cols)],
        "mines_count": mines_count,
        "lost": False,
        "won": False,
        "score": 0
    }
    games[user_id] = game
    
    await callback.message.edit_text(
        f"Игра началась! На поле {rows}×{cols} спрятано {mines_count} мин.\n\n"
        "Открывайте клетки и собирайте алмазы 💎. Избегайте бомб 💣!",
        reply_markup=create_keyboard(user_id)
    )

@dp.callback_query(F.data.startswith("cell_"))
async def cb_cell(callback: CallbackQuery):
    user_id = callback.from_user.id
    if user_id not in games:
        await callback.answer("Игра не найдена. Нажмите /start", show_alert=True)
        return
    
    game = games[user_id]
    if game["lost"] or game["won"]:
        await callback.answer("Игра уже завершена!", show_alert=True)
        return
    
    r, c = map(int, callback.data.split("_")[1:])
    game["opened"][r][c] = True
    
    if game["board"][r][c]:
        game["lost"] = True
        await callback.message.edit_text(
            "💥 БА-БАХ! Вы попали на мину!\n\n"
            "Не повезло, но можно попробовать снова.",
            reply_markup=create_keyboard(user_id, show_all=True)
        )
    else:
        base_score = 10
        multiplier = {2: 1.0, 3: 1.2, 5: 1.5, 7: 2.0}.get(game["mines_count"], 1.0)
        game["score"] += int(base_score * multiplier)
        
        total_cells = len(game["board"]) * len(game["board"])
        opened_safe = sum(
            1 for i in range(len(game["board"])) 
            for j in range(len(game["board"])) 
            if game["opened"][i][j] and not game["board"][i][j]
        )
        safe_cells = total_cells - game["mines_count"]
        
        if opened_safe == safe_cells:
            game["won"] = True
            await callback.message.edit_text(
                f"🎉 Победа! Вы открыли все безопасные клетки!\n\n"
                f"Ваш счёт: {game['score']} очков!",
                reply_markup=create_keyboard(user_id)
            )
        else:
            await callback.message.edit_text(
                f"✅ Безопасно! Вы нашли алмаз!\n\n"
                f"Текущий счёт: {game['score']} очков.",
                reply_markup=create_keyboard(user_id)
            )

@dp.callback_query(F.data == "collect")
async def cb_collect(callback: CallbackQuery):
    user_id = callback.from_user.id
    if user_id in games and games[user_id]["won"]:
        score = games[user_id]["score"]
        del games[user_id]
        await callback.message.edit_text(
            f"✨ Вы забрали свои {score} очков!\n\n"
            "Хотите сыграть ещё?",
            reply_markup=InlineKeyboardBuilder().button(text="🎮 Играть снова", callback_data="play").as_markup()
        )

@dp.callback_query(F.data == "menu")
async def cb_menu(callback: CallbackQuery):
    if callback.from_user.id in games:
        del games[callback.from_user.id]
    builder = InlineKeyboardBuilder()
    builder.button(text="🎮 Играть", callback_data="play")
    builder.button(text="ℹ️ Правила", callback_data="rules")
    await callback.message.edit_text(
        "Главное меню. Выберите действие:",
        reply_markup=builder.as_markup()
    )

@dp.callback_query(F.data == "rules")
async def cb_rules(callback: CallbackQuery):
    await callback.answer(
        "Правила игры «Mines»:\n\n"
        "1. Выберите количество мин (2, 3, 5 или 7).\n"
        "2. Открывайте клетки по одной.\n"
        "3. Если клетка безопасна — получите очки.\n"
        "4. Если попадёте на мину — игра окончена.\n"
        "5. Победа — когда открыты все безопасные клетки.",
        show_alert=True
    )

@dp.callback_query(F.data == "ignore")
async def cb_ignore(callback: CallbackQuery):
    await callback.answer()

async def main():
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
