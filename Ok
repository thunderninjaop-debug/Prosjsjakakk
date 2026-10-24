import sys
import subprocess
import os

# ---------------------------------------------------------------------------
# SELF-CONTAINED DEPENDENCY CHECK
# ---------------------------------------------------------------------------
# This file is intentionally standalone. If a required package is missing,
# it installs it automatically so only this single file is needed on the VPS.
# ---------------------------------------------------------------------------
REQUIRED_PACKAGES = {
    "telethon": "telethon",
    "aiosqlite": "aiosqlite",
    "aiohttp": "aiohttp",
    "qrcode": "qrcode",
    "PIL": "pillow",
    "requests": "requests",
}

for _module_name, _pip_name in REQUIRED_PACKAGES.items():
    try:
        __import__(_module_name)
    except ImportError:
        print(f"📦 Installing missing dependency: {_pip_name}...")
        subprocess.check_call([sys.executable, "-m", "pip", "install", _pip_name])

# ---------------------------------------------------------------------------
# IMPORTS
# ---------------------------------------------------------------------------
import asyncio
from contextlib import asynccontextmanager
import html
import io
import json
import logging
from functools import wraps
import re
import secrets
import string
import time
from datetime import datetime, timedelta, timezone
from decimal import Decimal, ROUND_HALF_UP
from typing import Optional, Dict, Any, List, Union, Callable, Awaitable, Tuple
from urllib.parse import quote

import aiohttp
import aiosqlite
import qrcode
import requests
from telethon import TelegramClient, events
from telethon import errors as tl_errors
from telethon.tl import types as tl_types

# ---------------------------------------------------------------------------
# TELETHON RUNTIME
# ---------------------------------------------------------------------------
# Everything below is the Telethon (MTProto) runtime for this bot: client
# wrapper, update dispatcher, FSM storage, filters, keyboards and message
# wrappers. Handlers further down are registered on `router` exactly as
# before and are executed by Telethon through this dispatcher.
# ---------------------------------------------------------------------------
import inspect
import functools

_rt_log = logging.getLogger("AdvancedSMM.runtime")


class ParseMode:
    HTML = "html"


class _Default:
    pass


_DEFAULT = _Default()


def _pm(parse_mode):
    if isinstance(parse_mode, _Default):
        return "html"
    return parse_mode or None


class TelegramObject:
    pass


class User:
    def __init__(self, id: int, first_name: Optional[str] = None,
                 username: Optional[str] = None, last_name: Optional[str] = None):
        self.id = int(id)
        self.first_name = first_name
        self.last_name = last_name
        self.username = username


class Chat:
    def __init__(self, id: int):
        self.id = int(id)


def _to_user(sender, fallback_id: Optional[int]) -> Optional[User]:
    if sender is not None and getattr(sender, "id", None) is not None:
        name = getattr(sender, "first_name", None) or getattr(sender, "title", None)
        return User(sender.id, name, getattr(sender, "username", None),
                    getattr(sender, "last_name", None))
    if fallback_id is not None:
        return User(fallback_id)
    return None


# ----------------------------- keyboards ----------------------------------
_STYLE_SUPPORT: Dict[Any, bool] = {}


def _tl_style_kwargs(cls, style) -> Dict[str, Any]:
    """Colored buttons (primary/success/danger) when this Telethon layer supports it."""
    if not style:
        return {}
    style_cls = getattr(tl_types, "KeyboardButtonStyle", None)
    if style_cls is None:
        return {}
    ok = _STYLE_SUPPORT.get(cls)
    if ok is None:
        try:
            ok = "style" in inspect.signature(cls.__init__).parameters
        except Exception:
            ok = False
        _STYLE_SUPPORT[cls] = ok
    if not ok:
        return {}
    flag = {"primary": "bg_primary", "danger": "bg_danger", "success": "bg_success"}.get(style)
    if not flag:
        return {}
    try:
        return {"style": style_cls(**{flag: True})}
    except Exception:
        return {}


class WebAppInfo:
    def __init__(self, url: str):
        self.url = url


class _TelegramInlineKeyboardButton:
    def __init__(self, text: str = "", callback_data: Optional[str] = None,
                 url: Optional[str] = None, style: Optional[str] = None,
                 web_app: Optional[WebAppInfo] = None, **kwargs):
        self.text = str(text)
        self.callback_data = callback_data
        self.url = url
        self.style = style
        self.web_app = web_app

    def to_tl(self):
        if self.web_app:
            # Telethon does not expose Bot API's KeyboardButtonWebView on all layers.
            # Use a normal URL button as a compatibility fallback; the dedicated
            # Mini App entry point is configured via BotMenuButton below.
            webview_cls = getattr(tl_types, "KeyboardButtonWebView", None)
            if webview_cls is not None:
                try:
                    return webview_cls(text=self.text, url=self.web_app.url)
                except Exception:
                    pass
            return tl_types.KeyboardButtonUrl(
                text=self.text, url=self.web_app.url,
                **_tl_style_kwargs(tl_types.KeyboardButtonUrl, self.style))
        if self.url:
            return tl_types.KeyboardButtonUrl(
                text=self.text, url=self.url,
                **_tl_style_kwargs(tl_types.KeyboardButtonUrl, self.style))
        data = str(self.callback_data if self.callback_data is not None else "noop").encode("utf-8")
        return tl_types.KeyboardButtonCallback(
            text=self.text, data=data,
            **_tl_style_kwargs(tl_types.KeyboardButtonCallback, self.style))


class _TelegramKeyboardButton:
    def __init__(self, text: str = "", style: Optional[str] = None,
                 web_app: Optional[WebAppInfo] = None, **kwargs):
        self.text = str(text)
        self.style = style
        self.web_app = web_app

    def to_tl(self):
        if self.web_app:
            # Reply-keyboard Mini App buttons are not supported by every Telethon TL layer.
            # Keep the menu usable on all layers; the inline/menu URL entry points open the app.
            type_cls = getattr(tl_types, "KeyboardButtonTypeText", None)
            if type_cls is not None:
                return tl_types.KeyboardButton(
                    text=self.text, type=type_cls(),
                    **_tl_style_kwargs(tl_types.KeyboardButton, self.style))
            return tl_types.KeyboardButton(text=self.text)
        # Current Telethon layers may require an explicit KeyboardButtonType.
        button_cls = getattr(tl_types, "KeyboardButton", None)
        if button_cls is None:
            raise RuntimeError("Installed Telethon does not provide KeyboardButton")
        kwargs = _tl_style_kwargs(button_cls, self.style)
        try:
            return button_cls(text=self.text, **kwargs)
        except TypeError as exc:
            if "type" not in str(exc):
                raise
            type_cls = getattr(tl_types, "KeyboardButtonTypeText", None)
            if type_cls is None:
                # Try a compatible plain-text button type from alternate schema naming.
                type_cls = getattr(tl_types, "KeyboardButtonTypeRequestPhone", None)
            if type_cls is None:
                raise RuntimeError(
                    "Incompatible Telethon TL schema: KeyboardButton requires a type, "
                    "but KeyboardButtonTypeText is unavailable. Reinstall requirements.txt."
                ) from exc
            return button_cls(text=self.text, type=type_cls(), **kwargs)


class InlineKeyboardMarkup:
    def __init__(self, inline_keyboard: Optional[List[List[Any]]] = None, **kwargs):
        self.inline_keyboard = inline_keyboard or []

    def to_tl(self):
        rows = [
            tl_types.KeyboardButtonRow(buttons=[b.to_tl() for b in row])
            for row in self.inline_keyboard if row
        ]
        if not rows:
            return None
        return tl_types.ReplyInlineMarkup(rows=rows)


class ReplyKeyboardMarkup:
    def __init__(self, keyboard: Optional[List[List[Any]]] = None, resize_keyboard: bool = False,
                 one_time_keyboard: bool = False, is_persistent: bool = False, **kwargs):
        self.keyboard = keyboard or []
        self.resize_keyboard = resize_keyboard
        self.one_time_keyboard = one_time_keyboard
        self.is_persistent = is_persistent

    def to_tl(self):
        rows = [
            tl_types.KeyboardButtonRow(buttons=[b.to_tl() for b in row])
            for row in self.keyboard if row
        ]
        try:
            return tl_types.ReplyKeyboardMarkup(
                rows=rows, resize=self.resize_keyboard,
                single_use=self.one_time_keyboard, persistent=self.is_persistent)
        except TypeError:
            return tl_types.ReplyKeyboardMarkup(
                rows=rows, resize=self.resize_keyboard, single_use=self.one_time_keyboard)


class ReplyKeyboardRemove:
    def to_tl(self):
        return tl_types.ReplyKeyboardHide()


def _markup(reply_markup):
    if reply_markup is None:
        return None
    if hasattr(reply_markup, "to_tl"):
        return reply_markup.to_tl()
    return reply_markup


class BufferedInputFile:
    def __init__(self, file: bytes, filename: str = "file"):
        self.data = file
        self.filename = filename

    def to_file(self):
        bio = io.BytesIO(self.data)
        bio.name = self.filename
        return bio


# ----------------------------- FSM ----------------------------------------
class State:
    def __init__(self, state: Optional[str] = None):
        self._state = state
        self._group = None
        self._name = None

    def __set_name__(self, owner, name):
        self._group = owner.__name__
        self._name = name

    @property
    def state(self) -> str:
        return self._state or f"{self._group}:{self._name}"

    def __str__(self):
        return self.state


class StatesGroup:
    @classmethod
    def _all_states(cls) -> List[str]:
        return [v.state for v in vars(cls).values() if isinstance(v, State)]


class MemoryStorage:
    def __init__(self):
        self.states: Dict[Any, Optional[str]] = {}
        self.data: Dict[Any, Dict[str, Any]] = {}


class FSMContext:
    def __init__(self, storage: MemoryStorage, key: Any):
        self.storage = storage
        self.key = key

    async def set_state(self, state: Any = None) -> None:
        if state is None:
            self.storage.states.pop(self.key, None)
        else:
            self.storage.states[self.key] = state.state if isinstance(state, State) else str(state)

    async def get_state(self) -> Optional[str]:
        return self.storage.states.get(self.key)

    async def update_data(self, data: Optional[Dict[str, Any]] = None, **kwargs) -> Dict[str, Any]:
        d = self.storage.data.setdefault(self.key, {})
        if data:
            d.update(data)
        d.update(kwargs)
        return dict(d)

    async def get_data(self) -> Dict[str, Any]:
        return dict(self.storage.data.get(self.key, {}))

    async def set_data(self, data: Dict[str, Any]) -> None:
        self.storage.data[self.key] = dict(data)

    async def clear(self) -> None:
        self.storage.states.pop(self.key, None)
        self.storage.data.pop(self.key, None)


# ----------------------------- filters ------------------------------------
class _Pred:
    def __init__(self, fn: Callable[[Any], bool]):
        self._fn = fn

    async def check(self, event, state) -> bool:
        return bool(self._fn(event))

    def __and__(self, other: "_Pred") -> "_Pred":
        return _Pred(lambda e: self._fn(e) and other._fn(e))

    def __or__(self, other: "_Pred") -> "_Pred":
        return _Pred(lambda e: self._fn(e) or other._fn(e))

    def __invert__(self) -> "_Pred":
        return _Pred(lambda e: not self._fn(e))


class _MagicAttr:
    def __init__(self, path: tuple = ()):
        object.__setattr__(self, "_path", path)

    def __getattr__(self, name: str) -> "_MagicAttr":
        return _MagicAttr(self._path + (name,))

    def _get(self, event):
        value = event
        for part in self._path:
            value = getattr(value, part, None)
            if value is None:
                return None
        return value

    def __eq__(self, other) -> _Pred:  # type: ignore[override]
        return _Pred(lambda e: self._get(e) == other)

    def __ne__(self, other) -> _Pred:  # type: ignore[override]
        return _Pred(lambda e: self._get(e) != other)

    __hash__ = object.__hash__

    def startswith(self, prefix: str) -> _Pred:
        def _fn(e):
            v = self._get(e)
            return isinstance(v, str) and v.startswith(prefix)
        return _Pred(_fn)

    def endswith(self, suffix: str) -> _Pred:
        def _fn(e):
            v = self._get(e)
            return isinstance(v, str) and v.endswith(suffix)
        return _Pred(_fn)

    def in_(self, items) -> _Pred:
        items = set(items)
        return _Pred(lambda e: self._get(e) in items)


F = _MagicAttr()


class StateFilter:
    def __init__(self, *states):
        self.states = states

    async def check(self, event, state) -> bool:
        for s in self.states:
            if s == "*":
                return True
            if s is None:
                if state is None:
                    return True
                continue
            if isinstance(s, State):
                if state == s.state:
                    return True
            elif isinstance(s, type) and issubclass(s, StatesGroup):
                if state in s._all_states():
                    return True
            elif isinstance(s, str) and state == s:
                return True
        return False


_BOT_INFO: Dict[str, Any] = {"username": None}


class Command:
    def __init__(self, *commands: str, prefix: str = "/"):
        self.commands = tuple(c.lower().lstrip("/") for c in commands)
        self.prefix = prefix

    async def check(self, event, state) -> bool:
        text = getattr(event, "text", None)
        if not text or not text.startswith(self.prefix):
            return False
        head = text.split(maxsplit=1)[0][len(self.prefix):]
        name, _, mention = head.partition("@")
        uname = _BOT_INFO.get("username")
        if mention and uname and mention.lower() != str(uname).lower():
            return False
        return name.lower() in self.commands


class CommandStart(Command):
    def __init__(self):
        super().__init__("start")


class BaseMiddleware:
    async def __call__(self, handler, event, data):
        return await handler(event, data)


class ErrorEvent:
    def __init__(self, exception: BaseException, update: Any = None):
        self.exception = exception
        self.update = update


def _is_not_modified(exc: BaseException) -> bool:
    cls = getattr(tl_errors, "MessageNotModifiedError", None)
    if cls is not None and isinstance(exc, cls):
        return True
    low = str(exc).lower()
    return "not modified" in low or "wasn't modified" in low


# ----------------------------- bot API wrapper ----------------------------
class Bot:
    def __init__(self, client: TelegramClient):
        self.client = client

    async def _peer(self, chat_id):
        if isinstance(chat_id, str):
            s = chat_id.strip()
            if re.fullmatch(r"-?\d+", s):
                chat_id = int(s)
            else:
                return s
        try:
            return await self.client.get_input_entity(chat_id)
        except (ValueError, TypeError):
            cid = int(chat_id)
            if cid < 0:
                s = str(cid)
                if s.startswith("-100"):
                    return tl_types.InputPeerChannel(channel_id=int(s[4:]), access_hash=0)
                return tl_types.InputPeerChat(chat_id=-cid)
            return tl_types.InputPeerUser(user_id=cid, access_hash=0)

    async def send_message(self, chat_id, text: str, reply_markup=None,
                           parse_mode=_DEFAULT, reply_to_message_id: Optional[int] = None,
                           **kwargs) -> "Message":
        peer = await self._peer(chat_id)
        raw = await self.client.send_message(
            peer, text, parse_mode=_pm(parse_mode),
            buttons=_markup(reply_markup), reply_to=reply_to_message_id)
        return Message.from_raw(self, raw)

    async def send_photo(self, chat_id, photo, caption: Optional[str] = None,
                         reply_markup=None, parse_mode=_DEFAULT, **kwargs) -> "Message":
        peer = await self._peer(chat_id)
        if isinstance(photo, BufferedInputFile):
            file = photo.to_file()
        elif isinstance(photo, str) and photo.lower().startswith(("http://", "https://")):
            file = tl_types.InputMediaPhotoExternal(url=photo)
        else:
            file = photo
        raw = await self.client.send_file(
            peer, file, caption=caption, parse_mode=_pm(parse_mode),
            buttons=_markup(reply_markup))
        return Message.from_raw(self, raw)

    async def edit_message_text(self, text: str, chat_id=None, message_id: Optional[int] = None,
                                reply_markup=None, parse_mode=_DEFAULT, **kwargs):
        peer = await self._peer(chat_id)
        return await self.client.edit_message(
            peer, message_id, text=text, parse_mode=_pm(parse_mode),
            buttons=_markup(reply_markup))

    async def edit_message_caption(self, chat_id=None, message_id: Optional[int] = None,
                                   caption: Optional[str] = None, reply_markup=None,
                                   parse_mode=_DEFAULT, **kwargs):
        peer = await self._peer(chat_id)
        return await self.client.edit_message(
            peer, message_id, text=caption or "", parse_mode=_pm(parse_mode),
            buttons=_markup(reply_markup))

    async def delete_message(self, chat_id, message_id: int) -> bool:
        peer = await self._peer(chat_id)
        await self.client.delete_messages(peer, [message_id])
        return True

    async def get_me(self):
        return await self.client.get_me()

    async def delete_webhook(self, drop_pending_updates: bool = False):
        return True


# ----------------------------- event wrappers -----------------------------
class Message(TelegramObject):
    def __init__(self, bot: Bot, chat_id: int, message_id: int, sender_id: Optional[int] = None):
        self.bot = bot
        self._raw = None
        self.message_id = message_id
        self.chat = Chat(chat_id)
        self.from_user = _to_user(None, sender_id)
        self.text = None
        self.caption = None
        self.photo = None
        self.video = None
        self.animation = None
        self.audio = None
        self.document = None

    @classmethod
    def stub(cls, bot: Bot, chat_id: int, message_id: int) -> "Message":
        return cls(bot, chat_id, message_id)

    @classmethod
    def from_raw(cls, bot: Bot, raw, sender=None) -> "Message":
        self = cls(bot, raw.chat_id, raw.id, getattr(raw, "sender_id", None))
        self._raw = raw
        self.from_user = _to_user(sender, getattr(raw, "sender_id", None))
        media = raw.media
        has_media = media is not None and not isinstance(media, tl_types.MessageMediaWebPage)
        body = raw.raw_text or ""
        if has_media:
            self.caption = body or None
            self.photo = raw.photo
            is_gif = bool(raw.gif)
            self.animation = raw.gif if is_gif else None
            self.video = raw.video if (raw.video and not is_gif) else None
            self.audio = raw.audio
            special = (raw.video or raw.audio or raw.gif or raw.voice
                       or raw.sticker or raw.video_note)
            self.document = raw.document if (raw.document and not special) else None
        else:
            self.text = body or None
        return self

    async def answer(self, text: str, reply_markup=None, parse_mode=_DEFAULT, **kwargs) -> "Message":
        return await self.bot.send_message(
            self.chat.id, text, reply_markup=reply_markup, parse_mode=parse_mode)

    async def reply(self, text: str, reply_markup=None, parse_mode=_DEFAULT, **kwargs) -> "Message":
        return await self.bot.send_message(
            self.chat.id, text, reply_markup=reply_markup, parse_mode=parse_mode,
            reply_to_message_id=self.message_id)

    async def answer_photo(self, photo, caption: Optional[str] = None, reply_markup=None,
                           parse_mode=_DEFAULT, **kwargs) -> "Message":
        return await self.bot.send_photo(
            self.chat.id, photo, caption=caption, reply_markup=reply_markup, parse_mode=parse_mode)

    async def edit_text(self, text: str, reply_markup=None, parse_mode=_DEFAULT, **kwargs):
        await self.bot.edit_message_text(
            text=text, chat_id=self.chat.id, message_id=self.message_id,
            reply_markup=reply_markup, parse_mode=parse_mode)
        return self

    async def edit_caption(self, caption: str, reply_markup=None, parse_mode=_DEFAULT, **kwargs):
        await self.bot.edit_message_caption(
            chat_id=self.chat.id, message_id=self.message_id, caption=caption,
            reply_markup=reply_markup, parse_mode=parse_mode)
        return self

    async def delete(self) -> bool:
        return await self.bot.delete_message(self.chat.id, self.message_id)

    async def copy_to(self, chat_id, **kwargs) -> "Message":
        if self._raw is None:
            raise RuntimeError("Cannot copy a message that was not received in full.")
        peer = await self.bot._peer(chat_id)
        raw = await self.bot.client.send_message(peer, self._raw)
        return Message.from_raw(self.bot, raw)


class CallbackQuery(TelegramObject):
    def __init__(self, bot: Bot, event, sender=None):
        self.bot = bot
        self._event = event
        self.id = getattr(getattr(event, "query", None), "query_id", None)
        raw_data = getattr(event, "data", None)
        self.data = raw_data.decode("utf-8", "ignore") if raw_data else None
        self.from_user = _to_user(sender, getattr(event, "sender_id", None))
        self.message = Message.stub(bot, event.chat_id, event.message_id)

    async def answer(self, text: Optional[str] = None, show_alert: bool = False, **kwargs) -> bool:
        try:
            await self._event.answer(message=text, alert=bool(show_alert))
            return True
        except Exception as e:
            _rt_log.debug("callback.answer failed: %s", e)
            return False


# ----------------------------- router / dispatcher ------------------------
class _Observer:
    def __init__(self, kind: str):
        self.kind = kind
        self.handlers: List[tuple] = []
        self.middlewares: List[Any] = []

    def __call__(self, *filters):
        def decorator(func):
            self.handlers.append((filters, func))
            return func
        return decorator

    def middleware(self, mw):
        self.middlewares.append(mw)
        return mw


class Router:
    def __init__(self):
        self.message = _Observer("message")
        self.callback_query = _Observer("callback_query")
        self.error_handlers: List[Any] = []

    def error(self, *filters):
        def decorator(func):
            self.error_handlers.append(func)
            return func
        return decorator


class Dispatcher:
    def __init__(self, storage: Optional[MemoryStorage] = None):
        self.storage = storage or MemoryStorage()
        self.message = _Observer("message")
        self.callback_query = _Observer("callback_query")
        self.routers: List[Router] = []
        self.bot: Optional[Bot] = None
        self._sigs: Dict[Any, Any] = {}

    def include_router(self, router: Router) -> None:
        self.routers.append(router)

    def attach(self, bot: Bot) -> None:
        self.bot = bot
        bot.client.add_event_handler(self._on_new_message, events.NewMessage(incoming=True))
        bot.client.add_event_handler(self._on_callback, events.CallbackQuery())

    async def _on_new_message(self, event) -> None:
        try:
            if getattr(event, "out", False):
                return
            if event.is_channel and not event.is_group:
                return
            if event.sender_id is None:
                return
            try:
                sender = await event.get_sender()
            except Exception:
                sender = None
            msg = Message.from_raw(self.bot, event.message, sender)
            await self._feed("message", msg, msg.from_user, msg.chat.id)
        except Exception as e:
            _rt_log.error("Update processing failed: %s", e, exc_info=True)

    async def _on_callback(self, event) -> None:
        try:
            if event.sender_id is None:
                return
            try:
                sender = await event.get_sender()
            except Exception:
                sender = None
            cb = CallbackQuery(self.bot, event, sender)
            await self._feed("callback_query", cb, cb.from_user, cb.message.chat.id)
        except Exception as e:
            _rt_log.error("Callback processing failed: %s", e, exc_info=True)

    async def _feed(self, kind: str, event, user: Optional[User], chat_id: int) -> None:
        key = (chat_id, user.id if user else 0)
        ctx = FSMContext(self.storage, key)
        current = await ctx.get_state()
        for router in self.routers:
            observer = getattr(router, kind)
            for filters, func in observer.handlers:
                if await self._matches(filters, event, current):
                    await self._execute(kind, func, event, user, ctx)
                    return

    async def _matches(self, filters, event, current_state) -> bool:
        for f in filters:
            if isinstance(f, State):
                if current_state != f.state:
                    return False
            elif hasattr(f, "check"):
                if not await f.check(event, current_state):
                    return False
            elif callable(f):
                res = f(event)
                if inspect.isawaitable(res):
                    res = await res
                if not res:
                    return False
        return True

    def _params(self, func) -> set:
        names = self._sigs.get(func)
        if names is None:
            try:
                sig = inspect.signature(func)
                names = {n for n, p in sig.parameters.items()
                         if p.kind in (p.POSITIONAL_OR_KEYWORD, p.KEYWORD_ONLY)}
            except (TypeError, ValueError):
                names = set()
            self._sigs[func] = names
        return names

    async def _execute(self, kind: str, func, event, user, ctx: FSMContext) -> None:
        data: Dict[str, Any] = {"state": ctx, "bot": self.bot, "event_from_user": user}

        async def final(ev, d):
            names = self._params(func)
            kwargs = {k: v for k, v in d.items() if k in names}
            return await func(ev, **kwargs)

        def wrap(mw, nxt):
            async def call(ev, d):
                return await mw(nxt, ev, d)
            return call

        chain = final
        own = getattr(self, kind).middlewares
        for mw in reversed(own):
            chain = wrap(mw, chain)
        try:
            await chain(event, data)
        except Exception as exc:
            for router in self.routers:
                if router.error_handlers:
                    try:
                        await router.error_handlers[0](ErrorEvent(exc, event))
                    except Exception as e2:
                        _rt_log.error("Error handler failed: %s", e2, exc_info=True)
                    return
            _rt_log.error("Unhandled handler exception: %s", exc, exc_info=True)


# ---------------------------------------------------------------------------
# TELEGRAM COLORED BUTTON UI
# ---------------------------------------------------------------------------
# Telegram Bot API supports primary (blue), success (green), and danger (red)
# button styles. We use the style field without custom emoji icons.
_EMOJI_RE = re.compile(
    "[\U0001F1E6-\U0001F1FF\U0001F300-\U0001FAFF\U00002700-\U000027BF\U00002600-\U000026FF\U00002300-\U000023FF\U00002B00-\U00002BFF\uFE0F\u200D]+"
)

# Enabled by default. Set BUTTON_COLORS=0 to disable color styles where the
# installed Telegram/Telethon schema supports them.
BUTTON_COLORS = os.getenv("BUTTON_COLORS", "1").strip().lower() not in {"0", "false", "no", "off"}


def _button_style(text: str, explicit: Optional[str] = None) -> Optional[str]:
    if not BUTTON_COLORS:
        return None
    if explicit in {"primary", "success", "danger"}:
        return explicit
    t = str(text or "").lower()
    danger_words = (
        "delete", "remove", "reject", "cancel", "close", "reset", "disable",
        "failed", "danger", "ban", "suspend", "no refund", "full reset"
    )
    success_words = (
        "approve", "confirm", "add", "save", "order", "enable", "submit",
        "check payment", "refill", "create", "verify", "apply", "import",
        "accept", "activate"
    )
    if any(w in t for w in danger_words):
        return "danger"
    if any(w in t for w in success_words):
        return "success"
    return "primary"

# Premium custom-emoji IDs are loaded from emojisid.txt when present. Telegram
# custom emoji IDs are message entities, not a supported field on every inline
# keyboard button schema, so they must not be injected into callback data or URLs.
_PREMIUM_EMOJI_IDS: Dict[str, List[str]] = {}

def _load_premium_emoji_ids(path: Optional[str] = None) -> Dict[str, List[str]]:
    source = path or os.path.join(os.path.dirname(os.path.abspath(__file__)), "emojisid.txt")
    mapping: Dict[str, List[str]] = {}
    try:
        with open(source, "r", encoding="utf-8", errors="ignore") as fh:
            content = fh.read()
        # File formats commonly look like: 1)💎 [6264791387032523779]
        for match in re.finditer(r"([^\s\[\]\d][^\[\]\n]{0,10}?)\s*\[([0-9]{10,})\]", content):
            emoji, document_id = match.group(1).strip(" )\t"), match.group(2)
            mapping.setdefault(emoji, []).append(document_id)
    except (OSError, UnicodeError):
        pass
    return mapping

_PREMIUM_EMOJI_IDS = _load_premium_emoji_ids()

def premium_emoji_id(emoji: str) -> Optional[str]:
    """Return a document ID from emojisid.txt for a matching emoji, if available."""
    ids = _PREMIUM_EMOJI_IDS.get(emoji, [])
    return ids[0] if ids else None

def InlineKeyboardButton(*args, **kwargs):
    text = kwargs.get("text", args[0] if args else "")
    if kwargs.get("style") not in {"primary", "success", "danger"}:
        kwargs["style"] = _button_style(text, kwargs.get("style"))
    try:
        return _TelegramInlineKeyboardButton(*args, **kwargs)
    except Exception:
        kwargs.pop("style", None)
        return _TelegramInlineKeyboardButton(*args, **kwargs)

def KeyboardButton(*args, **kwargs):
    text = kwargs.get("text", args[0] if args else "")
    if kwargs.get("style") not in {"primary", "success", "danger"}:
        kwargs["style"] = _button_style(text, kwargs.get("style"))
    try:
        return _TelegramKeyboardButton(*args, **kwargs)
    except Exception:
        kwargs.pop("style", None)
        return _TelegramKeyboardButton(*args, **kwargs)

# ---------------------------------------------------------------------------
# 1. CONFIGURATION
# ---------------------------------------------------------------------------
BOT_TOKEN = os.getenv("BOT_TOKEN", "8963617329:AAGEvTJDuXUqYI4Fdcb_c60pFq8_sPhccMg").strip()
API_ID = int(os.getenv("API_ID", "34098472") or 0)
API_HASH = os.getenv("API_HASH", "eb67cebd80cec5c397eb80b4cf4193b4").strip()
SESSION_NAME = os.getenv("SESSION_NAME", "smm_bot_session")
# Mini App web server. WEBAPP_URL must be the public HTTPS address that points to this server
# (reverse proxy / tunnel -> WEBAPP_HOST:WEBAPP_PORT). It can also be changed from the DB setting WEBAPP_URL.
WEBAPP_URL = os.getenv("WEBAPP_URL", "https://vericelapso.vercel.app/").strip()
WEBAPP_HOST = os.getenv("WEBAPP_HOST", "0.0.0.0")
WEBAPP_PORT = int(os.getenv("WEBAPP_PORT", os.getenv("PORT", "8080")) or 8080)
WEBAPP_PUBLIC_URL = WEBAPP_URL
ADMIN_IDS = [int(x.strip()) for x in os.getenv("ADMIN_IDS", "8363262755").split(",") if x.strip()]
DB_PATH = os.getenv("DB_PATH", "smm_reseller.db")
BOT_NAME = os.getenv("BOT_NAME", "SMM Store")

UPI_ID = os.getenv("UPI_ID", "paytm.s2znl0o@pty")
GMAIL = os.getenv("GMAIL", "alifaisalkhan62@gmail.com")
GMAIL_APP_PASSWORD = os.getenv("GMAIL_APP_PASSWORD", "xojvdwnxccqjkspk").strip()
VERIFY_API = os.getenv("VERIFY_API", "https://api.growfan.in/fampayapii.php/")
QR_BASE = os.getenv("QR_BASE", "https://quickchart.io/qr")

IST = timezone(timedelta(hours=5, minutes=30))
DECIMAL_2 = Decimal("0.01")
DEPOSIT_EXPIRY_SECONDS = 600
ORDER_CONFIRM_LOCKS: Dict[int, asyncio.Lock] = {}
ORDER_CONFIRM_LOCKS_GUARD = asyncio.Lock()

async def get_order_confirm_lock(user_id: int) -> asyncio.Lock:
    async with ORDER_CONFIRM_LOCKS_GUARD:
        return ORDER_CONFIRM_LOCKS.setdefault(user_id, asyncio.Lock())


def serialized_order_confirmation(func):
    """Serialize order confirmation callbacks per user in this bot process."""
    @wraps(func)
    async def wrapper(callback: CallbackQuery, state: FSMContext, *args, **kwargs):
        lock = await get_order_confirm_lock(callback.from_user.id)
        if lock.locked():
            await callback.answer("⏳ Your previous order is still processing.", show_alert=True)
            return
        async with lock:
            return await func(callback, state, *args, **kwargs)
    return wrapper


def get_ist_time() -> str:
    return datetime.now(IST).strftime("%d-%m-%Y %H:%M:%S IST")


def quantize_currency(val: Any) -> Decimal:
    try:
        return Decimal(str(val)).quantize(DECIMAL_2, rounding=ROUND_HALF_UP)
    except Exception:
        return Decimal("0.00")


def mask_api_key(val: str) -> str:
    if not val or len(val) <= 8:
        return "••••••••"
    return f"{val[:4]}••••••••{val[-4:]}"


def generate_ref() -> str:
    chars = string.ascii_uppercase + string.digits
    code = ''.join(secrets.choice(chars) for _ in range(8))
    return f"SMM{code}"


logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
)
logger = logging.getLogger("AdvancedSMM")


# ---------------------------------------------------------------------------
# SAFE MESSAGE EDITING HELPERS
# ---------------------------------------------------------------------------
async def safe_message_edit(message: Message, text: str,
                           reply_markup: Optional[InlineKeyboardMarkup] = None,
                           parse_mode=ParseMode.HTML) -> bool:
    """Edit a text or media (caption) message; fall back to a new message on failure."""
    try:
        await message.edit_text(text=text, reply_markup=reply_markup, parse_mode=parse_mode)
        return True
    except Exception as e:
        if _is_not_modified(e):
            return False
        logger.warning(f"safe_message_edit failed: {e}")
        try:
            await message.answer(text, reply_markup=reply_markup, parse_mode=parse_mode)
        except Exception:
            pass
        return False


async def safe_edit_message_text(bot: Bot, chat_id: int, message_id: int, text: str,
                                 reply_markup: Optional[InlineKeyboardMarkup] = None,
                                 parse_mode=ParseMode.HTML) -> bool:
    try:
        await bot.edit_message_text(
            text=text, chat_id=chat_id, message_id=message_id,
            reply_markup=reply_markup, parse_mode=parse_mode,
        )
        return True
    except Exception as e:
        if _is_not_modified(e):
            return False
        logger.warning(f"safe_edit_message_text failed for {chat_id}/{message_id}: {e}")
        return False




# ---------------------------------------------------------------------------
# 2. DATABASE SYSTEM
# ---------------------------------------------------------------------------
SCHEMA_TABLES = {
    "system_settings": """
        CREATE TABLE IF NOT EXISTS system_settings (
            key TEXT PRIMARY KEY,
            value TEXT NOT NULL
        )
    """,
    "providers": """
        CREATE TABLE IF NOT EXISTS providers (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            api_url TEXT NOT NULL,
            api_key TEXT NOT NULL,
            status INTEGER DEFAULT 1,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """,
    "categories": """
        CREATE TABLE IF NOT EXISTS categories (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            icon TEXT DEFAULT '📁',
            display_order INTEGER NOT NULL DEFAULT 0,
            is_active INTEGER NOT NULL DEFAULT 1
        )
    """,
    "services": """
        CREATE TABLE IF NOT EXISTS services (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            category_id INTEGER NOT NULL,
            provider_id INTEGER NOT NULL,
            provider_service_id INTEGER NOT NULL,
            name TEXT NOT NULL,
            min_quantity INTEGER NOT NULL,
            max_quantity INTEGER NOT NULL,
            provider_rate TEXT NOT NULL,
            profit_type TEXT NOT NULL DEFAULT 'PERCENT',
            profit_value TEXT NOT NULL DEFAULT '20.00',
            selling_rate TEXT NOT NULL,
            description TEXT,
            refill_available INTEGER DEFAULT 0,
            refill_days INTEGER DEFAULT 0,
            refill_duration TEXT DEFAULT '',
            cancel_available INTEGER DEFAULT 0,
            is_active INTEGER NOT NULL DEFAULT 1
        )
    """,
    "users": """
        CREATE TABLE IF NOT EXISTS users (
            user_id INTEGER PRIMARY KEY,
            first_name TEXT,
            username TEXT,
            balance TEXT NOT NULL DEFAULT '0.00',
            total_spent TEXT NOT NULL DEFAULT '0.00',
            total_deposited TEXT NOT NULL DEFAULT '0.00',
            referrer_id INTEGER,
            referral_earnings TEXT NOT NULL DEFAULT '0.00',
            is_banned INTEGER NOT NULL DEFAULT 0,
            currency TEXT NOT NULL DEFAULT 'INR',
            notifications_enabled INTEGER NOT NULL DEFAULT 0,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """,
    "orders": """
        CREATE TABLE IF NOT EXISTS orders (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            service_id INTEGER NOT NULL,
            provider_id INTEGER NOT NULL,
            provider_order_id TEXT,
            link TEXT NOT NULL,
            quantity INTEGER NOT NULL,
            charge TEXT NOT NULL,
            start_count INTEGER DEFAULT 0,
            remains INTEGER DEFAULT 0,
            status TEXT NOT NULL DEFAULT 'Pending',
            refill_available INTEGER DEFAULT 0,
            refill_days INTEGER DEFAULT 0,
            refill_status TEXT DEFAULT 'None',
            provider_refill_id TEXT,
            refund_state TEXT NOT NULL DEFAULT 'NOT_REFUNDED',
            refunded_amount TEXT NOT NULL DEFAULT '0.00',
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """,
    "refills": """
        CREATE TABLE IF NOT EXISTS refills (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            order_id INTEGER NOT NULL,
            user_id INTEGER NOT NULL,
            provider_id INTEGER NOT NULL,
            provider_order_id TEXT,
            provider_refill_id TEXT,
            status TEXT NOT NULL DEFAULT 'Pending',
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """,
    "deposits": """
        CREATE TABLE IF NOT EXISTS deposits (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            method TEXT NOT NULL DEFAULT 'UPI_AUTO',
            amount TEXT NOT NULL,
            crypto_amount TEXT,
            utr TEXT,
            purpose TEXT,
            reference TEXT,
            qr_msg_id INTEGER,
            expires_at REAL,
            status TEXT NOT NULL DEFAULT 'PENDING',
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            processed_at TIMESTAMP
        )
    """,
    "withdrawals": """
        CREATE TABLE IF NOT EXISTS withdrawals (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            amount TEXT NOT NULL,
            status TEXT NOT NULL DEFAULT 'PENDING',
            details TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            processed_at TIMESTAMP
        )
    """,
    "transactions": """
        CREATE TABLE IF NOT EXISTS transactions (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            type TEXT NOT NULL,
            amount TEXT NOT NULL,
            balance_before TEXT NOT NULL,
            balance_after TEXT NOT NULL,
            reference_id TEXT,
            description TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """,
    "refunds": """
        CREATE TABLE IF NOT EXISTS refunds (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            order_id INTEGER NOT NULL,
            user_id INTEGER NOT NULL,
            amount TEXT NOT NULL,
            reason TEXT NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """,
    "gift_cards": """
        CREATE TABLE IF NOT EXISTS gift_cards (
            code TEXT PRIMARY KEY,
            amount TEXT NOT NULL,
            max_uses INTEGER NOT NULL DEFAULT 1,
            used_count INTEGER NOT NULL DEFAULT 0,
            is_active INTEGER NOT NULL DEFAULT 1,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """,
    "gift_card_redemptions": """
        CREATE TABLE IF NOT EXISTS gift_card_redemptions (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            code TEXT NOT NULL,
            user_id INTEGER NOT NULL,
            amount TEXT NOT NULL,
            redeemed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """,
    "admin_logs": """
        CREATE TABLE IF NOT EXISTS admin_logs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            admin_id INTEGER NOT NULL,
            action TEXT NOT NULL,
            target TEXT,
            details TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """,
    "extra_admins": """
        CREATE TABLE IF NOT EXISTS extra_admins (
            user_id INTEGER PRIMARY KEY,
            added_by INTEGER NOT NULL,
            added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """
}

# ✅ FIXED: GLOBAL_MARKUP_PERCENT = DIRECT value (5% means 5%, not 25%)
DEFAULT_SETTINGS = {
    "MAINTENANCE_MODE": "0",
    "MAINT_NEW_ORDER": "0",
    "MAINT_ADD_BALANCE": "0",
    "MAINT_SERVICE": "0",
    "AUTO_UPI_ID": "paytm.s2znl0o@pty",
    "AUTO_UPI_NAME": "ALI",
    "AUTO_PAY_API_URL": "https://mixyox.in/zxox.php",
    "AUTO_PAY_MID": "hAsqwQ85709194299740",
    "UPI_ID": UPI_ID,
    "UPI_NAME": "SMM Store",
    "FAMPAY_GMAIL": GMAIL,
    "FAMPAY_APP_PASSWORD": GMAIL_APP_PASSWORD,
    "VERIFY_API": VERIFY_API,
    "QR_BASE": QR_BASE,
    "CURRENCY_SYMBOL": "₹",
    "MIN_DEPOSIT": "1.00",
    "MAX_DEPOSIT": "50000.00",
    "REFERRAL_PERCENT": "5.00",
    "REFERRAL_ENABLED": "1",
    "SUPPORT_USERNAME": "@telegram",
    "SUPPORT_CHANNEL": "https://t.me/telegram",
    "AUTO_SYNC_INTERVAL": "120",
    "API_CHECK_INTERVAL": "120",
    "API_TIMEOUT": "20",
    "API_RETRY_COUNT": "2",
    "PROVIDER_SERVICE_CHECK_INTERVAL": "60",
    "LOG_CHANNEL_ID": "",
    "LOG_CHANNEL_ENABLED": "0",
    "PAY_UPI_AUTO_ENABLED": "1",
    "PAY_UPI_MANUAL_ENABLED": "1",
    "PAY_USDT_ENABLED": "1",
    "GLOBAL_MARKUP_PERCENT": "20.00",       # DIRECT markup
    "GLOBAL_DISCOUNT_PERCENT": "0.00",      # Global discount
    "DISCOUNT_ENABLED": "0",                # Discount on/off
    "USDT_MIN_DEPOSIT": "1.00",             # USDT min in $
    "USDT_MAX_DEPOSIT": "10000.00",         # USDT max in $
    "USDT_BEP20_ADDRESS": "0x26db00D375C20A6b709B25e3A0c0B5067cffAcd4",
    "LIVE_RATE_ENABLED": "1",
    "RATE_CACHE_MINUTES": "1",
    "LIVE_USDT_RATE": "",
    "LIVE_USDT_TIMESTAMP": "0",
    "CACHED_USD_RATE": "",
    "CACHE_TIMESTAMP": "0",
    "SECURITY_RATE_LIMIT_ENABLED": "1",
    "SECURITY_CALLBACK_TTL": "900",
    "SECURITY_FSM_TIMEOUT": "900",
    "SECURITY_MAX_REQUESTS_PER_WINDOW": "25",
    "SECURITY_RATE_WINDOW_SECONDS": "10",
    "SECURITY_HEALTH_ENABLED": "1",
}


_DB_WAL_SET = False


@asynccontextmanager
async def get_db():
    global _DB_WAL_SET
    async with aiosqlite.connect(DB_PATH, timeout=30.0) as db:
        if not _DB_WAL_SET:
            # WAL mode is stored in the database file, so it only needs setting once.
            await db.execute("PRAGMA journal_mode = WAL;")
            _DB_WAL_SET = True
        await db.execute("PRAGMA foreign_keys = ON;")
        await db.execute("PRAGMA synchronous = NORMAL;")
        db.row_factory = aiosqlite.Row
        yield db


async def ensure_db_columns(db: aiosqlite.Connection):
    cols = [r[1] for r in await (await db.execute("PRAGMA table_info(deposits)")).fetchall()]
    for col, typ, default in [
        ("method", "TEXT", "'UPI_AUTO'"), ("purpose", "TEXT", None),
        ("crypto_amount", "TEXT", None), ("reference", "TEXT", None),
        ("qr_msg_id", "INTEGER", None), ("expires_at", "REAL", None),
    ]:
        if col not in cols:
            try:
                sql = f"ALTER TABLE deposits ADD COLUMN {col} {typ}"
                if default:
                    sql += f" DEFAULT {default}"
                await db.execute(sql)
            except Exception:
                pass

    cols = [r[1] for r in await (await db.execute("PRAGMA table_info(services)")).fetchall()]
    if "refill_days" not in cols:
        try:
            await db.execute("ALTER TABLE services ADD COLUMN refill_days INTEGER DEFAULT 0")
        except Exception:
            pass
    if "refill_duration" not in cols:
        try:
            await db.execute("ALTER TABLE services ADD COLUMN refill_duration TEXT DEFAULT ''")
        except Exception:
            pass

    cols = [r[1] for r in await (await db.execute("PRAGMA table_info(users)")).fetchall()]
    for col, typ, default in [
        ("currency", "TEXT", "'INR'"),
        ("notifications_enabled", "INTEGER", "0"),
    ]:
        if col not in cols:
            try:
                await db.execute(f"ALTER TABLE users ADD COLUMN {col} {typ} DEFAULT {default}")
            except Exception:
                pass

    cols = [r[1] for r in await (await db.execute("PRAGMA table_info(orders)")).fetchall()]
    order_cols = {
        "refill_available": "INTEGER DEFAULT 0",
        "refill_days": "INTEGER DEFAULT 0",
        "refill_status": "TEXT DEFAULT 'None'",
        "provider_refill_id": "TEXT",
        "service_name_snapshot": "TEXT",
        "service_description_snapshot": "TEXT",
        "service_category_snapshot": "TEXT",
        "service_provider_snapshot": "TEXT",
        "provider_service_id_snapshot": "TEXT",
        "service_rate_snapshot": "TEXT",
        "service_min_snapshot": "INTEGER",
        "service_max_snapshot": "INTEGER",
        "service_refill_snapshot": "INTEGER",
        "service_refill_days_snapshot": "INTEGER",
        "service_cancel_snapshot": "INTEGER",
        "discount_applied_percent": "TEXT DEFAULT '0.00'",
        "original_rate_snapshot": "TEXT",
    }
    for col, typ in order_cols.items():
        if col not in cols:
            try:
                await db.execute(f"ALTER TABLE orders ADD COLUMN {col} {typ}")
            except Exception:
                pass


async def init_db():
    async with get_db() as db:
        for sql in SCHEMA_TABLES.values():
            await db.execute(sql)

        await ensure_db_columns(db)

        for idx_sql in [
            "CREATE INDEX IF NOT EXISTS idx_users_uid ON users(user_id)",
            "CREATE INDEX IF NOT EXISTS idx_orders_uid ON orders(user_id)",
            "CREATE INDEX IF NOT EXISTS idx_orders_poid ON orders(provider_order_id)",
            "CREATE INDEX IF NOT EXISTS idx_deposits_utr ON deposits(utr)",
            "CREATE UNIQUE INDEX IF NOT EXISTS uq_deposits_approved_utr ON deposits(utr) WHERE utr IS NOT NULL AND status='APPROVED'",
            "CREATE UNIQUE INDEX IF NOT EXISTS uq_services_provider_sid ON services(provider_id, provider_service_id)",
            "CREATE INDEX IF NOT EXISTS idx_tx_uid ON transactions(user_id)",
        ]:
            try:
                await db.execute(idx_sql)
            except Exception:
                pass

        for k, v in DEFAULT_SETTINGS.items():
            await db.execute(
                "INSERT OR IGNORE INTO system_settings (key, value) VALUES (?, ?)",
                (k, v)
            )

        for admin_id in ADMIN_IDS:
            await db.execute(
                "INSERT OR IGNORE INTO users (user_id, first_name, username, balance) VALUES (?, 'Admin', 'admin', '0.00')",
                (admin_id,)
            )

        cat_count = (await (await db.execute("SELECT COUNT(*) FROM categories")).fetchone())[0]
        if cat_count == 0:
            sample_cats = [
                ("Instagram Followers", "👥", 1),
                ("Instagram Likes & Views", "❤️", 2),
                ("YouTube Services", "📹", 3),
                ("Telegram Members", "✈️", 4)
            ]
            await db.executemany(
                "INSERT INTO categories (name, icon, display_order) VALUES (?, ?, ?)",
                sample_cats
            )

        await db.commit()


_SETTINGS_CACHE: Dict[str, Tuple[Optional[str], float]] = {}
_SETTINGS_TTL = 30.0


async def get_setting(key: str, default: str = "") -> str:
    now = time.monotonic()
    hit = _SETTINGS_CACHE.get(key)
    if hit is not None and now - hit[1] < _SETTINGS_TTL:
        return hit[0] if hit[0] is not None else default
    async with get_db() as db:
        cursor = await db.execute("SELECT value FROM system_settings WHERE key = ?", (key,))
        row = await cursor.fetchone()
    val = row[0] if row else None
    _SETTINGS_CACHE[key] = (val, now)
    return val if val is not None else default


async def set_setting(key: str, value: str):
    async with get_db() as db:
        await db.execute(
            "INSERT OR REPLACE INTO system_settings (key, value) VALUES (?, ?)",
            (key, str(value))
        )
        await db.commit()
    _SETTINGS_CACHE[key] = (str(value), time.monotonic())


async def get_all_admin_ids() -> List[int]:
    admins = set(ADMIN_IDS)
    async with get_db() as db:
        try:
            rows = await (await db.execute("SELECT user_id FROM extra_admins")).fetchall()
            for r in rows:
                admins.add(int(r["user_id"]))
        except Exception:
            pass
    return list(admins)


async def is_admin(user_id: int) -> bool:
    if user_id in ADMIN_IDS:
        return True
    async with get_db() as db:
        try:
            row = await (await db.execute(
                "SELECT 1 FROM extra_admins WHERE user_id = ?", (user_id,)
            )).fetchone()
            return bool(row)
        except Exception:
            return False


# ---------------------------------------------------------------------------
# PRICING ENGINE — DIRECT markup (5% = 5%, NOT 25%)
# ---------------------------------------------------------------------------
async def calculate_selling_rate(provider_rate: Decimal) -> Decimal:
    """Calculate a new selling rate from the global markup/discount settings."""
    markup_pct = Decimal(await get_setting("GLOBAL_MARKUP_PERCENT", "20.00"))
    discount_pct = Decimal(await get_setting("GLOBAL_DISCOUNT_PERCENT", "0.00"))
    discount_on = await get_setting("DISCOUNT_ENABLED", "0") == "1"
    base_sell = provider_rate * (Decimal("1") + markup_pct / Decimal("100"))
    if discount_on and discount_pct > Decimal("0"):
        base_sell *= Decimal("1") - discount_pct / Decimal("100")
    return quantize_currency(base_sell)


async def calculate_service_selling_rate(service_row, provider_rate: Decimal) -> Decimal:
    """Respect service pricing mode: FIXED keeps its manual selling price;
    PERCENT uses that service's own markup value and then the global discount."""
    if str(service_row["profit_type"]).upper() == "FIXED":
        return quantize_currency(Decimal(str(service_row["selling_rate"])))
    try:
        markup_pct = Decimal(str(service_row["profit_value"] or "0"))
    except Exception:
        markup_pct = Decimal("0")
    sell = provider_rate * (Decimal("1") + markup_pct / Decimal("100"))
    if await get_setting("DISCOUNT_ENABLED", "0") == "1":
        try:
            discount_pct = Decimal(await get_setting("GLOBAL_DISCOUNT_PERCENT", "0.00"))
            if discount_pct > 0:
                sell *= Decimal("1") - discount_pct / Decimal("100")
        except Exception:
            pass
    return quantize_currency(sell)


async def refresh_provider_rates_and_reprice_all() -> Dict[str, int]:
    """Fetch every provider's current catalog, update provider_rate, then
    calculate selling_rate from that provider_rate + current global markup.

    This intentionally ignores each service's previous selling_rate and
    previous profit_value so changing Global Markup from e.g. 10% to 15%
    never compounds on an already-marked-up price.
    """
    stats = {"providers": 0, "updated_rates": 0, "repriced": 0, "missing": 0}
    async with get_db() as db:
        providers = await (await db.execute(
            "SELECT id FROM providers WHERE status=1 ORDER BY id"
        )).fetchall()

    for provider in providers:
        stats["providers"] += 1
        try:
            result = await ProviderAPIClient.fetch_services_list(provider["id"])
            if not result.get("success"):
                continue
            catalog = {}
            for item in result.get("services") or []:
                sid = _p_sid(item)
                rate = _p_rate(item)
                if sid is None or rate in (None, ""):
                    continue
                try:
                    catalog[str(sid)] = quantize_currency(rate)
                except Exception:
                    continue

            async with get_db() as db:
                local = await (await db.execute(
                    "SELECT id, provider_service_id, provider_rate, profit_type, profit_value, selling_rate FROM services WHERE provider_id=?",
                    (provider["id"],)
                )).fetchall()
                for svc in local:
                    rate = catalog.get(str(svc["provider_service_id"]))
                    if rate is None:
                        stats["missing"] += 1
                        continue
                    new_sell = await calculate_service_selling_rate(svc, rate)
                    if quantize_currency(svc["provider_rate"]) != rate:
                        stats["updated_rates"] += 1
                    await db.execute(
                        "UPDATE services SET provider_rate=?, selling_rate=? WHERE id=?",
                        (str(rate), str(new_sell), svc["id"])
                    )
                    stats["repriced"] += 1
                await db.commit()
        except Exception as e:
            logger.warning("Global provider refresh failed for #%s: %s", provider["id"], e)

    return stats


async def check_profit_safety(provider_rate: Decimal, discount_pct: Decimal) -> Dict[str, Any]:
    markup_pct = Decimal(await get_setting("GLOBAL_MARKUP_PERCENT", "20.00"))
    base_sell = provider_rate * (Decimal("1") + markup_pct / Decimal("100"))
    final_sell = base_sell * (Decimal("1") - discount_pct / Decimal("100"))
    profit = final_sell - provider_rate
    profit_pct = (profit / provider_rate * Decimal("100")) if provider_rate > 0 else Decimal("0")

    return {
        "safe": profit > Decimal("0"),
        "profit": quantize_currency(profit),
        "profit_pct": quantize_currency(profit_pct),
        "final_sell": quantize_currency(final_sell),
        "base_sell": quantize_currency(base_sell),
    }


# ---------------------------------------------------------------------------
# 3. LOGGING
# ---------------------------------------------------------------------------
async def log_admin_event(admin_id: int, action: str, target: str = "", details: str = ""):
    async with get_db() as db:
        await db.execute(
            "INSERT INTO admin_logs (admin_id, action, target, details) VALUES (?, ?, ?, ?)",
            (admin_id, action, str(target), str(details))
        )
        await db.commit()


_RECENT_LOG_KEYS: Dict[str, float] = {}


def _log_dedupe(text: str) -> bool:
    """True if this exact text was already posted to the log channel in the last 10 seconds."""
    now = time.time()
    for k in [k for k, t in _RECENT_LOG_KEYS.items() if now - t > 10]:
        _RECENT_LOG_KEYS.pop(k, None)
    key = hashlib.md5(text.encode("utf-8", "ignore")).hexdigest()
    if key in _RECENT_LOG_KEYS:
        return True
    _RECENT_LOG_KEYS[key] = now
    return False


async def send_channel_log(bot: Bot, category: str, text: str, reply_markup=None):
    """Post an event to the private log channel (LOG_CHANNEL_ID, enabled with LOG_CHANNEL_ENABLED)."""
    enabled = await get_setting("LOG_CHANNEL_ENABLED", "0")
    channel_id = (await get_setting("LOG_CHANNEL_ID", "") or "").strip()
    if not _flag_on(enabled) or not channel_id:
        return
    if _log_dedupe(text):
        return
    cat_icons = {
        "ORDER": "📦", "PAYMENT": "💳", "USER": "👤",
        "SERVICE": "🛍️", "REFILL": "♻️", "ADMIN": "👑", "ERROR": "⚠️"
    }
    icon = cat_icons.get(category.upper(), "📢")
    formatted_msg = f"{icon} <b>[{category.upper()} LOG]</b>\n\n{text}\n\n🕒 <i>{get_ist_time()}</i>"
    try:
        await bot.send_message(chat_id=channel_id, text=formatted_msg, reply_markup=reply_markup, parse_mode=ParseMode.HTML)
    except Exception as e:
        logger.error(f"Failed to deliver log to channel ({channel_id}): {e}. "
                     f"Make sure the bot is an ADMIN of the channel and LOG_CHANNEL_ID is correct (e.g. -100123456789).")


async def admin_alert_targets() -> List[int]:
    """Admin DMs are OFF by default (set ADMIN_DM_NOTIFICATIONS=1 in settings to turn them back on)."""
    if _flag_on(await get_setting("ADMIN_DM_NOTIFICATIONS", "0")):
        return await get_all_admin_ids()
    return []


async def notify_admins(bot: Bot, text: str, reply_markup=None):
    """Admin notifications: no DMs. Everything goes to the private log channel instead."""
    for admin_id in await admin_alert_targets():
        try:
            await bot.send_message(admin_id, text, reply_markup=reply_markup, parse_mode=ParseMode.HTML)
        except Exception as e:
            logger.warning(f"Admin notification failed for {admin_id}: {e}")
    await send_channel_log(bot, "ADMIN", text, reply_markup=reply_markup)

async def payment_method_enabled(method: str) -> bool:
    key = {"UPI_AUTO":"PAY_UPI_AUTO_ENABLED","UPI_MANUAL":"PAY_UPI_MANUAL_ENABLED","USDT_BEP20":"PAY_USDT_ENABLED"}.get(method)
    return bool(key and await get_setting(key, "0") == "1")



# ---------------------------------------------------------------------------
# 🔐 HIGH-SECURITY RUNTIME GUARDS
# ---------------------------------------------------------------------------
_SECURITY_BUCKETS: Dict[int, List[float]] = {}
_SECURITY_BUCKET_LOCK = asyncio.Lock()

async def security_rate_limit(user_id: int, cost: int = 1,
                              limit: Optional[int] = None,
                              window: Optional[int] = None) -> bool:
    if await get_setting("SECURITY_RATE_LIMIT_ENABLED", "1") != "1":
        return True
    limit = limit or int(await get_setting("SECURITY_MAX_REQUESTS_PER_WINDOW", "25"))
    window = window or int(await get_setting("SECURITY_RATE_WINDOW_SECONDS", "10"))
    now = time.monotonic()
    async with _SECURITY_BUCKET_LOCK:
        bucket = _SECURITY_BUCKETS.setdefault(int(user_id), [])
        bucket[:] = [t for t in bucket if t >= now - max(1, window)]
        if len(bucket) + max(1, cost) > max(1, limit):
            return False
        bucket.extend([now] * max(1, cost))
        return True

async def run_security_health_check(bot: Optional[Bot] = None) -> Dict[str, Any]:
    result = {
        "database": False, "telegram_api": False, "providers": 0,
        "payment_methods": {}, "secrets_ok": bool(BOT_TOKEN),
        "schema_ok": False, "wallet_ok": False, "security_score": 0,
        "warnings": []
    }
    try:
        async with get_db() as db:
            await (await db.execute("SELECT 1")).fetchone()
            result["database"] = True
            fk = await (await db.execute("PRAGMA foreign_keys")).fetchone()
            result["schema_ok"] = bool(fk and fk[0] == 1)
            neg = await (await db.execute(
                "SELECT COUNT(*) FROM users WHERE CAST(balance AS REAL) < 0"
            )).fetchone()
            result["wallet_ok"] = neg[0] == 0
            result["providers"] = (await (await db.execute(
                "SELECT COUNT(*) FROM providers WHERE status=1"
            )).fetchone())[0]
        if bot:
            await bot.get_me()
            result["telegram_api"] = True
        for method in ("UPI_AUTO", "UPI_MANUAL", "USDT_BEP20"):
            result["payment_methods"][method] = await payment_method_enabled(method)
        if not result["secrets_ok"]:
            result["warnings"].append("BOT_TOKEN is not configured in environment.")
        if not GMAIL_APP_PASSWORD:
            result["warnings"].append("GMAIL_APP_PASSWORD is not configured.")
        if not result["schema_ok"]:
            result["warnings"].append("SQLite foreign_keys is not enabled.")
        if not result["wallet_ok"]:
            result["warnings"].append("Negative wallet balance records detected.")
        if result["providers"] == 0:
            result["warnings"].append("No active provider configured.")
        checks = [result["database"], result["telegram_api"], result["secrets_ok"],
                  result["schema_ok"], result["wallet_ok"], bool(result["payment_methods"])]
        result["security_score"] = round(sum(bool(x) for x in checks) / len(checks) * 100)
    except Exception as exc:
        result["warnings"].append(f"Health check error: {type(exc).__name__}")
    return result

async def format_security_health(bot: Bot) -> str:
    h = await run_security_health_check(bot)
    mark = lambda v: "🟢" if v else "🔴"
    m = h["payment_methods"]
    lines = [
        "🛡️ <b>SECURITY & SYSTEM HEALTH</b>",
        "━━━━━━━━━━━━━━━━━━",
        f"{mark(h['database'])} Database",
        f"{mark(h['schema_ok'])} SQLite security/schema",
        f"{mark(h['wallet_ok'])} Wallet integrity",
        f"{mark(h['telegram_api'])} Telegram API",
        f"{mark(h['secrets_ok'])} Secret configuration",
        "",
        f"💳 UPI 1 Auto: {'ON' if m.get('UPI_AUTO') else 'OFF'}",
        f"💳 UPI 2 Manual: {'ON' if m.get('UPI_MANUAL') else 'OFF'}",
        f"🔵 USDT BEP-20: {'ON' if m.get('USDT_BEP20') else 'OFF'}",
        "",
        f"🔌 Active Providers: <b>{h['providers']}</b>",
        f"🛡️ Runtime Security Score: <b>{h['security_score']}/100</b>",
    ]
    if h["warnings"]:
        lines += ["", "⚠️ <b>Warnings</b>"]
        lines += [f"• {html.escape(str(w))}" for w in h["warnings"][:8]]
    lines += ["", "ℹ️ Live operational check; not a security certification."]
    return "\n".join(lines)

_KNOWN_USERS: set = set()


async def ensure_user_row(user) -> None:
    """Make sure a users row exists so Profile/Stats never say 'Profile not found'."""
    uid = getattr(user, "id", None)
    if not uid or uid in _KNOWN_USERS:
        return
    try:
        async with get_db() as db:
            row = await (await db.execute("SELECT 1 FROM users WHERE user_id = ?", (uid,))).fetchone()
            if not row:
                await db.execute(
                    "INSERT OR IGNORE INTO users (user_id, first_name, username) VALUES (?, ?, ?)",
                    (uid, getattr(user, "first_name", None) or "User", getattr(user, "username", None) or ""))
                await db.commit()
        _KNOWN_USERS.add(uid)
    except Exception as e:
        logger.warning(f"ensure_user_row failed for {uid}: {e}")


class HighSecurityMiddleware(BaseMiddleware):
    async def __call__(self, handler, event, data):
        user = getattr(event, "from_user", None)
        if user:
            txt = (getattr(event, "text", None) or "") if isinstance(event, Message) else ""
            if not txt.startswith("/start"):      # /start creates the user itself (referral + new-user alert)
                await ensure_user_row(user)
        if user and not await security_rate_limit(user.id):
            if isinstance(event, CallbackQuery):
                await event.answer("⚠️ Too many requests. Please wait a moment.", show_alert=True)
            elif isinstance(event, Message):
                await event.answer("⚠️ Too many requests. Please wait a moment.")
            return
        return await handler(event, data)

# ---------------------------------------------------------------------------
# 4. WALLET OPERATIONS
# ---------------------------------------------------------------------------
async def modify_wallet_atomic(
    db: aiosqlite.Connection,
    user_id: int,
    amount: Decimal,
    tx_type: str,
    reference_id: str = "",
    description: str = ""
) -> bool:
    cursor = await db.execute(
        "SELECT balance, total_spent, total_deposited FROM users WHERE user_id = ?",
        (user_id,)
    )
    row = await cursor.fetchone()
    if not row:
        await db.execute(
            "INSERT OR IGNORE INTO users (user_id, balance, total_spent, total_deposited) VALUES (?, '0.00', '0.00', '0.00')",
            (user_id,)
        )
        cursor = await db.execute(
            "SELECT balance, total_spent, total_deposited FROM users WHERE user_id = ?",
            (user_id,)
        )
        row = await cursor.fetchone()

    current_bal = Decimal(str(row["balance"] or "0.00"))
    new_bal = current_bal + amount
    if new_bal < Decimal("0.00"):
        return False

    spent_delta = abs(amount) if tx_type == "Order Deduction" else (-abs(amount) if "Refund" in tx_type else Decimal("0.00"))
    dep_delta = amount if tx_type == "Deposit" else Decimal("0.00")

    new_spent = Decimal(str(row["total_spent"] or "0.00")) + spent_delta
    new_dep = Decimal(str(row["total_deposited"] or "0.00")) + dep_delta

    await db.execute(
        "UPDATE users SET balance = ?, total_spent = ?, total_deposited = ? WHERE user_id = ?",
        (str(quantize_currency(new_bal)), str(quantize_currency(new_spent)), str(quantize_currency(new_dep)), user_id)
    )

    await db.execute(
        """
        INSERT INTO transactions (user_id, type, amount, balance_before, balance_after, reference_id, description)
        VALUES (?, ?, ?, ?, ?, ?, ?)
        """,
        (
            user_id, tx_type, str(quantize_currency(amount)),
            str(quantize_currency(current_bal)), str(quantize_currency(new_bal)),
            str(reference_id), description
        )
    )
    return True


async def credit_referral_bonus(db: aiosqlite.Connection, depositor_id: int, deposit_amt: Decimal):
    ref_enabled = await get_setting("REFERRAL_ENABLED", "1")
    if ref_enabled != "1":
        return
    cursor = await db.execute(
        "SELECT referrer_id, referral_earnings FROM users WHERE user_id = ?",
        (depositor_id,)
    )
    row = await cursor.fetchone()
    if not row or not row["referrer_id"]:
        return
    ref_id = row["referrer_id"]
    percent_str = await get_setting("REFERRAL_PERCENT", "5.00")
    commission = quantize_currency(deposit_amt * (Decimal(percent_str) / Decimal("100")))
    if commission > Decimal("0.00"):
        ok = await modify_wallet_atomic(
            db, ref_id, commission, "Referral Bonus",
            reference_id=f"DEP_{depositor_id}",
            description=f"{percent_str}% commission from user #{depositor_id}"
        )
        if ok:
            cur_earn = Decimal(str(row["referral_earnings"] or "0.00"))
            await db.execute(
                "UPDATE users SET referral_earnings = ? WHERE user_id = ?",
                (str(quantize_currency(cur_earn + commission)), ref_id)
            )


# ---------------------------------------------------------------------------
# 5. CURRENCY
# ---------------------------------------------------------------------------
async def fetch_live_usdt_inr_rate() -> Optional[Decimal]:
    """Fetch a current USDT/INR market rate.

    CoinGecko is the primary source. CryptoCompare is used as a secondary
    source so a temporary failure/rate-limit at one provider does not freeze
    the bot on an old/default value.
    """
    urls = [
        (
            "https://api.coingecko.com/api/v3/simple/price?ids=tether&vs_currencies=inr&precision=8",
            lambda data: data.get("tether", {}).get("inr"),
        ),
        (
            "https://min-api.cryptocompare.com/data/price?fsym=USDT&tsyms=INR",
            lambda data: data.get("INR"),
        ),
    ]
    timeout = aiohttp.ClientTimeout(total=10)
    for url, extractor in urls:
        try:
            async with aiohttp.ClientSession(timeout=timeout) as session:
                async with session.get(
                    url,
                    headers={"Accept": "application/json", "User-Agent": "CXS-LiveRate/1.0"}
                ) as resp:
                    if resp.status != 200:
                        continue
                    data = await resp.json(content_type=None)
                    raw = extractor(data)
                    if raw in (None, ""):
                        continue
                    rate = Decimal(str(raw))
                    if rate > 0:
                        return rate
        except Exception as e:
            logger.warning("Live USDT/INR fetch failed for %s: %s", url, e)
    return None


async def get_live_usdt_inr_rate(force: bool = False) -> Decimal:
    """Return the current USDT/INR rate, with a short cache and safe fallback.

    Stored balances/prices remain INR. This rate is only the conversion layer
    for USDT deposits and USD display.
    """
    cached_raw = await get_setting("LIVE_USDT_RATE", "")
    ts_raw = await get_setting("LIVE_USDT_TIMESTAMP", "0")
    cache_minutes = max(1, int(await get_setting("RATE_CACHE_MINUTES", "1")))

    cached: Optional[Decimal] = None
    try:
        if cached_raw:
            candidate = Decimal(str(cached_raw))
            if candidate > 0:
                cached = candidate
    except Exception:
        cached = None

    try:
        fresh = cached is not None and (time.time() - float(ts_raw) < cache_minutes * 60)
    except Exception:
        fresh = False

    # Do not return the old value when a caller explicitly asks for a refresh.
    if not force and fresh:
        return cached  # type: ignore[return-value]

    # Live conversion is mandatory for USDT/USD pricing. The old
    # LIVE_RATE_ENABLED switch is intentionally ignored so an old setting
    # cannot silently freeze all users on a stale rate.
    rate = await fetch_live_usdt_inr_rate()
    if rate is not None and rate > 0:
        now = str(time.time())
        await set_setting("LIVE_USDT_RATE", str(rate))
        await set_setting("LIVE_USDT_TIMESTAMP", now)
        await set_setting("CACHED_USD_RATE", str(rate))
        await set_setting("CACHE_TIMESTAMP", now)
        return rate

    # If live fetching fails, use the last successfully fetched live value.
    if cached is not None:
        return cached

    # Backward-compatible last cache. This is only used when there has never
    # been a successful live fetch; it is NOT used as the normal conversion rate.
    try:
        old = Decimal(str(await get_setting("CACHED_USD_RATE", "")))
        if old > 0:
            return old
    except Exception:
        pass

    # Final emergency fallback for a brand-new database/offline startup.
    try:
        legacy = Decimal(str(await get_setting("USDT_RATE", DEFAULT_SETTINGS.get("USDT_RATE", "99.00"))))
        if legacy > 0:
            return legacy
    except Exception:
        pass
    return Decimal("99.00")


async def get_live_usd_rate(force: bool = False) -> Decimal:
    # USD display and USDT deposits use the same live USDT/INR conversion rate.
    return await get_live_usdt_inr_rate(force=force)


async def live_usdt_rate_worker(bot: Bot):
    """Refresh live USDT/INR every 60 seconds, independent of deposits."""
    while True:
        try:
            await get_live_usdt_inr_rate(force=True)
        except asyncio.CancelledError:
            break
        except Exception as e:
            logger.error("Live USDT rate worker error: %s", e, exc_info=True)
        await asyncio.sleep(60)


async def get_user_currency(user_id: int) -> str:
    async with get_db() as db:
        row = await (await db.execute(
            "SELECT currency FROM users WHERE user_id = ?", (user_id,)
        )).fetchone()
        return (row["currency"] if row and row["currency"] else "INR")


def _fmt_decimal_trim(d: Decimal, max_places: int = 6) -> str:
    try:
        q = d.quantize(Decimal(1).scaleb(-max_places), rounding=ROUND_HALF_UP)
    except Exception:
        q = d
    s = format(q, 'f')
    if '.' in s:
        s = s.rstrip('0').rstrip('.')
    return s if s else "0"


async def format_price(amount_inr: Any, user_id: Optional[int] = None, currency: Optional[str] = None) -> str:
    try:
        amount = Decimal(str(amount_inr))
    except Exception:
        amount = Decimal("0.00")
    if currency is None and user_id is not None:
        currency = await get_user_currency(user_id)
    currency = currency or "INR"

    # Database amounts are canonical INR. For USD users, convert INR -> USD
    # using the same live USDT/INR rate used by crypto deposits.
    if currency == "USD":
        rate = await get_live_usdt_inr_rate()
        usd_amount = (amount / rate) if rate > 0 else amount
        usd_str = _fmt_decimal_trim(usd_amount, max_places=6)
        if "." in usd_str:
            a, b = usd_str.split(".", 1)
            usd_str = f"{a}.{b.ljust(2, '0')}"
        else:
            usd_str = f"{usd_str}.00"
        return f"${usd_str}"
    return f"₹{amount.quantize(DECIMAL_2, rounding=ROUND_HALF_UP)}"

async def format_rate_line(user_id: int) -> str:
    return ""

# ---------------------------------------------------------------------------
# 6. PROVIDER API
# ---------------------------------------------------------------------------
class ProviderAPIClient:
    @staticmethod
    async def call_api(api_url: str, api_key: str, params: Dict[str, Any]) -> Any:
        payload = dict(params)
        payload["key"] = api_key
        headers = {"User-Agent": "AdvancedSMM/4.0", "Accept": "application/json"}
        timeout_sec = int(await get_setting("API_TIMEOUT", "25"))
        retries = int(await get_setting("API_RETRY_COUNT", "2"))
        last_err = None
        for attempt in range(retries + 1):
            try:
                timeout = aiohttp.ClientTimeout(total=timeout_sec)
                async with aiohttp.ClientSession(timeout=timeout) as session:
                    async with session.post(api_url, data=payload, headers=headers) as resp:
                        raw_text = (await resp.text()).strip()
                        try:
                            return json.loads(raw_text)
                        except json.JSONDecodeError:
                            return {"error": "INVALID_PROVIDER_JSON", "raw": raw_text[:200]}
            except asyncio.TimeoutError:
                last_err = "PROVIDER_TIMEOUT"
            except Exception as e:
                last_err = f"CONNECTION_ERROR: {str(e)}"
            if attempt < retries:
                await asyncio.sleep(1)
        return {"error": last_err or "UNKNOWN_ERROR"}

    @classmethod
    async def _get_provider(cls, provider_id: int):
        async with get_db() as db:
            return await (await db.execute("SELECT * FROM providers WHERE id = ?", (provider_id,))).fetchone()

    @classmethod
    async def get_balance(cls, provider_id: int) -> Dict[str, Any]:
        p = await cls._get_provider(provider_id)
        if not p or not p["status"]:
            return {"error": "Provider disabled"}
        return await cls.call_api(p["api_url"], p["api_key"], {"action": "balance"})

    @classmethod
    async def create_order(cls, provider_id: int, service_id: int, link: str, quantity: int) -> Dict[str, Any]:
        p = await cls._get_provider(provider_id)
        if not p or not p["status"]:
            return {"error": "Provider disabled"}
        return await cls.call_api(p["api_url"], p["api_key"], {
            "action": "add", "service": str(service_id),
            "link": str(link).strip(), "quantity": str(quantity)
        })

    @classmethod
    async def get_order_status(cls, provider_id: int, provider_order_id: str) -> Dict[str, Any]:
        p = await cls._get_provider(provider_id)
        if not p:
            return {"error": "Provider missing"}
        return await cls.call_api(p["api_url"], p["api_key"], {
            "action": "status", "order": str(provider_order_id)
        })

    @classmethod
    async def request_refill(cls, provider_id: int, provider_order_id: str) -> Dict[str, Any]:
        p = await cls._get_provider(provider_id)
        if not p:
            return {"error": "Provider missing"}
        return await cls.call_api(p["api_url"], p["api_key"], {
            "action": "refill", "order": str(provider_order_id)
        })

    @classmethod
    async def get_refill_status(cls, provider_id: int, provider_refill_id: str) -> Dict[str, Any]:
        p = await cls._get_provider(provider_id)
        if not p:
            return {"error": "Provider missing"}
        return await cls.call_api(p["api_url"], p["api_key"], {
            "action": "refill_status", "refill": str(provider_refill_id)
        })

    @classmethod
    async def fetch_services_list(cls, provider_id: int) -> Dict[str, Any]:
        p = await cls._get_provider(provider_id)
        if not p:
            return {"success": False, "error": "Provider not found"}
        res = await cls.call_api(p["api_url"], p["api_key"], {"action": "services"})
        if isinstance(res, list):
            return {"success": True, "services": res}
        if isinstance(res, dict):
            catalog = res.get("services")
            if isinstance(catalog, list):
                return {"success": True, "services": catalog}
            for key in ("data", "result"):
                nested = res.get(key)
                if isinstance(nested, list):
                    return {"success": True, "services": nested}
                if isinstance(nested, dict) and isinstance(nested.get("services"), list):
                    return {"success": True, "services": nested["services"]}
            return {"success": False, "error": res.get("error", "Failed")}
        return {"success": False, "error": "Invalid response"}


# ---------------------------------------------------------------------------
# 7. UPI PAYMENT SYSTEM
# ---------------------------------------------------------------------------
async def generate_upi_qr_url(ref: str, amount: str = "") -> str:
    upi_id = await get_setting("UPI_ID", UPI_ID)
    upi_name = await get_setting("UPI_NAME", BOT_NAME)
    upi_link = f"upi://pay?pa={upi_id}&pn={quote(upi_name)}&tn={ref}&cu=INR"
    if amount:
        upi_link += f"&am={amount}"
    encoded = quote(upi_link, safe='')
    base = await get_setting("QR_BASE", QR_BASE)
    return f"{base}?text={encoded}&size=1080&margin=2"


async def check_payment_by_ref(ref: str) -> Dict[str, Any]:
    gmail = await get_setting("FAMPAY_GMAIL", GMAIL)
    app_pass = await get_setting("FAMPAY_APP_PASSWORD", GMAIL_APP_PASSWORD)
    verify_url = await get_setting("VERIFY_API", VERIFY_API)

    def request():
        try:
            params = {"gmail": gmail, "app_password": app_pass, "purpose": ref}
            response = requests.get(verify_url, params=params, timeout=45)
            raw = response.text
            logger.info("[UPI CHECK] ref=%s HTTP=%s", ref, response.status_code)
            try:
                data = response.json()
            except Exception:
                return {"status": "error", "raw": raw, "data": None}
            return {"status": "ok", "raw": raw, "data": data}
        except requests.Timeout:
            return {"status": "error", "raw": "Timeout", "data": None}
        except Exception as e:
            return {"status": "error", "raw": str(e), "data": None}

    return await asyncio.to_thread(request)


def parse_payment_response(data: Any) -> Dict[str, Any]:
    out = {"found": False, "amount": None, "utr": None, "duplicate": False, "raw_status": ""}
    if not isinstance(data, dict):
        return out
    payload = data.get("data") if isinstance(data.get("data"), dict) else data
    status_raw = str(payload.get("status", "")).strip().lower()
    out["raw_status"] = status_raw
    found_states = {"success", "ok", "verified", "found", "completed", "paid"}
    duplicate_states = {"repeat", "duplicate", "already_used"}
    if status_raw in found_states:
        out["found"] = True
    elif status_raw in duplicate_states:
        out["found"] = True
        out["duplicate"] = True
    for key in ("amount", "amt", "transaction_amount", "paid_amount"):
        if payload.get(key) not in (None, ""):
            try:
                out["amount"] = quantize_currency(payload[key])
                break
            except Exception:
                pass
    for key in ("utr", "transaction_id", "txn_id", "ref_id", "rrn"):
        if payload.get(key) not in (None, ""):
            out["utr"] = str(payload[key]).strip()
            break
    return out



# ---------------------------------------------------------------------------
# ⚡ UPI AUTO DEPOSIT — mixyox payment verification
# ---------------------------------------------------------------------------
# The QR carries a random 12-digit numeric transaction reference as UPI `tr`.
# After paying, the user taps "Check Payment"; the reference is looked up on the
# verification API and the wallet is credited with the paid TXNAMOUNT only.
AUTO_PAY_API_URL = os.getenv("AUTO_PAY_API_URL", "https://mixyox.in/zxox.php").strip()
AUTO_PAY_MID = os.getenv("AUTO_PAY_MID", "hAsqwQ85709194299740").strip()
AUTO_UPI_ID_DEFAULT = "paytm.s2znl0o@pty"
AUTO_UPI_NAME_DEFAULT = "ALI"


async def get_auto_upi() -> Tuple[str, str]:
    """UPI ID + payee name used in the auto-deposit QR."""
    upi_id = (await get_setting("AUTO_UPI_ID", AUTO_UPI_ID_DEFAULT) or AUTO_UPI_ID_DEFAULT).strip()
    name = (await get_setting("AUTO_UPI_NAME", AUTO_UPI_NAME_DEFAULT) or AUTO_UPI_NAME_DEFAULT).strip()
    return upi_id, name


async def new_upi_tr() -> str:
    """Random, unused 12-digit numeric transaction reference."""
    for _ in range(25):
        ref = str(secrets.randbelow(9 * 10 ** 11) + 10 ** 11)
        async with get_db() as db:
            used = await (await db.execute(
                "SELECT 1 FROM deposits WHERE reference = ? LIMIT 1", (ref,))).fetchone()
        if not used:
            return ref
    return str(secrets.randbelow(9 * 10 ** 11) + 10 ** 11)


def build_upi_pay_link(upi_id: str, payee: str, tr: str) -> str:
    """upi://pay?pa=<vpa>&pn=<name>&tn=Deposit&tr=<12-digit utr>"""
    return (f"upi://pay?pa={quote(str(upi_id), safe='@._-')}&pn={quote(str(payee), safe='')}"
            f"&tn=Deposit&tr={quote(str(tr), safe='')}")


@functools.lru_cache(maxsize=512)
def make_qr_matrix(text: str) -> Tuple[Tuple[bool, ...], ...]:
    """QR module matrix (no border). Cached: identical links are never re-encoded."""
    qr = qrcode.QRCode(version=None, error_correction=qrcode.constants.ERROR_CORRECT_M, box_size=1, border=0)
    qr.add_data(text)
    qr.make(fit=True)
    return tuple(tuple(bool(c) for c in row) for row in qr.get_matrix())


@functools.lru_cache(maxsize=512)
def make_qr_png_bytes(text: str, box: int = 10, border: int = 4) -> bytes:
    """QR -> PNG bytes using only the qrcode matrix + zlib (works even if Pillow is missing/broken)."""
    import zlib, struct
    matrix = make_qr_matrix(text)
    n = len(matrix)
    size = (n + 2 * border) * box
    white, black = b"\xff", b"\x00"
    pad = white * (border * box)
    blank_row = b"\x00" + white * size
    rows = [blank_row] * (border * box)
    for r in matrix:
        row = b"\x00" + pad + b"".join((black if cell else white) * box for cell in r) + pad
        rows.extend([row] * box)
    rows.extend([blank_row] * (border * box))
    raw = b"".join(rows)

    def chunk(tag: bytes, data: bytes) -> bytes:
        c = struct.pack(">I", len(data)) + tag + data
        return c + struct.pack(">I", zlib.crc32(tag + data) & 0xffffffff)

    return (b"\x89PNG\r\n\x1a\n" + chunk(b"IHDR", struct.pack(">IIBBBBB", size, size, 8, 0, 0, 0, 0))
            + chunk(b"IDAT", zlib.compress(raw, 6)) + chunk(b"IEND", b""))


@functools.lru_cache(maxsize=512)
def make_qr_svg(text: str, border: int = 4) -> str:
    """QR -> compact vector SVG (a few KB, instant to decode, razor sharp at any size)."""
    matrix = make_qr_matrix(text)
    n = len(matrix)
    parts = []
    for y, row in enumerate(matrix):
        x = 0
        while x < n:
            if row[x]:
                start = x
                while x < n and row[x]:
                    x += 1
                parts.append(f"M{start + border} {y + border}h{x - start}v1h-{x - start}z")
            else:
                x += 1
    total = n + 2 * border
    return (f'<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 {total} {total}" shape-rendering="crispEdges">'
            f'<rect width="{total}" height="{total}" fill="#fff"/><path fill="#000" d="{"".join(parts)}"/></svg>')


async def verify_auto_payment(ref: str) -> Dict[str, Any]:
    """Ask the verification API about a 12-digit reference.

    Returns {"state": "success" | "unpaid" | "pending" | "error", ...}
    Only a clear network/parse failure is "error"; anything that is not TXN_SUCCESS is "unpaid".
    """
    api_url = (await get_setting("AUTO_PAY_API_URL", AUTO_PAY_API_URL)) or AUTO_PAY_API_URL
    mid = ((await get_setting("AUTO_PAY_MID", AUTO_PAY_MID)) or AUTO_PAY_MID).strip()
    api_url = api_url.strip()

    def request() -> Dict[str, Any]:
        headers = {
            "User-Agent": "Mozilla/5.0 (Linux; Android 13; Pixel 7) AppleWebKit/537.36 (KHTML, like Gecko) "
                          "Chrome/124.0.0.0 Mobile Safari/537.36",
            "Accept": "application/json, text/plain, */*",
        }
        # send the reference under every name the gateway scripts commonly use
        params = {"mid": mid, "utr": ref, "orderid": ref, "order_id": ref}
        last_error = "Unknown error"
        for attempt in range(1, 4):
            try:
                response = requests.get(api_url, params=params, headers=headers, timeout=25)
                text = (response.text or "").strip().lstrip("\ufeff")
                logger.info("[UPI AUTO CHECK] GET %s mid=%s ref=%s -> HTTP %s | %s",
                            api_url, mid, ref, response.status_code, text[:600].replace("\n", " "))
                try:
                    data = json.loads(text)
                except ValueError:
                    a, b = text.find("{"), text.rfind("}")
                    if a == -1 or b <= a:
                        raise
                    data = json.loads(text[a:b + 1])
                return {"ok": True, "data": data}
            except requests.Timeout:
                last_error = "Timeout"
            except ValueError:
                last_error = "Invalid response from verification service"
            except Exception as e:
                last_error = str(e)
            logger.warning("[UPI AUTO CHECK] attempt %s failed for ref=%s: %s", attempt, ref, last_error)
            if attempt < 3:
                time.sleep(1)
        return {"ok": False, "error": last_error}

    res = await asyncio.to_thread(request)
    if not res["ok"]:
        return {"state": "error", "message": "Could not reach the verification service. Please try again in a moment."}
    data = res["data"]
    if not isinstance(data, dict):
        return {"state": "error", "message": "Unexpected response from the verification service. Please try again."}

    result = data.get("result") if isinstance(data.get("result"), dict) else None
    if result is None and "STATUS" in data:          # flat response
        result = data
    if not result:
        # success:false / empty result => transaction not found yet => not paid
        return {"state": "unpaid", "raw_status": "NOT_FOUND"}

    status = str(result.get("STATUS", "")).strip().upper()
    if status == "TXN_SUCCESS":
        order_id = str(result.get("ORDERID") or (data.get("request") or {}).get("ORDERID") or "").strip()
        if order_id and order_id != str(ref):
            logger.warning("[UPI AUTO CHECK] ORDERID %s differs from ref %s (credited anyway: API lookup was by ref)", order_id, ref)
        try:
            amount = quantize_currency(result.get("TXNAMOUNT"))
            if amount <= 0:
                raise ValueError
        except Exception:
            return {"state": "error", "message": "Payment amount could not be verified. Balance was not credited. Please contact support."}
        utr = str(result.get("BANKTXNID") or result.get("TXNID") or "").strip() or str(ref)
        return {"state": "success", "amount": amount, "utr": utr, "txnid": str(result.get("TXNID") or ""),
                "raw_status": status}
    if status in ("PENDING", "TXN_PENDING"):
        return {"state": "pending", "raw_status": status}
    # TXN_FAILURE / FAILURE / anything else => user has not paid yet
    return {"state": "unpaid", "raw_status": status or "UNKNOWN"}


UNPAID_MESSAGE = "Payment not received yet. Please pay first, then click Check Payment."


async def process_upi_auto_check(dep_id: int, uid: int, bot: Optional[Bot]) -> Dict[str, Any]:
    """Shared by the bot and the Mini App: verify an auto deposit and credit it if paid."""
    async with get_db() as db:
        dep = await (await db.execute("SELECT * FROM deposits WHERE id = ? AND user_id = ?", (dep_id, uid))).fetchone()
    if not dep:
        return {"state": "notfound"}
    status = dep["status"]
    if status == "APPROVED":
        return {"state": "already", "dep": dep}
    if status not in ("PENDING", "EXPIRED"):
        return {"state": "closed", "status": status, "dep": dep}
    is_expired = status == "EXPIRED" or bool(dep["expires_at"] and time.time() > float(dep["expires_at"]))
    ref = dep["reference"] or dep["purpose"]
    if not ref:
        return {"state": "noref", "dep": dep}

    v = await verify_auto_payment(str(ref))
    if v["state"] != "success":
        if is_expired:   # not paid and window over
            async with get_db() as db:
                await db.execute("UPDATE deposits SET status = 'EXPIRED' WHERE id = ? AND status = 'PENDING'", (dep_id,))
                await db.commit()
            return {"state": "expired", "dep": dep}
        return {"state": v["state"], "message": v.get("message", ""), "dep": dep, "ref": ref}

    received: Decimal = v["amount"]
    utr: str = v["utr"]

    async with get_db() as db:
        dup = await (await db.execute(
            "SELECT id FROM deposits WHERE utr = ? AND status = 'APPROVED' AND id != ? LIMIT 1", (utr, dep_id))).fetchone()
        if dup:
            await db.execute("UPDATE deposits SET status = 'DUPLICATE' WHERE id = ? AND status IN ('PENDING','EXPIRED')", (dep_id,))
            await db.commit()
    if dup:
        if bot is not None:
            event_text = await notify_user_deposit_event(
                bot, uid, "⚠️ <b>DUPLICATE DEPOSIT</b>", dep_id, received,
                "\n\nThis payment was already processed. Balance was not credited.")
            admin_text = event_text + f"\n👤 User: <code>{uid}</code>"
            await notify_admins(bot, admin_text)
            await send_channel_log(bot, "PAYMENT", admin_text)
        return {"state": "duplicate", "dep": dep, "ref": ref}

    try:
        async with get_db() as db:
            await db.execute("BEGIN IMMEDIATE")
            current = await (await db.execute("SELECT * FROM deposits WHERE id = ? AND status IN ('PENDING','EXPIRED')", (dep_id,))).fetchone()
            if not current:
                await db.rollback()
                return {"state": "closed", "status": "PROCESSED", "dep": dep}
            upd = await db.execute(
                "UPDATE deposits SET status = 'APPROVED', utr = ?, amount = ?, processed_at = CURRENT_TIMESTAMP "
                "WHERE id = ? AND status IN ('PENDING','EXPIRED')", (utr, str(received), dep_id))
            if upd.rowcount != 1:
                await db.rollback()
                return {"state": "closed", "status": "PROCESSED", "dep": dep}
            ok = await modify_wallet_atomic(
                db, current["user_id"], received, "Deposit", reference_id=utr,
                description=f"UPI 1 auto verified (tr {ref})")
            if not ok:
                await db.rollback()
                return {"state": "failed", "message": "Crediting failed. Contact support.", "dep": dep}
            await credit_referral_bonus(db, current["user_id"], received)
            await db.commit()
    except Exception as e:
        logger.error(f"UPI1 credit error: {e}", exc_info=True)
        return {"state": "failed", "message": "System error. Contact support.", "dep": dep}

    async with get_db() as bdb:
        brow = await (await bdb.execute("SELECT balance, first_name FROM users WHERE user_id = ?", (uid,))).fetchone()
    balance = brow["balance"] if brow else "0.00"
    name = (brow["first_name"] if brow and brow["first_name"] else "User")

    if bot is not None:
        diff = "" if received == quantize_currency(dep["amount"]) else f"\n• Requested: ₹{dep['amount']}"
        await send_channel_log(
            bot, "PAYMENT",
            f"⚡ <b>UPI 1 Auto Approved #{dep_id}</b>\n• User: <code>{uid}</code>\n• Credited: ₹{received}{diff}\n"
            f"• TR: <code>{ref}</code>\n• UTR: <code>{html.escape(utr)}</code>")
        await send_user_notification(
            bot, uid,
            "✅ <b>PAYMENT SUCCESSFUL</b>\n\n"
            f"🆔 Deposit: <b>#{dep_id}</b>\n"
            f"💰 Amount added: <b>₹{received}</b>\n"
            f"👛 New balance: <b>₹{balance}</b>\n"
            f"🔖 UTR: <code>{html.escape(utr)}</code>\n\n"
            "Your wallet has been credited successfully.")
        await notify_admins(
            bot,
            f"💰 <b>AUTO DEPOSIT SUCCESS #{dep_id}</b>\n\n👤 User: <code>{uid}</code>\n"
            f"💵 Amount: <b>₹{received}</b>\n🔢 UTR: <code>{html.escape(utr)}</code>\n"
            f"🆔 TR: <code>{ref}</code>\n🕒 {get_ist_time()}")
    return {"state": "success", "amount": received, "utr": utr, "balance": balance,
            "name": name, "dep": dep, "ref": ref}



# ---------------------------------------------------------------------------
# 8. KEYBOARDS
# ---------------------------------------------------------------------------
def get_user_main_reply_keyboard() -> ReplyKeyboardMarkup:
    kb = [
        [KeyboardButton(text="🛍️ Services"), KeyboardButton(text="📦 My Orders")],
        [KeyboardButton(text="💰 Add Funds"), KeyboardButton(text="👤 Profile")],
        [KeyboardButton(text="💬 Support")],
    ]
    if WEBAPP_PUBLIC_URL.lower().startswith("https://"):
        kb.append([KeyboardButton(text="🚀 Open Mini App", web_app=WebAppInfo(url=WEBAPP_PUBLIC_URL))])
    return ReplyKeyboardMarkup(keyboard=kb, resize_keyboard=True, is_persistent=True)


def get_user_profile_inline(currency: str) -> InlineKeyboardMarkup:
    # User notification controls were intentionally removed.
    cur_label = "₹ INR" if currency != "USD" else "$ USD"
    kb = [
        [InlineKeyboardButton(text="💰 Add Funds", callback_data="umenu_funds"),
         InlineKeyboardButton(text="🎁 Gift Card", callback_data="umenu_gift")],
        [InlineKeyboardButton(text="♻️ Refill", callback_data="umenu_refill"),
         InlineKeyboardButton(text="🔎 Track Order", callback_data="umenu_track")],
        [InlineKeyboardButton(text="📊 My Stats", callback_data="umenu_stats"),
         InlineKeyboardButton(text=f"💱 {cur_label}", callback_data="umenu_currency")],
        [InlineKeyboardButton(text="🤝 Refer & Earn", callback_data="umenu_referral")],
        [InlineKeyboardButton(text="⬅️ Back", callback_data="umenu_open_main")],
    ]
    return InlineKeyboardMarkup(inline_keyboard=kb)


def get_welcome_inline() -> InlineKeyboardMarkup:
    rows = []
    if WEBAPP_PUBLIC_URL.lower().startswith("https://"):
        rows.append([InlineKeyboardButton(text="🚀 Open Mini App", web_app=WebAppInfo(url=WEBAPP_PUBLIC_URL))])
    rows.append([InlineKeyboardButton(text="📜 Terms & Conditions", callback_data="uterms_open")])
    return InlineKeyboardMarkup(inline_keyboard=rows)


def get_terms_inline() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="⬅️ Back", callback_data="umenu_open_main")]
    ])


def get_currency_inline(current: str, rate_line: str) -> InlineKeyboardMarkup:
    inr_mark = "✅ " if current != "USD" else ""
    usd_mark = "✅ " if current == "USD" else ""
    kb = [
        [InlineKeyboardButton(text=f"{inr_mark}🇮🇳 INR (₹)", callback_data="ucur_set_INR")],
        [InlineKeyboardButton(text=f"{usd_mark}🇺🇸 USD ($)", callback_data="ucur_set_USD")],
        [InlineKeyboardButton(text="⬅️ Back", callback_data="umenu_profile")],
    ]
    return InlineKeyboardMarkup(inline_keyboard=kb)


def get_category_inline(categories: List, back_cb: str = "umenu_open_main", page: int = 1, total_pages: int = 1) -> InlineKeyboardMarkup:
    kb = []
    row = []
    for c in categories:
        row.append(InlineKeyboardButton(
            text=f"{c['icon'] or '📁'} {c['name'][:22]}",
            callback_data=f"ucat_sel_{c['id']}"
        ))
        if len(row) == 2:
            kb.append(row)
            row = []
    if row:
        kb.append(row)
    if total_pages > 1:
        kb.append(make_pagination(page, total_pages, "ucat_pg"))
    kb.append([InlineKeyboardButton(text="⬅️ Back", callback_data=back_cb)])
    return InlineKeyboardMarkup(inline_keyboard=kb)


def get_service_list_inline(services: List, cat_id: int, page: int = 1, total_pages: int = 1) -> InlineKeyboardMarkup:
    kb = []
    for s in services:
        kb.append([InlineKeyboardButton(
            text=f"{s['name'][:34]}",
            callback_data=f"usrv_sel_{s['id']}"
        )])
    if total_pages > 1:
        kb.append(make_pagination(page, total_pages, f"usvc_pg_{cat_id}"))
    kb.append([InlineKeyboardButton(text="⬅️ Back", callback_data="uorder_back_cats")])
    return InlineKeyboardMarkup(inline_keyboard=kb)


def get_service_card_inline(service_id: int, cat_id: int) -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🛒 Order Now", callback_data=f"uord_start_{service_id}")],
        [InlineKeyboardButton(text="⬅️ Back", callback_data=f"ucat_sel_{cat_id}")],
    ])


def get_order_confirm_inline() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ Confirm", callback_data="uord_do_confirm"),
         InlineKeyboardButton(text="❌ Cancel", callback_data="uord_do_cancel")],
    ])


def get_order_success_inline(order_id: int) -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔎 Track", callback_data=f"utrack_open_{order_id}"),
         InlineKeyboardButton(text="📦 My Orders", callback_data="uorders_open_main")],
        [InlineKeyboardButton(text="🏠 Menu", callback_data="umenu_open_main")],
    ])


def get_my_orders_inline(orders: List, page: int, total_pages: int) -> InlineKeyboardMarkup:
    kb = []
    for o in orders:
        svc = (o["service_name"] or "Service")[:26]
        kb.append([InlineKeyboardButton(
            text=f"#{o['id']} · {svc}",
            callback_data=f"uord_view_{o['id']}"
        )])
    if total_pages > 1:
        kb.append(make_pagination(page, total_pages, "uord_pg"))
    kb.append([InlineKeyboardButton(text="🔄 Refresh", callback_data=f"uorders_open_p_{page}"),
               InlineKeyboardButton(text="⬅️ Back", callback_data="umenu_open_main")])
    return InlineKeyboardMarkup(inline_keyboard=kb)


def get_order_detail_inline(order_id: int) -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔎 Track Full Details", callback_data=f"utrack_open_{order_id}")],
        [InlineKeyboardButton(text="📦 My Orders", callback_data="uorders_open_main"),
         InlineKeyboardButton(text="🏠 Menu", callback_data="umenu_open_main")],
    ])


def get_track_detail_inline(order_id: int, refill_available: bool) -> InlineKeyboardMarkup:
    rows = [
        [InlineKeyboardButton(text="🔄 Refresh Status", callback_data=f"utrack_refresh_{order_id}")],
    ]
    if refill_available:
        rows.append([InlineKeyboardButton(text="♻️ Refill", callback_data=f"urefill_do_{order_id}")])
    rows.append([
        InlineKeyboardButton(text="📦 My Orders", callback_data="uorders_open_main"),
        InlineKeyboardButton(text="🏠 Menu", callback_data="umenu_open_main"),
    ])
    return InlineKeyboardMarkup(inline_keyboard=rows)


async def get_add_funds_inline(currency: str) -> InlineKeyboardMarkup:
    kb=[]
    if currency=="USD":
        if await payment_method_enabled("USDT_BEP20"):
            kb.append([InlineKeyboardButton(text="🔵 CRYPTO (BEP-20 USDT)",callback_data="udep_usdt")])
        kb.append([InlineKeyboardButton(text="💱 Switch to INR",callback_data="umenu_currency")])
    else:
        if await payment_method_enabled("UPI_AUTO"):
            kb.append([InlineKeyboardButton(text="🟢 UPI 1 (Auto)",callback_data="udep_auto")])
        # UPI manual is intentionally disabled in the current production payment flow.
        if await payment_method_enabled("UPI_MANUAL"):
            kb.append([InlineKeyboardButton(text="🟡 UPI 2 (Manual)",callback_data="udep_manual")])
        if await payment_method_enabled("USDT_BEP20"):
            kb.append([InlineKeyboardButton(text="🔵 CRYPTO (BEP-20 USDT)",callback_data="udep_usdt")])
    if not kb: kb.append([InlineKeyboardButton(text="⚠️ No payment method available",callback_data="noop")])
    kb.append([InlineKeyboardButton(text="⬅️ Back",callback_data="umenu_open_main")])
    return InlineKeyboardMarkup(inline_keyboard=kb)


def get_upi1_qr_inline(dep_id: int, upi_id: str = "", amount: str = "") -> InlineKeyboardMarkup:
    # Telegram inline keyboard URL buttons do not support the `upi://` protocol.
    # Keeping such a URL here makes Telegram reject the entire keyboard with:
    # "Unsupported URL protocol". The QR itself contains the UPI deep-link, so
    # users can scan it from their payment app; verification remains API-based.
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ Check Payment", callback_data=f"upay1_check_{dep_id}")],
        [InlineKeyboardButton(text="❌ Cancel", callback_data=f"upay1_cancel_{dep_id}")],
    ])


def get_upi1_retry_inline(dep_id: int) -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔄 Check Again", callback_data=f"upay1_check_{dep_id}")],
        [InlineKeyboardButton(text="❌ Cancel", callback_data=f"upay1_cancel_{dep_id}")],
    ])


def get_upi1_expired_inline() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔄 New Deposit", callback_data="umenu_funds")],
        [InlineKeyboardButton(text="🏠 Menu", callback_data="umenu_open_main")],
    ])


def get_manual_upi_inline(dep_id: int) -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔢 Enter UTR", callback_data=f"umanual_enter_utr_{dep_id}")],
        [InlineKeyboardButton(text="❌ Cancel", callback_data=f"umanual_cancel_{dep_id}")],
    ])


def get_usdt_inline() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="⬅️ Back", callback_data="umenu_funds")],
    ])


def get_stats_inline() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔄 Refresh", callback_data="umenu_stats"),
         InlineKeyboardButton(text="⬅️ Back", callback_data="umenu_profile")],
    ])


def get_gift_card_inline() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="⬅️ Back", callback_data="umenu_profile")],
    ])


def get_refill_inline() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="⬅️ Back", callback_data="umenu_profile")],
    ])


def get_track_prompt_inline() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="⬅️ Back", callback_data="umenu_profile")],
    ])


def get_support_inline(support_username: str, support_channel: str) -> InlineKeyboardMarkup:
    clean = support_username.replace("@", "")
    rows = []
    if clean:
        rows.append([InlineKeyboardButton(text="💬 Contact Us", url=f"https://t.me/{clean}")])
    if support_channel:
        rows.append([InlineKeyboardButton(text="📢 Official Channel", url=support_channel)])
    rows.append([InlineKeyboardButton(text="⬅️ Back", callback_data="umenu_open_main")])
    return InlineKeyboardMarkup(inline_keyboard=rows)


def get_back_inline(cb: str) -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="⬅️ Back", callback_data=cb)]
    ])


def make_pagination(page: int, total_pages: int, prefix: str) -> List[InlineKeyboardButton]:
    row = []
    if page > 1:
        row.append(InlineKeyboardButton(text="⬅️ Prev", callback_data=f"{prefix}_p_{page - 1}"))
    row.append(InlineKeyboardButton(text=f"{page}/{max(1, total_pages)}", callback_data="noop"))
    if page < total_pages:
        row.append(InlineKeyboardButton(text="Next ➡️", callback_data=f"{prefix}_p_{page + 1}"))
    return row


async def render_user_categories(bot: Bot, chat_id: int, page: int = 1, message_id: Optional[int] = None):
    per_page = 8
    async with get_db() as db:
        total = (await (await db.execute(
            "SELECT COUNT(*) FROM categories WHERE is_active = 1"
        )).fetchone())[0]
        total_pages = max(1, (total + per_page - 1) // per_page)
        page = max(1, min(int(page), total_pages))
        offset = (page - 1) * per_page
        cats = await (await db.execute(
            "SELECT * FROM categories WHERE is_active = 1 ORDER BY display_order ASC, id ASC LIMIT ? OFFSET ?",
            (per_page, offset)
        )).fetchall()
    text = f"🛍️ <b>Choose a Category</b>\n\nPage <b>{page}/{total_pages}</b> · {total} categories"
    kb = get_category_inline(cats, back_cb="umenu_open_main", page=page, total_pages=total_pages)
    if message_id:
        await safe_edit_message_text(bot, chat_id, message_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    else:
        await bot.send_message(chat_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)


async def render_user_services(bot: Bot, chat_id: int, cat_id: int, page: int = 1, message_id: Optional[int] = None):
    per_page = 7
    async with get_db() as db:
        cat = await (await db.execute("SELECT name FROM categories WHERE id=? AND is_active=1", (cat_id,))).fetchone()
        total = (await (await db.execute(
            "SELECT COUNT(*) FROM services WHERE category_id=? AND is_active=1", (cat_id,)
        )).fetchone())[0]
        total_pages = max(1, (total + per_page - 1) // per_page)
        page = max(1, min(int(page), total_pages))
        offset = (page - 1) * per_page
        srvs = await (await db.execute(
            "SELECT * FROM services WHERE category_id=? AND is_active=1 ORDER BY id ASC LIMIT ? OFFSET ?",
            (cat_id, per_page, offset)
        )).fetchall()
    if not cat or not srvs:
        text = "⚠️ <b>No active services</b> in this category."
        kb = InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text="⬅️ Categories", callback_data="uorder_back_cats")]])
    else:
        text = f"🛍️ <b>{html.escape(cat['name'])}</b>\n\nSelect a service:\nPage <b>{page}/{total_pages}</b> · {total} services"
        kb = get_service_list_inline(srvs, cat_id, page=page, total_pages=total_pages)
    if message_id:
        await safe_edit_message_text(bot, chat_id, message_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    else:
        await bot.send_message(chat_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)


def get_status_emoji(st: str) -> str:
    mapping = {
        "Pending": "🟡", "Processing": "🔵", "In Progress": "🔵",
        "Completed": "🟢", "Partial": "🟠",
        "Cancelled": "🔴", "Failed": "❌",
    }
    return mapping.get(st, "⚪")


# ---------------------------------------------------------------------------
# 9. FSM STATES
# ---------------------------------------------------------------------------
class UserOrderFlow(StatesGroup):
    choosing_category = State()
    choosing_service = State()
    entering_link = State()
    entering_quantity = State()
    confirming_order = State()

class UserTrackOrderFlow(StatesGroup):
    entering_order_id = State()

class UserRefillFlow(StatesGroup):
    entering_order_id = State()

class UserDepositFlow(StatesGroup):
    entering_amount_auto = State()
    entering_utr_auto = State()
    entering_amount_manual = State()
    entering_utr_manual = State()
    entering_usdt_amount = State()
    entering_usdt_txid = State()

class UserGiftCardFlow(StatesGroup):
    entering_code = State()

class AdminOrderSearchFlow(StatesGroup):
    enter_order_id = State()
    enter_partial_refund_amount = State()

class AdminServiceEditFlow(StatesGroup):
    enter_value = State()

class AdminServiceAddFlow(StatesGroup):
    select_provider = State()
    enter_service_id = State()
    select_pricing_method = State()
    enter_fixed_rate = State()
    enter_markup_value = State()
    enter_description = State()
    select_category = State()
    confirm_save = State()

class AdminBulkImportFlow(StatesGroup):
    select_provider = State()
    enter_service_ids = State()
    select_pricing_method = State()
    enter_markup_value = State()
    select_category = State()
    enter_description = State()
    confirm_import = State()

class AdminProviderAddFlow(StatesGroup):
    enter_name = State()
    enter_url = State()
    enter_key = State()

class AdminCategoryAddFlow(StatesGroup):
    enter_name = State()
    enter_icon = State()

class AdminCategoryEditFlow(StatesGroup):
    enter_new_name = State()
    enter_new_icon = State()

class AdminUserActionFlow(StatesGroup):
    enter_search = State()
    enter_balance_amount = State()
    enter_balance_reason = State()
    enter_user_message = State()

class AdminBroadcastFlow(StatesGroup):
    enter_message = State()

class AdminGiftCardFlow(StatesGroup):
    enter_code = State()
    enter_amount = State()
    enter_uses = State()
    edit_amount = State()
    edit_uses = State()

class AdminSettingsFlow(StatesGroup):
    enter_value = State()

class AdminResetFlow(StatesGroup):
    confirm_reset = State()

class AdminDiscountFlow(StatesGroup):
    enter_discount = State()
    enter_markup = State()


# ---------------------------------------------------------------------------
# 10. BAN CHECK MIDDLEWARE
# ---------------------------------------------------------------------------
class BanCheckMiddleware(BaseMiddleware):
    async def __call__(
        self,
        handler: Callable[[TelegramObject, Dict[str, Any]], Awaitable[Any]],
        event: TelegramObject,
        data: Dict[str, Any],
    ) -> Any:
        user = data.get("event_from_user")
        if user:
            if not await is_admin(user.id):
                try:
                    async with get_db() as db:
                        row = await (await db.execute(
                            "SELECT is_banned FROM users WHERE user_id = ?", (user.id,)
                        )).fetchone()
                        if row and row["is_banned"]:
                            if isinstance(event, CallbackQuery):
                                try:
                                    await event.answer("❌ Your account is suspended.", show_alert=True)
                                except Exception:
                                    pass
                            elif isinstance(event, Message):
                                try:
                                    await event.answer("❌ Your account is suspended.")
                                except Exception:
                                    pass
                            return
                except Exception as e:
                    logger.error(f"Ban check middleware error: {e}")
        return await handler(event, data)


# ---------------------------------------------------------------------------
# 11. ROUTER & MAINTENANCE
# ---------------------------------------------------------------------------
router = Router()


TOGGLE_SETTING_KEYS = {
    "MAINTENANCE_MODE", "MAINT_NEW_ORDER", "MAINT_ADD_BALANCE", "MAINT_SERVICE",
    "PAY_UPI_AUTO_ENABLED", "PAY_UPI_MANUAL_ENABLED", "PAY_USDT_ENABLED",
    "REFERRAL_ENABLED", "LOG_CHANNEL_ENABLED", "DISCOUNT_ENABLED", "LIVE_RATE_ENABLED",
}


def _flag_on(value: Any) -> bool:
    return str(value or "").strip().lower() in ("1", "true", "on", "yes", "y", "enable", "enabled", "active")


def _normalize_flag(value: Any) -> Optional[str]:
    v = str(value or "").strip().lower()
    if v in ("1", "true", "on", "yes", "y", "enable", "enabled", "active"):
        return "1"
    if v in ("0", "false", "off", "no", "n", "disable", "disabled", "inactive"):
        return "0"
    return None


async def check_user_maintenance(event: Union[Message, CallbackQuery], feature: str = "FULL") -> bool:
    uid = event.from_user.id
    if await is_admin(uid):
        return False

    full_maint = await get_setting("MAINTENANCE_MODE", "0")
    feat_maint = "0"
    if feature == "ORDER":
        feat_maint = await get_setting("MAINT_NEW_ORDER", "0")
    elif feature == "DEPOSIT":
        feat_maint = await get_setting("MAINT_ADD_BALANCE", "0")
    elif feature == "SERVICE":
        feat_maint = await get_setting("MAINT_SERVICE", "0")

    if _flag_on(full_maint) or _flag_on(feat_maint):
        text = "🛠️ <b>Maintenance Mode</b>\n\nThis feature is temporarily under maintenance.\nPlease check back shortly."
        if isinstance(event, CallbackQuery):
            await event.answer("Feature currently under maintenance.", show_alert=True)
        else:
            await event.answer(text, parse_mode=ParseMode.HTML)
        return True
    return False


# ---------------------------------------------------------------------------
# 12. WELCOME / START
# ---------------------------------------------------------------------------
def build_welcome_text() -> str:
    return (
        f"🎉 <b>Welcome to {html.escape(BOT_NAME)}</b>\n\n"
        "Your all-in-one platform for social media growth and marketing services.\n\n"
        "✨ <b>What you can do:</b>\n"
        "• Order social media services from available categories\n"
        "• View live service prices, minimums and maximums\n"
        "• Track your orders and check current status\n"
        "• Request eligible refills where supported\n"
        "• Add balance using available payment methods\n"
        "• Get support whenever you need help\n\n"
        "Please read the Terms & Conditions before placing an order."
    )

MAIN_MENU_TEXT = (
    "🏠 <b>Main Menu</b>\n\n"
    "Choose an option from the keyboard below."
)

def build_terms_text() -> str:
    return (
        "📜 <b>Terms & Conditions</b>\n\n"
        f"Welcome to <b>{html.escape(BOT_NAME)}</b>. By using this bot, "
        "placing an order, adding funds, or continuing to use any service, "
        "you confirm that you have read and accepted these terms.\n\n"
        "<b>1. Account Responsibility</b>\n"
        "You are responsible for the Telegram account used to access this bot and for keeping your account secure.\n\n"
        "<b>2. Service Usage</b>\n"
        "Services must only be used for lawful purposes. You must not use the platform for scams, fraud, harassment, illegal activity, or harmful content.\n\n"
        "<b>3. Service Information</b>\n"
        "Service names, descriptions, minimums, maximums, refill availability, delivery speed, and other details are provided for guidance and may change according to the provider.\n\n"
        "<b>4. Prices</b>\n"
        "Service prices can change at any time because provider prices, availability, exchange rates, promotions, or platform settings may change. The price shown at checkout is the applicable price for that order.\n\n"
        "<b>5. Orders</b>\n"
        "Check the service, link, quantity, minimum, maximum, and price carefully before confirming. Once an order is submitted, cancellation is not guaranteed.\n\n"
        "<b>6. Order Delivery</b>\n"
        "Delivery times are estimates and are not guaranteed. Provider delays, platform issues, rate limits, service outages, or third-party changes can affect delivery.\n\n"
        "<b>7. Refill</b>\n"
        "Refill is available only on services that explicitly support it. Refill eligibility and duration are determined by the service/provider and may have additional conditions.\n\n"
        "<b>8. Refunds</b>\n"
        "Refunds are considered only according to the order status, provider response, and platform refund rules. Completed or successfully delivered orders are normally not refundable.\n\n"
        "<b>9. Deposits</b>\n"
        "Balance deposits are intended for use on this platform. Incorrect payment details, duplicate submissions, unsupported payment networks, or invalid transaction information may cause delays or rejection.\n\n"
        "<b>10. UPI Payments</b>\n"
        "For automatic UPI deposits, use the exact payment instructions shown by the bot and complete payment within the displayed validity period. Do not reuse expired payment references.\n\n"
        "<b>11. USDT Payments</b>\n"
        "USDT deposits are accepted only through the network and wallet address shown by the bot. Sending through an unsupported network or to an incorrect address can result in permanent loss and may not be recoverable.\n\n"
        "<b>12. Payment Verification</b>\n"
        "Payments may be checked automatically or manually. A deposit is credited only after successful verification/approval. Submitting a transaction does not itself guarantee credit.\n\n"
        "<b>13. Fraud & Abuse</b>\n"
        "Fake payment claims, duplicate transaction IDs, chargeback abuse, manipulation attempts, or other fraudulent activity can result in balance reversal, order cancellation, account suspension, and permanent ban.\n\n"
        "<b>14. Provider Changes</b>\n"
        "Third-party providers can change rates, limits, availability, delivery behavior, or service status without notice. The platform may update, disable, or remove services when required.\n\n"
        "<b>15. Account Suspension</b>\n"
        "Accounts may be suspended or restricted when there is reasonable evidence of abuse, fraud, illegal use, payment misuse, or violation of these terms.\n\n"
        "<b>16. Support</b>\n"
        "If you have a genuine issue, contact support with the relevant order/deposit ID and accurate details. Do not submit repeated or false claims.\n\n"
        "<b>17. Availability</b>\n"
        "The bot and its services are provided on an availability basis. Temporary downtime, maintenance, provider outages, or API failures may occur.\n\n"
        "<b>18. Liability</b>\n"
        "To the maximum extent permitted by applicable law, the platform's liability for a specific order is limited to the amount paid for that order.\n\n"
        "<b>19. Changes to Terms</b>\n"
        "These terms may be updated when platform features, providers, payment methods, or policies change. Continued use after an update means you accept the revised terms.\n\n"
        "<b>20. Acceptance</b>\n"
        "By continuing to use the bot, you confirm that you understand and accept these Terms & Conditions."
    )

# Backward-compatible variables for code that may reference them.
WELCOME_TEXT = build_welcome_text()
TERMS_TEXT = build_terms_text()


@router.message(Command("app"), StateFilter("*"))
async def cmd_open_app(message: Message, state: FSMContext):
    url = WEBAPP_PUBLIC_URL
    if not url.lower().startswith("https://"):
        await message.answer("🚀 The Mini App is not configured yet. Please contact support.", parse_mode=ParseMode.HTML)
        return
    kb = InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text="🚀 Open Mini App", web_app=WebAppInfo(url=url))]])
    await message.answer("✨ <b>Open the Mini App</b>\n\nOrder, add balance and track everything in a beautiful app.",
                         reply_markup=kb, parse_mode=ParseMode.HTML)


@router.message(CommandStart(), StateFilter("*"))
async def cmd_start(message: Message, state: FSMContext):
    await state.clear()
    if await check_user_maintenance(message):
        return

    uid = message.from_user.id
    first_name = message.from_user.first_name or "User"
    username = message.from_user.username or ""

    referrer_id = None
    parts = message.text.split()
    if len(parts) > 1 and parts[1].startswith("ref_"):
        try:
            cand = int(parts[1].replace("ref_", ""))
            if cand != uid:
                referrer_id = cand
        except ValueError:
            pass

    is_new = False
    async with get_db() as db:
        u = await (await db.execute("SELECT * FROM users WHERE user_id = ?", (uid,))).fetchone()
        if not u:
            is_new = True
            await db.execute(
                "INSERT INTO users (user_id, first_name, username, referrer_id) VALUES (?, ?, ?, ?)",
                (uid, first_name, username, referrer_id)
            )
            await db.commit()
            u = await (await db.execute("SELECT * FROM users WHERE user_id = ?", (uid,))).fetchone()
        else:
            await db.execute(
                "UPDATE users SET first_name = ?, username = ? WHERE user_id = ?",
                (first_name, username, uid)
            )
            await db.commit()

        if u["is_banned"] and not await is_admin(uid):
            await message.answer("❌ Your account is suspended.")
            return

    # Notify admins about new user
    if is_new:
        asyncio.create_task(notify_new_user_to_admins(
            message.bot, uid, first_name, username, referrer_id
        ))

    try:
        await message.answer(build_welcome_text(), reply_markup=get_welcome_inline(), parse_mode=ParseMode.HTML)
    except Exception as e:
        logger.error(f"Welcome msg error: {e}")

    try:
        await message.answer(MAIN_MENU_TEXT, reply_markup=get_user_main_reply_keyboard(), parse_mode=ParseMode.HTML)
    except Exception as e:
        logger.error(f"Main menu msg error: {e}")


@router.callback_query(F.data == "uterms_open", StateFilter("*"))
async def cb_terms_open(callback: CallbackQuery):
    await safe_message_edit(callback.message, build_terms_text(), reply_markup=get_terms_inline(), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data == "umenu_open_main", StateFilter("*"))
async def cb_umenu_open_main(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    try:
        await safe_message_edit(callback.message, MAIN_MENU_TEXT, reply_markup=None, parse_mode=ParseMode.HTML)
    except Exception:
        pass
    await callback.message.answer(MAIN_MENU_TEXT, reply_markup=get_user_main_reply_keyboard(), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data == "noop", StateFilter("*"))
async def cb_noop(callback: CallbackQuery):
    await callback.answer()
# ===========================================================================
async def notify_new_user_to_admins(bot: Bot, uid: int, first_name: str, username: str, referrer_id=None):
    text=(f"👤 <b>NEW USER</b>\n\n🆔 User ID: <code>{uid}</code>\n"
          f"👤 Name: <b>{html.escape(first_name or 'User')}</b>\n"
          f"🔗 Username: @{html.escape(username) if username else 'N/A'}\n"
          f"🤝 Referrer: <code>{referrer_id or 'None'}</code>\n🕒 {get_ist_time()}")
    # New-user alerts go ONLY to admins, never to the logs channel.
    await notify_admins(bot, text)

# 13. SERVICE ORDERING FLOW
# ===========================================================================
@router.message(F.text == "🛍️ Services", StateFilter("*"))
async def handle_new_order_start(message: Message, state: FSMContext):
    await state.clear()
    if await check_user_maintenance(message, "ORDER"):
        return
    async with get_db() as db:
        total = (await (await db.execute("SELECT COUNT(*) FROM categories WHERE is_active=1")).fetchone())[0]
    if not total:
        await message.answer("⚠️ No service categories available currently.")
        return
    await render_user_categories(message.bot, message.chat.id, page=1)
    await state.set_state(UserOrderFlow.choosing_category)


@router.callback_query(F.data == "umenu_services", StateFilter("*"))
async def cb_umenu_services(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    if await check_user_maintenance(callback, "ORDER"):
        return
    async with get_db() as db:
        total = (await (await db.execute("SELECT COUNT(*) FROM categories WHERE is_active=1")).fetchone())[0]
    if not total:
        await callback.answer("No categories available.", show_alert=True)
        return
    await render_user_categories(callback.bot, callback.message.chat.id, page=1, message_id=callback.message.message_id)
    await state.set_state(UserOrderFlow.choosing_category)
    await callback.answer()


@router.callback_query(F.data.startswith("ucat_pg_p_"), StateFilter("*"))
async def cb_user_categories_page(callback: CallbackQuery, state: FSMContext):
    page = int(callback.data.rsplit("_", 1)[1])
    await render_user_categories(callback.bot, callback.message.chat.id, page=page, message_id=callback.message.message_id)
    await state.set_state(UserOrderFlow.choosing_category)
    await callback.answer()


@router.callback_query(F.data == "uorder_back_cats", StateFilter("*"))
async def cb_user_order_back_cats(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    await render_user_categories(callback.bot, callback.message.chat.id, page=1, message_id=callback.message.message_id)
    await state.set_state(UserOrderFlow.choosing_category)
    await callback.answer()


@router.callback_query(F.data.startswith("ucat_sel_"), StateFilter("*"))
async def cb_user_select_category(callback: CallbackQuery, state: FSMContext):
    cat_id = int(callback.data.split("_")[2])
    await render_user_services(callback.bot, callback.message.chat.id, cat_id, page=1, message_id=callback.message.message_id)
    await state.set_state(UserOrderFlow.choosing_service)
    await callback.answer()


@router.callback_query(F.data.startswith("usvc_pg_"), StateFilter("*"))
async def cb_user_services_page(callback: CallbackQuery, state: FSMContext):
    parts = callback.data.split("_")
    cat_id = int(parts[2])
    page = int(parts[4])
    await render_user_services(callback.bot, callback.message.chat.id, cat_id, page=page, message_id=callback.message.message_id)
    await state.set_state(UserOrderFlow.choosing_service)
    await callback.answer()


@router.callback_query(F.data.startswith("usrv_sel_"), StateFilter("*"))
async def cb_user_show_service_card(callback: CallbackQuery, state: FSMContext):
    srv_id = int(callback.data.split("_")[2])
    uid = callback.from_user.id
    async with get_db() as db:
        s = await (await db.execute("SELECT * FROM services WHERE id = ?", (srv_id,))).fetchone()

    if not s or not s["is_active"]:
        await callback.answer("Service unavailable.", show_alert=True)
        return

    price = await format_price(s["selling_rate"], uid)
    refill_line = f"♻️ Refill: {refill_display(s['refill_available'], s['refill_days'], s['refill_duration'])}"
    desc = html.escape(s["description"] or "High quality, fast delivery.")

    # Show discount if applied
    discount_line = ""
    if await get_setting("DISCOUNT_ENABLED", "0") == "1":
        disc_pct = await get_setting("GLOBAL_DISCOUNT_PERCENT", "0.00")
        if Decimal(disc_pct) > 0:
            # Calculate original price
            try:
                orig = quantize_currency(Decimal(str(s["selling_rate"])) / (Decimal("1") - Decimal(disc_pct) / Decimal("100")))
                orig_disp = await format_price(orig, uid)
                discount_line = f"🎁 <b>{disc_pct}% OFF!</b> <s>{orig_disp}</s>\n"
            except Exception:
                pass

    text = (
        f"📦 <b>{html.escape(s['name'])}</b>\n\n"
        f"{discount_line}"
        f"💰 <b>Price:</b> {price} / 1000\n"
        f"📉 <b>Min:</b> {s['min_quantity']:,}\n"
        f"📈 <b>Max:</b> {s['max_quantity']:,}\n"
        f"{refill_line}\n\n"
        f"📝 <b>Description:</b>\n{desc}"
    )
    await safe_message_edit(
        callback.message, text,
        reply_markup=get_service_card_inline(srv_id, s["category_id"]),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data.startswith("uord_start_"), StateFilter("*"))
async def cb_uord_ask_link(callback: CallbackQuery, state: FSMContext):
    srv_id = int(callback.data.split("_")[2])
    await state.update_data(chosen_srv_id=srv_id)
    await safe_message_edit(
        callback.message,
        "🔗 <b>Enter Link</b>\n\nSend the target link for your order:\n<i>Example: instagram.com/username</i>",
        reply_markup=None, parse_mode=ParseMode.HTML
    )
    await state.set_state(UserOrderFlow.entering_link)
    await callback.answer()


@router.message(UserOrderFlow.entering_link)
async def handle_uord_link_input(message: Message, state: FSMContext):
    link = (message.text or "").strip()
    if not (link.startswith("http://") or link.startswith("https://") or link.startswith("tg://")):
        await message.answer("❌ Invalid link. Must start with http:// or https://. Try again:")
        return

    data = await state.get_data()
    srv_id = data.get("chosen_srv_id")
    if not srv_id:
        await state.clear()
        await message.answer("❌ Session lost. Please start again.")
        return

    async with get_db() as db:
        s = await (await db.execute(
            "SELECT min_quantity, max_quantity FROM services WHERE id = ?", (srv_id,)
        )).fetchone()

    await state.update_data(order_link=link)
    await message.answer(
        f"🔢 <b>Enter Quantity</b>\n\n📉 Min: {s['min_quantity']:,} | 📈 Max: {s['max_quantity']:,}\n\n<i>Example: 1000</i>",
        parse_mode=ParseMode.HTML
    )
    await state.set_state(UserOrderFlow.entering_quantity)


@router.message(UserOrderFlow.entering_quantity)
async def handle_uord_qty_input(message: Message, state: FSMContext):
    try:
        qty = int((message.text or "").strip().replace(",", ""))
    except ValueError:
        await message.answer("❌ Enter a valid numerical quantity:")
        return

    data = await state.get_data()
    srv_id = data.get("chosen_srv_id")
    link = data.get("order_link")
    uid = message.from_user.id

    async with get_db() as db:
        s = await (await db.execute("SELECT * FROM services WHERE id = ?", (srv_id,))).fetchone()
        u = await (await db.execute("SELECT balance FROM users WHERE user_id = ?", (uid,))).fetchone()

    if not s or not u:
        await state.clear()
        await message.answer("❌ Session lost. Please start again.")
        return

    if qty < s["min_quantity"] or qty > s["max_quantity"]:
        await message.answer(f"❌ Quantity must be between {s['min_quantity']:,} and {s['max_quantity']:,}:")
        return

    unit_rate = Decimal(str(s["selling_rate"]))
    total_cost = quantize_currency((Decimal(qty) / Decimal("1000")) * unit_rate)
    user_bal = Decimal(str(u["balance"]))
    price_disp = await format_price(total_cost, uid)
    bal_disp = await format_price(user_bal, uid)

    if user_bal < total_cost:
        await message.answer(
            f"❌ <b>Insufficient Balance</b>\n\nOrder total: <b>{price_disp}</b>\nYour balance: <b>{bal_disp}</b>\n\nPlease add funds first.",
            parse_mode=ParseMode.HTML,
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="💰 Add Funds", callback_data="umenu_funds")],
                [InlineKeyboardButton(text="🏠 Menu", callback_data="umenu_open_main")]
            ])
        )
        await state.clear()
        return

    await state.update_data(order_qty=qty, order_cost=str(total_cost))
    await state.set_state(UserOrderFlow.confirming_order)

    refill_txt = refill_display(s['refill_available'], s['refill_days'], s['refill_duration'])

    preview = (
        f"📦 <b>Confirm Order</b>\n\n"
        f"🛍️ Service: <b>{html.escape(s['name'])}</b>\n"
        f"🔗 Link: <code>{html.escape(link)}</code>\n"
        f"🔢 Quantity: <b>{qty:,}</b>\n\n"
        f"💰 Total: <b>{price_disp}</b>\n"
        f"♻️ Refill: {refill_txt}\n\n"
        "Confirm?"
    )
    await message.answer(preview, reply_markup=get_order_confirm_inline(), parse_mode=ParseMode.HTML)


@router.callback_query(F.data == "uord_do_cancel", StateFilter("*"))
async def cb_uord_do_cancel(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    await safe_message_edit(callback.message, "❌ Order cancelled.", reply_markup=None)
    await callback.answer()


@router.callback_query(F.data == "uord_do_confirm", StateFilter("*"))
@serialized_order_confirmation
async def cb_uord_do_confirm(callback: CallbackQuery, state: FSMContext):
    await callback.answer()
    data = await state.get_data()
    await state.clear()

    uid = callback.from_user.id
    srv_id = data.get("chosen_srv_id")
    link = data.get("order_link")
    qty = data.get("order_qty")
    cost = Decimal(data.get("order_cost", "0.00"))

    if not srv_id or not link or not qty:
        await safe_message_edit(callback.message, "❌ Session expired. Please start again.")
        return

    # Atomic deduction
    async with get_db() as db:
        await db.execute("BEGIN IMMEDIATE")
        try:
            s = await (await db.execute("SELECT * FROM services WHERE id = ?", (srv_id,))).fetchone()
            u = await (await db.execute("SELECT balance FROM users WHERE user_id = ?", (uid,))).fetchone()

            if not s or not s["is_active"]:
                await db.rollback()
                await safe_message_edit(callback.message, "❌ Service is currently unavailable.")
                return
            if Decimal(str(u["balance"])) < cost:
                await db.rollback()
                await safe_message_edit(callback.message, "❌ Insufficient balance.")
                return

            deducted = await modify_wallet_atomic(
                db, uid, -cost, "Order Deduction",
                description=f"Order for {s['name']}"
            )
            if not deducted:
                await db.rollback()
                await safe_message_edit(callback.message, "❌ Balance deduction error.")
                return
            await db.commit()
        except Exception as e:
            await db.rollback()
            logger.error(f"Order deduction error: {e}", exc_info=True)
            await safe_message_edit(callback.message, "❌ Order failed. Please try again.")
            return

    # Get discount info
    discount_applied = "0.00"
    original_rate = str(s["selling_rate"])
    if await get_setting("DISCOUNT_ENABLED", "0") == "1":
        discount_applied = await get_setting("GLOBAL_DISCOUNT_PERCENT", "0.00")
        try:
            disc = Decimal(discount_applied)
            if disc > 0:
                orig = quantize_currency(Decimal(str(s["selling_rate"])) / (Decimal("1") - disc / Decimal("100")))
                original_rate = str(orig)
        except Exception:
            pass

    # Call provider
    prov_res = await ProviderAPIClient.create_order(s["provider_id"], s["provider_service_id"], link, qty)
    prov_order_id = prov_res.get("order") if isinstance(prov_res, dict) else None

    async with get_db() as db:
        try:
            if prov_order_id:
                cursor = await db.execute(
                    """
                    INSERT INTO orders (
                        user_id, service_id, provider_id, provider_order_id, link, quantity, charge,
                        status, refill_available, refill_days,
                        service_name_snapshot, service_description_snapshot, service_category_snapshot,
                        service_provider_snapshot, provider_service_id_snapshot, service_rate_snapshot,
                        service_min_snapshot, service_max_snapshot, service_refill_snapshot,
                        service_refill_days_snapshot, service_cancel_snapshot,
                        discount_applied_percent, original_rate_snapshot
                    )
                    VALUES (?, ?, ?, ?, ?, ?, ?, 'Processing', ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                    """,
                    (
                        uid, srv_id, s["provider_id"], str(prov_order_id), link, qty, str(cost),
                        s["refill_available"], s["refill_days"],
                        s["name"], s["description"] or "", str(s["category_id"] or ""),
                        str(s["provider_id"] or ""), str(s["provider_service_id"] or ""),
                        str(s["selling_rate"] or s["provider_rate"] or ""),
                        int(s["min_quantity"] or 0), int(s["max_quantity"] or 0),
                        int(s["refill_available"] or 0), int(s["refill_days"] or 0),
                        int(s["cancel_available"] or 0),
                        str(discount_applied), str(original_rate)
                    )
                )
                bot_order_id = cursor.lastrowid
                await db.commit()

                price_disp = await format_price(cost, uid)
                refill_str = "✅ Yes" if s["refill_available"] else "❌ No"

                success = (
                    f"✅ <b>Order Placed</b>\n\n"
                    f"🆔 <b>#{bot_order_id}</b>\n"
                    f"🛍️ {html.escape(s['name'])}\n"
                    f"🔢 {qty:,}\n"
                    f"💰 {price_disp}\n"
                    f"📊 🔵 Processing\n"
                    f"♻️ Refill: {refill_str}"
                )
                await safe_message_edit(
                    callback.message, success,
                    reply_markup=get_order_success_inline(bot_order_id),
                    parse_mode=ParseMode.HTML
                )
                await notify_admins(callback.bot,f"📦 <b>NEW ORDER #{bot_order_id}</b>\n👤 User: <code>{uid}</code>\n🛍️ {html.escape(s['name'])}\n🔢 {qty:,}\n💰 ₹{cost}\n🕒 {get_ist_time()}")
                await send_channel_log(
                    callback.bot, "ORDER",
                    f"📦 <b>New Order #{bot_order_id}</b>\n• User: <code>{uid}</code>\n• Service: <b>{html.escape(s['name'])}</b>\n• Quantity: {qty:,}\n• Amount: ₹{cost}"
                )
            else:
                err_msg = prov_res.get("error", "Provider API rejection") if isinstance(prov_res, dict) else str(prov_res)
                if str(err_msg) == "PROVIDER_TIMEOUT":
                    # A timeout is ambiguous: the provider may have accepted the order.
                    # Do NOT refund blindly. Keep the charge reserved in a local Pending order.
                    cursor = await db.execute(
                        """INSERT INTO orders (user_id, service_id, provider_id, link, quantity, charge, status, refill_available, refill_days, service_name_snapshot, service_description_snapshot, service_category_snapshot, service_provider_snapshot, provider_service_id_snapshot, service_rate_snapshot, service_min_snapshot, service_max_snapshot, service_refill_snapshot, service_refill_days_snapshot, service_cancel_snapshot, discount_applied_percent, original_rate_snapshot)
                        VALUES (?, ?, ?, ?, ?, ?, 'Pending', ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)""",
                        (uid, srv_id, s["provider_id"], link, qty, str(cost), s["refill_available"], s["refill_days"], s["name"], s["description"] or "", str(s["category_id"] or ""), str(s["provider_id"] or ""), str(s["provider_service_id"] or ""), str(s["selling_rate"] or s["provider_rate"] or ""), int(s["min_quantity"] or 0), int(s["max_quantity"] or 0), int(s["refill_available"] or 0), int(s["refill_days"] or 0), int(s["cancel_available"] or 0), str(discount_applied), str(original_rate))
                    )
                    bot_order_id = cursor.lastrowid
                    await db.commit()
                    await safe_message_edit(callback.message, f"⏳ <b>Order Processing</b>\n\n🆔 <b>#{bot_order_id}</b>\n\nProvider did not respond in time. Your balance was kept reserved while the order is reconciled. Please check My Orders shortly.", reply_markup=InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text="📦 My Orders", callback_data="uorders_open_main")],[InlineKeyboardButton(text="💬 Support", callback_data="umenu_support")]]), parse_mode=ParseMode.HTML)
                else:
                    await modify_wallet_atomic(
                        db, uid, cost, "Full Refund",
                        description=f"Auto-Refund: Order creation rejected ({err_msg})"
                    )
                    await db.commit()
                    price_disp = await format_price(cost, uid)
                    await safe_message_edit(
                        callback.message,
                        f"❌ <b>Order Failed</b>\n\nReason: <code>{html.escape(str(err_msg))}</code>\n\n💰 <b>{price_disp}</b> refunded to your wallet.",
                    reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                        [InlineKeyboardButton(text="🏠 Menu", callback_data="umenu_open_main")]
                    ]),
                    parse_mode=ParseMode.HTML
                )
        except Exception as db_err:
            logger.error(f"DB insert after provider success failed: {db_err}", exc_info=True)
            # Do not issue an automatic refund here: the provider may already have
            # accepted the order. A refund could create a free provider order.
            # Alert admins so the provider order can be reconciled manually.
            await notify_admins(
                callback.bot,
                f"🚨 <b>CRITICAL ORDER RECONCILIATION</b>\n\n"
                f"👤 User: <code>{uid}</code>\n"
                f"🛍️ Service: <code>{srv_id}</code>\n"
                f"🔢 Quantity: <b>{qty:,}</b>\n"
                f"💰 Charged: <b>₹{cost}</b>\n"
                f"⚠️ Provider accepted/DB insert may be inconsistent.\n"
                f"Please reconcile this transaction before refunding."
            )
            await safe_message_edit(
                callback.message,
                "⚠️ <b>Order Reconciliation Required</b>\n\n"
                "The provider response was received, but the local order record could not be saved safely. "
                "Your balance has not been automatically refunded to avoid duplicate/free orders. Support has been alerted.",
                reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                    [InlineKeyboardButton(text="💬 Support", callback_data="umenu_support")],
                    [InlineKeyboardButton(text="🏠 Menu", callback_data="umenu_open_main")]
                ]),
                parse_mode=ParseMode.HTML
            )


# ===========================================================================
# 14. MY ORDERS
# ===========================================================================
@router.message(F.text == "📦 My Orders", StateFilter("*"))
async def handle_user_my_orders_msg(message: Message, state: FSMContext):
    await state.clear()
    await render_my_orders_list(message.bot, message.chat.id, message.from_user.id, page=1, message_id=None)


@router.callback_query(F.data == "uorders_open_main", StateFilter("*"))
async def cb_user_my_orders(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    await render_my_orders_list(callback.bot, callback.message.chat.id, callback.from_user.id, page=1, message_id=callback.message.message_id)
    await callback.answer()


@router.callback_query(F.data.startswith("uorders_open_p_"), StateFilter("*"))
async def cb_user_my_orders_page(callback: CallbackQuery):
    page = int(callback.data.rsplit("_", 1)[1])
    await render_my_orders_list(callback.bot, callback.message.chat.id, callback.from_user.id, page=page, message_id=callback.message.message_id)
    await callback.answer()


@router.callback_query(F.data.startswith("uord_pg_p_"), StateFilter("*"))
async def cb_user_orders_paginate(callback: CallbackQuery):
    page = int(callback.data.split("_")[3])
    await render_my_orders_list(callback.bot, callback.message.chat.id, callback.from_user.id, page=page, message_id=callback.message.message_id)
    await callback.answer()


async def render_my_orders_list(bot: Bot, chat_id: int, user_id: int, page: int = 1, message_id: Optional[int] = None):
    per_page = 5
    offset = (page - 1) * per_page

    async with get_db() as db:
        tot = (await (await db.execute(
            "SELECT COUNT(*) FROM orders WHERE user_id = ?", (user_id,)
        )).fetchone())[0]
        orders = await (await db.execute(
            """
            SELECT o.*, COALESCE(o.service_name_snapshot, s.name, 'Service') AS service_name
            FROM orders o
            LEFT JOIN services s ON o.service_id = s.id
            WHERE o.user_id = ?
            ORDER BY o.id DESC LIMIT ? OFFSET ?
            """,
            (user_id, per_page, offset)
        )).fetchall()

    if not orders:
        text = (
            "📦 <b>My Orders</b>\n\n"
            "You haven't placed any orders yet.\n\n"
            "Start by browsing our services!"
        )
        kb = InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🛍️ Browse Services", callback_data="umenu_services")],
            [InlineKeyboardButton(text="🏠 Menu", callback_data="umenu_open_main")]
        ])
    else:
        lines = ["📦 <b>My Orders</b>\n"]
        for o in orders:
            ico = get_status_emoji(o["status"])
            amount = await format_price(o["charge"], user_id)
            svc = html.escape((o["service_name"] or "Service")[:30])
            lines.append(f"<b>#{o['id']}</b> · {svc}\n{ico} {o['status']} · {amount}")
        text = "\n\n".join(lines)

        tot_pages = (tot + per_page - 1) // per_page
        kb = get_my_orders_inline(orders, page, tot_pages)

    if message_id:
        await safe_edit_message_text(bot, chat_id, message_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    else:
        await bot.send_message(chat_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)


@router.callback_query(F.data.startswith("uord_view_"), StateFilter("*"))
async def cb_user_order_short_card(callback: CallbackQuery):
    oid = int(callback.data.split("_")[2])
    uid = callback.from_user.id
    async with get_db() as db:
        o = await (await db.execute(
            "SELECT * FROM orders WHERE id = ? AND user_id = ?", (oid, uid)
        )).fetchone()

    if not o:
        await callback.answer("Order not found.", show_alert=True)
        return

    amount = await format_price(o["charge"], uid)
    ico = get_status_emoji(o["status"])
    svc = html.escape(o["service_name_snapshot"] or "Service")
    date_str = str(o["created_at"])[:16]

    text = (
        f"📦 <b>Order #{o['id']}</b>\n\n"
        f"🛍️ {svc}\n"
        f"🔗 <code>{html.escape(o['link'])}</code>\n"
        f"🔢 {o['quantity']:,}\n"
        f"💰 {amount}\n"
        f"📊 {ico} {o['status']}\n"
        f"📅 {date_str}\n\n"
        "For full details, use Track Order."
    )
    await safe_message_edit(
        callback.message, text,
        reply_markup=get_order_detail_inline(oid),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


# ===========================================================================
# 15. TRACK ORDER
# ===========================================================================
@router.callback_query(F.data == "umenu_track", StateFilter("*"))
async def cb_umenu_track(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    await safe_message_edit(
        callback.message,
        "🔎 <b>Track Order</b>\n\nEnter your Order ID:\n<i>Example: 125</i>",
        reply_markup=get_track_prompt_inline(),
        parse_mode=ParseMode.HTML
    )
    await state.set_state(UserTrackOrderFlow.entering_order_id)
    await callback.answer()


@router.message(UserTrackOrderFlow.entering_order_id)
async def handle_user_track_order_input(message: Message, state: FSMContext):
    raw = (message.text or "").strip().replace("#", "")
    if not raw.isdigit():
        await message.answer("❌ Invalid format. Enter a numerical Order ID (e.g. 125):")
        return
    order_id = int(raw)
    await state.clear()
    await render_user_track_card(message.bot, message.chat.id, message.from_user.id, order_id, message_id=None)


@router.callback_query(F.data.startswith("utrack_open_"), StateFilter("*"))
async def cb_user_track_direct(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    oid = int(callback.data.split("_")[2])
    await render_user_track_card(callback.bot, callback.message.chat.id, callback.from_user.id, oid, message_id=callback.message.message_id)
    await callback.answer()


async def render_user_track_card(bot: Bot, chat_id: int, user_id: int, order_id: int, message_id: Optional[int] = None):
    async with get_db() as db:
        o = await (await db.execute(
            """
            SELECT o.*,
                   COALESCE(o.service_name_snapshot, s.name, 'Service') AS service_name,
                   COALESCE(o.service_refill_snapshot, s.refill_available, 0) AS refill_available_snapshot,
                   COALESCE(o.service_cancel_snapshot, s.cancel_available, 0) AS cancel_snapshot
            FROM orders o
            LEFT JOIN services s ON o.service_id = s.id
            WHERE o.id = ? AND o.user_id = ?
            """,
            (order_id, user_id)
        )).fetchone()

    if not o:
        text = "❌ Order not found."
        if message_id:
            await safe_edit_message_text(bot, chat_id, message_id, text)
        else:
            await bot.send_message(chat_id, text)
        return

    amount = await format_price(o["charge"], user_id)
    ico = get_status_emoji(o["status"])
    refill_str = f"✅ Yes ({o['refill_days_snapshot']} days)" if o["refill_available_snapshot"] else "❌ No"
    cancel_str = "✅ Yes" if o["cancel_snapshot"] else "❌ No"
    delivered = max(0, int(o["quantity"]) - int(o["remains"] or 0))

    text = (
        f"📦 <b>Order #{o['id']} — Full Details</b>\n\n"
        f"🛍️ Service: <b>{html.escape(o['service_name'])}</b>\n"
        f"🔗 Link: <code>{html.escape(o['link'])}</code>\n"
        f"🔢 Quantity: <b>{o['quantity']:,}</b>\n"
        f"💰 Amount: <b>{amount}</b>\n"
        f"📊 Status: {ico} <b>{o['status']}</b>\n\n"
        f"📥 Start Count: {o['start_count']:,}\n"
        f"📉 Remains: {o['remains']:,}\n"
        f"✅ Delivered: {delivered:,}\n\n"
        f"♻️ Refill: {refill_str}\n"
        f"❌ Cancel: {cancel_str}\n\n"
        f"📅 Created: {str(o['created_at'])[:16]}\n"
        f"🔄 Updated: {str(o['updated_at'])[:16]}"
    )
    kb = get_track_detail_inline(order_id, bool(o["refill_available_snapshot"]))

    if message_id:
        await safe_edit_message_text(bot, chat_id, message_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    else:
        await bot.send_message(chat_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)


@router.callback_query(F.data.startswith("utrack_refresh_"), StateFilter("*"))
async def cb_user_track_refresh(callback: CallbackQuery):
    order_id = int(callback.data.split("_")[2])
    async with get_db() as db:
        o = await (await db.execute(
            "SELECT * FROM orders WHERE id = ? AND user_id = ?",
            (order_id, callback.from_user.id)
        )).fetchone()

    if not o or not o["provider_order_id"]:
        await callback.answer("Unable to sync with provider.", show_alert=True)
        return

    res = await ProviderAPIClient.get_order_status(o["provider_id"], o["provider_order_id"])
    if isinstance(res, dict) and "status" in res:
        raw_st = str(res.get("status", "")).strip().lower()
        remains = int(res.get("remains", 0) or 0)
        start_count = int(res.get("start_count", 0) or 0)

        mapped = "Processing"
        if raw_st == "pending": mapped = "Pending"
        elif raw_st in ("in progress", "processing"): mapped = "Processing"
        elif raw_st == "completed": mapped = "Completed"
        elif raw_st == "partial": mapped = "Partial"
        elif raw_st in ("canceled", "cancelled"): mapped = "Cancelled"
        elif raw_st == "failed": mapped = "Failed"

        notify_refund = Decimal("0.00")
        async with get_db() as db:
            if mapped == "Partial" and o["refund_state"] == "NOT_REFUNDED":
                ordered_qty = Decimal(str(o["quantity"]))
                charge_amt = Decimal(str(o["charge"]))
                rem_qty = Decimal(str(remains))
                refund_val = quantize_currency((rem_qty / ordered_qty) * charge_amt) if ordered_qty > 0 else Decimal("0.00")
                if refund_val > Decimal("0.00"):
                    notify_refund = refund_val
                    await modify_wallet_atomic(db, o["user_id"], refund_val, "Partial Refund",
                        reference_id=str(o["id"]),
                        description=f"Partial refund for order #{o['id']}")
                    await db.execute(
                        "INSERT INTO refunds (order_id, user_id, amount, reason) VALUES (?, ?, ?, 'Partial Order Unfinished Quantity')",
                        (o["id"], o["user_id"], str(refund_val))
                    )
                    await db.execute(
                        "UPDATE orders SET refund_state = 'PARTIALLY_REFUNDED', refunded_amount = ? WHERE id = ?",
                        (str(refund_val), o["id"])
                    )
            elif mapped in ("Cancelled", "Failed") and o["refund_state"] == "NOT_REFUNDED":
                charge_amt = Decimal(str(o["charge"]))
                notify_refund = charge_amt
                await modify_wallet_atomic(db, o["user_id"], charge_amt, "Full Refund",
                    reference_id=str(o["id"]),
                    description=f"Full refund for {mapped.lower()} order #{o['id']}")
                await db.execute(
                    "INSERT INTO refunds (order_id, user_id, amount, reason) VALUES (?, ?, ?, ?)",
                    (o["id"], o["user_id"], str(charge_amt), f"{mapped} Order by Provider")
                )
                await db.execute(
                    "UPDATE orders SET refund_state = 'FULLY_REFUNDED', refunded_amount = ? WHERE id = ?",
                    (str(charge_amt), o["id"])
                )

            await db.execute(
                "UPDATE orders SET status = ?, start_count = ?, remains = ?, updated_at = CURRENT_TIMESTAMP WHERE id = ?",
                (mapped, start_count, remains, o["id"])
            )
            await db.commit()

        if o["status"] != mapped:
            await notify_user_order_status_change(
                callback.bot, o["id"], o["user_id"],
                o["service_name_snapshot"] or "Service", mapped, notify_refund
            )
        await callback.answer(f"Status: {mapped}")
    else:
        await callback.answer("Status check timed out.", show_alert=True)

    await render_user_track_card(callback.bot, callback.message.chat.id, callback.from_user.id, order_id, message_id=callback.message.message_id)


# ===========================================================================
# 16. REFILL
# ===========================================================================
@router.callback_query(F.data == "umenu_refill", StateFilter("*"))
async def cb_umenu_refill(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    await safe_message_edit(
        callback.message,
        "♻️ <b>Refill Request</b>\n\nEnter your Order ID:\n<i>Example: 125</i>",
        reply_markup=get_refill_inline(),
        parse_mode=ParseMode.HTML
    )
    await state.set_state(UserRefillFlow.entering_order_id)
    await callback.answer()


@router.callback_query(F.data.startswith("urefill_do_"), StateFilter("*"))
async def cb_urefill_do(callback: CallbackQuery):
    order_id = int(callback.data.split("_")[2])
    await process_refill_request(callback.message, callback.from_user.id, order_id)
    await callback.answer()


@router.message(UserRefillFlow.entering_order_id)
async def handle_user_refill_execute(message: Message, state: FSMContext):
    raw = (message.text or "").strip().replace("#", "")
    if not raw.isdigit():
        await message.answer("❌ Invalid format. Enter a valid numerical Order ID:")
        return
    order_id = int(raw)
    await state.clear()
    await process_refill_request(message, message.from_user.id, order_id)


async def process_refill_request(message: Message, uid: int, order_id: int):
    async with get_db() as db:
        o = await (await db.execute(
            "SELECT * FROM orders WHERE id = ? AND user_id = ?", (order_id, uid)
        )).fetchone()

    if not o:
        await message.answer("❌ Order not found.")
        return
    if not o["refill_available"]:
        await message.answer("❌ This service does not support refills.")
        return
    if o["status"] not in ("Completed", "Partial"):
        await message.answer(
            f"❌ Refills only allowed for completed orders. Current: <b>{o['status']}</b>",
            parse_mode=ParseMode.HTML
        )
        return

    # Atomically claim the refill slot so double-clicks cannot create duplicate provider refills.
    async with get_db() as db:
        claimed = await db.execute(
            "UPDATE orders SET refill_status='Processing' WHERE id=? AND user_id=? AND refill_status NOT IN ('Pending','Processing')",
            (order_id, uid)
        )
        await db.commit()
        if claimed.rowcount != 1:
            await message.answer("⏳ A refill request is already pending or being processed.")
            return

    try:
        created_dt = datetime.strptime(str(o["created_at"])[:19], "%Y-%m-%d %H:%M:%S").replace(tzinfo=timezone.utc)
        refill_days = int(o["refill_days"] or 0)
        if refill_days > 0:
            allowed = timedelta(days=refill_days)
            if datetime.now(timezone.utc) - created_dt > allowed:
                await message.answer(f"❌ Refill period expired. Valid within {o['refill_days']} days.")
                return
    except Exception:
        logger.debug("Could not evaluate refill expiry for order #%s", order_id, exc_info=True)

    wait = await message.answer("♻️ <i>Requesting refill...</i>", parse_mode=ParseMode.HTML)
    res = await ProviderAPIClient.request_refill(o["provider_id"], o["provider_order_id"])
    try:
        await wait.delete()
    except Exception:
        pass

    ok = isinstance(res, dict) and (res.get("refill") or res.get("status") == "success" or not res.get("error"))
    if ok:
        pref_id = str(res.get("refill", ""))
        async with get_db() as db:
            await db.execute(
                "INSERT INTO refills (order_id, user_id, provider_id, provider_order_id, provider_refill_id, status) VALUES (?, ?, ?, ?, ?, 'Pending')",
                (o["id"], uid, o["provider_id"], o["provider_order_id"], pref_id)
            )
            await db.execute(
                "UPDATE orders SET refill_status = 'Pending', provider_refill_id = ? WHERE id = ?",
                (pref_id, o["id"])
            )
            await db.commit()
        await send_channel_log(message.bot, "REFILL", f"♻️ <b>Refill Requested:</b> Order #{o['id']}\n• User: <code>{uid}</code>")
        await message.answer(
            f"✅ <b>Refill Requested</b>\n\n📦 Order #{o['id']}\n⏳ Processing...",
            parse_mode=ParseMode.HTML,
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="🔎 Track Order", callback_data=f"utrack_open_{o['id']}")],
                [InlineKeyboardButton(text="🏠 Menu", callback_data="umenu_open_main")]
            ])
        )
    else:
        err = res.get("error", "Provider rejected refill.") if isinstance(res, dict) else str(res)
        async with get_db() as db:
            await db.execute("UPDATE orders SET refill_status='None' WHERE id=? AND refill_status='Processing'", (o["id"],))
            await db.commit()
        await message.answer(f"❌ <b>Refill Failed</b>\n\n{html.escape(str(err))}", parse_mode=ParseMode.HTML)


# ===========================================================================
# 17. PROFILE
# ===========================================================================
@router.message(F.text == "👤 Profile", StateFilter("*"))
async def handle_user_profile_msg(message: Message, state: FSMContext):
    await state.clear()
    await render_user_profile(message.bot, message.chat.id, message.from_user.id, message_id=None)


@router.callback_query(F.data == "umenu_profile", StateFilter("*"))
async def cb_umenu_profile(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    await render_user_profile(callback.bot, callback.message.chat.id, callback.from_user.id, message_id=callback.message.message_id)
    await callback.answer()


async def render_user_profile(bot: Bot, chat_id: int, user_id: int, message_id: Optional[int] = None):
    async with get_db() as db:
        u = await (await db.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))).fetchone()
        o_cnt = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE user_id = ?", (user_id,))).fetchone())[0]

    if not u:
        # self-heal: create the missing row instead of telling the user to /start again
        async with get_db() as db:
            await db.execute("INSERT OR IGNORE INTO users (user_id, first_name, username) VALUES (?, 'User', '')", (user_id,))
            await db.commit()
            u = await (await db.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))).fetchone()
            o_cnt = 0
        if not u:
            text = "❌ Profile not found. Please /start again."
            if message_id:
                await safe_edit_message_text(bot, chat_id, message_id, text)
            else:
                await bot.send_message(chat_id, text)
            return

    currency = u["currency"] or "INR"
    bal = await format_price(u["balance"], user_id, currency)
    spent = await format_price(u["total_spent"], user_id, currency)
    dep = await format_price(u["total_deposited"], user_id, currency)
    ref_earn = await format_price(u["referral_earnings"], user_id, currency)

    text = (
        f"👤 <b>{html.escape(u['first_name'] or 'User')}</b>\n\n"
        f"🆔 <code>{u['user_id']}</code>\n"
        f"💰 Balance: <b>{bal}</b>\n"
        f"📦 Orders: <b>{o_cnt}</b>\n"
        f"💵 Spent: <b>{spent}</b>\n"
        f"💳 Deposited: <b>{dep}</b>\n"
        f"🤝 Referral: <b>{ref_earn}</b>"
    )
    kb = get_user_profile_inline(currency)

    if message_id:
        await safe_edit_message_text(bot, chat_id, message_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    else:
        await bot.send_message(chat_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)


# ===========================================================================
# 18. REFERRAL (moved to profile)
# ===========================================================================
@router.callback_query(F.data == "umenu_referral", StateFilter("*"))
async def cb_umenu_referral(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    uid = callback.from_user.id

    async with get_db() as db:
        u = await (await db.execute(
            "SELECT referral_earnings FROM users WHERE user_id = ?", (uid,)
        )).fetchone()
        ref_count = (await (await db.execute(
            "SELECT COUNT(*) FROM users WHERE referrer_id = ?", (uid,)
        )).fetchone())[0]

    if not u:
        await ensure_user_row(callback.from_user)
        await callback.answer("Profile created. Please tap again.", show_alert=True)
        return

    percent = await get_setting("REFERRAL_PERCENT", "5.00")
    enabled = await get_setting("REFERRAL_ENABLED", "1")

    try:
        bot_info = await callback.bot.get_me()
        bot_username = bot_info.username
        ref_link = f"https://t.me/{bot_username}?start=ref_{uid}"
    except Exception:
        ref_link = f"https://t.me/Bot?start=ref_{uid}"

    earn = await format_price(u["referral_earnings"], uid)

    text = (
        "🤝 <b>REFER & EARN</b>\n\n"
        f"💰 Your Earnings: <b>{earn}</b>\n"
        f"👥 Total Referrals: <b>{ref_count}</b>\n"
        f"📊 Commission: <b>{percent}%</b> per deposit\n"
        f"📊 Status: <b>{'🟢 Active' if enabled == '1' else '🔴 Disabled'}</b>\n\n"
        f"━━━━━━━━━━━━━━━━━━━━\n"
        f"🔗 <b>Your Referral Link:</b>\n"
        f"<code>{ref_link}</code>\n"
        f"━━━━━━━━━━━━━━━━━━━━\n\n"
        f"<b>How it works:</b>\n"
        f"1. Share your link with friends\n"
        f"2. They sign up & deposit\n"
        f"3. You earn <b>{percent}%</b> commission instantly!\n\n"
        f"<i>Unlimited earning potential 💸</i>"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(
            text="📤 Share Link",
            url=f"https://t.me/share/url?url={ref_link}&text=Join%20this%20amazing%20SMM%20bot!"
        )],
        [InlineKeyboardButton(text="⬅️ Back", callback_data="umenu_profile")]
    ])
    await safe_message_edit(callback.message, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()


# ===========================================================================
# 19. MY STATS
# ===========================================================================
@router.callback_query(F.data == "umenu_stats", StateFilter("*"))
async def cb_umenu_stats(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    uid = callback.from_user.id
    async with get_db() as db:
        u = await (await db.execute("SELECT * FROM users WHERE user_id = ?", (uid,))).fetchone()
        tot = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE user_id = ?", (uid,))).fetchone())[0]
        comp = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE user_id = ? AND status = 'Completed'", (uid,))).fetchone())[0]
        proc = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE user_id = ? AND status IN ('Pending','Processing','In Progress')", (uid,))).fetchone())[0]
        part = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE user_id = ? AND status = 'Partial'", (uid,))).fetchone())[0]
        canc = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE user_id = ? AND status IN ('Cancelled','Failed')", (uid,))).fetchone())[0]

    if not u:
        await ensure_user_row(callback.from_user)
        await callback.answer("Profile created. Please tap again.", show_alert=True)
        return

    spent = await format_price(u["total_spent"], uid)
    dep = await format_price(u["total_deposited"], uid)

    text = (
        "📊 <b>My Stats</b>\n\n"
        f"📦 Total Orders: <b>{tot:,}</b>\n"
        f"🟢 Completed: <b>{comp:,}</b>\n"
        f"🔵 In Progress: <b>{proc:,}</b>\n"
        f"🟠 Partial: <b>{part:,}</b>\n"
        f"🔴 Cancelled/Failed: <b>{canc:,}</b>\n\n"
        f"💰 Total Spent: <b>{spent}</b>\n"
        f"💳 Total Deposited: <b>{dep}</b>"
    )
    await safe_message_edit(callback.message, text, reply_markup=get_stats_inline(), parse_mode=ParseMode.HTML)
    await callback.answer()


# ===========================================================================
# 20. CURRENCY TOGGLE
# ===========================================================================
@router.callback_query(F.data == "umenu_currency", StateFilter("*"))
async def cb_umenu_currency(callback: CallbackQuery, state: Optional[FSMContext] = None):
    if state is not None:
        await state.clear()
    uid = callback.from_user.id
    cur = await get_user_currency(uid)
    rate = await get_live_usd_rate(force=True)

    text = (
        "💱 <b>Choose Currency</b>\n\n"
        f"Current: <b>{'$ USD' if cur == 'USD' else '₹ INR'}</b>\n"
        f"Live rate: <b>1 USD = ₹{rate}</b>\n\n"
        "Note: USD accounts support crypto payments only."
    )
    await safe_message_edit(callback.message, text, reply_markup=get_currency_inline(cur, str(rate)), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("ucur_set_"), StateFilter("*"))
async def cb_ucur_set(callback: CallbackQuery):
    new_cur = callback.data.split("_")[2].upper()
    if new_cur not in ("INR", "USD"):
        await callback.answer("Invalid currency.", show_alert=True)
        return
    uid = callback.from_user.id
    async with get_db() as db:
        await db.execute("UPDATE users SET currency = ? WHERE user_id = ?", (new_cur, uid))
        await db.commit()
    await callback.answer(f"Currency set to {new_cur}")
    await cb_umenu_currency(callback, None)


# ===========================================================================
# 21. GIFT CARD
# ===========================================================================
@router.callback_query(F.data == "umenu_gift", StateFilter("*"))
async def cb_umenu_gift(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    await safe_message_edit(
        callback.message,
        "🎁 <b>Gift Card</b>\n\nEnter your gift card code:\n<i>Example: WELCOME50</i>",
        reply_markup=get_gift_card_inline(),
        parse_mode=ParseMode.HTML
    )
    await state.set_state(UserGiftCardFlow.entering_code)
    await callback.answer()


@router.message(UserGiftCardFlow.entering_code)
async def handle_user_gift_card_redeem(message: Message, state: FSMContext):
    code = (message.text or "").strip().upper()
    uid = message.from_user.id
    if not code or len(code) > 50:
        await message.answer("❌ Invalid gift card code.")
        await state.clear()
        return

    async with get_db() as db:
        await db.execute("BEGIN IMMEDIATE")
        try:
            gc = await (await db.execute(
                "SELECT * FROM gift_cards WHERE UPPER(code) = ? AND is_active = 1", (code,)
            )).fetchone()
            if not gc:
                await db.rollback()
                await message.answer("❌ Invalid or inactive gift card code.")
                await state.clear()
                return

            redemption = await (await db.execute(
                "SELECT id FROM gift_card_redemptions WHERE UPPER(code) = ? AND user_id = ?", (code, uid)
            )).fetchone()
            if redemption:
                await db.rollback()
                await message.answer("❌ You have already redeemed this gift card.")
                await state.clear()
                return

            if gc["used_count"] >= gc["max_uses"]:
                await db.rollback()
                await message.answer("❌ This gift card has reached its limit.")
                await state.clear()
                return

            amt = Decimal(str(gc["amount"]))
            await db.execute(
                "INSERT INTO gift_card_redemptions (code, user_id, amount) VALUES (?, ?, ?)",
                (code, uid, str(amt))
            )
            await db.execute(
                "UPDATE gift_cards SET used_count = used_count + 1 WHERE UPPER(code) = ?", (code,)
            )
            ok = await modify_wallet_atomic(
                db, uid, amt, "Gift Card Redemption",
                reference_id=code, description=f"Redeemed code {code}"
            )
            if not ok:
                await db.rollback()
                await message.answer("❌ Failed to add balance. Try again.")
                await state.clear()
                return
            await db.commit()
        except Exception as e:
            await db.rollback()
            logger.error(f"Gift card error: {e}", exc_info=True)
            await message.answer("❌ System error. Try again.")
            await state.clear()
            return

    await state.clear()
    amt_disp = await format_price(amt, uid)
    await send_channel_log(
        message.bot, "PAYMENT",
        f"🎁 <b>Gift Card Redeemed</b>\n• User: <code>{uid}</code>\n• Code: <code>{code}</code>\n• Amount: {amt_disp}"
    )
    await message.answer(
        f"🎉 <b>Gift Card Redeemed!</b>\n\n💰 <b>{amt_disp}</b> added to your balance.",
        parse_mode=ParseMode.HTML,
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🏠 Menu", callback_data="umenu_open_main")]
        ])
    )


# ===========================================================================
# 23. SUPPORT
# ===========================================================================
@router.message(F.text == "💬 Support", StateFilter("*"))
async def handle_user_support_msg(message: Message, state: FSMContext):
    await state.clear()
    await render_support(message.bot, message.chat.id, message_id=None)


@router.callback_query(F.data == "umenu_support", StateFilter("*"))
async def cb_umenu_support(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    await render_support(callback.bot, callback.message.chat.id, message_id=callback.message.message_id)
    await callback.answer()


async def render_support(bot: Bot, chat_id: int, message_id: Optional[int] = None):
    support_user = await get_setting("SUPPORT_USERNAME", "@telegram")
    support_chan = await get_setting("SUPPORT_CHANNEL", "https://t.me/telegram")

    text = (
        "💬 <b>Support</b>\n\n"
        "Need help? Reach out to us:\n\n"
        f"📞 Contact: <code>{html.escape(support_user)}</code>\n"
        f"📢 Channel: {html.escape(support_chan)}\n"
        "🕒 Hours: 10 AM – 10 PM IST"
    )
    kb = get_support_inline(support_user, support_chan)

    if message_id:
        await safe_edit_message_text(bot, chat_id, message_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    else:
        await bot.send_message(chat_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)


# ===========================================================================
# 24. ADD FUNDS MENU
# ===========================================================================
@router.callback_query(F.data == "umenu_funds", StateFilter("*"))
async def cb_umenu_funds(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    if await check_user_maintenance(callback, "DEPOSIT"):
        return
    uid = callback.from_user.id
    currency = await get_user_currency(uid)

    if currency == "USD":
        text = (
            "💰 <b>Add Funds</b>\n\n"
            "USD accounts support crypto payments only.\n\n"
            "🔵 CRYPTO (BEP-20 USDT) — deposit in USDT"
        )
    else:
        text = (
            "💰 <b>Add Funds</b>\n\n"
            "Choose a payment method:\n\n"
            "🟢 UPI — instant, auto verified\n"
            "🔵 USDT/Crypto — manual admin approval (BEP-20)"
        )
    await safe_message_edit(callback.message, text, reply_markup=await get_add_funds_inline(currency), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.message(F.text == "💰 Add Funds", StateFilter("*"))
async def handle_user_add_funds_msg(message: Message, state: FSMContext):
    await state.clear()
    uid = message.from_user.id
    if await check_user_maintenance(message, "DEPOSIT"):
        return
    currency = await get_user_currency(uid)

    if currency == "USD":
        text = (
            "💰 <b>Add Funds</b>\n\n"
            "USD accounts support crypto payments only.\n\n"
            "🔵 CRYPTO (BEP-20 USDT) — deposit in USDT"
        )
    else:
        text = (
            "💰 <b>Add Funds</b>\n\n"
            "Choose a payment method:\n\n"
            "🟢 UPI — instant, auto verified\n"
            "🔵 USDT/Crypto — manual admin approval (BEP-20)"
        )
    await message.answer(text, reply_markup=await get_add_funds_inline(currency), parse_mode=ParseMode.HTML)


# ===========================================================================
# 25. UPI 1 — AUTO DEPOSIT
# ===========================================================================
@router.callback_query(F.data == "udep_auto", StateFilter("*"))
async def cb_deposit_auto(callback: CallbackQuery, state: FSMContext):
    if not await payment_method_enabled("UPI_AUTO"):
        await callback.answer("UPI 1 is disabled by admin.", show_alert=True)
        return
    await state.clear()
    if await check_user_maintenance(callback, "DEPOSIT"):
        return
    min_dep = await get_setting("MIN_DEPOSIT", "1.00")
    max_dep = await get_setting("MAX_DEPOSIT", "50000.00")
    await safe_message_edit(
        callback.message,
        f"🟢 <b>Deposit</b>\n\nEnter amount:\n\n📉 Min: ₹{min_dep}\n📈 Max: ₹{max_dep}",
        reply_markup=get_back_inline("umenu_funds"),
        parse_mode=ParseMode.HTML
    )
    await state.set_state(UserDepositFlow.entering_amount_auto)
    await callback.answer()


@router.message(UserDepositFlow.entering_amount_auto)
async def handle_dep_amount_auto_in(message: Message, state: FSMContext):
    raw_amt = (message.text or "").strip()
    if not re.fullmatch(r"[0-9]{1,9}", raw_amt) or int(raw_amt) <= 0:
        await state.clear()
        await message.answer(
            "❌ <b>Invalid amount</b>\n\nOnly a positive whole number is allowed (example: <code>100</code>).\n"
            "Your deposit request was cancelled. Please start again.",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="🔄 Start Again", callback_data="umenu_funds")]]),
            parse_mode=ParseMode.HTML)
        return
    amt = quantize_currency(raw_amt)

    min_dep = Decimal(await get_setting("MIN_DEPOSIT", "1.00"))
    max_dep = Decimal(await get_setting("MAX_DEPOSIT", "50000.00"))
    if amt < min_dep or amt > max_dep:
        await message.answer(f"❌ Amount must be between ₹{min_dep} and ₹{max_dep}:")
        return

    uid = message.from_user.id
    ref = await new_upi_tr()
    expires_at = time.time() + DEPOSIT_EXPIRY_SECONDS

    try:
        async with get_db() as db:
            cursor = await db.execute(
                """
                INSERT INTO deposits (user_id, method, amount, status, reference, expires_at, purpose)
                VALUES (?, 'UPI_AUTO', ?, 'PENDING', ?, ?, ?)
                """,
                (uid, str(amt), ref, expires_at, ref)
            )
            dep_id = cursor.lastrowid
            await db.commit()
    except Exception as e:
        logger.error(f"Error creating UPI1 deposit: {e}", exc_info=True)
        await message.answer("❌ System error. Please try again.")
        await state.clear()
        return

    await state.clear()

    upi_id_val, upi_name_val = await get_auto_upi()

    # Generate the QR completely locally. This avoids any external QR API/image
    # fetch, so the QR is generated even when the QR API is unavailable.
    upi_link = build_upi_pay_link(upi_id_val, upi_name_val, ref)

    qr_file = None
    qr_error = None
    try:
        qr_file = BufferedInputFile(make_qr_png_bytes(upi_link), filename=f"upi_{ref}.png")
    except Exception as e:
        qr_error = e
        logger.error(f"UPI 1 QR generation failed for deposit #{dep_id}: {e}", exc_info=True)

    caption = (
        f"⚡ <b>Auto Payment</b>\n\n"
        f"━━━━━━━━━━━━━━━━━━━━\n"
        f"💰 <b>Amount:</b> ₹{amt}\n"
        f"🆔 <b>Transaction Ref:</b> <code>{ref}</code>\n"
        f"━━━━━━━━━━━━━━━━━━━━\n\n"
        f"🏦 <b>PAYMENT DETAILS</b>\n"
        f"👤 <b>Payee:</b> {html.escape(upi_name_val)}\n"
        f"📱 <b>UPI ID:</b> <code>{html.escape(upi_id_val)}</code>\n"
        f"💵 <b>Amount:</b> ₹{amt}\n\n"
        f"━━━━━━━━━━━━━━━━━━━━\n"
        f"⚠️ <b>IMPORTANT:</b>\n"
        f"• Scan this QR with any UPI app\n"
        f"• Enter and pay ₹{amt} in your UPI app\n"
        f"• QR expires in <b>10 minutes</b>\n"
        f"• Tap <b>Check Payment</b> after paying\n"
        f"━━━━━━━━━━━━━━━━━━━━"
    )
    kb = get_upi1_qr_inline(dep_id, upi_id=upi_id_val, amount=str(amt))

    sent_msg = None
    if qr_file is not None:
        try:
            sent_msg = await message.answer_photo(
                photo=qr_file, caption=caption, reply_markup=kb, parse_mode=ParseMode.HTML
            )
        except Exception as e:
            logger.error(f"QR photo send failed: {e}; falling back to text", exc_info=True)

    if sent_msg is None:
        # Never leave the user without a response if QR creation/sending fails.
        fallback_caption = caption + (
            "\n\n⚠️ <b>QR could not be generated right now.</b>\n"
            "Please contact support if the QR does not appear."
            if qr_error is not None else ""
        )
        sent_msg = await message.answer(
            fallback_caption, reply_markup=kb, parse_mode=ParseMode.HTML
        )

    if sent_msg:
        async with get_db() as db:
            await db.execute("UPDATE deposits SET qr_msg_id = ? WHERE id = ?", (sent_msg.message_id, dep_id))
            await db.commit()
    await notify_admins(message.bot,f"🟢 <b>NEW UPI 1 DEPOSIT</b>\n👤 <code>{uid}</code>\n💰 ₹{amt}\n🆔 #{dep_id}\n🔖 <code>{ref}</code>")
    await send_channel_log(message.bot,"PAYMENT",f"🟢 <b>New UPI 1 Deposit</b>\n• User: <code>{uid}</code>\n• Amount: ₹{amt}\n• Deposit: #{dep_id}")


@router.callback_query(F.data.startswith("upay1_check_"), StateFilter("*"))
async def cb_upi1_check(callback: CallbackQuery):
    dep_id = int(callback.data.split("_")[2])
    uid = callback.from_user.id
    await callback.answer("Checking payment...")

    try:
        res = await process_upi_auto_check(dep_id, uid, callback.bot)
    except Exception as e:
        logger.error(f"UPI1 check crashed for dep #{dep_id}: {e}", exc_info=True)
        res = {"state": "error", "message": "Temporary error while checking. Please click Check Again."}
    st = res["state"]
    dep = res.get("dep")
    expired_text = "⏰ <b>QR Expired</b>\n\nThe 10-minute window has passed.\nPlease start a new deposit."

    async def edit_keep(text, kb):
        """Edit the QR message; if Telegram refuses, send a fresh message so the Check button never disappears."""
        try:
            await callback.message.edit_text(text=text, reply_markup=kb, parse_mode=ParseMode.HTML)
        except Exception as e:
            if _is_not_modified(e):
                return
            logger.warning(f"upi1 edit failed ({e}); sending new message")
            try:
                await callback.message.answer(text, reply_markup=kb, parse_mode=ParseMode.HTML)
            except Exception as e2:
                logger.error(f"upi1 fallback send failed: {e2}")

    if st == "notfound":
        await safe_message_edit(callback.message, "❌ Deposit not found.")
    elif st == "already":
        await safe_message_edit(callback.message, "✅ Payment already credited.")
    elif st == "expired":
        await edit_keep(expired_text, get_upi1_expired_inline())
    elif st == "closed":
        await safe_message_edit(callback.message, f"❌ Deposit status: {res.get('status')}")
    elif st == "noref":
        await safe_message_edit(callback.message, "❌ Reference missing. Please start again.")
    elif st in ("unpaid", "pending"):
        upi_id_val, upi_name_val = await get_auto_upi()
        head = ("❌ <b>Payment Not Received</b>\n\nPlease pay first, then click <b>Check Payment</b>."
                if st == "unpaid" else
                "⏳ <b>Payment Processing</b>\n\nYour payment is still being confirmed. Please wait a moment, then click <b>Check Payment</b>.")
        text = (
            f"{head}\n\n"
            f"━━━━━━━━━━━━━━━━━━━━\n"
            f"🏦 <b>PAYMENT DETAILS</b>\n"
            f"👤 <b>Payee:</b> {html.escape(upi_name_val)}\n"
            f"📱 <b>UPI ID:</b> <code>{html.escape(upi_id_val)}</code>\n"
            f"🆔 <b>Transaction Ref:</b> <code>{res.get('ref')}</code>\n"
            f"━━━━━━━━━━━━━━━━━━━━\n\n"
            f"🕒 Last checked: {get_ist_time()}"
        )
        await edit_keep(text, get_upi1_retry_inline(dep_id))
    elif st == "error":
        await edit_keep(
            f"⚠️ <b>Could Not Verify</b>\n\n{html.escape(res.get('message') or 'Please try again in a moment.')}\n\n🕒 {get_ist_time()}",
            get_upi1_retry_inline(dep_id))
    elif st == "duplicate":
        await edit_keep(
            "⚠️ <b>Payment Already Used</b>\n\nThis payment was already processed.\nBalance not credited.",
            get_upi1_expired_inline())
    elif st == "failed":
        await edit_keep(f"❌ {html.escape(res.get('message') or 'System error. Contact support.')}",
                        get_upi1_retry_inline(dep_id))
    elif st == "success":
        success_text = (
            f"🎉 <b>Payment Successful!</b>\n\n"
            f"━━━━━━━━━━━━━━━━━━━━\n"
            f"👤 <b>Name:</b> {html.escape(res['name'])}\n"
            f"🆔 <b>User ID:</b> <code>{uid}</code>\n"
            f"💰 <b>Amount Added:</b> ₹{res['amount']}\n"
            f"👛 <b>New Balance:</b> ₹{res['balance']}\n"
            f"🔢 <b>UTR:</b> <code>{html.escape(res['utr'])}</code>\n"
            f"🆔 <b>Transaction Ref:</b> <code>{res['ref']}</code>\n"
            f"🕒 <b>Time:</b> {get_ist_time()}\n"
            f"━━━━━━━━━━━━━━━━━━━━\n\n"
            f"✅ Your wallet has been updated successfully.\n"
            f"<i>You can now place orders instantly.</i>"
        )
        success_kb = InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🛍️ Browse Services", callback_data="umenu_services")],
            [InlineKeyboardButton(text="👤 My Profile", callback_data="umenu_profile")],
        ])
        await edit_keep(success_text, success_kb)


@router.callback_query(F.data.startswith("upay1_cancel_"), StateFilter("*"))
async def cb_upi1_cancel(callback: CallbackQuery):
    dep_id = int(callback.data.split("_")[2])
    uid = callback.from_user.id
    async with get_db() as db:
        dep = await (await db.execute(
            "SELECT * FROM deposits WHERE id = ? AND user_id = ? AND status = 'PENDING'",
            (dep_id, uid)
        )).fetchone()
        await db.execute(
            "UPDATE deposits SET status = 'CANCELLED' WHERE id = ? AND user_id = ? AND status = 'PENDING'",
            (dep_id, uid)
        )
        await db.commit()
    if dep:
        event_text = await notify_user_deposit_event(
            callback.bot, uid, "❌ <b>DEPOSIT CANCELLED</b>", dep_id, dep["amount"],
            "\n\nYour deposit request was cancelled."
        )
        admin_text = event_text + f"\n👤 User: <code>{uid}</code>"
        await notify_admins(callback.bot, admin_text)
        await send_channel_log(callback.bot, "PAYMENT", admin_text)
    try:
        await callback.message.delete()
    except Exception:
        pass
    await callback.message.answer(
        "❌ Deposit cancelled.",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🏠 Menu", callback_data="umenu_open_main")]
        ])
    )
    await callback.answer()


# ===========================================================================
# 26. UPI 2 — MANUAL DEPOSIT
# ===========================================================================
@router.callback_query(F.data == "udep_manual", StateFilter("*"))
async def cb_deposit_manual(callback: CallbackQuery, state: FSMContext):
    if not await payment_method_enabled("UPI_MANUAL"):
        await callback.answer("UPI 2 is disabled by admin.", show_alert=True)
        return
    await state.clear()
    if await check_user_maintenance(callback, "DEPOSIT"):
        return
    min_dep = await get_setting("MIN_DEPOSIT", "1.00")
    max_dep = await get_setting("MAX_DEPOSIT", "50000.00")
    await safe_message_edit(
        callback.message,
        f"🟡 <b>UPI 2 Deposit</b>\n\nEnter amount:\n\n📉 Min: ₹{min_dep}\n📈 Max: ₹{max_dep}",
        reply_markup=get_back_inline("umenu_funds"),
        parse_mode=ParseMode.HTML
    )
    await state.set_state(UserDepositFlow.entering_amount_manual)
    await callback.answer()


@router.message(UserDepositFlow.entering_amount_manual)
async def handle_manual_amount(message: Message, state: FSMContext):
    raw_amt = (message.text or "").strip()
    if not re.fullmatch(r"[0-9]{1,9}", raw_amt) or int(raw_amt) <= 0:
        await state.clear()
        await message.answer(
            "❌ <b>Invalid amount</b>\n\nOnly a positive whole number is allowed (example: <code>100</code>).\n"
            "Your deposit request was cancelled. Please start again.",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="🔄 Start Again", callback_data="umenu_funds")]]),
            parse_mode=ParseMode.HTML)
        return
    amt = quantize_currency(raw_amt)

    min_dep = Decimal(await get_setting("MIN_DEPOSIT", "1.00"))
    max_dep = Decimal(await get_setting("MAX_DEPOSIT", "50000.00"))
    if amt < min_dep or amt > max_dep:
        await message.answer(f"❌ Amount must be between ₹{min_dep} and ₹{max_dep}:")
        return

    uid = message.from_user.id
    ref = generate_ref()
    async with get_db() as db:
        cur = await db.execute(
            "INSERT INTO deposits (user_id, method, amount, status, reference, purpose) VALUES (?, 'UPI_MANUAL', ?, 'PENDING', ?, 'MANUAL_UPI')",
            (uid, str(amt), ref)
        )
        dep_id = cur.lastrowid
        await db.commit()

    await state.update_data(manual_dep_id=dep_id)

    upi_id_val = await get_setting("UPI_ID", UPI_ID)
    upi_name_val = await get_setting("UPI_NAME", BOT_NAME)
    qr_url = await generate_upi_qr_url(ref, str(amt))

    caption = (
        f"🟡 <b>UPI 2 — Manual Payment</b>\n\n"
        f"━━━━━━━━━━━━━━━━━━━━\n"
        f"🏦 <b>PAYMENT DETAILS</b>\n"
        f"👤 <b>Payee:</b> {html.escape(upi_name_val)}\n"
        f"📱 <b>UPI ID:</b> <code>{html.escape(upi_id_val)}</code>\n"
        f"💰 <b>Amount:</b> ₹{amt}\n"
        f"🆔 <b>Reference:</b> <code>{ref}</code>\n"
        f"━━━━━━━━━━━━━━━━━━━━\n\n"
        f"Pay exact amount, then submit UTR.\n"
        f"Admin will review & approve."
    )
    kb = get_manual_upi_inline(dep_id)

    try:
        await message.answer_photo(photo=qr_url, caption=caption, reply_markup=kb, parse_mode=ParseMode.HTML)
    except Exception:
        await message.answer(caption, reply_markup=kb, parse_mode=ParseMode.HTML)


@router.callback_query(F.data.startswith("umanual_enter_utr_"), StateFilter("*"))
async def cb_manual_enter_utr(callback: CallbackQuery, state: FSMContext):
    dep_id = int(callback.data.split("_")[-1])
    await state.update_data(manual_dep_id=dep_id)
    await safe_message_edit(
        callback.message,
        "🔢 <b>Enter UTR</b>\n\nSend numeric UTR from payment app:",
        reply_markup=get_back_inline("umenu_funds"),
        parse_mode=ParseMode.HTML
    )
    await state.set_state(UserDepositFlow.entering_utr_manual)
    await callback.answer()


@router.message(UserDepositFlow.entering_utr_manual)
async def handle_manual_utr(message: Message, state: FSMContext):
    utr = (message.text or "").strip()
    if not (utr.isdigit() and 4 <= len(utr) <= 30):
        await message.answer("❌ Invalid UTR. Numeric 4-30 digits:")
        return

    data = await state.get_data()
    dep_id = data.get("manual_dep_id")
    if not dep_id:
        await message.answer("❌ Session not found. Please start again.")
        await state.clear()
        return

    async with get_db() as db:
        dep = await (await db.execute(
            "SELECT * FROM deposits WHERE id = ? AND user_id = ?", (int(dep_id), message.from_user.id)
        )).fetchone()
        if not dep or dep["status"] != "PENDING":
            await message.answer("❌ Deposit is no longer pending.")
            await state.clear()
            return
        dup = await (await db.execute(
            "SELECT id FROM deposits WHERE utr = ? AND status IN ('PENDING','APPROVED') LIMIT 1", (utr,)
        )).fetchone()
        if dup:
            await message.answer("⚠️ This UTR is already submitted. Use a different UTR.")
            return
        await db.execute("UPDATE deposits SET utr = ? WHERE id = ? AND status = 'PENDING'", (utr, int(dep_id)))
        await db.commit()

    await state.clear()
    await message.answer(
        f"✅ <b>Request Submitted</b>\n\n💰 ₹{dep['amount']}\n🔢 UTR: <code>{utr}</code>\n⏳ Admin approval pending",
        parse_mode=ParseMode.HTML,
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🏠 Menu", callback_data="umenu_open_main")]
        ])
    )

    await send_channel_log(message.bot,"PAYMENT",f"🟡 <b>New UPI 2 Deposit</b>\n• User: <code>{message.from_user.id}</code>\n• Amount: ₹{dep['amount']}\n• UTR: <code>{utr}</code>\n• Deposit: #{dep_id}")
    for admin_id in await admin_alert_targets():
        try:
            await message.bot.send_message(
                admin_id,
                f"🟡 <b>NEW UPI 2 DEPOSIT</b>\n\n"
                f"👤 User: <code>{message.from_user.id}</code>\n"
                f"💰 Amount: <b>₹{dep['amount']}</b>\n"
                f"🔢 UTR: <code>{utr}</code>\n"
                f"🆔 Deposit: #{dep_id}",
                reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                    [InlineKeyboardButton(text="✅ Approve", callback_data=f"amanual_approve_{dep_id}"),
                     InlineKeyboardButton(text="❌ Reject", callback_data=f"amanual_reject_{dep_id}")],
                ]),
                parse_mode=ParseMode.HTML
            )
        except Exception:
            pass


@router.callback_query(F.data.startswith("umanual_cancel_"), StateFilter("*"))
async def cb_manual_cancel(callback: CallbackQuery, state: FSMContext):
    dep_id = int(callback.data.split("_")[-1])
    async with get_db() as db:
        dep = await (await db.execute(
            "SELECT * FROM deposits WHERE id = ? AND user_id = ? AND status = 'PENDING'",
            (dep_id, callback.from_user.id)
        )).fetchone()
        await db.execute(
            "UPDATE deposits SET status = 'CANCELLED' WHERE id = ? AND user_id = ? AND status = 'PENDING'",
            (dep_id, callback.from_user.id)
        )
        await db.commit()
    if dep:
        event_text = await notify_user_deposit_event(
            callback.bot, callback.from_user.id, "❌ <b>DEPOSIT CANCELLED</b>",
            dep_id, dep["amount"], "\n\nYour deposit request was cancelled."
        )
        admin_text = event_text + f"\n👤 User: <code>{callback.from_user.id}</code>"
        await notify_admins(callback.bot, admin_text)
        await send_channel_log(callback.bot, "PAYMENT", admin_text)
    await state.clear()
    await safe_message_edit(
        callback.message, "❌ Manual deposit cancelled.",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🏠 Menu", callback_data="umenu_open_main")]
        ])
    )
    await callback.answer()


# ===========================================================================
# 27. USDT CRYPTO DEPOSIT — live-rate conversion + separate min/max
# ===========================================================================
@router.callback_query(F.data == "udep_usdt", StateFilter("*"))
async def cb_deposit_usdt(callback: CallbackQuery, state: FSMContext):
    if not await payment_method_enabled("USDT_BEP20"):
        await callback.answer("USDT payment is disabled by admin.", show_alert=True)
        return
    await state.clear()
    if await check_user_maintenance(callback, "DEPOSIT"):
        return
    rate = await get_live_usdt_inr_rate(force=True)
    min_usdt = await get_setting("USDT_MIN_DEPOSIT", "1.00")
    max_usdt = await get_setting("USDT_MAX_DEPOSIT", "10000.00")
    address = await get_setting("USDT_BEP20_ADDRESS", DEFAULT_SETTINGS["USDT_BEP20_ADDRESS"])

    text = (
        "🔵 <b>CRYPTO (BEP-20 USDT)</b>\n\n"
        f"📉 <b>Minimum:</b> {min_usdt} USDT\n"
        f"📈 <b>Maximum:</b> {max_usdt} USDT\n"
        f"💱 <b>Live rate:</b> 1 USDT ≈ ₹{_fmt_decimal_trim(rate, 4)}\n\n"
        f"📥 <b>Send USDT (BEP-20 network only) to:</b>\n"
        f"<code>{html.escape(address)}</code>\n\n"
        f"⚠️ <b>Only BEP-20 (BSC) network accepted.</b>\n"
        "⚠️ Send only to the address shown above.\n\n"
        "After sending, enter the exact USDT amount you paid."
    )
    await safe_message_edit(callback.message, text, reply_markup=get_usdt_inline(), parse_mode=ParseMode.HTML)
    await state.set_state(UserDepositFlow.entering_usdt_amount)
    await callback.answer()


@router.message(UserDepositFlow.entering_usdt_amount)
async def handle_usdt_amount(message: Message, state: FSMContext):
    try:
        crypto = Decimal((message.text or "").strip()).quantize(Decimal("0.000001"), rounding=ROUND_HALF_UP)
        if crypto <= 0:
            raise ValueError
    except Exception:
        await state.clear()
        await message.answer(
            "❌ <b>Invalid amount</b>\n\nEnter a positive number (example: <code>5</code>).\n"
            "Your deposit request was cancelled. Please start again.",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="🔄 Start Again", callback_data="umenu_funds")]]),
            parse_mode=ParseMode.HTML)
        return

    min_usdt = Decimal(await get_setting("USDT_MIN_DEPOSIT", "1.00"))
    max_usdt = Decimal(await get_setting("USDT_MAX_DEPOSIT", "10000.00"))
    if crypto < min_usdt or crypto > max_usdt:
        await message.answer(
            f"❌ <b>Invalid USDT Amount</b>\n\n"
            f"📉 Min: <b>{min_usdt} USDT</b>\n"
            f"📈 Max: <b>{max_usdt} USDT</b>\n\n"
            f"You entered: <b>{crypto} USDT</b>",
            parse_mode=ParseMode.HTML
        )
        return

    # Capture the live rate at the moment the user submits the amount.
    # The INR credit is then stored on the deposit, so later rate changes do
    # not alter an already-submitted deposit.
    rate = await get_live_usdt_inr_rate(force=True)
    inr = quantize_currency(crypto * rate)
    uid = message.from_user.id

    async with get_db() as db:
        cur = await db.execute(
            "INSERT INTO deposits (user_id, method, amount, crypto_amount, status, purpose) VALUES (?, 'USDT_BEP20', ?, ?, 'PENDING', 'USDT_BEP20')",
            (uid, str(inr), str(crypto))
        )
        dep_id = cur.lastrowid
        await db.commit()

    await state.update_data(usdt_dep_id=dep_id)
    await message.answer(
        f"💵 <b>USDT Payment</b>\n\n"
        f"💰 USDT Amount: <b>{crypto}</b>\n\n"
        f"🔗 Now send the <b>transaction hash / TXID</b>.\n"
        "Your deposit will be credited after admin verification.",
        parse_mode=ParseMode.HTML
    )
    await state.set_state(UserDepositFlow.entering_usdt_txid)


@router.message(UserDepositFlow.entering_usdt_txid)
async def handle_usdt_txid(message: Message, state: FSMContext):
    txid = (message.text or "").strip()
    if len(txid) < 10 or len(txid) > 120:
        await message.answer("❌ Invalid TXID. Send the transaction hash.")
        return

    data = await state.get_data()
    dep_id = data.get("usdt_dep_id")
    if not dep_id:
        await message.answer("❌ Session not found.")
        await state.clear()
        return

    async with get_db() as db:
        dep = await (await db.execute(
            "SELECT * FROM deposits WHERE id = ? AND user_id = ?", (dep_id, message.from_user.id)
        )).fetchone()
        if not dep or dep["status"] != "PENDING":
            await message.answer("❌ Deposit not found or already processed.")
            await state.clear()
            return
        dup = await (await db.execute(
            "SELECT id FROM deposits WHERE utr = ? AND status IN ('PENDING','APPROVED') LIMIT 1", (txid,)
        )).fetchone()
        if dup:
            await message.answer("⚠️ This TXID is already submitted.")
            return
        await db.execute("UPDATE deposits SET utr = ? WHERE id = ? AND status = 'PENDING'", (txid, dep_id))
        await db.commit()

    await state.clear()
    await message.answer(
        f"✅ <b>USDT Deposit Submitted</b>\n\n"
        f"💰 USDT: <b>{dep['crypto_amount']}</b>\n"
        f"💳 Credit: <b>₹{dep['amount']}</b>\n"
        f"🔗 TXID: <code>{txid}</code>\n"
        f"⏳ Admin approval pending",
        parse_mode=ParseMode.HTML,
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🏠 Menu", callback_data="umenu_open_main")]
        ])
    )

    await send_channel_log(message.bot,"PAYMENT",f"🔵 <b>New USDT Deposit</b>\n• User: <code>{message.from_user.id}</code>\n• Credit: ₹{dep['amount']}\n• TXID: <code>{txid}</code>\n• Deposit: #{dep_id}")
    for admin_id in await admin_alert_targets():
        try:
            await message.bot.send_message(
                admin_id,
                f"🔵 <b>NEW USDT DEPOSIT</b>\n\n"
                f"👤 User: <code>{message.from_user.id}</code>\n"
                f"💰 USDT: <b>{dep['crypto_amount']}</b>\n"
                f"💳 Credit: <b>₹{dep['amount']}</b>\n"
                f"🔗 TXID: <code>{txid}</code>\n"
                f"🆔 Deposit: #{dep_id}",
                reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                    [InlineKeyboardButton(text="✅ Approve", callback_data=f"amanual_approve_{dep_id}"),
                     InlineKeyboardButton(text="❌ Reject", callback_data=f"amanual_reject_{dep_id}")],
                ]),
                parse_mode=ParseMode.HTML
            )
        except Exception:
            pass

# ---------------------------------------------------------------------------
# 🛡️ SECURITY & SYSTEM HEALTH
# ---------------------------------------------------------------------------
@router.callback_query(F.data == "asec_health", StateFilter("*"))
async def cb_security_health(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    text = await format_security_health(callback.bot)
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔄 Run Again", callback_data="asec_health")],
        [InlineKeyboardButton(text="⬅️ Back", callback_data="ahub_back")]
    ])
    await safe_message_edit(callback.message, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await log_admin_event(callback.from_user.id, "SECURITY_HEALTH_CHECK", "", "Live runtime audit")
    await callback.answer("Security health checked.")

# ===========================================================================
# 👑 COMPLETE ADMIN PANEL
# ===========================================================================
@router.message(Command("admin"), StateFilter("*"))
async def cmd_admin_hub(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    await state.clear()
    await render_admin_panel(message.bot, message.chat.id)


async def render_admin_panel(bot: Bot, chat_id: int, message_id: Optional[int] = None):
    async with get_db() as db:
        users = (await (await db.execute("SELECT COUNT(*) FROM users")).fetchone())[0]
        orders = (await (await db.execute("SELECT COUNT(*) FROM orders")).fetchone())[0]
        pending_orders = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE status='Pending'")).fetchone())[0]
        completed_orders = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE status='Completed'")).fetchone())[0]
        failed_orders = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE status='Failed'")).fetchone())[0]
        cancelled_orders = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE status='Cancelled'")).fetchone())[0]
        revenue = (await (await db.execute("SELECT COALESCE(SUM(CAST(charge AS REAL)),0) FROM orders WHERE status NOT IN ('Cancelled','Failed')")).fetchone())[0] or 0
        refunds = (await (await db.execute("SELECT COALESCE(SUM(CAST(amount AS REAL)),0) FROM refunds")).fetchone())[0] or 0
        deposits = (await (await db.execute("SELECT COALESCE(SUM(CAST(amount AS REAL)),0) FROM deposits WHERE status='APPROVED'")).fetchone())[0] or 0
        pending_deposits = (await (await db.execute("SELECT COUNT(*) FROM deposits WHERE status='PENDING'")).fetchone())[0]
        services = (await (await db.execute("SELECT COUNT(*) FROM services WHERE is_active=1")).fetchone())[0]
        providers = (await (await db.execute("SELECT COUNT(*) FROM providers WHERE status=1")).fetchone())[0]
        categories = (await (await db.execute("SELECT COUNT(*) FROM categories WHERE is_active=1")).fetchone())[0]
        gift_cards = (await (await db.execute("SELECT COUNT(*) FROM gift_cards WHERE is_active=1")).fetchone())[0]

    text = (
        "👑 <b>ADMIN PANEL</b>\n\n"
        "📊 <b>DASHBOARD</b>\n"
        "━━━━━━━━━━━━━━━━━━\n"
        f"👥 Users: <b>{users:,}</b>\n"
        f"📦 Orders: <b>{orders:,}</b>\n"
        f"• 🟡 Pending: <b>{pending_orders}</b>\n"
        f"• 🟢 Completed: <b>{completed_orders}</b>\n"
        f"• 🔴 Cancelled: <b>{cancelled_orders}</b>\n"
        f"• ❌ Failed: <b>{failed_orders}</b>\n\n"
        "💰 <b>FINANCIALS</b>\n"
        f"• 💵 Net Sales: <b>₹{revenue:,.2f}</b>\n"
        f"• ↩️ Refunds: <b>₹{refunds:,.2f}</b>\n"
        f"• 💳 Deposits: <b>₹{deposits:,.2f}</b>\n"
        f"• ⏳ Pending Deposits: <b>{pending_deposits}</b>\n\n"
        "⚙️ <b>SYSTEM</b>\n"
        f"• Services: <b>{services}</b> | Providers: <b>{providers}</b>\n"
        f"• Categories: <b>{categories}</b> | Gift Cards: <b>{gift_cards}</b>\n"
        "━━━━━━━━━━━━━━━━━━\n\n"
        "👇 <b>Select a module:</b>"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="📊 Dashboard", callback_data="ahub_dash"),
         InlineKeyboardButton(text="📦 Orders", callback_data="aord_menu")],
        [InlineKeyboardButton(text="👥 Users", callback_data="ahub_users"),
         InlineKeyboardButton(text="💰 Payments", callback_data="apay_menu")],
        [InlineKeyboardButton(text="Services", callback_data="ahub_srv_p_1"),
         InlineKeyboardButton(text="Add Service", callback_data="asrv_add")],
        [InlineKeyboardButton(text="Bulk Import", callback_data="asrv_bulk")],
        [InlineKeyboardButton(text="📂 Categories", callback_data="ahub_cat_p_1"),
         InlineKeyboardButton(text="💳 Payment Methods", callback_data="apay_methods")],
        [InlineKeyboardButton(text="🔌 Providers", callback_data="ahub_prov_p_1"),
         InlineKeyboardButton(text="🎁 Gift Cards", callback_data="ahub_gc_p_1")],
        [InlineKeyboardButton(text="♻️ Refills", callback_data="arefill_menu"),
         InlineKeyboardButton(text="📢 Broadcast", callback_data="ahub_broadcast")],
        [InlineKeyboardButton(text="🚨 Reports", callback_data="arep_menu"),
         InlineKeyboardButton(text="🛠 Maintenance", callback_data="ahub_maint")],
        [InlineKeyboardButton(text="🎯 Discount Manager", callback_data="adisc_menu"),
         InlineKeyboardButton(text="⚙️ Settings", callback_data="ahub_settings")],
        [InlineKeyboardButton(text="🛡️ Security & Health", callback_data="asec_health")],
        [InlineKeyboardButton(text="🔐 Admin Manager", callback_data="aadmin_mgmt")],
        [InlineKeyboardButton(text="📢 Log Channel", callback_data="alog_channel_cfg"),
         InlineKeyboardButton(text="🗑️ FULL RESET", callback_data="areset_menu")],
        [InlineKeyboardButton(text="❌ Close", callback_data="ahub_close")],
    ])
    if message_id:
        await safe_edit_message_text(bot, chat_id, message_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    else:
        await bot.send_message(chat_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)


@router.callback_query(F.data == "ahub_close", StateFilter("*"))
async def cb_admin_close(callback: CallbackQuery):
    try:
        await callback.message.delete()
    except Exception:
        pass
    await callback.answer()


@router.callback_query(F.data == "ahub_back", StateFilter("*"))
async def cb_admin_back(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    await render_admin_panel(callback.bot, callback.message.chat.id, message_id=callback.message.message_id)
    await callback.answer()


# ---------------------------------------------------------------------------
# 📊 DASHBOARD
# ---------------------------------------------------------------------------
@router.callback_query(F.data == "ahub_dash", StateFilter("*"))
async def cb_admin_dashboard(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    async with get_db() as db:
        u_tot = (await (await db.execute("SELECT COUNT(*) FROM users")).fetchone())[0]
        o_tot = (await (await db.execute("SELECT COUNT(*) FROM orders")).fetchone())[0]
        o_pend = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE status = 'Pending'")).fetchone())[0]
        o_proc = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE status IN ('Processing', 'In Progress')")).fetchone())[0]
        o_comp = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE status = 'Completed'")).fetchone())[0]
        o_part = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE status = 'Partial'")).fetchone())[0]
        o_canc = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE status = 'Cancelled'")).fetchone())[0]
        o_fail = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE status = 'Failed'")).fetchone())[0]
        dep_tot = (await (await db.execute("SELECT SUM(CAST(amount AS REAL)) FROM deposits WHERE status = 'APPROVED'")).fetchone())[0] or 0.0
        ref_tot = (await (await db.execute("SELECT SUM(CAST(amount AS REAL)) FROM refunds")).fetchone())[0] or 0.0
        rev_tot = (await (await db.execute("SELECT SUM(CAST(charge AS REAL)) FROM orders")).fetchone())[0] or 0.0
        srv_act = (await (await db.execute("SELECT COUNT(*) FROM services WHERE is_active = 1")).fetchone())[0]
        prov_act = (await (await db.execute("SELECT COUNT(*) FROM providers WHERE status = 1")).fetchone())[0]

    text = (
        "📊 <b>ADMIN LIVE DASHBOARD</b>\n\n"
        f"👥 <b>Users:</b> <b>{u_tot:,}</b>\n\n"
        "📦 <b>Orders Breakdown:</b>\n"
        f"• Total: <b>{o_tot:,}</b>\n"
        f"• 🟡 Pending: <b>{o_pend}</b>\n"
        f"• 🔵 Processing: <b>{o_proc}</b>\n"
        f"• 🟢 Completed: <b>{o_comp}</b>\n"
        f"• 🟠 Partial: <b>{o_part}</b>\n"
        f"• 🔴 Cancelled: <b>{o_canc}</b>\n"
        f"• ❌ Failed: <b>{o_fail}</b>\n\n"
        "💰 <b>Financials:</b>\n"
        f"• 💰 Deposits: <b>₹{dep_tot:,.2f}</b>\n"
        f"• ↩️ Refunds: <b>₹{ref_tot:,.2f}</b>\n"
        f"• 📈 Revenue: <b>₹{rev_tot:,.2f}</b>\n\n"
        "🔌 <b>Infrastructure:</b>\n"
        f"• 🛍️ Active Services: <b>{srv_act}</b>\n"
        f"• 🔌 Active Providers: <b>{prov_act}</b>"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔄 Refresh", callback_data="ahub_dash"),
         InlineKeyboardButton(text="⬅️ Back", callback_data="ahub_back")]
    ])
    await safe_message_edit(callback.message, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()


# ---------------------------------------------------------------------------
# 🎯 DISCOUNT MANAGER
# ---------------------------------------------------------------------------
@router.callback_query(F.data == "adisc_menu", StateFilter("*"))
async def admin_discount_menu(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    enabled = await get_setting("DISCOUNT_ENABLED", "0")
    discount = await get_setting("GLOBAL_DISCOUNT_PERCENT", "0.00")
    markup = await get_setting("GLOBAL_MARKUP_PERCENT", "20.00")

    status = "🟢 ACTIVE" if enabled == "1" else "🔴 INACTIVE"
    net_profit = Decimal(markup) - Decimal(discount) if enabled == "1" else Decimal(markup)

    text = (
        "🎯 <b>GLOBAL DISCOUNT MANAGER</b>\n\n"
        f"📈 Global Markup: <b>{markup}%</b>\n"
        f"🎁 Discount: <b>{discount}%</b>\n"
        f"📊 Status: <b>{status}</b>\n"
        f"💵 Net Admin Profit: <b>{net_profit}%</b>\n\n"
        f"<i>Discount applies on final selling price.\n"
        f"Admin profit must stay above 0%.</i>\n\n"
        "👇 Select option:"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(
            text=f"🔁 Toggle Discount ({'ON' if enabled == '1' else 'OFF'})",
            callback_data="adisc_toggle"
        )],
        [InlineKeyboardButton(text="✏️ Set Discount %", callback_data="adisc_set")],
        [InlineKeyboardButton(text="📈 Set Markup % (Direct)", callback_data="amarkup_set")],
        [InlineKeyboardButton(text="🔄 Recalculate All Services", callback_data="adisc_recalc")],
        [InlineKeyboardButton(text="⬅️ Back", callback_data="ahub_back")]
    ])
    await safe_message_edit(callback.message, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data == "adisc_toggle", StateFilter("*"))
async def admin_discount_toggle(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    cur = await get_setting("DISCOUNT_ENABLED", "0")
    await set_setting("DISCOUNT_ENABLED", "0" if cur == "1" else "1")
    await log_admin_event(callback.from_user.id, "TOGGLE_DISCOUNT", "GLOBAL", f"→ {'OFF' if cur == '1' else 'ON'}")
    await callback.answer("Discount toggled!")
    await admin_discount_menu(callback)


@router.callback_query(F.data == "adisc_set", StateFilter("*"))
async def admin_discount_set_prompt(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    await state.update_data(setting_key="GLOBAL_DISCOUNT_PERCENT")
    await state.set_state(AdminSettingsFlow.enter_value)
    markup = await get_setting("GLOBAL_MARKUP_PERCENT", "20.00")
    cur = await get_setting("GLOBAL_DISCOUNT_PERCENT", "0.00")
    await callback.message.answer(
        f"🎁 <b>Set Global Discount %</b>\n\n"
        f"📈 Current Markup: <b>{markup}%</b>\n"
        f"🎁 Current Discount: <b>{cur}%</b>\n\n"
        f"⚠️ <b>IMPORTANT:</b> Discount must be LESS than markup, "
        f"otherwise admin profit will go negative.\n\n"
        f"Max allowed: <b>{Decimal(markup) - Decimal('0.01')}%</b>\n\n"
        f"Send the new discount %:",
        parse_mode=ParseMode.HTML,
        reply_markup=get_back_inline("adisc_menu")
    )
    await callback.answer()


@router.callback_query(F.data == "amarkup_set", StateFilter("*"))
async def admin_markup_set_prompt(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    await state.update_data(setting_key="GLOBAL_MARKUP_PERCENT")
    await state.set_state(AdminSettingsFlow.enter_value)
    cur = await get_setting("GLOBAL_MARKUP_PERCENT", "20.00")
    await callback.message.answer(
        f"📈 <b>Set Global Markup %</b>\n\n"
        f"Current: <b>{cur}%</b>\n\n"
        f"⚠️ This is the <b>DIRECT markup value</b>.\n"
        f"Example: Provider rate ₹100, markup 5% → selling ₹105.\n"
        f"Provider rate ₹100, markup 20% → selling ₹120.\n\n"
        f"Send the new markup %:",
        parse_mode=ParseMode.HTML,
        reply_markup=get_back_inline("adisc_menu")
    )
    await callback.answer()


@router.callback_query(F.data == "adisc_recalc", StateFilter("*"))
async def admin_discount_recalc(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    wait = await callback.message.answer("🔄 <i>Recalculating all service prices...</i>", parse_mode=ParseMode.HTML)
    updated = 0
    skipped = 0
    async with get_db() as db:
        services = await (await db.execute("SELECT * FROM services")).fetchall()
        for s in services:
            try:
                pr = Decimal(str(s["provider_rate"]))
                old_sell = Decimal(str(s["selling_rate"]))
                if str(s["profit_type"]).upper() == "FIXED":
                    skipped += 1
                    continue
                new_sell = await calculate_service_selling_rate(s, pr)
                if new_sell != old_sell:
                    await db.execute(
                        "UPDATE services SET selling_rate=? WHERE id=?",
                        (str(new_sell), s["id"])
                    )
                    updated += 1
                else:
                    skipped += 1
            except Exception as e:
                logger.warning(f"Recalc failed for #{s['id']}: {e}")
                skipped += 1
        await db.commit()

    await log_admin_event(callback.from_user.id, "RECALC_PRICES", "ALL", f"Updated={updated}, Skipped={skipped}")
    await wait.edit_text(
        f"✅ <b>Recalculation Complete</b>\n\n"
        f"🔄 Updated: <b>{updated}</b>\n"
        f"⏭️ Skipped: <b>{skipped}</b>\n\n"
        f"<i>All services now follow global markup + discount.</i>",
        parse_mode=ParseMode.HTML,
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="⬅️ Back", callback_data="adisc_menu")]
        ])
    )
    await callback.answer()


# ---------------------------------------------------------------------------
# 🔐 ADMIN MANAGER (Add/Remove Admin)
# ---------------------------------------------------------------------------
@router.callback_query(F.data == "aadmin_mgmt", StateFilter("*"))
async def admin_management(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    all_admins = await get_all_admin_ids()
    base_admins = "\n".join(f"• <code>{x}</code> (env)" for x in ADMIN_IDS)
    async with get_db() as db:
        extras = await (await db.execute("SELECT user_id FROM extra_admins")).fetchall()
    extra_admins = "\n".join(f"• <code>{r['user_id']}</code> (added)" for r in extras) if extras else "<i>None</i>"

    text = (
        "🔐 <b>ADMIN MANAGEMENT</b>\n\n"
        f"<b>Base Admins (env):</b>\n{base_admins}\n\n"
        f"<b>Extra Admins (DB):</b>\n{extra_admins}\n\n"
        f"Total: <b>{len(all_admins)}</b>"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="➕ Add Admin", callback_data="aadmin_add")],
        [InlineKeyboardButton(text="➖ Remove Admin", callback_data="aadmin_remove")],
        [InlineKeyboardButton(text="⬅️ Back", callback_data="ahub_back")]
    ])
    await safe_message_edit(callback.message, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data == "aadmin_add", StateFilter("*"))
async def admin_add_prompt(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    await state.update_data(admin_action="add")
    await state.set_state(AdminSettingsFlow.enter_value)
    await callback.message.answer(
        "➕ <b>Add Admin</b>\n\n"
        "Send the Telegram User ID to add as admin:\n"
        "<i>Example: 123456789</i>\n\n"
        "⚠️ Make sure the user has started the bot first.",
        parse_mode=ParseMode.HTML,
        reply_markup=get_back_inline("aadmin_mgmt")
    )
    await callback.answer()


@router.callback_query(F.data == "aadmin_remove", StateFilter("*"))
async def admin_remove_prompt(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    if len(ADMIN_IDS) <= 1:
        async with get_db() as db:
            cnt = (await (await db.execute("SELECT COUNT(*) FROM extra_admins")).fetchone())[0]
        if cnt == 0:
            await callback.answer("Cannot remove the last admin!", show_alert=True)
            return
    await state.update_data(admin_action="remove")
    await state.set_state(AdminSettingsFlow.enter_value)
    await callback.message.answer(
        "➖ <b>Remove Admin</b>\n\n"
        "Send the Telegram User ID to remove:\n"
        "<i>Example: 123456789</i>",
        parse_mode=ParseMode.HTML,
        reply_markup=get_back_inline("aadmin_mgmt")
    )
    await callback.answer()


# ---------------------------------------------------------------------------
# 📦 ORDER MANAGEMENT
# ---------------------------------------------------------------------------
@router.callback_query(F.data == "aord_menu", StateFilter("*"))
async def cb_admin_orders_menu(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔎 Search Order", callback_data="aord_search_start")],
        [InlineKeyboardButton(text="📋 All Orders", callback_data="ahub_ord_p_1")],
        [InlineKeyboardButton(text="🟡 Pending", callback_data="aord_filt_Pending"),
         InlineKeyboardButton(text="🔵 Processing", callback_data="aord_filt_Processing")],
        [InlineKeyboardButton(text="🟢 Completed", callback_data="aord_filt_Completed"),
         InlineKeyboardButton(text="🟠 Partial", callback_data="aord_filt_Partial")],
        [InlineKeyboardButton(text="🔴 Cancelled", callback_data="aord_filt_Cancelled"),
         InlineKeyboardButton(text="❌ Failed", callback_data="aord_filt_Failed")],
        [InlineKeyboardButton(text="⬅️ Back", callback_data="ahub_back")]
    ])
    await safe_message_edit(
        callback.message,
        "📦 <b>ORDER MANAGEMENT</b>\n\nSelect an option or search by Order ID:",
        reply_markup=kb, parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data == "aord_search_start", StateFilter("*"))
async def cb_aord_search_prompt(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    await safe_message_edit(
        callback.message,
        "🔎 <b>SEARCH ORDER</b>\n\nEnter the Bot Order ID (e.g. <code>125</code>):",
        reply_markup=get_back_inline("aord_menu"),
        parse_mode=ParseMode.HTML
    )
    await state.set_state(AdminOrderSearchFlow.enter_order_id)
    await callback.answer()


@router.message(AdminOrderSearchFlow.enter_order_id)
async def handle_aord_search_input(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    raw = (message.text or "").strip().replace("#", "")
    if not raw.isdigit():
        await message.answer("❌ Invalid numeric Order ID:")
        return
    order_id = int(raw)
    await state.clear()
    await render_admin_order_card(message.bot, message.chat.id, order_id)


async def render_admin_order_card(bot: Bot, chat_id: int, order_id: int, message_id: Optional[int] = None):
    async with get_db() as db:
        o = await (await db.execute(
            """
            SELECT o.*, COALESCE(s.name, 'Archived Service') as service_name,
                   COALESCE(p.name, 'N/A') as provider_name,
                   COALESCE(u.first_name, 'Unknown') as user_first_name
            FROM orders o
            LEFT JOIN services s ON o.service_id = s.id
            LEFT JOIN providers p ON o.provider_id = p.id
            LEFT JOIN users u ON o.user_id = u.user_id
            WHERE o.id = ?
            """,
            (order_id,)
        )).fetchone()

    if not o:
        text = f"❌ Order #{order_id} not found."
        if message_id:
            await safe_edit_message_text(bot, chat_id, message_id, text)
        else:
            await bot.send_message(chat_id, text)
        return

    st_ico = get_status_emoji(o["status"])
    refill_str = "✅ Yes" if o["refill_available"] else "❌ No"
    refill_days = f"{o['refill_days']} Days" if o["refill_available"] else "N/A"

    text = (
        f"📦 <b>ORDER #{o['id']}</b>\n\n"
        f"👤 <b>User:</b> {html.escape(o['user_first_name'])}\n"
        f"🆔 <b>User ID:</b> <code>{o['user_id']}</code>\n\n"
        f"📦 <b>Service:</b> {html.escape(o['service_name'])}\n"
        f"🔢 <b>Quantity:</b> {o['quantity']:,}\n"
        f"🔗 <b>Link:</b> <code>{html.escape(o['link'])}</code>\n\n"
        f"💰 <b>Charge:</b> ₹{o['charge']}\n"
        f"📊 <b>Status:</b> {st_ico} {o['status']}\n\n"
        f"🔢 <b>Start Count:</b> {o['start_count']:,}\n"
        f"📉 <b>Remains:</b> {o['remains']:,}\n\n"
        f"♻️ <b>Refill:</b> {refill_str}\n"
        f"⏳ <b>Refill Days:</b> {refill_days}\n\n"
        f"🔌 <b>Provider:</b> {html.escape(o['provider_name'])}\n"
        f"🆔 <b>Provider Order ID:</b> <code>{o['provider_order_id']}</code>\n\n"
        f"💳 <b>Refunded:</b> ₹{o['refunded_amount']}\n\n"
        f"📅 <b>Created:</b> {o['created_at']}\n"
        f"🔄 <b>Updated:</b> {o['updated_at']}"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔄 Sync Status", callback_data=f"aord_sync_{order_id}")],
        [InlineKeyboardButton(text="🟡 Pending", callback_data=f"aord_set_Pending_{order_id}"),
         InlineKeyboardButton(text="🔵 Processing", callback_data=f"aord_set_Processing_{order_id}"),
         InlineKeyboardButton(text="🟢 Completed", callback_data=f"aord_set_Completed_{order_id}")],
        [InlineKeyboardButton(text="🟠 Partial", callback_data=f"aord_set_Partial_{order_id}"),
         InlineKeyboardButton(text="🔴 Cancel Order", callback_data=f"aord_cancel_menu_{order_id}"),
         InlineKeyboardButton(text="❌ Failed", callback_data=f"aord_set_Failed_{order_id}")],
        [InlineKeyboardButton(text="💯 Full Refund", callback_data=f"aord_ref_full_{order_id}"),
         InlineKeyboardButton(text="🟠 Partial Refund", callback_data=f"aord_ref_part_{order_id}")],
        [InlineKeyboardButton(text="♻️ Request Refill", callback_data=f"aord_req_refill_{order_id}"),
         InlineKeyboardButton(text="🚫 Cancel Refill", callback_data=f"aord_canc_refill_{order_id}")],
        [InlineKeyboardButton(text="👤 View User", callback_data=f"ausr_view_{o['user_id']}"),
         InlineKeyboardButton(text="📜 History", callback_data=f"aord_hist_{o['user_id']}")],
        [InlineKeyboardButton(text="⬅️ Back", callback_data="aord_menu")]
    ])
    if message_id:
        await safe_edit_message_text(bot, chat_id, message_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    else:
        await bot.send_message(chat_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)


@router.callback_query(F.data.startswith("aord_sync_"), StateFilter("*"))
async def cb_aord_sync(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    oid = int(callback.data.split("_")[2])
    async with get_db() as db:
        o = await (await db.execute("SELECT * FROM orders WHERE id = ?", (oid,))).fetchone()

    if not o or not o["provider_order_id"]:
        await callback.answer("Unable to sync order.", show_alert=True)
        return

    res = await ProviderAPIClient.get_order_status(o["provider_id"], o["provider_order_id"])
    if isinstance(res, dict) and "status" in res:
        raw_st = str(res.get("status", "")).strip().lower()
        remains = int(res.get("remains", 0) or 0)
        start_count = int(res.get("start_count", 0) or 0)
        mapped = "Processing"
        if raw_st == "pending": mapped = "Pending"
        elif raw_st in ("in progress", "processing"): mapped = "Processing"
        elif raw_st == "completed": mapped = "Completed"
        elif raw_st == "partial": mapped = "Partial"
        elif raw_st in ("canceled", "cancelled"): mapped = "Cancelled"
        elif raw_st == "failed": mapped = "Failed"

        notify_refund = Decimal("0.00")
        async with get_db() as db:
            if mapped == "Partial" and o["refund_state"] == "NOT_REFUNDED":
                ordered_qty = Decimal(str(o["quantity"]))
                charge_amt = Decimal(str(o["charge"]))
                rem_qty = Decimal(str(remains))
                refund_val = quantize_currency((rem_qty / ordered_qty) * charge_amt) if ordered_qty > 0 else Decimal("0.00")
                if refund_val > Decimal("0.00"):
                    notify_refund = refund_val
                    await modify_wallet_atomic(db, o["user_id"], refund_val, "Partial Refund",
                        reference_id=str(oid), description=f"Partial refund for order #{oid}")
                    await db.execute(
                        "INSERT INTO refunds (order_id, user_id, amount, reason) VALUES (?, ?, ?, 'Partial unfulfilled refund')",
                        (oid, o["user_id"], str(refund_val))
                    )
                    await db.execute(
                        "UPDATE orders SET refund_state = 'PARTIALLY_REFUNDED', refunded_amount = ? WHERE id = ?",
                        (str(refund_val), oid)
                    )
            elif mapped in ("Cancelled", "Failed") and o["refund_state"] == "NOT_REFUNDED":
                charge_amt = Decimal(str(o["charge"]))
                notify_refund = charge_amt
                await modify_wallet_atomic(db, o["user_id"], charge_amt, "Full Refund",
                    reference_id=str(oid), description=f"Full refund for {mapped} order #{oid}")
                await db.execute(
                    "INSERT INTO refunds (order_id, user_id, amount, reason) VALUES (?, ?, ?, ?)",
                    (oid, o["user_id"], str(charge_amt), f"{mapped} Order by Provider")
                )
                await db.execute(
                    "UPDATE orders SET refund_state = 'FULLY_REFUNDED', refunded_amount = ? WHERE id = ?",
                    (str(charge_amt), oid)
                )
            await db.execute(
                "UPDATE orders SET status = ?, start_count = ?, remains = ?, updated_at = CURRENT_TIMESTAMP WHERE id = ?",
                (mapped, start_count, remains, oid)
            )
            await db.commit()

        if o["status"] != mapped:
            await notify_user_order_status_change(
                callback.bot, oid, o["user_id"],
                o["service_name_snapshot"] or "Service", mapped, notify_refund
            )
        await callback.answer(f"Synced: {mapped}")
    else:
        await callback.answer("Provider sync failed.", show_alert=True)

    await render_admin_order_card(callback.bot, callback.message.chat.id, oid, message_id=callback.message.message_id)


@router.callback_query(F.data.startswith("aord_set_"), StateFilter("*"))
async def cb_aord_set_status(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    parts = callback.data.split("_")
    new_status = parts[2]
    oid = int(parts[3])

    if new_status in ("Cancelled", "Failed"):
        await callback.answer("⚠️ Use 'Cancel Order' button for refund options.", show_alert=True)
        return

    async with get_db() as db:
        old_row = await (await db.execute(
            "SELECT user_id, status, service_name_snapshot FROM orders WHERE id = ?", (oid,)
        )).fetchone()
        await db.execute(
            "UPDATE orders SET status = ?, updated_at = CURRENT_TIMESTAMP WHERE id = ?",
            (new_status, oid)
        )
        await db.commit()

    if old_row and old_row["status"] != new_status:
        await notify_user_order_status_change(
            callback.bot, oid, old_row["user_id"],
            old_row["service_name_snapshot"] or "Service", new_status, Decimal("0.00")
        )
    if old_row and old_row["status"] != new_status:
        await send_channel_log(callback.bot,"ORDER",f"🔄 <b>Order #{oid}</b>\n• User: <code>{old_row['user_id']}</code>\n• {html.escape(old_row['status'])} → {html.escape(new_status)}\n• Admin: <code>{callback.from_user.id}</code>")
    await log_admin_event(callback.from_user.id, "SET_ORDER_STATUS", str(oid), f"Set to {new_status}")
    await callback.answer(f"Order #{oid} → {new_status}")
    await render_admin_order_card(callback.bot, callback.message.chat.id, oid, message_id=callback.message.message_id)


@router.callback_query(F.data.startswith("aord_cancel_menu_"), StateFilter("*"))
async def cb_aord_cancel_menu(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    oid = int(callback.data.split("_")[3])
    async with get_db() as db:
        o = await (await db.execute("SELECT * FROM orders WHERE id = ?", (oid,))).fetchone()
    if not o:
        await callback.answer("Order not found.", show_alert=True)
        return
    text = (
        f"⚠️ <b>CANCEL ORDER #{oid}</b>\n\n"
        f"Current Status: <b>{o['status']}</b>\n"
        f"Order Amount: <b>₹{o['charge']}</b>\n\n"
        "Choose refund action:"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="💯 Cancel + Full Refund", callback_data=f"aord_do_canc_full_{oid}")],
        [InlineKeyboardButton(text="🟠 Cancel + Partial Refund", callback_data=f"aord_do_canc_part_{oid}")],
        [InlineKeyboardButton(text="❌ Cancel Without Refund", callback_data=f"aord_do_canc_noref_{oid}")],
        [InlineKeyboardButton(text="⬅️ Back", callback_data=f"aord_search_view_{oid}")]
    ])
    await safe_message_edit(callback.message, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("aord_do_canc_full_"), StateFilter("*"))
@router.callback_query(F.data.startswith("aord_ref_full_"), StateFilter("*"))
async def cb_aord_cancel_full_refund(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    oid = int(callback.data.split("_")[-1])
    async with get_db() as db:
        o = await (await db.execute("SELECT * FROM orders WHERE id = ?", (oid,))).fetchone()
        if not o:
            await callback.answer("Order not found.", show_alert=True)
            return
        if o["refund_state"] == "FULLY_REFUNDED":
            await callback.answer("Already fully refunded.", show_alert=True)
            return
        charge = Decimal(str(o["charge"]))
        already = Decimal(str(o["refunded_amount"]))
        net = charge - already
        if net > Decimal("0.00"):
            await modify_wallet_atomic(db, o["user_id"], net, "Admin Full Refund",
                reference_id=str(oid), description=f"Full refund for order #{oid}")
            await db.execute(
                "INSERT INTO refunds (order_id, user_id, amount, reason) VALUES (?, ?, ?, 'Admin Full Refund')",
                (oid, o["user_id"], str(net))
            )
            await db.execute(
                "UPDATE orders SET status = 'Cancelled', refund_state = 'FULLY_REFUNDED', refunded_amount = ? WHERE id = ?",
                (str(charge), oid)
            )
            await db.commit()

    await log_admin_event(callback.from_user.id, "CANCEL_FULL_REFUND", str(oid), f"Refunded ₹{net}")
    await callback.answer("Cancelled & refunded.")
    await render_admin_order_card(callback.bot, callback.message.chat.id, oid, message_id=callback.message.message_id)


@router.callback_query(F.data.startswith("aord_do_canc_noref_"), StateFilter("*"))
async def cb_aord_cancel_no_refund(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    oid = int(callback.data.split("_")[4])
    async with get_db() as db:
        await db.execute(
            "UPDATE orders SET status = 'Cancelled', updated_at = CURRENT_TIMESTAMP WHERE id = ?",
            (oid,)
        )
        await db.commit()
    await log_admin_event(callback.from_user.id, "CANCEL_NO_REFUND", str(oid), "No refund")
    await callback.answer("Cancelled without refund.")
    await render_admin_order_card(callback.bot, callback.message.chat.id, oid, message_id=callback.message.message_id)


@router.callback_query(F.data.startswith("aord_do_canc_part_"), StateFilter("*"))
@router.callback_query(F.data.startswith("aord_ref_part_"), StateFilter("*"))
async def cb_aord_prompt_partial_refund(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    oid = int(callback.data.split("_")[-1])
    await state.update_data(target_refund_oid=oid)
    await callback.message.answer(
        f"Enter partial refund amount for Order #{oid}:",
        reply_markup=get_back_inline(f"aord_search_view_{oid}")
    )
    await state.set_state(AdminOrderSearchFlow.enter_partial_refund_amount)
    await callback.answer()


@router.message(AdminOrderSearchFlow.enter_partial_refund_amount)
async def handle_aord_exec_partial_refund(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    data = await state.get_data()
    oid = data.get("target_refund_oid")
    await state.clear()
    if not oid:
        await message.answer("❌ Session expired.")
        return
    try:
        amt = quantize_currency((message.text or "").strip())
        if amt <= Decimal("0.00"):
            raise ValueError
    except Exception:
        await message.answer("❌ Invalid numeric refund amount.")
        return

    async with get_db() as db:
        o = await (await db.execute("SELECT * FROM orders WHERE id = ?", (oid,))).fetchone()
        if not o:
            await message.answer("❌ Order not found.")
            return
        charge = Decimal(str(o["charge"]))
        already = Decimal(str(o["refunded_amount"]))
        max_ref = charge - already
        if amt > max_ref:
            await message.answer(f"❌ Cannot exceed remaining refundable (₹{max_ref}).")
            return
        new_total = already + amt
        new_state = "FULLY_REFUNDED" if new_total >= charge else "PARTIALLY_REFUNDED"
        await modify_wallet_atomic(db, o["user_id"], amt, "Admin Partial Refund",
            reference_id=str(oid), description=f"Admin partial refund #{oid}")
        await db.execute(
            "INSERT INTO refunds (order_id, user_id, amount, reason) VALUES (?, ?, ?, 'Admin Partial Refund')",
            (oid, o["user_id"], str(amt))
        )
        await db.execute(
            "UPDATE orders SET refund_state = ?, refunded_amount = ?, updated_at = CURRENT_TIMESTAMP WHERE id = ?",
            (new_state, str(new_total), oid)
        )
        await db.commit()

    await log_admin_event(message.from_user.id, "PARTIAL_REFUND", str(oid), f"₹{amt}")
    await message.answer(f"✅ Refund ₹{amt} credited to user for Order #{oid}!")
    await render_admin_order_card(message.bot, message.chat.id, oid)


@router.callback_query(F.data.startswith("aord_req_refill_"), StateFilter("*"))
async def cb_aord_req_refill_admin(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    oid = int(callback.data.split("_")[3])
    async with get_db() as db:
        o = await (await db.execute("SELECT * FROM orders WHERE id = ?", (oid,))).fetchone()
    if not o or not o["provider_order_id"]:
        await callback.answer("Cannot refill.", show_alert=True)
        return
    res = await ProviderAPIClient.request_refill(o["provider_id"], o["provider_order_id"])
    ok = isinstance(res, dict) and (res.get("refill") or res.get("status") == "success" or not res.get("error"))
    if ok:
        pref_id = str(res.get("refill", ""))
        async with get_db() as db:
            await db.execute(
                "INSERT INTO refills (order_id, user_id, provider_id, provider_order_id, provider_refill_id, status) VALUES (?, ?, ?, ?, ?, 'Pending')",
                (oid, o["user_id"], o["provider_id"], o["provider_order_id"], pref_id)
            )
            await db.execute(
                "UPDATE orders SET refill_status = 'Pending', provider_refill_id = ? WHERE id = ?",
                (pref_id, oid)
            )
            await db.commit()
        await callback.answer("Refill requested!", show_alert=True)
    else:
        err = res.get("error", "Rejected") if isinstance(res, dict) else str(res)
        await callback.answer(f"Refill error: {err}", show_alert=True)
    await render_admin_order_card(callback.bot, callback.message.chat.id, oid, message_id=callback.message.message_id)


@router.callback_query(F.data.startswith("aord_canc_refill_"), StateFilter("*"))
async def cb_aord_canc_refill_admin(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    oid = int(callback.data.split("_")[3])
    async with get_db() as db:
        await db.execute("UPDATE orders SET refill_status = 'Cancelled' WHERE id = ?", (oid,))
        await db.execute("UPDATE refills SET status = 'Cancelled' WHERE order_id = ?", (oid,))
        await db.commit()
    await callback.answer("Refill cancelled.")
    await render_admin_order_card(callback.bot, callback.message.chat.id, oid, message_id=callback.message.message_id)


@router.callback_query(F.data.startswith("aord_search_view_"), StateFilter("*"))
async def cb_aord_search_view(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    oid = int(callback.data.split("_")[3])
    await render_admin_order_card(callback.bot, callback.message.chat.id, oid, message_id=callback.message.message_id)
    await callback.answer()


@router.callback_query(F.data.startswith("ahub_ord_p_"), StateFilter("*"))
async def cb_admin_orders_paginate(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    page = int(callback.data.split("_")[3])
    per_page = 6
    offset = (page - 1) * per_page
    async with get_db() as db:
        tot = (await (await db.execute("SELECT COUNT(*) FROM orders")).fetchone())[0]
        orders = await (await db.execute(
            "SELECT o.*, COALESCE(s.name, 'Service') as service_name FROM orders o LEFT JOIN services s ON o.service_id = s.id ORDER BY o.id DESC LIMIT ? OFFSET ?",
            (per_page, offset)
        )).fetchall()

    buttons = []
    for o in orders:
        buttons.append([InlineKeyboardButton(
            text=f"#{o['id']} | {o['status']} | ₹{o['charge']} | {(o['service_name'] or '')[:14]}",
            callback_data=f"aord_search_view_{o['id']}"
        )])
    tot_pages = (tot + per_page - 1) // per_page
    if tot_pages > 1:
        buttons.append(make_pagination(page, tot_pages, "ahub_ord"))
    buttons.append([InlineKeyboardButton(text="⬅️ Back", callback_data="aord_menu")])

    await safe_message_edit(
        callback.message,
        f"📦 <b>ALL ORDERS ({tot}):</b>",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=buttons),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data.startswith("aord_filtpg_"), StateFilter("*"))
async def cb_aord_filtered_page(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    parts=callback.data.split("_")
    filt_status=parts[2]
    page=max(1,int(parts[-1]))
    per_page=6
    async with get_db() as db:
        tot=(await (await db.execute("SELECT COUNT(*) FROM orders WHERE status=?",(filt_status,))).fetchone())[0]
        total_pages=max(1,(tot+per_page-1)//per_page)
        page=min(page,total_pages)
        offset=(page-1)*per_page
        orders=await (await db.execute(
            "SELECT o.*, COALESCE(s.name,'Service') AS service_name FROM orders o LEFT JOIN services s ON o.service_id=s.id WHERE o.status=? ORDER BY o.id DESC LIMIT ? OFFSET ?",
            (filt_status,per_page,offset)
        )).fetchall()
    buttons=[[InlineKeyboardButton(text=f"#{o['id']} | ₹{o['charge']} | {(o['service_name'] or '')[:16]}", callback_data=f"aord_search_view_{o['id']}")] for o in orders]
    if total_pages>1:
        buttons.append(make_pagination(page,total_pages,f"aord_filtpg_{filt_status}"))
    buttons.append([InlineKeyboardButton(text="⬅️ Back",callback_data="aord_menu")])
    await safe_message_edit(callback.message,f"📦 <b>ORDERS: {filt_status.upper()} ({tot})</b>\nPage <b>{page}/{total_pages}</b>",reply_markup=InlineKeyboardMarkup(inline_keyboard=buttons),parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("aord_filt_"), StateFilter("*"))
async def cb_aord_filtered_list(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    callback.data = "aord_filtpg_" + callback.data.replace("aord_filt_", "") + "_1"
    await cb_aord_filtered_page(callback)


# ---------------------------------------------------------------------------
# 👥 USER MANAGEMENT
# ---------------------------------------------------------------------------
@router.callback_query(F.data.in_({"ahub_users", "ahub_usr_p_1"}), StateFilter("*"))
async def cb_admin_users_search_only(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return

    # SEARCH-ONLY: never load or render an inline user list here.
    buttons = [
        [InlineKeyboardButton(text="🔎 Search User", callback_data="ausr_search_start")],
        [InlineKeyboardButton(text="⬅️ Back to Hub", callback_data="ahub_back")]
    ]
    await safe_message_edit(
        callback.message,
        "👥 <b>USER MANAGEMENT</b>\n\n"
        "Search for a specific user by Telegram User ID.",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=buttons),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data == "ausr_search_start", StateFilter("*"))
async def cb_ausr_search_start(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    await safe_message_edit(
        callback.message,
        "🔎 <b>SEARCH USER</b>\n\nEnter User Telegram ID:",
        reply_markup=get_back_inline("ahub_users"),
        parse_mode=ParseMode.HTML
    )
    await state.set_state(AdminUserActionFlow.enter_search)
    await callback.answer()


@router.message(AdminUserActionFlow.enter_search)
async def handle_ausr_search_input(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    raw = (message.text or "").strip()
    if not raw.isdigit():
        await message.answer("❌ Invalid numeric User ID:")
        return
    uid = int(raw)
    await state.clear()
    await render_admin_user_card(message.bot, message.chat.id, uid)


async def render_admin_user_card(bot: Bot, chat_id: int, user_id: int, message_id: Optional[int] = None):
    async with get_db() as db:
        u = await (await db.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))).fetchone()
        if not u:
            text = f"❌ User #{user_id} not found."
            if message_id:
                await safe_edit_message_text(bot, chat_id, message_id, text)
            else:
                await bot.send_message(chat_id, text)
            return
        o_cnt = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE user_id = ?", (user_id,))).fetchone())[0]
        r_cnt = (await (await db.execute("SELECT COUNT(*) FROM users WHERE referrer_id = ?", (user_id,))).fetchone())[0]

    text = (
        f"👤 <b>USER #{u['user_id']}</b>\n\n"
        f"<b>Name:</b> {html.escape(u['first_name'] or 'N/A')}\n"
        f"<b>Username:</b> @{u['username'] or 'N/A'}\n"
        f"<b>Currency:</b> {u['currency'] or 'INR'}\n"
        f"💰 <b>Balance:</b> ₹{u['balance']}\n"
        f"💵 <b>Total Spent:</b> ₹{u['total_spent']}\n"
        f"💳 <b>Total Deposited:</b> ₹{u['total_deposited']}\n\n"
        f"📦 <b>Orders:</b> {o_cnt}\n"
        f"👥 <b>Referrals:</b> {r_cnt} (Earnings: ₹{u['referral_earnings']})\n"
        f"📊 <b>Status:</b> {'🔴 Banned' if u['is_banned'] else '🟢 Active'}\n"
        f"📅 <b>Joined:</b> {u['created_at']}"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="➕ Add Balance", callback_data=f"aubal_add_{user_id}"),
         InlineKeyboardButton(text="➖ Deduct Balance", callback_data=f"aubal_sub_{user_id}")],
        [InlineKeyboardButton(text="📦 Orders", callback_data=f"aord_hist_{user_id}"),
         InlineKeyboardButton(text="💳 Payments", callback_data=f"ausr_tx_{user_id}")],
        [InlineKeyboardButton(text="🚫 Toggle Ban", callback_data=f"auban_tog_{user_id}"),
         InlineKeyboardButton(text="📢 Message", callback_data=f"ausr_msg_{user_id}")],
        [InlineKeyboardButton(text="⬅️ Back", callback_data="ahub_users")]
    ])
    if message_id:
        await safe_edit_message_text(bot, chat_id, message_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    else:
        await bot.send_message(chat_id, text, reply_markup=kb, parse_mode=ParseMode.HTML)


@router.callback_query(F.data.startswith("ausr_view_"), StateFilter("*"))
async def cb_ausr_view(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    try:
        uid = int(callback.data.split("_")[2])
    except (ValueError, IndexError):
        await callback.answer("Invalid user.", show_alert=True)
        return
    await render_admin_user_card(callback.bot, callback.message.chat.id, uid, message_id=callback.message.message_id)
    await callback.answer()


@router.callback_query(F.data.startswith("auban_tog_"), StateFilter("*"))
async def cb_ausr_toggle_ban(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    uid = int(callback.data.split("_")[2])
    async with get_db() as db:
        await db.execute(
            "UPDATE users SET is_banned = CASE WHEN is_banned = 1 THEN 0 ELSE 1 END WHERE user_id = ?",
            (uid,)
        )
        await db.commit()
    await log_admin_event(callback.from_user.id, "TOGGLE_BAN", str(uid), "Ban toggled")
    await callback.answer("Ban status updated!")
    await render_admin_user_card(callback.bot, callback.message.chat.id, uid, message_id=callback.message.message_id)


@router.callback_query(F.data.startswith("aubal_"), StateFilter("*"))
async def cb_ausr_balance_prompt(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    parts = callback.data.split("_")
    action = parts[1]
    uid = int(parts[2])
    await state.update_data(adj_action=action, adj_uid=uid)
    await callback.message.answer(
        f"Enter amount to {'ADD to' if action == 'add' else 'DEDUCT from'} User #{uid}:",
        reply_markup=get_back_inline(f"ausr_view_{uid}")
    )
    await state.set_state(AdminUserActionFlow.enter_balance_amount)
    await callback.answer()


@router.message(AdminUserActionFlow.enter_balance_amount)
async def handle_ausr_bal_amount_in(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    try:
        amt = quantize_currency((message.text or "").strip())
        if amt <= Decimal("0.00"):
            raise ValueError
    except Exception:
        await message.answer("❌ Invalid positive numeric amount:")
        return
    await state.update_data(adj_amt=str(amt))
    await message.answer("Enter reason / note for this adjustment:")
    await state.set_state(AdminUserActionFlow.enter_balance_reason)


@router.message(AdminUserActionFlow.enter_balance_reason)
async def handle_ausr_bal_reason_in(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    reason = (message.text or "").strip()
    data = await state.get_data()
    uid = data.get("adj_uid")
    amt = Decimal(data.get("adj_amt", "0.00"))
    act = data.get("adj_action")
    await state.clear()
    if not uid or not act:
        await message.answer("❌ Session expired.")
        return
    final_delta = amt if act == "add" else -amt
    async with get_db() as db:
        ok = await modify_wallet_atomic(
            db, uid, final_delta, "Admin Balance Adjustment",
            description=f"Admin #{message.from_user.id} Adj: {reason}"
        )
        await db.commit()
    if ok:
        await log_admin_event(message.from_user.id, "BAL_ADJ", str(uid), f"{act.upper()} ₹{amt} | {reason}")
        await message.answer(f"✅ User #{uid} balance updated by ₹{final_delta}!")
        await render_admin_user_card(message.bot, message.chat.id, uid)
    else:
        await message.answer("❌ Adjustment failed (balance cannot become negative).")


@router.callback_query(F.data.startswith("ausr_msg_"), StateFilter("*"))
async def cb_ausr_msg_prompt(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    uid = int(callback.data.split("_")[2])
    await state.update_data(target_msg_uid=uid)
    await callback.message.answer(
        f"Enter direct message to send to User #{uid}:",
        reply_markup=get_back_inline(f"ausr_view_{uid}")
    )
    await state.set_state(AdminUserActionFlow.enter_user_message)
    await callback.answer()


@router.message(AdminUserActionFlow.enter_user_message)
async def handle_ausr_direct_message(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    data = await state.get_data()
    uid = data.get("target_msg_uid")
    text = message.text or ""
    await state.clear()
    if not uid:
        await message.answer("❌ Session expired.")
        return
    try:
        await message.bot.send_message(
            uid,
            f"📩 <b>Support Message:</b>\n\n{html.escape(text)}",
            parse_mode=ParseMode.HTML
        )
        await message.answer(f"✅ Message delivered to User #{uid}!")
    except Exception as e:
        await message.answer(f"❌ Failed to deliver: {e}")


@router.callback_query(F.data.startswith("aord_hist_"), StateFilter("*"))
async def cb_admin_user_order_history(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    uid = int(callback.data.split("_")[2])
    async with get_db() as db:
        orders = await (await db.execute(
            "SELECT o.*, COALESCE(s.name, 'Service') as service_name FROM orders o LEFT JOIN services s ON o.service_id = s.id WHERE o.user_id = ? ORDER BY o.id DESC LIMIT 8",
            (uid,)
        )).fetchall()
    buttons = []
    for o in orders:
        buttons.append([InlineKeyboardButton(
            text=f"#{o['id']} | {o['status']} | ₹{o['charge']}",
            callback_data=f"aord_search_view_{o['id']}"
        )])
    buttons.append([InlineKeyboardButton(text="⬅️ Back", callback_data=f"ausr_view_{uid}")])
    await safe_message_edit(
        callback.message,
        f"📜 <b>ORDER HISTORY — User #{uid}:</b>",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=buttons),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data.startswith("ausr_tx_"), StateFilter("*"))
async def cb_admin_user_tx_history(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    uid = int(callback.data.split("_")[2])
    async with get_db() as db:
        txs = await (await db.execute(
            "SELECT * FROM transactions WHERE user_id = ? ORDER BY id DESC LIMIT 10",
            (uid,)
        )).fetchall()
    lines = [f"💳 <b>PAYMENT HISTORY — User #{uid}:</b>\n"]
    for t in txs:
        lines.append(
            f"• <b>{t['type']}</b>: ₹{t['amount']} (Bal: ₹{t['balance_after']})\n"
            f"  <i>{html.escape(t['description'] or '')}</i> — {str(t['created_at'])[:16]}"
        )
    if not txs:
        lines.append("No transactions recorded.")
    kb = InlineKeyboardMarkup(inline_keyboard=[[
        InlineKeyboardButton(text="⬅️ Back", callback_data=f"ausr_view_{uid}")
    ]])
    await safe_message_edit(callback.message, "\n".join(lines), reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()


# ---------------------------------------------------------------------------
# 💰 PAYMENTS
# ---------------------------------------------------------------------------
@router.callback_query(F.data == "apay_methods", StateFilter("*"))
async def admin_payment_methods(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id): return
    vals={k:await get_setting(k,"1")=="1" for k in ("PAY_UPI_AUTO_ENABLED","PAY_UPI_MANUAL_ENABLED","PAY_USDT_ENABLED")}
    text=("💳 <b>PAYMENT METHODS</b>\n\nOFF = hidden from users + callback blocked.\n\n"
          f"🟢 UPI 1 Auto: <b>{'ON' if vals['PAY_UPI_AUTO_ENABLED'] else 'OFF'}</b>\n"
          f"🟡 UPI 2 Manual: <b>{'ON' if vals['PAY_UPI_MANUAL_ENABLED'] else 'OFF'}</b>\n"
          f"🔵 USDT BEP-20: <b>{'ON' if vals['PAY_USDT_ENABLED'] else 'OFF'}</b>")
    kb=InlineKeyboardMarkup(inline_keyboard=[
      [InlineKeyboardButton(text=f"{'🟢' if vals['PAY_UPI_AUTO_ENABLED'] else '🔴'} UPI 1 Auto",callback_data="apay_toggle_UPI_AUTO")],
      [InlineKeyboardButton(text=f"{'🟢' if vals['PAY_UPI_MANUAL_ENABLED'] else '🔴'} UPI 2 Manual",callback_data="apay_toggle_UPI_MANUAL")],
      [InlineKeyboardButton(text=f"{'🟢' if vals['PAY_USDT_ENABLED'] else '🔴'} USDT BEP-20",callback_data="apay_toggle_USDT_BEP20")],
      [InlineKeyboardButton(text="⬅️ Back",callback_data="ahub_back")]])
    await safe_message_edit(callback.message,text,reply_markup=kb,parse_mode=ParseMode.HTML); await callback.answer()

@router.callback_query(F.data.startswith("apay_toggle_"), StateFilter("*"))
async def admin_payment_toggle(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id): return
    method=callback.data.replace("apay_toggle_","")
    key={"UPI_AUTO":"PAY_UPI_AUTO_ENABLED","UPI_MANUAL":"PAY_UPI_MANUAL_ENABLED","USDT_BEP20":"PAY_USDT_ENABLED"}.get(method)
    if not key: await callback.answer("Invalid payment method.",show_alert=True); return
    cur=await get_setting(key,"1"); new="0" if cur=="1" else "1"; await set_setting(key,new)
    await log_admin_event(callback.from_user.id,"TOGGLE_PAYMENT_METHOD",method,new)
    await send_channel_log(callback.bot,"PAYMENT",f"💳 <b>Payment {method}: {'ON' if new=='1' else 'OFF'}</b>\n• Admin: <code>{callback.from_user.id}</code>")
    await admin_payment_methods(callback)

@router.callback_query(F.data == "apay_menu", StateFilter("*"))
async def cb_admin_payments_menu(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="💳 Deposits Ledger", callback_data="ahub_dep_p_1")],
        [InlineKeyboardButton(text="🟡 Pending", callback_data="adep_filt_PENDING"),
         InlineKeyboardButton(text="✅ Approved", callback_data="adep_filt_APPROVED")],
        [InlineKeyboardButton(text="❌ Rejected", callback_data="adep_filt_REJECTED")],
        [InlineKeyboardButton(text="📊 Payment Stats", callback_data="apay_stats")],
        [InlineKeyboardButton(text="⬅️ Back", callback_data="ahub_back")]
    ])
    await safe_message_edit(
        callback.message,
        "💰 <b>PAYMENTS & WALLET</b>\n\nSelect a ledger or filter:",
        reply_markup=kb, parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data.startswith("adep_filt_"), StateFilter("*"))
async def cb_adep_filtered_list(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    st = callback.data.replace("adep_filt_", "")
    async with get_db() as db:
        tot = (await (await db.execute(
            "SELECT COUNT(*) FROM deposits WHERE status = ?", (st,)
        )).fetchone())[0]
        deps = await (await db.execute(
            "SELECT * FROM deposits WHERE status = ? ORDER BY id DESC LIMIT 8", (st,)
        )).fetchall()
    buttons = []
    for d in deps:
        ref = d["utr"] or d["reference"] or d["purpose"] or "N/A"
        buttons.append([InlineKeyboardButton(
            text=f"#{d['id']} | {d['method']} | ₹{d['amount']} | {ref[:14]}",
            callback_data=f"adep_view_{d['id']}"
        )])
    buttons.append([InlineKeyboardButton(text="⬅️ Back", callback_data="apay_menu")])
    await safe_message_edit(
        callback.message,
        f"💳 <b>DEPOSITS: {st} ({tot})</b>",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=buttons),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data == "apay_stats", StateFilter("*"))
async def cb_apay_stats(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    async with get_db() as db:
        tot_dep = (await (await db.execute(
            "SELECT SUM(CAST(amount AS REAL)) FROM deposits WHERE status = 'APPROVED'"
        )).fetchone())[0] or 0.0
        tot_ref = (await (await db.execute(
            "SELECT SUM(CAST(amount AS REAL)) FROM refunds"
        )).fetchone())[0] or 0.0
        dep_cnt = (await (await db.execute(
            "SELECT COUNT(*) FROM deposits WHERE status = 'APPROVED'"
        )).fetchone())[0]
    text = (
        "📊 <b>PAYMENT STATISTICS</b>\n\n"
        f"• Approved Deposits: <b>₹{tot_dep:,.2f}</b> ({dep_cnt} total)\n"
        f"• Total Refunds: <b>₹{tot_ref:,.2f}</b>\n"
        f"• Net Deposited: <b>₹{(tot_dep - tot_ref):,.2f}</b>"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[[
        InlineKeyboardButton(text="⬅️ Back", callback_data="apay_menu")
    ]])
    await safe_message_edit(callback.message, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("ahub_dep_p_"), StateFilter("*"))
async def cb_admin_deposits_list(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    page = int(callback.data.split("_")[3])
    per_page = 6
    offset = (page - 1) * per_page
    async with get_db() as db:
        tot = (await (await db.execute("SELECT COUNT(*) FROM deposits")).fetchone())[0]
        deps = await (await db.execute(
            "SELECT * FROM deposits ORDER BY id DESC LIMIT ? OFFSET ?",
            (per_page, offset)
        )).fetchall()
    buttons = []
    for d in deps:
        st_ico = "⏳" if d["status"] == "PENDING" else ("✅" if d["status"] == "APPROVED" else "❌")
        buttons.append([InlineKeyboardButton(
            text=f"{st_ico} #{d['id']} | {d['method']} | ₹{d['amount']}",
            callback_data=f"adep_view_{d['id']}"
        )])
    tot_pages = (tot + per_page - 1) // per_page
    if tot_pages > 1:
        buttons.append(make_pagination(page, tot_pages, "ahub_dep"))
    buttons.append([InlineKeyboardButton(text="⬅️ Back", callback_data="apay_menu")])
    await safe_message_edit(
        callback.message,
        f"💳 <b>DEPOSITS LEDGER ({tot}):</b>",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=buttons),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data.startswith("adep_view_"), StateFilter("*"))
async def cb_adep_view(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    did = int(callback.data.split("_")[2])
    async with get_db() as db:
        d = await (await db.execute(
            "SELECT d.*, u.first_name, u.username FROM deposits d JOIN users u ON d.user_id = u.user_id WHERE d.id = ?",
            (did,)
        )).fetchone()
    if not d:
        await callback.answer("Deposit not found.", show_alert=True)
        return
    ref_val = d["utr"] or d["reference"] or "N/A"
    text = (
        f"💳 <b>DEPOSIT #{d['id']}</b>\n\n"
        f"<b>Method:</b> <code>{d['method']}</code>\n"
        f"<b>User:</b> {html.escape(d['first_name'] or 'N/A')} (@{d['username'] or 'N/A'}) [<code>{d['user_id']}</code>]\n"
        f"<b>Amount:</b> <b>₹{d['amount']}</b>\n"
        f"<b>Reference:</b> <code>{ref_val}</code>\n"
        f"<b>Status:</b> <b>{d['status']}</b>\n"
        f"<b>Created:</b> {d['created_at']}\n"
        f"<b>Processed:</b> {d['processed_at'] or 'N/A'}"
    )
    rows = []
    if d["status"] == "PENDING" and d["method"] in ("UPI_MANUAL", "USDT_BEP20"):
        rows.append([
            InlineKeyboardButton(text="✅ Approve", callback_data=f"amanual_approve_{did}"),
            InlineKeyboardButton(text="❌ Reject", callback_data=f"amanual_reject_{did}")
        ])
    rows.append([InlineKeyboardButton(text="⬅️ Back", callback_data="ahub_dep_p_1")])
    await safe_message_edit(
        callback.message, text,
        reply_markup=InlineKeyboardMarkup(inline_keyboard=rows),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data.startswith("amanual_approve_"), StateFilter("*"))
async def cb_admin_manual_approve(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    did = int(callback.data.split("_")[2])
    user_id_credited = None
    amt_credited = Decimal("0.00")
    try:
        async with get_db() as db:
            await db.execute("BEGIN IMMEDIATE")
            dep = await (await db.execute(
                "SELECT * FROM deposits WHERE id = ? AND status = 'PENDING'", (did,)
            )).fetchone()
            if not dep:
                await db.rollback()
                await callback.answer("Deposit is not pending.", show_alert=True)
                return
            amt = quantize_currency(dep["amount"])
            updated = await db.execute(
                "UPDATE deposits SET status = 'APPROVED', processed_at = CURRENT_TIMESTAMP WHERE id = ? AND status = 'PENDING'",
                (did,)
            )
            if updated.rowcount != 1:
                await db.rollback()
                await callback.answer("Deposit state changed.", show_alert=True)
                return
            ok = await modify_wallet_atomic(
                db, dep["user_id"], amt, "Deposit",
                reference_id=dep["utr"] or dep["reference"] or str(did),
                description=f"Manual deposit #{did} approved by admin"
            )
            if not ok:
                await db.rollback()
                await callback.answer("Crediting failed.", show_alert=True)
                return
            await credit_referral_bonus(db, dep["user_id"], amt)
            await db.commit()
            user_id_credited = dep["user_id"]
            amt_credited = amt
    except Exception as e:
        logger.error(f"Manual approve error: {e}", exc_info=True)
        await callback.answer("System error.", show_alert=True)
        return

    event_text = await notify_user_deposit_event(
        callback.bot, int(user_id_credited), "✅ <b>DEPOSIT APPROVED</b>",
        did, amt_credited, "\n\nYour balance has been credited successfully."
    )
    await notify_admins(callback.bot, event_text + f"\n👤 User: <code>{user_id_credited}</code>")
    await log_admin_event(callback.from_user.id, "APPROVE_DEPOSIT", str(did), f"₹{amt_credited}")
    await send_channel_log(callback.bot,"PAYMENT",f"✅ <b>Deposit Approved #{did}</b>\n• User: <code>{user_id_credited}</code>\n• Amount: ₹{amt_credited}\n• Admin: <code>{callback.from_user.id}</code>")

    await callback.answer("Deposit approved!", show_alert=True)
    callback.data = f"adep_view_{did}"
    await cb_adep_view(callback)


@router.callback_query(F.data.startswith("amanual_reject_"), StateFilter("*"))
async def cb_admin_manual_reject(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    did = int(callback.data.split("_")[2])
    async with get_db() as db:
        dep = await (await db.execute(
            "SELECT * FROM deposits WHERE id = ? AND status = 'PENDING'", (did,)
        )).fetchone()
        if not dep:
            await callback.answer("Deposit is not pending.", show_alert=True)
            return
        await db.execute(
            "UPDATE deposits SET status = 'REJECTED', processed_at = CURRENT_TIMESTAMP WHERE id = ? AND status = 'PENDING'",
            (did,)
        )
        await db.commit()
    event_text = await notify_user_deposit_event(
        callback.bot, int(dep["user_id"]), "❌ <b>DEPOSIT REJECTED</b>",
        did, dep["amount"], "\n\nYour deposit was rejected by admin. Please contact support if needed."
    )
    await notify_admins(callback.bot, event_text + f"\n👤 User: <code>{dep['user_id']}</code>")
    await log_admin_event(callback.from_user.id, "REJECT_DEPOSIT", str(did), "Rejected")
    await send_channel_log(callback.bot,"PAYMENT",f"❌ <b>Deposit Rejected #{did}</b>\n• User: <code>{dep['user_id']}</code>\n• Amount: ₹{dep['amount']}\n• Admin: <code>{callback.from_user.id}</code>")
    await callback.answer("Deposit rejected.", show_alert=True)
    callback.data = f"adep_view_{did}"
    await cb_adep_view(callback)


# ---------------------------------------------------------------------------
# ➕ ADD SERVICE
# ---------------------------------------------------------------------------
@router.callback_query(F.data=="asrv_add",StateFilter("*"))
async def admin_add_service_start(callback:CallbackQuery,state:FSMContext):
    if not await is_admin(callback.from_user.id): return
    await state.clear()
    async with get_db() as db:
        providers=await (await db.execute("SELECT id,name,status FROM providers ORDER BY id")).fetchall()
    if not providers:
        await callback.answer("Add provider first.",show_alert=True); return
    kb=[[InlineKeyboardButton(text=f"{'🟢' if p['status'] else '🔴'} #{p['id']} {p['name']}",callback_data=f"asrv_provider_{p['id']}")] for p in providers]
    kb.append([InlineKeyboardButton(text="⬅️ Back",callback_data="ahub_srv_p_1")])
    await safe_message_edit(callback.message,"➕ <b>ADD SERVICE</b>\n\nSelect provider:",reply_markup=InlineKeyboardMarkup(inline_keyboard=kb),parse_mode=ParseMode.HTML)
    await state.set_state(AdminServiceAddFlow.select_provider); await callback.answer()

@router.callback_query(F.data.startswith("asrv_provider_"),StateFilter(AdminServiceAddFlow.select_provider))
async def admin_add_service_provider(callback:CallbackQuery,state:FSMContext):
    if not await is_admin(callback.from_user.id): return
    pid=int(callback.data.rsplit("_",1)[1])
    async with get_db() as db:
        p=await (await db.execute("SELECT status,name FROM providers WHERE id=?",(pid,))).fetchone()
    if not p or not p["status"]:
        await callback.answer("Provider disabled.",show_alert=True); return
    await state.update_data(provider_id=pid,provider_name=p["name"])
    await state.set_state(AdminServiceAddFlow.enter_service_id)
    await callback.message.answer("🆔 <b>Provider Service ID</b>\n\nService ID dalo — provider se name, min, max, rate aur description automatically fetch hoga.",parse_mode=ParseMode.HTML)
    await callback.answer()

@router.message(AdminServiceAddFlow.enter_service_id)
async def admin_add_service_id(message:Message,state:FSMContext):
    if not await is_admin(message.from_user.id): return
    raw=(message.text or "").strip()
    if not raw.isdigit():
        await message.answer("❌ Numeric service ID required."); return
    d=await state.get_data(); pid=int(d["provider_id"])
    await message.answer("⏳ Provider se service fetch kar raha hoon...")
    try:
        res=await ProviderAPIClient.fetch_services_list(pid)
    except Exception as e:
        logger.error(f"Add service fetch failed: {e}",exc_info=True)
        await message.answer("❌ Provider fetch failed. Provider/API check karo."); return
    if not res.get("success"):
        await message.answer(f"❌ Provider se services fetch nahi hui.\n\n{html.escape(str(res.get('error','Unknown error')))}",parse_mode=ParseMode.HTML); return
    wanted=str(int(raw)); item=None
    for x in (res.get("services") or []):
        sid=_p_sid(x)
        if sid is not None and str(sid)==wanted:
            item=x; break
    if not item:
        await message.answer(f"❌ Service ID <code>{wanted}</code> provider list me nahi mili.",parse_mode=ParseMode.HTML); return
    try:
        provider_rate=quantize_currency(item.get("rate",item.get("price")))
        if provider_rate<=0: raise ValueError
        min_q=int(item.get("min",item.get("minimum",0)) or 0)
        max_q=int(item.get("max",item.get("maximum",0)) or 0)
        if min_q<1 or max_q<min_q: raise ValueError
    except Exception:
        await message.answer("❌ Provider response me valid rate/min/max nahi mila."); return
    name=str(item.get("name") or item.get("service_name") or item.get("title") or f"Service {wanted}").strip()
    desc=_p_desc(item)
    refill_available, refill_days, refill_duration = _p_refill(item)
    cancel_available = _p_cancel(item)
    await state.update_data(
        provider_service_id=int(wanted), name=name, min_quantity=min_q, max_quantity=max_q,
        provider_rate=str(provider_rate), description=desc,
        refill_available=refill_available, refill_days=refill_days, refill_duration=refill_duration,
        cancel_available=cancel_available
    )
    preview=(f"📥 <b>Service Fetched</b>\n\n📛 <b>{html.escape(name)}</b>\n🆔 Provider ID: <code>{wanted}</code>\n"
             f"🔢 Quantity: <b>{min_q:,} - {max_q:,}</b>\n💰 Provider Rate: <b>₹{provider_rate}</b>\n"
             f"📝 Description: {'Available' if desc else 'Not available'}\n\n💵 Ab selling price ka method select karo:")
    kb=InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="💵 Fixed Rate",callback_data="asrv_price_fixed")],
        [InlineKeyboardButton(text="📈 Markup %",callback_data="asrv_price_percent")],
        [InlineKeyboardButton(text="❌ Cancel",callback_data="asrv_add_cancel")]
    ])
    await state.set_state(AdminServiceAddFlow.select_pricing_method)
    await message.answer(preview,reply_markup=kb,parse_mode=ParseMode.HTML)

@router.callback_query(F.data=="asrv_price_fixed",StateFilter(AdminServiceAddFlow.select_pricing_method))
async def admin_add_service_fixed(callback:CallbackQuery,state:FSMContext):
    if not await is_admin(callback.from_user.id): return
    await state.update_data(profit_type="FIXED",profit_value="0")
    await state.set_state(AdminServiceAddFlow.enter_fixed_rate)
    await callback.message.answer("💵 <b>Service Selling Rate</b>\n\nJo amount dalo wahi user ko service rate dikhega.\nExample: provider rate ₹10 → <code>11</code> = ₹11, <code>5</code> = ₹5.",parse_mode=ParseMode.HTML)
    await callback.answer()

@router.callback_query(F.data=="asrv_price_percent",StateFilter(AdminServiceAddFlow.select_pricing_method))
async def admin_add_service_percent(callback:CallbackQuery,state:FSMContext):
    if not await is_admin(callback.from_user.id): return
    await state.set_state(AdminServiceAddFlow.enter_markup_value)
    await callback.message.answer("📈 <b>Markup %</b>\n\nExample: provider rate ₹10 + 10% = ₹11.",parse_mode=ParseMode.HTML)
    await callback.answer()

async def _admin_add_service_description_prompt(message:Message,state:FSMContext):
    d=await state.get_data()
    desc=str(d.get("description") or "").strip()
    if desc:
        await message.answer(f"📝 <b>Description</b>\n\nProvider description already fetched:\n{html.escape(desc)}\n\nAgar change karna hai to naya description bhejo, warna <code>skip</code> bhejo.",parse_mode=ParseMode.HTML)
    else:
        await message.answer("📝 <b>Description</b>\n\nDescription likho, ya <code>skip</code> karke next step par jao.",parse_mode=ParseMode.HTML)
    await state.set_state(AdminServiceAddFlow.enter_description)

@router.message(AdminServiceAddFlow.enter_fixed_rate)
async def admin_add_service_fixed_rate(message:Message,state:FSMContext):
    if not await is_admin(message.from_user.id): return
    try: sell=quantize_currency((message.text or "").strip())
    except: await message.answer("❌ Invalid selling rate."); return
    if sell<=0: await message.answer("❌ Selling rate must be > 0."); return
    await state.update_data(selling_rate=str(sell))
    await _admin_add_service_description_prompt(message,state)

@router.message(AdminServiceAddFlow.enter_markup_value)
async def admin_add_service_markup(message:Message,state:FSMContext):
    if not await is_admin(message.from_user.id): return
    try:v=Decimal((message.text or "").strip())
    except: await message.answer("❌ Invalid markup."); return
    if v<0: await message.answer("❌ Markup cannot be negative."); return
    d=await state.get_data()
    sell=quantize_currency(Decimal(d["provider_rate"])*(Decimal("1")+v/Decimal("100")))
    await state.update_data(markup=str(v),profit_type="PERCENT",profit_value=str(v),selling_rate=str(sell))
    await _admin_add_service_description_prompt(message,state)

@router.message(AdminServiceAddFlow.enter_description)
async def admin_add_service_description(message:Message,state:FSMContext):
    if not await is_admin(message.from_user.id): return
    d=await state.get_data(); text=(message.text or "").strip()
    if text.lower() not in {"skip","-","none"}:
        desc=text
    else:
        desc=str(d.get("description") or "").strip()
    await state.update_data(description=desc)
    async with get_db() as db:
        cats=await (await db.execute("SELECT id,name,icon FROM categories WHERE is_active=1 ORDER BY display_order,id")).fetchall()
    if not cats:
        await message.answer("❌ Create an active category first."); await state.clear(); return
    kb=[[InlineKeyboardButton(text=f"{c['icon'] or '📁'} {c['name'][:28]}",callback_data=f"asrv_cat_{c['id']}")] for c in cats]
    await state.set_state(AdminServiceAddFlow.select_category)
    await message.answer("📂 <b>Select Category</b>",reply_markup=InlineKeyboardMarkup(inline_keyboard=kb),parse_mode=ParseMode.HTML)

@router.callback_query(F.data.startswith("asrv_cat_"),StateFilter(AdminServiceAddFlow.select_category))
async def admin_add_service_category(callback:CallbackQuery,state:FSMContext):
    if not await is_admin(callback.from_user.id): return
    d=await state.get_data(); cid=int(callback.data.rsplit("_",1)[1])
    await state.update_data(category_id=cid)
    await state.set_state(AdminServiceAddFlow.confirm_save)
    await callback.message.answer(
        f"➕ <b>Confirm Service</b>\n\n📛 {html.escape(d['name'])}\n🆔 {d['provider_service_id']}\n"
        f"🔢 {d['min_quantity']:,}-{d['max_quantity']:,}\n💰 Provider Rate: ₹{d['provider_rate']}\n"
        f"💵 Selling Rate: ₹{d['selling_rate']}\n📝 {html.escape(d.get('description') or 'N/A')}",
        parse_mode=ParseMode.HTML,
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="✅ Add Service",callback_data="asrv_save")],
            [InlineKeyboardButton(text="❌ Cancel",callback_data="asrv_add_cancel")]
        ])
    )
    await callback.answer()

@router.callback_query(F.data=="asrv_add_cancel",StateFilter("*"))
async def admin_add_service_cancel(callback:CallbackQuery,state:FSMContext):
    if not await is_admin(callback.from_user.id): return
    await state.clear(); callback.data="ahub_srv_p_1"; await cb_admin_services_list(callback); await callback.answer("Cancelled.")

@router.callback_query(F.data=="asrv_save",StateFilter(AdminServiceAddFlow.confirm_save))
async def admin_add_service_save(callback:CallbackQuery,state:FSMContext):
    if not await is_admin(callback.from_user.id): return
    d=await state.get_data()
    async with get_db() as db:
        duplicate = await (await db.execute(
            "SELECT id FROM services WHERE provider_id=? AND provider_service_id=? LIMIT 1",
            (d["provider_id"], d["provider_service_id"])
        )).fetchone()
        if duplicate:
            await state.clear()
            await callback.answer(f"Service already exists as #{duplicate['id']}.", show_alert=True)
            return
        cur=await db.execute("""INSERT INTO services(category_id,provider_id,provider_service_id,name,min_quantity,max_quantity,provider_rate,profit_type,profit_value,selling_rate,description,refill_available,refill_days,refill_duration,cancel_available,is_active)
        VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,1)""",(
            d["category_id"],d["provider_id"],d["provider_service_id"],d["name"],d["min_quantity"],d["max_quantity"],
            d["provider_rate"],d.get("profit_type","FIXED"),d.get("profit_value","0"),d["selling_rate"],d.get("description",""),
            d.get("refill_available",0),d.get("refill_days",0),d.get("refill_duration",""),d.get("cancel_available",0)
        ))
        sid=cur.lastrowid; await db.commit()
    await state.clear(); await log_admin_event(callback.from_user.id,"ADD_SERVICE",str(sid),d["name"])
    await send_channel_log(callback.bot,"SERVICE",f"➕ <b>Service Added #{sid}</b>\n• {html.escape(d['name'])}\n• Admin: <code>{callback.from_user.id}</code>")
    await notify_admins(
        callback.bot,
        "➕ <b>NEW SERVICE ADDED</b>\n\n"
        f"🆔 Service: <b>#{sid}</b>\n"
        f"🛍️ <b>{html.escape(str(d['name']))}</b>\n"
        f"👤 Admin: <code>{callback.from_user.id}</code>"
    )
    await safe_message_edit(callback.message,f"✅ <b>Service #{sid} added.</b>",reply_markup=InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text="🛍️ Services",callback_data="ahub_srv_p_1")],[InlineKeyboardButton(text="➕ Add Another",callback_data="asrv_add")]]),parse_mode=ParseMode.HTML); await callback.answer("Saved.",show_alert=True)

# ---------------------------------------------------------------------------
# 📥 BULK SERVICE IMPORT
# ---------------------------------------------------------------------------
@router.callback_query(F.data == "asrv_bulk", StateFilter("*"))
async def admin_bulk_import_start(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    await state.clear()
    async with get_db() as db:
        providers = await (await db.execute(
            "SELECT id,name,status FROM providers ORDER BY id"
        )).fetchall()
    if not providers:
        await callback.answer("Add a provider first.", show_alert=True)
        return
    kb = [
        [InlineKeyboardButton(
            text=f"#{p['id']} {p['name']}",
            callback_data=f"abulk_provider_{p['id']}"
        )]
        for p in providers
    ]
    kb.append([InlineKeyboardButton(text="Back", callback_data="ahub_srv_p_1")])
    await safe_message_edit(
        callback.message,
        "Bulk Import\n\nSelect provider:",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=kb),
        parse_mode=ParseMode.HTML
    )
    await state.set_state(AdminBulkImportFlow.select_provider)
    await callback.answer()


@router.callback_query(
    F.data.startswith("abulk_provider_"),
    StateFilter(AdminBulkImportFlow.select_provider)
)
async def admin_bulk_import_provider(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    pid = int(callback.data.rsplit("_", 1)[1])
    async with get_db() as db:
        provider = await (await db.execute(
            "SELECT id,name,status FROM providers WHERE id=?", (pid,)
        )).fetchone()
    if not provider or not provider["status"]:
        await callback.answer("Provider disabled.", show_alert=True)
        return

    await state.update_data(provider_id=pid, provider_name=provider["name"])
    await state.set_state(AdminBulkImportFlow.enter_service_ids)
    await callback.message.answer(
        "Bulk Import\n\n"
        "Send provider service IDs separated by comma, space, or new line.\n\n"
        "Example:\n"
        "<code>1, 2, 5, 10</code>\n\n"
        "The provider catalog will be fetched once and all matching IDs will be imported.",
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.message(AdminBulkImportFlow.enter_service_ids)
async def admin_bulk_import_ids(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    raw = (message.text or "").strip()
    ids = [x for x in re.split(r"[\s,;]+", raw) if x]
    normalized = []
    seen = set()
    for item in ids:
        if not item.isdigit():
            await message.answer(
                f"Invalid service ID: <code>{html.escape(item)}</code>\n"
                "Only numeric IDs are allowed.",
                parse_mode=ParseMode.HTML
            )
            return
        sid = str(int(item))
        if sid not in seen:
            normalized.append(sid)
            seen.add(sid)

    if not normalized:
        await message.answer("Send at least one numeric service ID.")
        return
    if len(normalized) > 200:
        await message.answer("Maximum 200 service IDs per bulk import.")
        return

    d = await state.get_data()
    pid = int(d["provider_id"])
    await message.answer("Fetching provider catalog and matching service IDs...")
    res = await ProviderAPIClient.fetch_services_list(pid)
    if not res.get("success"):
        await message.answer(
            f"Provider fetch failed.\n\n{html.escape(str(res.get('error', 'Unknown error')))}",
            parse_mode=ParseMode.HTML
        )
        return

    catalog = {}
    for item in res.get("services") or []:
        sid = _p_sid(item)
        if sid is not None:
            catalog[str(int(sid))] = item

    found = []
    missing = []
    for sid in normalized:
        item = catalog.get(sid)
        if item is None:
            missing.append(sid)
            continue
        try:
            rate = quantize_currency(_p_rate(item))
            min_q = int(_p_min(item) or 0)
            max_q = int(_p_max(item) or 0)
            if rate <= 0 or min_q < 1 or max_q < min_q:
                missing.append(sid)
                continue
        except Exception:
            missing.append(sid)
            continue
        found.append((sid, item, rate, min_q, max_q))

    if not found:
        await message.answer(
            "No valid matching services were found.\n"
            f"Missing/invalid IDs: <code>{', '.join(normalized)}</code>",
            parse_mode=ParseMode.HTML
        )
        return

    await state.update_data(
        requested_ids=normalized,
        found_ids=[x[0] for x in found],
        missing_ids=missing,
        catalog_items={
            sid: {
                "service": item,
                "rate": str(rate),
                "min": min_q,
                "max": max_q,
                "name": str(item.get("name") or item.get("service_name") or item.get("title") or f"Service {sid}").strip(),
                "description": _p_desc(item),
                "refill": _p_refill(item),
                "cancel": _p_cancel(item),
            }
            for sid, item, rate, min_q, max_q in found
        }
    )

    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="Global Markup %", callback_data="abulk_price_global")],
        [InlineKeyboardButton(text="Provider Rate", callback_data="abulk_price_provider")],
        [InlineKeyboardButton(text="Cancel", callback_data="abulk_cancel")]
    ])
    await state.set_state(AdminBulkImportFlow.select_pricing_method)
    await message.answer(
        "Bulk Import Ready\n\n"
        f"Found: <b>{len(found)}</b>\n"
        f"Missing/invalid: <b>{len(missing)}</b>\n\n"
        "Choose how selling prices should be calculated:",
        reply_markup=kb,
        parse_mode=ParseMode.HTML
    )


@router.callback_query(
    F.data.in_(["abulk_price_global", "abulk_price_provider"]),
    StateFilter(AdminBulkImportFlow.select_pricing_method)
)
async def admin_bulk_import_pricing(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    mode = "GLOBAL" if callback.data.endswith("global") else "PROVIDER"
    await state.update_data(bulk_price_mode=mode)

    if mode == "GLOBAL":
        await state.set_state(AdminBulkImportFlow.enter_markup_value)
        await callback.message.answer(
            "Global Markup %\n\n"
            "Send the markup percentage for all imported services.\n"
            "Example: <code>20</code> = provider rate + 20%.",
            parse_mode=ParseMode.HTML
        )
    else:
        await state.update_data(markup="0", profit_type="FIXED", profit_value="0")
        await admin_bulk_import_ask_category(callback.message, state)
    await callback.answer()


@router.message(AdminBulkImportFlow.enter_markup_value)
async def admin_bulk_import_markup(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    try:
        markup = Decimal((message.text or "").strip())
        if markup < 0 or markup > 10000:
            raise ValueError
    except Exception:
        await message.answer("Invalid markup. Enter a number from 0 to 10000.")
        return
    await state.update_data(
        markup=str(markup),
        profit_type="PERCENT",
        profit_value=str(markup)
    )
    await admin_bulk_import_ask_category(message, state)


async def admin_bulk_import_ask_category(message: Message, state: FSMContext):
    async with get_db() as db:
        cats = await (await db.execute(
            "SELECT id,name FROM categories WHERE is_active=1 ORDER BY display_order,id"
        )).fetchall()
    if not cats:
        await message.answer("Create an active category first.")
        await state.clear()
        return
    kb = [[InlineKeyboardButton(
        text=str(c["name"])[:40],
        callback_data=f"abulk_cat_{c['id']}"
    )] for c in cats]
    kb.append([InlineKeyboardButton(text="Cancel", callback_data="abulk_cancel")])
    await state.set_state(AdminBulkImportFlow.select_category)
    await message.answer(
        "Select Category\n\n"
        "All services in this bulk import will be added to the selected category.",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=kb),
        parse_mode=ParseMode.HTML
    )


@router.callback_query(
    F.data.startswith("abulk_cat_"),
    StateFilter(AdminBulkImportFlow.select_category)
)
async def admin_bulk_import_category(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    cid = int(callback.data.rsplit("_", 1)[1])
    await state.update_data(category_id=cid)
    d = await state.get_data()
    items = d.get("catalog_items", {})
    mode = d.get("bulk_price_mode", "GLOBAL")
    markup = Decimal(str(d.get("markup", "0")))

    preview_lines = []
    for sid, item in list(items.items())[:12]:
        rate = Decimal(str(item["rate"]))
        sell = rate if mode == "PROVIDER" else quantize_currency(rate * (Decimal("1") + markup / Decimal("100")))
        preview_lines.append(
            f"• {sid} — {html.escape(item['name'][:35])} — ₹{sell}"
        )
    more = len(items) - len(preview_lines)
    if more > 0:
        preview_lines.append(f"... and {more} more")

    await state.set_state(AdminBulkImportFlow.confirm_import)
    await callback.message.answer(
        "Confirm Bulk Import\n\n"
        + "\n".join(preview_lines)
        + "\n\n"
        f"Total: <b>{len(items)}</b>\n"
        f"Pricing: <b>{'Provider Rate' if mode == 'PROVIDER' else f'Global Markup {markup}%'} </b>",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="Confirm Import", callback_data="abulk_confirm")],
            [InlineKeyboardButton(text="Cancel", callback_data="abulk_cancel")]
        ]),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data == "abulk_confirm", StateFilter(AdminBulkImportFlow.confirm_import))
async def admin_bulk_import_confirm(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    d = await state.get_data()
    pid = int(d["provider_id"])
    cid = int(d["category_id"])
    mode = d.get("bulk_price_mode", "GLOBAL")
    markup = Decimal(str(d.get("markup", "0")))
    items = d.get("catalog_items", {})

    added = 0
    updated = 0
    skipped = []
    async with get_db() as db:
        for sid, item in items.items():
            try:
                provider_rate = Decimal(str(item["rate"]))
                selling_rate = (
                    provider_rate if mode == "PROVIDER"
                    else quantize_currency(provider_rate * (Decimal("1") + markup / Decimal("100")))
                )
                refill_available, refill_days, refill_duration = item.get("refill", (0, 0, ""))
                cancel_available = int(item.get("cancel", 0))
                existing = await (await db.execute(
                    "SELECT id FROM services WHERE provider_id=? AND provider_service_id=?",
                    (pid, int(sid))
                )).fetchone()
                if existing:
                    await db.execute(
                        """UPDATE services SET category_id=?, name=?, min_quantity=?, max_quantity=?,
                        provider_rate=?, profit_type=?, profit_value=?, selling_rate=?, description=?,
                        refill_available=?, refill_days=?, refill_duration=?, cancel_available=?, is_active=1 WHERE id=?""",
                        (
                            cid, item["name"], item["min"], item["max"], str(provider_rate),
                            "FIXED" if mode == "PROVIDER" else "PERCENT",
                            "0" if mode == "PROVIDER" else str(markup),
                            str(selling_rate), item.get("description", ""),
                            refill_available, refill_days, refill_duration, cancel_available, existing["id"]
                        )
                    )
                    updated += 1
                else:
                    await db.execute(
                        """INSERT INTO services(
                        category_id,provider_id,provider_service_id,name,min_quantity,max_quantity,
                        provider_rate,profit_type,profit_value,selling_rate,description,
                        refill_available,refill_days,refill_duration,cancel_available,is_active)
                        VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,1)""",
                        (
                            cid, pid, int(sid), item["name"], item["min"], item["max"],
                            str(provider_rate),
                            "FIXED" if mode == "PROVIDER" else "PERCENT",
                            "0" if mode == "PROVIDER" else str(markup),
                            str(selling_rate), item.get("description", ""),
                            refill_available, refill_days, refill_duration, cancel_available
                        )
                    )
                    added += 1
            except Exception as e:
                skipped.append(f"{sid}: {e}")
        await db.commit()

    await log_admin_event(
        callback.from_user.id, "BULK_IMPORT_SERVICES", str(pid),
        f"added={added}, updated={updated}, skipped={len(skipped)}"
    )
    if added or updated:
        bulk_service_text = (
            "🛍️ <b>SERVICES UPDATED</b>\n\n"
            f"➕ Added: <b>{added}</b>\n"
            f"🔄 Updated: <b>{updated}</b>\n"
            f"👤 Admin: <code>{callback.from_user.id}</code>"
        )
        await notify_admins(callback.bot, bulk_service_text)
        await send_channel_log(callback.bot, "SERVICE", bulk_service_text)
    await state.clear()
    result = (
        "Bulk Import Complete\n\n"
        f"Added: <b>{added}</b>\n"
        f"Updated: <b>{updated}</b>\n"
        f"Missing/invalid from provider: <b>{len(d.get('missing_ids', []))}</b>\n"
        f"Errors: <b>{len(skipped)}</b>"
    )
    if d.get("missing_ids"):
        result += "\n\nMissing IDs: <code>" + ", ".join(d["missing_ids"]) + "</code>"
    if skipped:
        result += "\n\nErrors:\n<code>" + html.escape("\n".join(skipped[:10])) + "</code>"

    await safe_message_edit(
        callback.message,
        result,
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="Services", callback_data="ahub_srv_p_1")],
            [InlineKeyboardButton(text="Bulk Import Again", callback_data="asrv_bulk")]
        ]),
        parse_mode=ParseMode.HTML
    )
    await callback.answer("Bulk import completed.", show_alert=True)


@router.callback_query(F.data == "abulk_cancel", StateFilter("*"))
async def admin_bulk_import_cancel(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    await state.clear()
    callback.data = "ahub_srv_p_1"
    await cb_admin_services_list(callback)
    await callback.answer("Cancelled.")


# ---------------------------------------------------------------------------
# 🛍️ SERVICES MANAGEMENT (basic)
# ---------------------------------------------------------------------------
@router.callback_query(F.data.startswith("ahub_srv_p_"), StateFilter("*"))
async def cb_admin_services_list(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    page = int(callback.data.split("_")[3])
    per_page = 6
    offset = (page - 1) * per_page
    async with get_db() as db:
        tot = (await (await db.execute("SELECT COUNT(*) FROM services")).fetchone())[0]
        srvs = await (await db.execute(
            "SELECT * FROM services ORDER BY id DESC LIMIT ? OFFSET ?",
            (per_page, offset)
        )).fetchall()
    buttons = []
    for s in srvs:
        st = "🟢" if s["is_active"] else "🔴"
        name = (s["name"] or "Service")[:18]
        buttons.append([InlineKeyboardButton(
            text=f"{st} #{s['id']} • {name}",
            callback_data=f"asrv_view_{s['id']}"
        )])
    tot_pages = (tot + per_page - 1) // per_page
    if tot_pages > 1:
        buttons.append(make_pagination(page, tot_pages, "ahub_srv"))
    buttons.append([InlineKeyboardButton(text="🗑️ Delete All", callback_data="asrv_delete_all_confirm")])
    buttons.append([InlineKeyboardButton(text="⬅️ Back to Hub", callback_data="ahub_back")])
    await safe_message_edit(
        callback.message,
        f"🛍️ <b>SERVICES ({tot}):</b>",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=buttons),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data.startswith("asrv_view_"), StateFilter("*"))
async def admin_service_view(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    sid = int(callback.data.rsplit("_", 1)[1])
    async with get_db() as db:
        s = await (await db.execute("SELECT * FROM services WHERE id=?", (sid,))).fetchone()
    if not s:
        await callback.answer("Service not found.", show_alert=True)
        return
    status = "🟢 Active" if s["is_active"] else "🔴 Disabled"
    text = (
        f"🛍️ <b>SERVICE #{s['id']}</b>\n\n"
        f"📛 <b>{html.escape(s['name'])}</b>\n"
        f"🔌 Provider ID: <code>{s['provider_id']}</code>\n"
        f"🆔 Provider Service ID: <code>{s['provider_service_id']}</code>\n"
        f"💰 Provider Rate: <b>{s['provider_rate']}</b>\n"
        f"💵 Selling Rate: <b>₹{s['selling_rate']}</b>\n"
        f"📈 Markup: <b>{s['profit_value']}{'%' if s['profit_type']=='PERCENT' else ''}</b>\n"
        f"📂 Category ID: <b>{s['category_id']}</b>\n"
        f"🔢 Min/Max: <b>{s['min_quantity']} / {s['max_quantity']}</b>\n"
        f"♻️ Refill: <b>{'Yes' if s['refill_available'] else 'No'}</b>\n"
        f"📊 Status: <b>{status}</b>"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✏️ Edit Service", callback_data=f"asrv_edit_{sid}")],
        [InlineKeyboardButton(text="🔁 Enable/Disable", callback_data=f"asrv_toggle_{sid}"),
         InlineKeyboardButton(text="🗑️ Delete", callback_data=f"asrv_delete_{sid}")],
        [InlineKeyboardButton(text="⬅️ Services", callback_data="ahub_srv_p_1")]
    ])
    await safe_message_edit(callback.message, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("asrv_edit_"), StateFilter("*"))
async def admin_service_edit_menu(callback: CallbackQuery, state: Optional[FSMContext] = None):
    if not await is_admin(callback.from_user.id):
        return
    sid = int(callback.data.rsplit("_", 1)[1])
    async with get_db() as db:
        s = await (await db.execute("SELECT * FROM services WHERE id=?", (sid,))).fetchone()
    if not s:
        await callback.answer("Service not found.", show_alert=True); return
    text = (
        f"✏️ <b>EDIT SERVICE #{sid}</b>\n\n"
        f"📛 Name: <b>{html.escape(s['name'])}</b>\n"
        f"💵 Selling Rate: <b>₹{s['selling_rate']}</b>\n"
        f"📝 Description: <b>{html.escape((s['description'] or 'N/A')[:120])}</b>\n"
        f"🔢 Min/Max: <b>{s['min_quantity']} / {s['max_quantity']}</b>\n"
        f"📂 Category ID: <b>{s['category_id']}</b>\n"
        f"♻️ Refill: <b>{'Yes' if s['refill_available'] else 'No'}</b>\n"
        f"🚫 Cancel: <b>{'Yes' if s['cancel_available'] else 'No'}</b>\n\n"
        "Choose what you want to edit:"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="📛 Name", callback_data=f"asedit_field_name_{sid}"), InlineKeyboardButton(text="💵 Price", callback_data=f"asedit_field_price_{sid}")],
        [InlineKeyboardButton(text="📝 Description", callback_data=f"asedit_field_desc_{sid}"), InlineKeyboardButton(text="🔢 Min/Max", callback_data=f"asedit_field_minmax_{sid}")],
        [InlineKeyboardButton(text="📂 Category", callback_data=f"asedit_field_category_{sid}"), InlineKeyboardButton(text="⚙️ Other", callback_data=f"asedit_field_other_{sid}")],
        [InlineKeyboardButton(text="⬅️ Service", callback_data=f"asrv_view_{sid}")]
    ])
    if state is not None:
        await state.clear()
    await safe_message_edit(callback.message, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("asedit_field_"), StateFilter("*"))
async def admin_service_edit_field(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id): return
    parts = callback.data.split("_")
    field = parts[2]
    sid = int(parts[3])
    if field == "category":
        async with get_db() as db:
            cats = await (await db.execute("SELECT id,name,icon FROM categories WHERE is_active=1 ORDER BY display_order,id")).fetchall()
        kb = [[InlineKeyboardButton(text=f"{c['icon'] or '📁'} {c['name'][:28]}", callback_data=f"asedit_cat_{sid}_{c['id']}")] for c in cats]
        kb.append([InlineKeyboardButton(text="⬅️ Back", callback_data=f"asrv_edit_{sid}")])
        await safe_message_edit(callback.message, "📂 <b>Select New Category</b>", reply_markup=InlineKeyboardMarkup(inline_keyboard=kb), parse_mode=ParseMode.HTML)
        await callback.answer(); return
    if field == "other":
        async with get_db() as db:
            s = await (await db.execute("SELECT refill_available,refill_days,refill_duration,cancel_available FROM services WHERE id=?", (sid,))).fetchone()
        if not s:
            await callback.answer("Service not found.", show_alert=True); return
        kb = InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="♻️ Toggle Refill", callback_data=f"asedit_toggle_refill_{sid}")],
            [InlineKeyboardButton(text="🚫 Toggle Cancel", callback_data=f"asedit_toggle_cancel_{sid}")],
            [InlineKeyboardButton(text="📅 Refill Days", callback_data=f"asedit_refilldays_{sid}")],
            [InlineKeyboardButton(text="⬅️ Back", callback_data=f"asrv_edit_{sid}")]
        ])
        await safe_message_edit(callback.message, f"⚙️ <b>Other Settings</b>\n\n♻️ Refill: {'Yes' if s['refill_available'] else 'No'}\n📅 Days: {s['refill_days']}\n🚫 Cancel: {'Yes' if s['cancel_available'] else 'No'}", reply_markup=kb, parse_mode=ParseMode.HTML)
        await callback.answer(); return
    prompts = {
        "name": "📛 Send the new service name:",
        "price": "💵 Send the new selling price (e.g. 15 or 15.50):",
        "desc": "📝 Send the new description. Send <code>-</code> to clear it:",
        "minmax": "🔢 Send Min and Max like: <code>10 100000</code>"
    }
    await state.update_data(edit_service_id=sid, edit_field=field)
    await state.set_state(AdminServiceEditFlow.enter_value)
    await callback.message.answer(prompts[field], parse_mode=ParseMode.HTML, reply_markup=get_back_inline(f"asrv_edit_{sid}"))
    await callback.answer()


@router.callback_query(F.data.startswith("asedit_cat_"), StateFilter("*"))
async def admin_service_edit_category(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id): return
    _,_,_,sid,cid = callback.data.split("_")
    async with get_db() as db:
        svc = await (await db.execute("SELECT name FROM services WHERE id=?", (int(sid),))).fetchone()
        await db.execute("UPDATE services SET category_id=? WHERE id=?", (int(cid), int(sid))); await db.commit()
    if svc:
        await notify_service_change(callback.bot, int(sid), svc["name"], "Category changed.")
    await callback.answer("Category updated!")
    callback.data=f"asrv_view_{sid}"; await admin_service_view(callback)


@router.message(AdminServiceEditFlow.enter_value)
async def admin_service_edit_save(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id): return
    d=await state.get_data(); sid=int(d["edit_service_id"]); field=d["edit_field"]; raw=(message.text or "").strip()
    try:
        if field == "name":
            if not raw or len(raw)>150: raise ValueError
            sql,args="UPDATE services SET name=? WHERE id=?",(raw,sid)
        elif field == "price":
            val=quantize_currency(raw)
            if val<=0: raise ValueError
            sql,args="UPDATE services SET selling_rate=?, profit_type='FIXED', profit_value='0' WHERE id=?",(str(val),sid)
        elif field == "desc":
            sql,args="UPDATE services SET description=? WHERE id=?",("" if raw.lower() in {"-","none","clear"} else raw,sid)
        elif field == "refilldays":
            days=int(raw)
            if days < 0: raise ValueError
            sql,args="UPDATE services SET refill_days=?, refill_duration=? WHERE id=?",(days, "Lifetime" if days == 0 else f"{days} Days", sid)
        else:
            parts=raw.replace(","," ").split()
            if len(parts)!=2: raise ValueError
            mn,mx=int(parts[0]),int(parts[1])
            if mn<1 or mx<mn: raise ValueError
            sql,args="UPDATE services SET min_quantity=?, max_quantity=? WHERE id=?",(mn,mx,sid)
    except Exception:
        await message.answer("❌ Invalid value. Please try again."); return
    async with get_db() as db:
        before = await (await db.execute("SELECT * FROM services WHERE id=?", (sid,))).fetchone()
        cur=await db.execute(sql,args)
        if cur.rowcount!=1: await db.rollback(); await state.clear(); await message.answer("❌ Service not found."); return
        await db.commit()
    if before and field == "price":
        await notify_users_price_change(
            message.bot, sid, before["name"], before["selling_rate"], str(val)
        )
    elif before and field == "name":
        txt = (
            "🛍️ <b>SERVICE UPDATED</b>\n\n"
            f"🆔 Service: <b>#{sid}</b>\n"
            f"📛 Old: <b>{html.escape(str(before['name']))}</b>\n"
            f"📛 New: <b>{html.escape(str(raw))}</b>"
        )
        await notify_admins(message.bot, txt)
        await send_channel_log(message.bot, "SERVICE", txt)
    elif before:
        labels = {"desc": "Description changed.", "minmax": "Minimum/maximum quantity changed.", "refilldays": "Refill days changed."}
        await notify_service_change(message.bot, sid, before["name"], labels.get(field, "Service settings changed."))
    await state.clear()
    await log_admin_event(message.from_user.id,"EDIT_SERVICE",str(sid),field)
    await message.answer("✅ Service updated successfully.", reply_markup=InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text="✏️ Continue Editing",callback_data=f"asrv_edit_{sid}")],[InlineKeyboardButton(text="⬅️ Service",callback_data=f"asrv_view_{sid}")]]))


@router.callback_query(F.data.startswith("asedit_toggle_refill_"), StateFilter("*"))
async def admin_service_toggle_refill(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id): return
    sid=int(callback.data.rsplit("_",1)[1])
    async with get_db() as db:
        r=await (await db.execute("SELECT refill_available FROM services WHERE id=?",(sid,))).fetchone()
        if not r: await callback.answer("Not found",show_alert=True); return
        await db.execute("UPDATE services SET refill_available=? WHERE id=?",(0 if r["refill_available"] else 1,sid)); await db.commit()
    await callback.answer("Refill updated!"); callback.data=f"asrv_edit_{sid}"; await admin_service_edit_menu(callback, None)


@router.callback_query(F.data.startswith("asedit_toggle_cancel_"), StateFilter("*"))
async def admin_service_toggle_cancel(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id): return
    sid=int(callback.data.rsplit("_",1)[1])
    async with get_db() as db:
        r=await (await db.execute("SELECT cancel_available FROM services WHERE id=?",(sid,))).fetchone()
        if not r: await callback.answer("Not found",show_alert=True); return
        await db.execute("UPDATE services SET cancel_available=? WHERE id=?",(0 if r["cancel_available"] else 1,sid)); await db.commit()
    await callback.answer("Cancel option updated!"); callback.data=f"asrv_edit_{sid}"; await admin_service_edit_menu(callback, None)


@router.callback_query(F.data.startswith("asedit_refilldays_"), StateFilter("*"))
async def admin_service_edit_refilldays(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id): return
    sid=int(callback.data.rsplit("_",1)[1])
    await state.update_data(edit_service_id=sid, edit_field="refilldays")
    await state.set_state(AdminServiceEditFlow.enter_value)
    await callback.message.answer("📅 Send refill days. Use <code>0</code> for Lifetime:",parse_mode=ParseMode.HTML,reply_markup=get_back_inline(f"asrv_edit_{sid}"))
    await callback.answer()

@router.callback_query(F.data.startswith("asrv_toggle_"), StateFilter("*"))
async def admin_service_toggle(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    sid = int(callback.data.rsplit("_", 1)[1])
    async with get_db() as db:
        r = await (await db.execute("SELECT is_active, name FROM services WHERE id=?", (sid,))).fetchone()
        if not r:
            await callback.answer("Service not found.", show_alert=True)
            return
        await db.execute("UPDATE services SET is_active=? WHERE id=?", (0 if r["is_active"] else 1, sid))
        await db.commit()
    await log_admin_event(callback.from_user.id,"TOGGLE_SERVICE",str(sid),r["name"])
    await send_channel_log(callback.bot,"SERVICE",f"🔁 <b>Service Toggled #{sid}</b>\n• {html.escape(r['name'])}\n• Admin: <code>{callback.from_user.id}</code>")
    await callback.answer("Service toggled!")
    callback.data = f"asrv_view_{sid}"
    await admin_service_view(callback)


@router.callback_query(F.data.startswith("asrv_delete_") & ~F.data.startswith("asrv_delete_all"), StateFilter("*"))
async def admin_service_delete(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    sid = int(callback.data.rsplit("_", 1)[1])
    async with get_db() as db:
        srow = await (await db.execute(
            "SELECT id, name FROM services WHERE id=?", (sid,)
        )).fetchone()
        if not srow:
            await callback.answer("Service not found.", show_alert=True)
            return
        await db.execute("DELETE FROM services WHERE id=?", (sid,))
        await db.commit()
    await log_admin_event(callback.from_user.id, "DELETE_SERVICE", str(sid), srow["name"])
    await send_channel_log(callback.bot,"SERVICE",f"🗑️ <b>Service Deleted #{sid}</b>\n• {html.escape(srow['name'])}\n• Admin: <code>{callback.from_user.id}</code>")
    await callback.answer("Service deleted.", show_alert=True)
    callback.data = "ahub_srv_p_1"
    await cb_admin_services_list(callback)


@router.callback_query(F.data == "asrv_delete_all_confirm", StateFilter("*"))
async def admin_services_delete_all_confirm(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    async with get_db() as db:
        total = (await (await db.execute("SELECT COUNT(*) FROM services")).fetchone())[0]
    if not total:
        await callback.answer("No services to delete.", show_alert=True)
        return
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text=f"⚠️ DELETE ALL ({total})", callback_data="asrv_delete_all")],
        [InlineKeyboardButton(text="⬅️ Cancel", callback_data="ahub_srv_p_1")]
    ])
    await safe_message_edit(
        callback.message,
        f"⚠️ <b>DELETE ALL SERVICES?</b>\n\nThis will permanently remove <b>{total}</b> services.\n\n<b>Cannot be undone.</b>",
        reply_markup=kb, parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data == "asrv_delete_all", StateFilter("*"))
async def admin_services_delete_all(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    async with get_db() as db:
        total = (await (await db.execute("SELECT COUNT(*) FROM services")).fetchone())[0]
        await db.execute("DELETE FROM services")
        await db.commit()
    await log_admin_event(callback.from_user.id, "DELETE_ALL_SERVICES", "ALL", f"Deleted={total}")
    await safe_message_edit(
        callback.message,
        f"✅ <b>ALL SERVICES DELETED</b>\n\n🗑️ Removed: <b>{total}</b> services.",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="⬅️ Admin Panel", callback_data="ahub_back")]
        ]),
        parse_mode=ParseMode.HTML
    )
    await callback.answer("All services deleted.", show_alert=True)


# ---------------------------------------------------------------------------
# 📂 CATEGORIES MANAGEMENT — with UP/DOWN ordering
# ---------------------------------------------------------------------------
@router.callback_query(F.data.startswith("ahub_cat_p_"), StateFilter("*"))
async def admin_categories(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    page = int(callback.data.rsplit("_", 1)[1])
    per_page = 6
    async with get_db() as db:
        total = (await (await db.execute("SELECT COUNT(*) FROM categories")).fetchone())[0]
        total_pages = max(1, (total + per_page - 1) // per_page)
        page = max(1, min(page, total_pages))
        offset = (page - 1) * per_page
        rows = await (await db.execute(
            "SELECT id,name,icon,display_order,is_active FROM categories ORDER BY display_order ASC,id ASC LIMIT ? OFFSET ?",
            (per_page, offset)
        )).fetchall()
    text = f"📂 <b>CATEGORIES</b>\n\nPage <b>{page}/{total_pages}</b> · {total} total"
    if rows:
        text += "\n\n" + "\n".join(
            f"{offset+i}. {r['icon'] or '📁'} <b>#{r['id']}</b> {html.escape(r['name'])} — {'🟢' if r['is_active'] else '🔴'}"
            for i,r in enumerate(rows,1)
        )
    else:
        text += "\n\nNo categories found."
    kb=[]
    for r in rows:
        kb.append([InlineKeyboardButton(text=f"{'🟢' if r['is_active'] else '🔴'} #{r['id']} {r['name'][:22]}", callback_data=f"acat_view_{r['id']}")])
    if total_pages > 1:
        kb.append(make_pagination(page, total_pages, "ahub_cat"))
    kb.append([InlineKeyboardButton(text="➕ Add Category", callback_data="acat_add")])
    kb.append([InlineKeyboardButton(text="⬅️ Back", callback_data="ahub_back")])
    await safe_message_edit(callback.message, text, reply_markup=InlineKeyboardMarkup(inline_keyboard=kb), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data == "acat_add", StateFilter("*"))
async def admin_category_add_start(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    await state.clear()
    await state.set_state(AdminCategoryAddFlow.enter_name)
    await callback.message.answer("📂 Enter category name:")
    await callback.answer()


@router.message(AdminCategoryAddFlow.enter_name)
async def admin_category_add_name(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    name = (message.text or "").strip()
    if not name or len(name) > 60:
        await message.answer("❌ Name must be 1-60 characters.")
        return
    await state.update_data(name=name)
    await state.set_state(AdminCategoryAddFlow.enter_icon)
    await message.answer("🖼️ Send icon/emoji, or send SKIP:")


@router.message(AdminCategoryAddFlow.enter_icon)
async def admin_category_add_icon(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    d = await state.get_data()
    icon = (message.text or "").strip()
    icon = "📁" if icon.upper() == "SKIP" or not icon else icon[:8]
    async with get_db() as db:
        mx = (await (await db.execute("SELECT COALESCE(MAX(display_order),0) FROM categories")).fetchone())[0] or 0
        cur = await db.execute(
            "INSERT INTO categories(name, icon, display_order, is_active) VALUES(?, ?, ?, 1)",
            (d["name"], icon, int(mx) + 1)
        )
        cid = cur.lastrowid
        await db.commit()
    await state.clear()
    await message.answer(
        f"✅ Category <b>#{cid}</b> created.",
        parse_mode=ParseMode.HTML,
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="📂 Categories", callback_data="ahub_cat_p_1")]
        ])
    )


@router.callback_query(F.data.startswith("acat_view_"), StateFilter("*"))
async def admin_category_view(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    cid = int(callback.data.rsplit("_", 1)[1])
    async with get_db() as db:
        r = await (await db.execute("SELECT * FROM categories WHERE id=?", (cid,))).fetchone()
        used = (await (await db.execute("SELECT COUNT(*) FROM services WHERE category_id=?", (cid,))).fetchone())[0] if r else 0
    if not r:
        await callback.answer("Category not found.", show_alert=True)
        return
    text = (
        f"📂 <b>CATEGORY #{cid}</b>\n\n"
        f"📛 Name: <b>{html.escape(r['name'])}</b>\n"
        f"🖼️ Icon: {html.escape(r['icon'] or '📁')}\n"
        f"🔢 Display Order: <b>{r['display_order']}</b>\n"
        f"🛍️ Services: <b>{used}</b>\n"
        f"📊 Status: <b>{'Active' if r['is_active'] else 'Disabled'}</b>"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="⬆️ Move Up", callback_data=f"acat_up_{cid}"),
         InlineKeyboardButton(text="⬇️ Move Down", callback_data=f"acat_down_{cid}")],
        [InlineKeyboardButton(text="✏️ Edit Name", callback_data=f"acat_edit_name_{cid}"),
         InlineKeyboardButton(text="🖼️ Edit Icon", callback_data=f"acat_edit_icon_{cid}")],
        [InlineKeyboardButton(text="🔁 Enable/Disable", callback_data=f"acat_toggle_{cid}")],
        [InlineKeyboardButton(text="🗑️ Delete", callback_data=f"acat_delete_{cid}")],
        [InlineKeyboardButton(text="⬅️ Categories", callback_data="ahub_cat_p_1")]
    ])
    await safe_message_edit(callback.message, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("acat_edit_name_"), StateFilter("*"))
async def admin_category_edit_name_start(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    cid = int(callback.data.rsplit("_", 1)[1])
    await state.clear()
    await state.update_data(category_id=cid)
    await state.set_state(AdminCategoryEditFlow.enter_new_name)
    await callback.message.answer("✏️ Send the new category name (1-60 characters):")
    await callback.answer()


@router.message(AdminCategoryEditFlow.enter_new_name)
async def admin_category_edit_name_save(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    name = (message.text or "").strip()
    if not name or len(name) > 60:
        await message.answer("❌ Name must be 1-60 characters.")
        return
    d = await state.get_data()
    cid = int(d["category_id"])
    async with get_db() as db:
        await db.execute("UPDATE categories SET name=? WHERE id=?", (name, cid))
        await db.commit()
    await state.clear()
    await message.answer("✅ Category name updated.")
    # Re-open category page.
    async with get_db() as db:
        exists = await (await db.execute("SELECT id FROM categories WHERE id=?", (cid,))).fetchone()
    if exists:
        await message.answer("📂 Category updated. Open Categories to view it.",
                             reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                                 [InlineKeyboardButton(text="📂 Categories", callback_data="ahub_cat_p_1")]
                             ]))


@router.callback_query(F.data.startswith("acat_edit_icon_"), StateFilter("*"))
async def admin_category_edit_icon_start(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    cid = int(callback.data.rsplit("_", 1)[1])
    await state.clear()
    await state.update_data(category_id=cid)
    await state.set_state(AdminCategoryEditFlow.enter_new_icon)
    await callback.message.answer("🖼️ Send the new category icon/emoji:")
    await callback.answer()


@router.message(AdminCategoryEditFlow.enter_new_icon)
async def admin_category_edit_icon_save(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    icon = (message.text or "").strip()
    if not icon:
        await message.answer("❌ Icon cannot be empty.")
        return
    d = await state.get_data()
    cid = int(d["category_id"])
    async with get_db() as db:
        await db.execute("UPDATE categories SET icon=? WHERE id=?", (icon[:8], cid))
        await db.commit()
    await state.clear()
    await message.answer("✅ Category icon updated.",
                         reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                             [InlineKeyboardButton(text="📂 Categories", callback_data="ahub_cat_p_1")]
                         ]))


@router.callback_query(F.data.startswith("acat_up_"), StateFilter("*"))
async def admin_category_move_up(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    cid = int(callback.data.rsplit("_", 1)[1])
    async with get_db() as db:
        cur = await (await db.execute(
            "SELECT id, display_order FROM categories WHERE id=?", (cid,)
        )).fetchone()
        if not cur:
            await callback.answer("Not found.", show_alert=True)
            return
        # Find category above
        prev = await (await db.execute(
            "SELECT id, display_order FROM categories WHERE display_order < ? ORDER BY display_order DESC LIMIT 1",
            (cur["display_order"],)
        )).fetchone()
        if not prev:
            await callback.answer("Already at top.", show_alert=True)
            return
        await db.execute("UPDATE categories SET display_order=? WHERE id=?", (prev["display_order"], cid))
        await db.execute("UPDATE categories SET display_order=? WHERE id=?", (cur["display_order"], prev["id"]))
        await db.commit()
    await callback.answer("Moved up!")
    callback.data = f"acat_view_{cid}"
    await admin_category_view(callback)


@router.callback_query(F.data.startswith("acat_down_"), StateFilter("*"))
async def admin_category_move_down(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    cid = int(callback.data.rsplit("_", 1)[1])
    async with get_db() as db:
        cur = await (await db.execute(
            "SELECT id, display_order FROM categories WHERE id=?", (cid,)
        )).fetchone()
        if not cur:
            await callback.answer("Not found.", show_alert=True)
            return
        nxt = await (await db.execute(
            "SELECT id, display_order FROM categories WHERE display_order > ? ORDER BY display_order ASC LIMIT 1",
            (cur["display_order"],)
        )).fetchone()
        if not nxt:
            await callback.answer("Already at bottom.", show_alert=True)
            return
        await db.execute("UPDATE categories SET display_order=? WHERE id=?", (nxt["display_order"], cid))
        await db.execute("UPDATE categories SET display_order=? WHERE id=?", (cur["display_order"], nxt["id"]))
        await db.commit()
    await callback.answer("Moved down!")
    callback.data = f"acat_view_{cid}"
    await admin_category_view(callback)


@router.callback_query(F.data.startswith("acat_toggle_"), StateFilter("*"))
async def admin_category_toggle(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    cid = int(callback.data.rsplit("_", 1)[1])
    async with get_db() as db:
        r = await (await db.execute("SELECT is_active FROM categories WHERE id=?", (cid,))).fetchone()
        if not r:
            await callback.answer("Not found.", show_alert=True)
            return
        await db.execute("UPDATE categories SET is_active=? WHERE id=?", (0 if r["is_active"] else 1, cid))
        await db.commit()
    callback.data = f"acat_view_{cid}"
    await admin_category_view(callback)


@router.callback_query(F.data.startswith("acat_delete_"), StateFilter("*"))
async def admin_category_delete(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    cid = int(callback.data.rsplit("_", 1)[1])
    async with get_db() as db:
        r = await (await db.execute("SELECT id, name FROM categories WHERE id=?", (cid,))).fetchone()
        if not r:
            await callback.answer("Category not found.", show_alert=True)
            return
        await db.execute("DELETE FROM categories WHERE id=?", (cid,))
        await db.commit()
    await log_admin_event(callback.from_user.id, "DELETE_CATEGORY", str(cid), r["name"])
    callback.data = "ahub_cat_p_1"
    await admin_categories(callback)


# ---------------------------------------------------------------------------
# ⚙️ SETTINGS (with new USDT keys + discount)
# ---------------------------------------------------------------------------
SETTINGS_PAGE_SIZE = 6


SETTINGS_KEYS = [
    ("GLOBAL_MARKUP_PERCENT", "📈 Global Markup %"),
    ("GLOBAL_DISCOUNT_PERCENT", "🎁 Global Discount %"),
    ("DISCOUNT_ENABLED", "🎯 Discount ON/OFF"),
    ("USDT_MIN_DEPOSIT", "🔵 USDT Min ($)"),
    ("USDT_MAX_DEPOSIT", "🔵 USDT Max ($)"),
    ("MIN_DEPOSIT", "💰 INR Min Deposit"),
    ("MAX_DEPOSIT", "💰 INR Max Deposit"),
    ("UPI_ID", "💳 UPI ID"),
    ("UPI_NAME", "👤 UPI Payee Name"),
    ("AUTO_UPI_ID", "⚡ Auto UPI ID"),
    ("AUTO_UPI_NAME", "⚡ Auto UPI Name"),
    ("AUTO_PAY_MID", "⚡ Auto Pay MID"),
    ("AUTO_PAY_API_URL", "⚡ Auto Pay API URL"),
    ("FAMPAY_GMAIL", "📧 Payment Gmail"),
    ("FAMPAY_APP_PASSWORD", "🔐 Gmail App Password"),
    ("USDT_BEP20_ADDRESS", "🔵 USDT BEP20 Address"),
    ("CURRENCY_SYMBOL", "💱 Currency Symbol"),
    ("REFERRAL_PERCENT", "🤝 Referral %"),
    ("REFERRAL_ENABLED", "🤝 Referral ON/OFF"),
    ("SUPPORT_USERNAME", "🎫 Support Username"),
    ("SUPPORT_CHANNEL", "📢 Support Channel"),
    ("AUTO_SYNC_INTERVAL", "⏱ Order Sync (sec)"),
    ("API_TIMEOUT", "⌛ API Timeout (sec)"),
    ("API_RETRY_COUNT", "🔁 API Retries"),
    ("LIVE_RATE_ENABLED", "💱 Live Rate ON/OFF"),
    ("RATE_CACHE_MINUTES", "⏱ Rate Cache (min)"),
    ("LOG_CHANNEL_ID", "🧾 Log Channel ID"),
    ("LOG_CHANNEL_ENABLED", "🧾 Logs ON/OFF"),
    ("PAY_UPI_AUTO_ENABLED", "🟢 UPI Auto ON/OFF"),
    ("PAY_UPI_MANUAL_ENABLED", "🟡 UPI Manual ON/OFF"),
    ("PAY_USDT_ENABLED", "🔵 USDT ON/OFF"),
    ("MAINTENANCE_MODE", "🛠️ Global Maintenance"),
    ("MAINT_NEW_ORDER", "🛒 Order Maintenance"),
    ("MAINT_ADD_BALANCE", "💰 Deposit Maintenance"),
    ("MAINT_SERVICE", "🛍️ Service Maintenance"),
]


async def _render_admin_settings(callback: CallbackQuery, page: int = 1):
    total_pages = max(1, (len(SETTINGS_KEYS) + SETTINGS_PAGE_SIZE - 1) // SETTINGS_PAGE_SIZE)
    page = max(1, min(int(page), total_pages))
    start = (page - 1) * SETTINGS_PAGE_SIZE
    chunk = SETTINGS_KEYS[start:start + SETTINGS_PAGE_SIZE]
    live_rate = await get_live_usdt_inr_rate()
    lines = [
        "⚙️ <b>SETTINGS CENTER</b>",
        f"Page <b>{page}/{total_pages}</b> · {len(SETTINGS_KEYS)} settings",
        f"Live USDT/INR: <code>₹{_fmt_decimal_trim(live_rate, 4)}</code>",
        "",
    ]
    buttons=[]
    secret_keys = {"FAMPAY_GMAIL", "FAMPAY_APP_PASSWORD"}
    for key,label in chunk:
        val=await get_setting(key, DEFAULT_SETTINGS.get(key, ""))
        raw = str(val or "")
        if key == "FAMPAY_APP_PASSWORD":
            display = "••••••••" if raw else "Not set"
        elif key == "FAMPAY_GMAIL":
            if "@" in raw:
                local, domain = raw.split("@", 1)
                display = (local[:2] + "***@" + domain) if len(local) > 2 else ("***@" + domain)
            else:
                display = "Not set" if not raw else "***"
        else:
            display=html.escape(raw)
            if len(display)>24: display=display[:21]+"…"
        lines.append(f"{label}: <code>{html.escape(display)}</code>")
        row=[InlineKeyboardButton(text=f"✏️ {label}", callback_data=f"aset_edit:{key}")]
        if key in secret_keys:
            row.append(InlineKeyboardButton(text="🗑️ Delete", callback_data=f"aset_del:{key}"))
        buttons.append(row)
    if total_pages > 1:
        buttons.append(make_pagination(page, total_pages, "aset_pg"))
    buttons.append([InlineKeyboardButton(text="🔄 Refresh", callback_data=f"aset_pg_p_{page}"), InlineKeyboardButton(text="⬅️ Back", callback_data="ahub_back")])
    await safe_message_edit(callback.message, "\n".join(lines), reply_markup=InlineKeyboardMarkup(inline_keyboard=buttons), parse_mode=ParseMode.HTML)


@router.callback_query(F.data == "ahub_settings", StateFilter("*"))
async def admin_settings_menu(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    await _render_admin_settings(callback, 1)
    await callback.answer()


@router.callback_query(F.data.startswith("aset_pg_p_"), StateFilter("*"))
async def admin_settings_page(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    page=int(callback.data.rsplit("_",1)[1])
    await _render_admin_settings(callback, page)
    await callback.answer()


@router.callback_query(F.data.startswith("aset_edit:"), StateFilter("*"))
async def admin_setting_edit(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    key = callback.data.split(":", 1)[1]
    allowed = {
        "CURRENCY_SYMBOL", "GLOBAL_MARKUP_PERCENT", "GLOBAL_DISCOUNT_PERCENT", "DISCOUNT_ENABLED",
        "USDT_MIN_DEPOSIT", "USDT_MAX_DEPOSIT",
        "MIN_DEPOSIT", "MAX_DEPOSIT", "UPI_ID", "UPI_NAME",
        "AUTO_UPI_ID", "AUTO_UPI_NAME", "AUTO_PAY_MID", "AUTO_PAY_API_URL",
        "USDT_BEP20_ADDRESS", "AUTO_SYNC_INTERVAL", "PROVIDER_SERVICE_CHECK_INTERVAL",
        "API_TIMEOUT", "API_RETRY_COUNT", "REFERRAL_PERCENT", "REFERRAL_ENABLED",
        "SUPPORT_USERNAME", "SUPPORT_CHANNEL", "LIVE_RATE_ENABLED", "RATE_CACHE_MINUTES",
        "LOG_CHANNEL_ID", "LOG_CHANNEL_ENABLED", "PAY_UPI_AUTO_ENABLED",
        "PAY_UPI_MANUAL_ENABLED", "PAY_USDT_ENABLED", "MAINTENANCE_MODE",
        "MAINT_NEW_ORDER", "MAINT_ADD_BALANCE", "MAINT_SERVICE",
        "FAMPAY_GMAIL", "FAMPAY_APP_PASSWORD",
    }
    if key not in allowed:
        await callback.answer("Not editable.", show_alert=True)
        return
    await state.update_data(setting_key=key)
    await state.set_state(AdminSettingsFlow.enter_value)
    current = await get_setting(key, DEFAULT_SETTINGS.get(key, ""))
    if key == "FAMPAY_APP_PASSWORD":
        shown = "••••••••" if current else "Not set"
        instruction = "Send the new Gmail App Password. To remove it, use the 🗑️ Delete button in Settings."
    elif key == "FAMPAY_GMAIL":
        shown = str(current) if current else "Not set"
        instruction = "Send the Gmail address. To remove it, use the 🗑️ Delete button in Settings."
    else:
        shown = str(current)
        instruction = "Send the new value."
    await callback.message.answer(
        f"⚙️ <b>Edit {html.escape(key)}</b>\n\n"
        f"Current: <code>{html.escape(shown)}</code>\n\n"
        f"{instruction}",
        parse_mode=ParseMode.HTML,
        reply_markup=get_back_inline("ahub_settings")
    )
    await callback.answer()


@router.callback_query(F.data.startswith("aset_del:"), StateFilter("*"))
async def admin_setting_delete(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    key = callback.data.split(":", 1)[1]
    if key not in {"FAMPAY_GMAIL", "FAMPAY_APP_PASSWORD"}:
        await callback.answer("Delete is only available for payment Gmail credentials.", show_alert=True)
        return
    await set_setting(key, "")
    await state.clear()
    await log_admin_event(callback.from_user.id, "DELETE_SETTING", key, "Cleared payment credential")
    label = "Payment Gmail" if key == "FAMPAY_GMAIL" else "Gmail App Password"
    await callback.answer(f"{label} deleted.", show_alert=True)
    await _render_admin_settings(callback, 1)


@router.message(AdminSettingsFlow.enter_value)
async def admin_setting_save(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    data = await state.get_data()

    # Admin add/remove action
    admin_action = data.get("admin_action")
    if admin_action:
        await state.clear()
        raw = (message.text or "").strip()
        if not raw.isdigit():
            await message.answer("❌ Invalid Telegram ID. Send numeric ID.")
            return
        new_admin_id = int(raw)

        if admin_action == "add":
            if new_admin_id in ADMIN_IDS:
                await message.answer("⚠️ Already a base admin (env).")
                return
            async with get_db() as db:
                exists = await (await db.execute(
                    "SELECT 1 FROM extra_admins WHERE user_id=?", (new_admin_id,)
                )).fetchone()
                if exists:
                    await message.answer("⚠️ Already an extra admin.")
                    return
                await db.execute(
                    "INSERT INTO extra_admins (user_id, added_by) VALUES (?, ?)",
                    (new_admin_id, message.from_user.id)
                )
                await db.commit()
            await log_admin_event(message.from_user.id, "ADD_ADMIN", str(new_admin_id), "Added")
            await message.answer(
                f"✅ <b>Admin Added</b>\n\n🆔 <code>{new_admin_id}</code>",
                parse_mode=ParseMode.HTML,
                reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                    [InlineKeyboardButton(text="🔐 Admin Manager", callback_data="aadmin_mgmt")]
                ])
            )
            try:
                await message.bot.send_message(
                    new_admin_id,
                    "🎉 <b>You've been promoted to Admin!</b>\n\nUse /admin to access the admin panel.",
                    parse_mode=ParseMode.HTML
                )
            except Exception:
                pass
        else:  # remove
            if new_admin_id in ADMIN_IDS:
                await message.answer("❌ Cannot remove base admin (env).")
                return
            async with get_db() as db:
                r = await (await db.execute(
                    "SELECT 1 FROM extra_admins WHERE user_id=?", (new_admin_id,)
                )).fetchone()
                if not r:
                    await message.answer("❌ Not an extra admin.")
                    return
                await db.execute("DELETE FROM extra_admins WHERE user_id=?", (new_admin_id,))
                await db.commit()
            await log_admin_event(message.from_user.id, "REMOVE_ADMIN", str(new_admin_id), "Removed")
            await message.answer(
                f"✅ <b>Admin Removed</b>\n\n🆔 <code>{new_admin_id}</code>",
                parse_mode=ParseMode.HTML,
                reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                    [InlineKeyboardButton(text="🔐 Admin Manager", callback_data="aadmin_mgmt")]
                ])
            )
        return

    key = data.get("setting_key")
    if not key:
        await state.clear()
        return
    value = (message.text or "").strip()

    if key in TOGGLE_SETTING_KEYS:
        norm = _normalize_flag(value)
        if norm is None:
            await message.answer("❌ Send <b>1</b>/<b>on</b> to enable or <b>0</b>/<b>off</b> to disable.", parse_mode=ParseMode.HTML)
            return
        value = norm

    # Special validation for discount
    if key == "GLOBAL_DISCOUNT_PERCENT":
        try:
            disc = Decimal(value)
            if disc < 0 or disc > 100:
                raise ValueError
        except Exception:
            await message.answer("❌ Invalid discount. Enter 0-100.")
            return
        markup = Decimal(await get_setting("GLOBAL_MARKUP_PERCENT", "20.00"))
        if disc >= markup:
            await message.answer(
                f"❌ <b>Invalid Discount!</b>\n\n"
                f"📈 Markup: <b>{markup}%</b>\n"
                f"🎁 Discount: <b>{disc}%</b>\n\n"
                f"⚠️ Discount must be LESS than markup, "
                f"otherwise admin profit becomes <b>negative</b>.\n\n"
                f"Max allowed: <b>{markup - Decimal('0.01')}%</b>",
                parse_mode=ParseMode.HTML
            )
            return
        net = markup - disc
        await set_setting(key, value)
        await state.clear()
        # Notify all admins
        for admin_id in await admin_alert_targets():
            try:
                await message.bot.send_message(
                    admin_id,
                    f"⚠️ <b>DISCOUNT CHANGED</b>\n\n"
                    f"🎁 New Discount: <b>{disc}%</b>\n"
                    f"📈 Markup: <b>{markup}%</b>\n"
                    f"💵 Net Profit: <b>{net}%</b>\n\n"
                    f"👑 By: <code>{message.from_user.id}</code>\n"
                    f"🕒 {get_ist_time()}",
                    parse_mode=ParseMode.HTML
                )
            except Exception:
                pass
        # Notify users
        asyncio.create_task(notify_users_discount_change(message.bot, disc, net))
        await message.answer(
            f"✅ <b>Discount Updated</b>\n\n"
            f"🎁 New Discount: <b>{disc}%</b>\n"
            f"💵 Net Admin Profit: <b>{net}%</b>\n\n"
            f"💡 Tap <b>Recalculate All Services</b> in Discount Manager to apply.",
            parse_mode=ParseMode.HTML
        )
        return

    # Special validation for markup
    if key == "GLOBAL_MARKUP_PERCENT":
        try:
            mk = Decimal(value)
            if mk < 0 or mk > 10000:
                raise ValueError
        except Exception:
            await message.answer("❌ Invalid markup. Enter 0-10000.")
            return
        disc = Decimal(await get_setting("GLOBAL_DISCOUNT_PERCENT", "0.00"))
        if mk <= disc:
            await message.answer(
                f"❌ <b>Invalid Markup!</b>\n\n"
                f"🎁 Current Discount: <b>{disc}%</b>\n"
                f"📈 New Markup: <b>{mk}%</b>\n\n"
                f"⚠️ Markup must be GREATER than discount, "
                f"otherwise admin profit becomes <b>negative</b>.\n\n"
                f"Min allowed: <b>{disc + Decimal('0.01')}%</b>",
                parse_mode=ParseMode.HTML
            )
            return
        await set_setting(key, value)
        await state.clear()

        # Always refresh provider prices first, then calculate from provider cost.
        stats = await refresh_provider_rates_and_reprice_all()
        net = mk - disc
        await message.answer(
            f"✅ <b>Global Markup Updated</b>\n\n"
            f"📈 New Markup: <b>{mk}%</b>\n"
            f"🎁 Discount: <b>{disc}%</b>\n"
            f"💵 Net Profit: <b>{net}%</b>\n\n"
            f"🔄 Provider rates refreshed: <b>{stats['updated_rates']}</b>\n"
            f"💰 Services repriced: <b>{stats['repriced']}</b>\n\n"
            f"<i>Every price is now calculated from the latest provider rate, not the old selling price.</i>",
            parse_mode=ParseMode.HTML
        )
        return

    # Standard validation
    numeric = {
        "USDT_MIN_DEPOSIT", "USDT_MAX_DEPOSIT",
        "MIN_DEPOSIT", "MAX_DEPOSIT", "REFERRAL_PERCENT"
    }
    integer = {"AUTO_SYNC_INTERVAL", "PROVIDER_SERVICE_CHECK_INTERVAL", "API_TIMEOUT", "API_RETRY_COUNT", "RATE_CACHE_MINUTES"}
    toggle = {"REFERRAL_ENABLED", "LIVE_RATE_ENABLED", "DISCOUNT_ENABLED"}
    try:
        if key in numeric:
            Decimal(value)
        elif key in integer:
            n = int(value)
            if n < 1:
                raise ValueError
            value = str(n)
        elif key in toggle:
            if value.lower() in {"on", "yes", "true", "1"}:
                value = "1"
            elif value.lower() in {"off", "no", "false", "0"}:
                value = "0"
            else:
                raise ValueError
        elif key == "CURRENCY_SYMBOL" and len(value) > 5:
            raise ValueError
    except Exception:
        await message.answer("❌ Invalid value. Try again.")
        return
    await set_setting(key, value)
    await state.clear()
    await message.answer(
        f"✅ <b>{html.escape(key)}</b> updated to <code>{html.escape(value)}</code>.",
        parse_mode=ParseMode.HTML
    )


# ---------------------------------------------------------------------------
# 🗑️ FULL RESET — Wipe everything except settings
# ---------------------------------------------------------------------------
@router.callback_query(F.data == "areset_menu", StateFilter("*"))
async def cb_admin_reset_menu(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    await safe_message_edit(
        callback.message,
        "⚠️ <b>FULL RESET — DANGER ZONE</b>\n\n"
        "This will <b>PERMANENTLY DELETE</b>:\n\n"
        "• 👥 All users (except admins)\n"
        "• 📦 All orders\n"
        "• 💰 All deposits & withdrawals\n"
        "• 💳 All transactions & refunds\n"
        "• 🎁 All gift cards & redemptions\n"
        "• ♻️ All refills\n"
        "• 📊 All admin logs\n"
        "• 🛍️ All services & categories\n"
        "• 🔌 All providers\n\n"
        "<b>Settings will be kept.</b>\n\n"
        "The bot will become <b>fresh like new</b>.\n\n"
        "Type <code>CONFIRM RESET</code> to proceed:",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="❌ Cancel", callback_data="ahub_back")]
        ]),
        parse_mode=ParseMode.HTML
    )
    await state.set_state(AdminResetFlow.confirm_reset)
    await callback.answer()


@router.message(AdminResetFlow.confirm_reset)
async def handle_admin_reset_confirm(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    text = (message.text or "").strip().upper()
    if text != "CONFIRM RESET":
        await message.answer(
            "❌ Reset cancelled. You must type exactly <code>CONFIRM RESET</code>.",
            parse_mode=ParseMode.HTML
        )
        await state.clear()
        return
    await state.clear()
    wait = await message.answer("🗑️ <b>Resetting bot...</b>", parse_mode=ParseMode.HTML)
    try:
        async with get_db() as db:
            await db.execute("DELETE FROM transactions")
            await db.execute("DELETE FROM refunds")
            await db.execute("DELETE FROM refills")
            await db.execute("DELETE FROM orders")
            await db.execute("DELETE FROM deposits")
            await db.execute("DELETE FROM withdrawals")
            await db.execute("DELETE FROM gift_card_redemptions")
            await db.execute("DELETE FROM gift_cards")
            await db.execute("DELETE FROM admin_logs")
            if ADMIN_IDS:
                placeholders = ",".join("?" * len(ADMIN_IDS))
                await db.execute(
                    f"DELETE FROM users WHERE user_id NOT IN ({placeholders})",
                    ADMIN_IDS
                )
            await db.execute("DELETE FROM services")
            await db.execute("DELETE FROM categories")
            await db.execute("DELETE FROM providers")
            for admin_id in ADMIN_IDS:
                await db.execute(
                    "UPDATE users SET balance='0.00', total_spent='0.00', "
                    "total_deposited='0.00', referral_earnings='0.00' "
                    "WHERE user_id=?",
                    (admin_id,)
                )
            await db.commit()
        # Re-seed default categories
        async with get_db() as db:
            sample_cats = [
                ("Instagram Followers", "👥", 1),
                ("Instagram Likes & Views", "❤️", 2),
                ("YouTube Services", "📹", 3),
                ("Telegram Members", "✈️", 4)
            ]
            await db.executemany(
                "INSERT INTO categories (name, icon, display_order) VALUES (?, ?, ?)",
                sample_cats
            )
            await db.commit()
        try:
            await wait.edit_text(
                "✅ <b>FULL RESET COMPLETE</b>\n\n"
                "🤖 Bot is now fresh like new.\n"
                "👥 All users removed (admins kept)\n"
                "📦 All orders/services/providers wiped\n"
                "📂 Default categories re-seeded\n"
                "⚙️ Settings preserved\n\n"
                "<i>Ready for fresh start!</i>",
                parse_mode=ParseMode.HTML,
                reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                    [InlineKeyboardButton(text="👑 Admin Panel", callback_data="ahub_back")]
                ])
            )
        except Exception:
            await message.answer("✅ Full reset complete!")
        await log_admin_event(message.from_user.id, "FULL_RESET", "ALL", "Bot wiped")
        logger.warning(f"⚠️ FULL RESET by admin {message.from_user.id}")
    except Exception as e:
        logger.error(f"Full reset failed: {e}", exc_info=True)
        try:
            await wait.edit_text(f"❌ Reset failed: {html.escape(str(e))}")
        except Exception:
            await message.answer(f"❌ Reset failed: {e}")


# ---------------------------------------------------------------------------
# 📢 BROADCAST
# ---------------------------------------------------------------------------
@router.callback_query(F.data == "ahub_broadcast", StateFilter("*"))
async def admin_broadcast_start(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    await state.clear()
    await state.set_state(AdminBroadcastFlow.enter_message)
    await callback.message.answer(
        "📢 <b>Broadcast</b>\n\nSend the message to broadcast to all users.",
        parse_mode=ParseMode.HTML,
        reply_markup=get_back_inline("ahub_back")
    )
    await callback.answer()


@router.message(AdminBroadcastFlow.enter_message)
async def admin_broadcast_send(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    if not (message.text or message.caption or message.photo or message.video or message.document or message.animation or message.audio):
        await message.answer("❌ Send a text or supported media message.")
        return
    await state.clear()
    async with get_db() as db:
        rows = await (await db.execute("SELECT user_id FROM users WHERE is_banned = 0")).fetchall()
    sent = failed = 0
    status_msg = await message.answer(f"📢 Broadcasting to {len(rows)} users...")
    for i, row in enumerate(rows):
        try:
            await message.copy_to(chat_id=row["user_id"])
            sent += 1
        except Exception:
            failed += 1
        await asyncio.sleep(0.05)
        if (i + 1) % 25 == 0:
            try:
                await status_msg.edit_text(f"📢 Progress: {i+1}/{len(rows)}\n✅ {sent} | ❌ {failed}")
            except Exception:
                pass
    await status_msg.edit_text(
        f"✅ <b>Broadcast Complete</b>\n\n"
        f"👥 Total: {len(rows)}\n"
        f"✅ Sent: {sent}\n"
        f"❌ Failed: {failed}",
        parse_mode=ParseMode.HTML
    )


# ---------------------------------------------------------------------------
# 🛠 MAINTENANCE
# ---------------------------------------------------------------------------
@router.callback_query(F.data == "ahub_maint", StateFilter("*"))
async def cb_admin_maint_menu(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    full_m = await get_setting("MAINTENANCE_MODE", "0")
    order_m = await get_setting("MAINT_NEW_ORDER", "0")
    dep_m = await get_setting("MAINT_ADD_BALANCE", "0")
    srv_m = await get_setting("MAINT_SERVICE", "0")
    def fmt(v): return "🔴 Active" if _flag_on(v) else "🟢 Inactive"
    text = (
        "🛠️ <b>MAINTENANCE CONTROL</b>\n\n"
        f"• Full Bot: <b>{fmt(full_m)}</b>\n"
        f"• New Orders: <b>{fmt(order_m)}</b>\n"
        f"• Add Funds: <b>{fmt(dep_m)}</b>\n"
        f"• Services: <b>{fmt(srv_m)}</b>\n\n"
        "Tap to toggle:"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🟢 Turn ALL maintenance OFF", callback_data="atog_maint_ALLOFF")],
        [InlineKeyboardButton(text="🔄 Toggle Full Maintenance", callback_data="atog_maint_MAINTENANCE_MODE")],
        [InlineKeyboardButton(text="🔄 Toggle Orders", callback_data="atog_maint_MAINT_NEW_ORDER"),
         InlineKeyboardButton(text="🔄 Toggle Add Funds", callback_data="atog_maint_MAINT_ADD_BALANCE")],
        [InlineKeyboardButton(text="🔄 Toggle Services", callback_data="atog_maint_MAINT_SERVICE")],
        [InlineKeyboardButton(text="⬅️ Back", callback_data="ahub_back")]
    ])
    await safe_message_edit(callback.message, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("atog_maint_"), StateFilter("*"))
async def cb_admin_toggle_maint_key(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    key = callback.data.replace("atog_maint_", "")
    all_keys = ("MAINTENANCE_MODE", "MAINT_NEW_ORDER", "MAINT_ADD_BALANCE", "MAINT_SERVICE")
    if key == "ALLOFF":
        for k in all_keys:
            await set_setting(k, "0")
        await log_admin_event(callback.from_user.id, "MAINT_TOGGLE", "ALL", "→ 0")
        await callback.answer("🟢 All maintenance is now OFF")
        await cb_admin_maint_menu(callback)
        return
    if key not in all_keys:
        await callback.answer("Unknown setting.", show_alert=True)
        return
    cur_val = await get_setting(key, "0")
    new_val = "0" if _flag_on(cur_val) else "1"
    await set_setting(key, new_val)
    await log_admin_event(callback.from_user.id, "MAINT_TOGGLE", key, f"→ {new_val}")
    await callback.answer("🔴 Maintenance is now ON" if new_val == "1" else "🟢 Maintenance is now OFF")
    await cb_admin_maint_menu(callback)


# ---------------------------------------------------------------------------
# 🚨 REPORTS
# ---------------------------------------------------------------------------
@router.callback_query(F.data == "arep_menu", StateFilter("*"))
async def admin_reports(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    async with get_db() as db:
        total_orders = (await (await db.execute("SELECT COUNT(*) FROM orders")).fetchone())[0]
        pending = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE status='Pending'")).fetchone())[0]
        processing = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE status IN ('Processing','In Progress')")).fetchone())[0]
        completed = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE status='Completed'")).fetchone())[0]
        partial = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE status='Partial'")).fetchone())[0]
        cancelled = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE status='Cancelled'")).fetchone())[0]
        failed = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE status='Failed'")).fetchone())[0]
        revenue = (await (await db.execute("SELECT COALESCE(SUM(CAST(charge AS REAL)),0) FROM orders")).fetchone())[0] or 0
        approved_dep = (await (await db.execute("SELECT COALESCE(SUM(CAST(amount AS REAL)),0) FROM deposits WHERE status='APPROVED'")).fetchone())[0] or 0
        pending_dep = (await (await db.execute("SELECT COUNT(*) FROM deposits WHERE status='PENDING'")).fetchone())[0]
        users = (await (await db.execute("SELECT COUNT(*) FROM users")).fetchone())[0]
    text = (
        "🚨 <b>SMM REPORTS</b>\n\n"
        f"👥 Total Users: <b>{users:,}</b>\n\n"
        "📦 <b>ORDER REPORT</b>\n"
        f"• Total: <b>{total_orders:,}</b>\n"
        f"• 🟡 Pending: <b>{pending}</b>\n"
        f"• 🔵 Processing: <b>{processing}</b>\n"
        f"• 🟢 Completed: <b>{completed}</b>\n"
        f"• 🟠 Partial: <b>{partial}</b>\n"
        f"• 🔴 Cancelled: <b>{cancelled}</b>\n"
        f"• ❌ Failed: <b>{failed}</b>\n\n"
        "💰 <b>PAYMENT REPORT</b>\n"
        f"• Revenue: <b>₹{revenue:,.2f}</b>\n"
        f"• Approved Deposits: <b>₹{approved_dep:,.2f}</b>\n"
        f"• ⏳ Pending Deposits: <b>{pending_dep}</b>"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔄 Refresh", callback_data="arep_menu"),
         InlineKeyboardButton(text="⬅️ Back", callback_data="ahub_back")]
    ])
    await safe_message_edit(callback.message, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()


# ---------------------------------------------------------------------------
# ♻️ REFILLS MENU
# ---------------------------------------------------------------------------
@router.callback_query(F.data == "arefill_menu", StateFilter("*"))
async def admin_refill_menu(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    async with get_db() as db:
        total = (await (await db.execute("SELECT COUNT(*) FROM refills")).fetchone())[0]
        pending = (await (await db.execute("SELECT COUNT(*) FROM refills WHERE status='Pending'")).fetchone())[0]
        completed = (await (await db.execute("SELECT COUNT(*) FROM refills WHERE status='Completed'")).fetchone())[0]
        failed = (await (await db.execute("SELECT COUNT(*) FROM refills WHERE status='Failed'")).fetchone())[0]
    text = (
        "♻️ <b>REFILL MANAGEMENT</b>\n\n"
        f"📊 Total: <b>{total}</b>\n"
        f"⏳ Pending: <b>{pending}</b>\n"
        f"🟢 Completed: <b>{completed}</b>\n"
        f"❌ Failed: <b>{failed}</b>"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔄 Refresh", callback_data="arefill_menu")],
        [InlineKeyboardButton(text="⬅️ Back", callback_data="ahub_back")]
    ])
    await safe_message_edit(callback.message, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()


# ---------------------------------------------------------------------------
# 🔌 PROVIDERS
# ---------------------------------------------------------------------------
@router.callback_query(F.data == "ahub_prov_p_1", StateFilter("*"))
async def admin_providers(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    async with get_db() as db:
        rows = await (await db.execute("SELECT id, name, status FROM providers ORDER BY id ASC")).fetchall()
    lines = ["🔌 <b>PROVIDERS</b>\n"]
    if rows:
        for r in rows:
            state = "🟢" if r["status"] else "🔴"
            lines.append(f"{state} <b>#{r['id']}</b> {html.escape(r['name'])}")
    else:
        lines.append("No providers configured.")
    buttons = []
    for r in rows:
        buttons.append([InlineKeyboardButton(
            text=f"🔌 #{r['id']} {r['name'][:22]}",
            callback_data=f"aprov_view_{r['id']}"
        )])
    buttons.append([InlineKeyboardButton(text="➕ Add Provider", callback_data="aprov_add")])
    buttons.append([InlineKeyboardButton(text="⬅️ Back", callback_data="ahub_back")])
    await safe_message_edit(
        callback.message, "\n".join(lines),
        reply_markup=InlineKeyboardMarkup(inline_keyboard=buttons),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data == "aprov_add", StateFilter("*"))
async def admin_provider_add_start(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    await state.clear()
    await state.set_state(AdminProviderAddFlow.enter_name)
    await callback.message.answer("🔌 <b>Add Provider</b>\n\nEnter provider name:", parse_mode=ParseMode.HTML)
    await callback.answer()


@router.message(AdminProviderAddFlow.enter_name)
async def admin_provider_add_name(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    name = (message.text or "").strip()
    if not name or len(name) > 80:
        await message.answer("❌ Invalid name (1-80 chars).")
        return
    await state.update_data(provider_name=name)
    await state.set_state(AdminProviderAddFlow.enter_url)
    await message.answer("🌐 Enter API URL:\n\nExample: <code>https://example.com/api/v2</code>", parse_mode=ParseMode.HTML)


@router.message(AdminProviderAddFlow.enter_url)
async def admin_provider_add_url(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    url = (message.text or "").strip()
    if not url.startswith("https://") or len(url) > 500:
        await message.answer("❌ Invalid URL. Must start with https://")
        return
    await state.update_data(provider_url=url)
    await state.set_state(AdminProviderAddFlow.enter_key)
    await message.answer("🔑 Enter API key:")


@router.message(AdminProviderAddFlow.enter_key)
async def admin_provider_add_key(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    api_key = (message.text or "").strip()
    data = await state.get_data()
    name = data.get("provider_name", "").strip()
    url = data.get("provider_url", "").strip()
    if not api_key:
        await message.answer("❌ API key cannot be empty.")
        return
    async with get_db() as db:
        cur = await db.execute(
            "INSERT INTO providers (name, api_url, api_key, status) VALUES (?, ?, ?, 1)",
            (name, url, api_key)
        )
        provider_id = cur.lastrowid
        await db.commit()
    await state.clear()
    await message.answer(
        f"✅ <b>Provider Added</b>\n\n🆔 ID: <code>{provider_id}</code>\n🔌 Name: <b>{html.escape(name)}</b>\n🌐 URL: <code>{html.escape(url)}</code>\n🟢 Status: Active",
        parse_mode=ParseMode.HTML
    )


@router.callback_query(F.data.startswith("aprov_view_"), StateFilter("*"))
async def admin_provider_view(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    try:
        pid = int(callback.data.rsplit("_", 1)[1])
    except (ValueError, IndexError):
        await callback.answer("Invalid provider.", show_alert=True)
        return
    async with get_db() as db:
        p = await (await db.execute("SELECT * FROM providers WHERE id = ?", (pid,))).fetchone()
        svc_count = (await (await db.execute("SELECT COUNT(*) FROM services WHERE provider_id = ?", (pid,))).fetchone())[0] if p else 0
    if not p:
        await callback.answer("Provider not found.", show_alert=True)
        return
    status = "🟢 Active" if p["status"] else "🔴 Disabled"
    text = (
        "🔌 <b>PROVIDER DETAILS</b>\n\n"
        f"🆔 ID: <code>{p['id']}</code>\n"
        f"📛 Name: <b>{html.escape(p['name'])}</b>\n"
        f"🌐 URL: <code>{html.escape(p['api_url'])}</code>\n"
        f"🔑 Key: <code>{mask_api_key(p['api_key'])}</code>\n"
        f"📊 Status: <b>{status}</b>\n"
        f"🛍️ Services: <b>{svc_count}</b>"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="💰 Check Balance", callback_data=f"aprov_balance_{pid}"),
         InlineKeyboardButton(text="🧪 Test API", callback_data=f"aprov_test_{pid}")],
        [InlineKeyboardButton(text="🔁 Toggle", callback_data=f"aprov_toggle_{pid}"),
         InlineKeyboardButton(text="🗑️ Delete", callback_data=f"aprov_delete_{pid}")],
        [InlineKeyboardButton(text="⬅️ Providers", callback_data="ahub_prov_p_1")]
    ])
    await safe_message_edit(callback.message, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("aprov_toggle_"), StateFilter("*"))
async def admin_provider_toggle(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    pid = int(callback.data.rsplit("_", 1)[1])
    async with get_db() as db:
        p = await (await db.execute("SELECT status FROM providers WHERE id = ?", (pid,))).fetchone()
        if not p:
            await callback.answer("Provider not found.", show_alert=True)
            return
        await db.execute("UPDATE providers SET status = ? WHERE id = ?", (0 if p["status"] else 1, pid))
        await db.commit()
    callback.data = f"aprov_view_{pid}"
    await admin_provider_view(callback)


@router.callback_query(F.data.startswith("aprov_delete_yes_"), StateFilter("*"))
async def admin_provider_delete_execute(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    pid = int(callback.data.rsplit("_", 1)[1])
    async with get_db() as db:
        linked = (await (await db.execute("SELECT COUNT(*) FROM services WHERE provider_id = ?", (pid,))).fetchone())[0]
        # Historical order snapshots remain intact; remove current catalog entries with the provider.
        if linked:
            await db.execute("DELETE FROM services WHERE provider_id = ?", (pid,))
        await db.execute("DELETE FROM providers WHERE id = ?", (pid,))
        await db.commit()
    await callback.answer("Provider deleted.", show_alert=True)
    await admin_providers(callback)


@router.callback_query(F.data.startswith("aprov_delete_"), StateFilter("*"))
async def admin_provider_delete_confirm(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    if "delete_yes_" in callback.data:
        return
    pid = int(callback.data.rsplit("_", 1)[1])
    async with get_db() as db:
        p = await (await db.execute("SELECT name FROM providers WHERE id = ?", (pid,))).fetchone()
        linked = (await (await db.execute("SELECT COUNT(*) FROM services WHERE provider_id = ?", (pid,))).fetchone())[0]
    if not p:
        await callback.answer("Provider not found.", show_alert=True)
        return
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="⚠️ Yes, Delete", callback_data=f"aprov_delete_yes_{pid}")],
        [InlineKeyboardButton(text="⬅️ Cancel", callback_data=f"aprov_view_{pid}")]
    ])
    await safe_message_edit(
        callback.message,
        f"⚠️ <b>DELETE PROVIDER?</b>\n\n🔌 <b>{html.escape(p['name'])}</b>\n\n⚠️ Linked current services: <b>{linked}</b>\nThey will also be removed; historical orders keep their snapshots.\n\nCannot be undone.",
        reply_markup=kb, parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data.startswith("aprov_balance_"), StateFilter("*"))
async def admin_provider_balance(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    pid = int(callback.data.rsplit("_", 1)[1])
    res = await ProviderAPIClient.get_balance(pid)
    if isinstance(res, dict) and not res.get("error"):
        bal = res.get("balance", "N/A")
        currency = res.get("currency", "")
        await callback.answer(f"Balance: {bal} {currency}".strip(), show_alert=True)
    else:
        err = res.get("error", "Unknown") if isinstance(res, dict) else "Invalid"
        await callback.answer(f"❌ {str(err)[:180]}", show_alert=True)


@router.callback_query(F.data.startswith("aprov_test_"), StateFilter("*"))
async def admin_provider_test(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    pid = int(callback.data.rsplit("_", 1)[1])
    res = await ProviderAPIClient.get_balance(pid)
    if isinstance(res, dict) and not res.get("error"):
        await callback.answer("✅ API connected.", show_alert=True)
    else:
        await callback.answer("❌ API connection failed.", show_alert=True)


# ---------------------------------------------------------------------------
# 🎁 GIFT CARDS
# ---------------------------------------------------------------------------
@router.callback_query(F.data == "ahub_gc_p_1", StateFilter("*"))
async def admin_gift_cards(callback:CallbackQuery):
    if not await is_admin(callback.from_user.id): return
    async with get_db() as db: rows=await (await db.execute("SELECT code,amount,max_uses,used_count,is_active FROM gift_cards ORDER BY created_at DESC LIMIT 30")).fetchall()
    text="🎁 <b>GIFT CARD MANAGEMENT</b>\n\n"+("No gift cards found." if not rows else "".join(f"{'🟢' if r['is_active'] else '🔴'} <code>{html.escape(r['code'])}</code> — ₹{r['amount']} — {r['used_count']}/{r['max_uses']}\n" for r in rows))
    kb=[[InlineKeyboardButton(text=f"{'🟢' if r['is_active'] else '🔴'} {r['code'][:18]}",callback_data=f"agift_view_{r['code']}")] for r in rows]
    kb += [[InlineKeyboardButton(text="➕ Create Gift Card",callback_data="agift_add")],[InlineKeyboardButton(text="⬅️ Back",callback_data="ahub_back")]]
    await safe_message_edit(callback.message,text,reply_markup=InlineKeyboardMarkup(inline_keyboard=kb),parse_mode=ParseMode.HTML); await callback.answer()

@router.callback_query(F.data.startswith("agift_view_"),StateFilter("*"))
async def admin_gift_view(callback:CallbackQuery):
    if not await is_admin(callback.from_user.id): return
    code=callback.data[len("agift_view_"):]
    async with get_db() as db:r=await (await db.execute("SELECT * FROM gift_cards WHERE code=?",(code,))).fetchone()
    if not r: await callback.answer("Not found.",show_alert=True); return
    kb=InlineKeyboardMarkup(inline_keyboard=[
      [InlineKeyboardButton(text=f"{'🔴 Disable' if r['is_active'] else '🟢 Enable'}",callback_data=f"agift_toggle_{code}")],
      [InlineKeyboardButton(text="✏️ Edit",callback_data=f"agift_edit_{code}")],
      [InlineKeyboardButton(text="🗑️ Delete",callback_data=f"agift_delete_{code}")],
      [InlineKeyboardButton(text="⬅️ Gift Cards",callback_data="ahub_gc_p_1")]])
    await safe_message_edit(callback.message,f"🎁 <b>GIFT CARD</b>\n\n🔑 <code>{html.escape(code)}</code>\n💰 ₹{r['amount']}\n🔢 {r['used_count']}/{r['max_uses']}\n📊 {'🟢 ON' if r['is_active'] else '🔴 OFF'}",reply_markup=kb,parse_mode=ParseMode.HTML); await callback.answer()

@router.callback_query(F.data.startswith("agift_toggle_"),StateFilter("*"))
async def admin_gift_toggle(callback:CallbackQuery):
    if not await is_admin(callback.from_user.id): return
    code=callback.data[len("agift_toggle_"):]
    async with get_db() as db:
        r=await (await db.execute("SELECT is_active FROM gift_cards WHERE code=?",(code,))).fetchone()
        if not r: await callback.answer("Not found.",show_alert=True); return
        await db.execute("UPDATE gift_cards SET is_active=? WHERE code=?",(0 if r["is_active"] else 1,code)); await db.commit()
    await log_admin_event(callback.from_user.id,"TOGGLE_GIFT_CARD",code,""); await send_channel_log(callback.bot,"PAYMENT",f"🎁 Gift card <code>{html.escape(code)}</code> toggled")
    callback.data=f"agift_view_{code}"; await admin_gift_view(callback)

@router.callback_query(F.data.startswith("agift_delete_"),StateFilter("*"))
async def admin_gift_delete(callback:CallbackQuery):
    if not await is_admin(callback.from_user.id): return
    code=callback.data[len("agift_delete_"):]
    await safe_message_edit(callback.message,f"⚠️ <b>Delete gift card?</b>\n\n<code>{html.escape(code)}</code>",reply_markup=InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text="⚠️ Yes, Delete",callback_data=f"agift_delete_yes_{code}")],[InlineKeyboardButton(text="⬅️ Cancel",callback_data=f"agift_view_{code}")]]),parse_mode=ParseMode.HTML); await callback.answer()

@router.callback_query(F.data.startswith("agift_delete_yes_"),StateFilter("*"))
async def admin_gift_delete_yes(callback:CallbackQuery):
    if not await is_admin(callback.from_user.id): return
    code=callback.data[len("agift_delete_yes_"):]
    async with get_db() as db: await db.execute("DELETE FROM gift_cards WHERE code=?",(code,)); await db.commit()
    await log_admin_event(callback.from_user.id,"DELETE_GIFT_CARD",code,""); await send_channel_log(callback.bot,"PAYMENT",f"🗑️ Gift card <code>{html.escape(code)}</code> deleted")
    callback.data="ahub_gc_p_1"; await admin_gift_cards(callback)

@router.callback_query(F.data.startswith("agift_edit_"),StateFilter("*"))
async def admin_gift_edit_start(callback:CallbackQuery,state:FSMContext):
    if not await is_admin(callback.from_user.id): return
    code=callback.data[len("agift_edit_"):]
    async with get_db() as db:r=await (await db.execute("SELECT amount,max_uses FROM gift_cards WHERE code=?",(code,))).fetchone()
    if not r: await callback.answer("Not found.",show_alert=True); return
    await state.update_data(gift_edit_code=code); await state.set_state(AdminGiftCardFlow.edit_amount)
    await callback.message.answer(f"✏️ <b>Edit {html.escape(code)}</b>\nCurrent amount: ₹{r['amount']}\nSend new amount:",parse_mode=ParseMode.HTML); await callback.answer()

@router.message(AdminGiftCardFlow.edit_amount)
async def admin_gift_edit_amount(message:Message,state:FSMContext):
    if not await is_admin(message.from_user.id): return
    try:v=quantize_currency((message.text or "").strip())
    except: await message.answer("❌ Invalid amount."); return
    if v<=0: await message.answer("❌ Amount must be > 0."); return
    await state.update_data(edit_amount=str(v)); await state.set_state(AdminGiftCardFlow.edit_uses); await message.answer("🔢 Send new maximum uses:")

@router.message(AdminGiftCardFlow.edit_uses)
async def admin_gift_edit_uses(message:Message,state:FSMContext):
    if not await is_admin(message.from_user.id): return
    try:uses=int((message.text or "").strip())
    except: await message.answer("❌ Invalid uses."); return
    d=await state.get_data(); code=d.get("gift_edit_code")
    async with get_db() as db:
        row=await (await db.execute("SELECT used_count FROM gift_cards WHERE code=?",(code,))).fetchone()
        if not row or uses<row["used_count"]: await message.answer("❌ Uses cannot be below current used count."); return
        await db.execute("UPDATE gift_cards SET amount=?,max_uses=? WHERE code=?",(d["edit_amount"],uses,code)); await db.commit()
    await state.clear(); await log_admin_event(message.from_user.id,"EDIT_GIFT_CARD",code,f"₹{d['edit_amount']} / {uses}")
    await send_channel_log(message.bot,"PAYMENT",f"✏️ <b>Gift Card Edited</b>\n• Code: <code>{html.escape(code)}</code>\n• Amount: ₹{d['edit_amount']}\n• Uses: {uses}")
    await message.answer("✅ Gift card updated.",reply_markup=InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text="🎁 Gift Cards",callback_data="ahub_gc_p_1")]]))

@router.callback_query(F.data == "agift_add", StateFilter("*"))
async def admin_gift_add_start(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    await state.clear()
    await state.set_state(AdminGiftCardFlow.enter_code)
    await callback.message.answer("🎁 Enter gift card code:")
    await callback.answer()


@router.message(AdminGiftCardFlow.enter_code)
async def admin_gift_code(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    code = (message.text or "").strip().upper()
    if not code or len(code) > 50:
        await message.answer("❌ Invalid code.")
        return
    async with get_db() as db:
        exists = await (await db.execute("SELECT 1 FROM gift_cards WHERE code=?", (code,))).fetchone()
    if exists:
        await message.answer("❌ Code already exists.")
        return
    await state.update_data(code=code)
    await state.set_state(AdminGiftCardFlow.enter_amount)
    await message.answer("💰 Enter gift card amount:")


@router.message(AdminGiftCardFlow.enter_amount)
async def admin_gift_amount(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    try:
        amt = Decimal((message.text or "").strip())
    except Exception:
        await message.answer("❌ Invalid amount.")
        return
    if amt <= 0:
        await message.answer("❌ Amount must be > 0.")
        return
    await state.update_data(amount=str(quantize_currency(amt)))
    await state.set_state(AdminGiftCardFlow.enter_uses)
    await message.answer("🔢 Enter maximum uses (1 = single use):")


@router.message(AdminGiftCardFlow.enter_uses)
async def admin_gift_uses(message: Message, state: FSMContext):
    if not await is_admin(message.from_user.id):
        return
    try:
        uses = int((message.text or "").strip())
    except Exception:
        await message.answer("❌ Invalid uses.")
        return
    if uses < 1 or uses > 100000:
        await message.answer("❌ Uses must be 1-100000.")
        return
    d = await state.get_data()
    try:
        async with get_db() as db:
            await db.execute(
                "INSERT INTO gift_cards(code, amount, max_uses, used_count, is_active) VALUES(?, ?, ?, 0, 1)",
                (d["code"], d["amount"], uses)
            )
            await db.commit()
    except Exception as e:
        logger.warning(f"Gift card create failed: {e}")
        await message.answer("❌ Could not create.")
        return
    await state.clear()
    await log_admin_event(message.from_user.id, "CREATE_GIFT_CARD", d["code"], f"₹{d['amount']} / {uses}")
    await send_channel_log(message.bot,"PAYMENT",f"🎁 <b>Gift Card Created</b>\n• Code: <code>{html.escape(d['code'])}</code>\n• Amount: ₹{d['amount']}\n• Uses: {uses}\n• Admin: <code>{message.from_user.id}</code>")
    await message.answer(
        f"🎉 <b>Gift Card Created</b>\n\n"
        f"🔑 Code: <code>{html.escape(d['code'])}</code>\n"
        f"💰 Amount: <b>₹{d['amount']}</b>\n"
        f"🔢 Max Uses: <b>{uses}</b>",
        parse_mode=ParseMode.HTML,
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🎁 Gift Cards", callback_data="ahub_gc_p_1")]
        ])
    )


@router.callback_query(F.data.startswith("agift_toggle_"), StateFilter("*"))
async def admin_gift_toggle(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    code = callback.data[len("agift_toggle_"):]
    async with get_db() as db:
        r = await (await db.execute("SELECT is_active FROM gift_cards WHERE code=?", (code,))).fetchone()
        if not r:
            await callback.answer("Not found.", show_alert=True)
            return
        await db.execute(
            "UPDATE gift_cards SET is_active=? WHERE code=?",
            (0 if r["is_active"] else 1, code)
        )
        await db.commit()
    callback.data = "ahub_gc_p_1"
    await admin_gift_cards(callback)


# ---------------------------------------------------------------------------
# 📢 LOG CHANNEL
# ---------------------------------------------------------------------------
@router.callback_query(F.data == "alog_channel_cfg", StateFilter("*"))
async def admin_log_channel(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    enabled = await get_setting("LOG_CHANNEL_ENABLED", "0")
    cid = await get_setting("LOG_CHANNEL_ID", "")
    text = (
        "📢 <b>LOG CHANNEL</b>\n\n"
        f"Status: <b>{'🟢 ON' if enabled == '1' else '🔴 OFF'}</b>\n"
        f"Channel: <code>{html.escape(cid or 'Not set')}</code>"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔄 Toggle", callback_data="alog_toggle")],
        [InlineKeyboardButton(text="✏️ Set Channel ID", callback_data="alog_set_id")],
        [InlineKeyboardButton(text="⬅️ Back", callback_data="ahub_back")]
    ])
    await safe_message_edit(callback.message, text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data == "alog_toggle", StateFilter("*"))
async def admin_log_toggle(callback: CallbackQuery):
    if not await is_admin(callback.from_user.id):
        return
    cur = await get_setting("LOG_CHANNEL_ENABLED", "0")
    await set_setting("LOG_CHANNEL_ENABLED", "0" if cur == "1" else "1")
    await callback.answer("Toggled!")
    await admin_log_channel(callback)


@router.callback_query(F.data == "alog_set_id", StateFilter("*"))
async def admin_log_set_id(callback: CallbackQuery, state: FSMContext):
    if not await is_admin(callback.from_user.id):
        return
    await state.update_data(setting_key="LOG_CHANNEL_ID")
    await state.set_state(AdminSettingsFlow.enter_value)
    await callback.message.answer(
        "📢 Enter channel ID (e.g. <code>-1001234567890</code>):",
        parse_mode=ParseMode.HTML,
        reply_markup=get_back_inline("alog_channel_cfg")
    )
    await callback.answer()


# ===========================================================================
# BACKGROUND WORKERS
# ===========================================================================
async def background_sync_daemon(bot: Bot):
    while True:
        try:
            interval = int(await get_setting("API_CHECK_INTERVAL",
                             await get_setting("AUTO_SYNC_INTERVAL", "120")))
            await asyncio.sleep(max(15, min(interval, 86400)))
            async with get_db() as db:
                active_orders = await (await db.execute(
                    """
                    SELECT id, user_id, provider_id, provider_order_id, status,
                           charge, quantity, refund_state, refunded_amount
                    FROM orders
                    WHERE status IN ('Pending', 'Processing', 'In Progress')
                      AND provider_order_id IS NOT NULL
                    """
                )).fetchall()
                notify_jobs = []
                for o in active_orders:
                    st_res = await ProviderAPIClient.get_order_status(
                        o["provider_id"], o["provider_order_id"]
                    )
                    if not (isinstance(st_res, dict) and "status" in st_res):
                        continue
                    raw_st = str(st_res.get("status", "")).strip().lower()
                    remains = int(st_res.get("remains", 0) or 0)
                    start_c = int(st_res.get("start_count", 0) or 0)
                    nst = "Processing"
                    if raw_st == "pending": nst = "Pending"
                    elif raw_st in ("in progress", "processing"): nst = "Processing"
                    elif raw_st == "completed": nst = "Completed"
                    elif raw_st == "partial": nst = "Partial"
                    elif raw_st in ("canceled", "cancelled"): nst = "Cancelled"
                    elif raw_st == "failed": nst = "Failed"
                    if nst == o["status"]:
                        continue
                    notify_refund = Decimal("0.00")
                    if nst == "Partial" and o["refund_state"] == "NOT_REFUNDED":
                        ord_qty = Decimal(str(o["quantity"]))
                        charge_amt = Decimal(str(o["charge"]))
                        rem_qty = Decimal(str(remains))
                        refund_val = quantize_currency((rem_qty / ord_qty) * charge_amt) if ord_qty > 0 else Decimal("0.00")
                        if refund_val > Decimal("0.00"):
                            notify_refund = refund_val
                            await modify_wallet_atomic(
                                db, o["user_id"], refund_val, "Partial Refund",
                                reference_id=str(o["id"]),
                                description=f"Auto Partial refund for Order #{o['id']}"
                            )
                            await db.execute(
                                "INSERT INTO refunds (order_id, user_id, amount, reason) VALUES (?, ?, ?, 'Auto Partial Refund')",
                                (o["id"], o["user_id"], str(refund_val))
                            )
                            await db.execute(
                                "UPDATE orders SET refund_state = 'PARTIALLY_REFUNDED', refunded_amount = ? WHERE id = ?",
                                (str(refund_val), o["id"])
                            )
                    elif nst in ("Cancelled", "Failed") and o["refund_state"] == "NOT_REFUNDED":
                        charge_amt = Decimal(str(o["charge"]))
                        notify_refund = charge_amt
                        await modify_wallet_atomic(
                            db, o["user_id"], charge_amt, "Full Refund",
                            reference_id=str(o["id"]),
                            description=f"Auto {nst} refund for Order #{o['id']}"
                        )
                        await db.execute(
                            "INSERT INTO refunds (order_id, user_id, amount, reason) VALUES (?, ?, ?, ?)",
                            (o["id"], o["user_id"], str(charge_amt), f"Auto Provider {nst}")
                        )
                        await db.execute(
                            "UPDATE orders SET refund_state = 'FULLY_REFUNDED', refunded_amount = ? WHERE id = ?",
                            (str(charge_amt), o["id"])
                        )
                    await db.execute(
                        "UPDATE orders SET status = ?, start_count = ?, remains = ?, updated_at = CURRENT_TIMESTAMP WHERE id = ?",
                        (nst, start_c, remains, o["id"])
                    )
                    notify_jobs.append((o["id"], o["user_id"], nst, notify_refund))
                await db.commit()
            for _oid, _uid, _nst, _refund in notify_jobs:
                try:
                    async with get_db() as ndb:
                        _row = await (await ndb.execute(
                            "SELECT service_name_snapshot FROM orders WHERE id = ?", (_oid,)
                        )).fetchone()
                    await notify_user_order_status_change(
                        bot, _oid, _uid,
                        (_row["service_name_snapshot"] if _row else "Service"),
                        _nst, _refund
                    )
                except Exception as _ne:
                    logger.warning(f"Status notify failed #{_oid}: {_ne}")
            # Check refills
            async with get_db() as db:
                active_refills = await (await db.execute(
                    "SELECT * FROM refills WHERE status IN ('Pending', 'Processing') AND provider_refill_id IS NOT NULL"
                )).fetchall()
                for r in active_refills:
                    rf_res = await ProviderAPIClient.get_refill_status(
                        r["provider_id"], r["provider_refill_id"]
                    )
                    if isinstance(rf_res, dict) and "status" in rf_res:
                        rst = str(rf_res.get("status", "")).capitalize()
                        if rst in ("Completed", "Success"):
                            await db.execute(
                                "UPDATE refills SET status = 'Completed', updated_at = CURRENT_TIMESTAMP WHERE id = ?",
                                (r["id"],)
                            )
                            await db.execute(
                                "UPDATE orders SET refill_status = 'Completed' WHERE id = ?",
                                (r["order_id"],)
                            )
                        elif rst in ("Rejected", "Failed", "Canceled"):
                            await db.execute(
                                "UPDATE refills SET status = 'Failed', updated_at = CURRENT_TIMESTAMP WHERE id = ?",
                                (r["id"],)
                            )
                            await db.execute(
                                "UPDATE orders SET refill_status = 'Failed' WHERE id = ?",
                                (r["order_id"],)
                            )
                await db.commit()
        except asyncio.CancelledError:
            break
        except Exception as e:
            logger.error(f"Background sync error: {e}", exc_info=True)


def _p_sid(x): return (x.get("service") or x.get("service_id") or x.get("id")) if isinstance(x, dict) else None
def _p_desc(x): return str((x or {}).get("description") or (x or {}).get("desc") or "").strip()
def _p_rate(x): return (x or {}).get("rate") if isinstance(x, dict) else None
def _p_min(x): return (x or {}).get("min", (x or {}).get("minimum", 0)) if isinstance(x, dict) else 0
def _p_max(x): return (x or {}).get("max", (x or {}).get("maximum", 0)) if isinstance(x, dict) else 0
def _p_refill(x):
    """Normalize provider refill data.

    Supports common provider formats:
    refill=true/yes, refill_days=30
    refill="lifetime", refill_days="lifetime"
    refill="30 days", refill_duration="30 days", etc.
    A refill service with no numeric duration is treated as Lifetime.
    """
    x = x or {}
    raw = x.get("refill")
    raw_days = x.get("refill_days", x.get("refill_duration", x.get("warranty", "")))

    truthy = raw is True or str(raw).strip().lower() in {
        "1", "true", "yes", "y", "on", "lifetime", "life", "forever"
    }
    duration = str(raw_days or "").strip()

    # Some APIs put the duration directly in refill.
    if not truthy and isinstance(raw, str):
        low = raw.strip().lower()
        if "lifetime" in low or low in {"life", "forever"}:
            truthy = True
            duration = "Lifetime"
        elif any(ch.isdigit() for ch in low):
            truthy = True
            duration = raw.strip()

    if not truthy:
        return 0, 0, ""

    if "lifetime" in duration.lower() or duration.lower() in {"life", "forever"}:
        return 1, 0, "Lifetime"

    m = re.search(r"(\d+)", duration)
    if m:
        days = int(m.group(1))
        return 1, days, f"{days} days"

    # refill=true with no duration => lifetime, not 0 days.
    return 1, 0, "Lifetime"


def refill_display(refill_available: Any, refill_days: Any = 0, refill_duration: str = "") -> str:
    if not int(refill_available or 0):
        return "❌ No"
    duration = str(refill_duration or "").strip()
    if duration:
        return f"✅ Yes ({html.escape(duration)})"
    try:
        days = int(refill_days or 0)
    except Exception:
        days = 0
    return f"✅ Yes ({days} days)" if days > 0 else "✅ Yes (Lifetime)"
def _p_cancel(x):
    return 1 if (x or {}).get("cancel") in [True, 1, "1", "true", "True", "yes", "Yes"] else 0


async def background_provider_service_monitor(bot: Bot):
    """Detect provider rate changes — reapply markup + discount."""
    while True:
        try:
            interval = int(await get_setting("PROVIDER_SERVICE_CHECK_INTERVAL", "60"))
            await asyncio.sleep(max(30, min(interval, 3600)))
            async with get_db() as db:
                providers = await (await db.execute(
                    "SELECT id, name, status FROM providers WHERE status=1 ORDER BY id"
                )).fetchall()
            for p in providers:
                try:
                    res = await ProviderAPIClient.fetch_services_list(p["id"])
                    if not res.get("success"):
                        continue
                    items = res.get("services") or []
                    catalog = {str(_p_sid(x)): x for x in items if _p_sid(x) is not None}
                    async with get_db() as db:
                        local = await (await db.execute(
                            "SELECT * FROM services WHERE provider_id=?", (p["id"],)
                        )).fetchall()
                        price_jobs = []
                        for svc in local:
                            psid = str(svc["provider_service_id"])
                            item = catalog.get(psid)
                            if item is None:
                                if svc["is_active"]:
                                    await db.execute("UPDATE services SET is_active=0 WHERE id=?", (svc["id"],))
                                    service_removed_text = (
                                        f"🚫 <b>Provider Service Removed</b>\n"
                                        f"• Local Service: #{svc['id']}\n"
                                        f"• Provider: #{p['id']} {html.escape(p['name'])}\n"
                                        f"• Action: <b>Disabled</b>"
                                    )
                                    await notify_admins(bot, service_removed_text)
                                    await send_channel_log(bot, "SERVICE", service_removed_text)
                                continue
                            raw_rate = item.get("rate", item.get("price", svc["provider_rate"]))
                            try:
                                new_rate = quantize_currency(raw_rate)
                                old_rate = quantize_currency(svc["provider_rate"])
                            except Exception:
                                continue
                            refill_available, refill_days, refill_duration = _p_refill(item)
                            # Keep provider-side refill/cancel data synchronized too.
                            await db.execute(
                                "UPDATE services SET refill_available=?, refill_days=?, refill_duration=?, cancel_available=? WHERE id=?",
                                (refill_available, refill_days, refill_duration, _p_cancel(item), svc["id"])
                            )
                            if new_rate == old_rate:
                                continue
                            old_sell = quantize_currency(svc["selling_rate"])
                            # Fixed-rate services keep their manual selling price.
                            new_sell = await calculate_service_selling_rate(svc, new_rate)
                            await db.execute(
                                "UPDATE services SET provider_rate=?, selling_rate=? WHERE id=?",
                                (str(new_rate), str(new_sell), svc["id"])
                            )
                            direction = "UP" if new_rate > old_rate else "DOWN"
                            await send_channel_log(
                                bot, "SERVICE",
                                f"💰 <b>Provider Rate {direction}</b>\n"
                                f"• Service: #{svc['id']} {html.escape(svc['name'])}\n"
                                f"• Old Provider Rate: <b>₹{old_rate}</b>\n"
                                f"• New Provider Rate: <b>₹{new_rate}</b>\n"
                                f"• Selling: ₹{old_sell} → ₹{new_sell}"
                            )
                            if new_sell != old_sell:
                                price_jobs.append((svc["id"], svc["name"], str(old_sell), str(new_sell)))
                        await db.commit()
                    for _job in price_jobs:
                        asyncio.create_task(notify_users_price_change(bot, *_job))
                except Exception as pe:
                    logger.warning(f"Provider monitor failed #{p['id']}: {pe}")
        except asyncio.CancelledError:
            break
        except Exception as e:
            logger.error(f"Provider monitor exception: {e}", exc_info=True)


async def send_user_notification(bot: Bot, user_id: int, text: str):
    """Send an event notification to a specific user. Fail silently if blocked/deleted."""
    try:
        await bot.send_message(user_id, text, parse_mode=ParseMode.HTML)
    except Exception as e:
        logger.debug(f"User notification failed for {user_id}: {e}")


async def broadcast_user_notification(bot: Bot, text: str):
    """Disabled for users: service/platform broadcasts are admin/log-only."""
    return


async def notify_service_change(bot: Bot, service_id: int, service_name: str, change: str):
    """Service changes are visible only to admins and the configured log channel."""
    text = (
        "🛠️ <b>SERVICE UPDATED</b>\n\n"
        f"🆔 Service: <b>#{service_id}</b>\n"
        f"🛍️ <b>{html.escape(str(service_name))}</b>\n"
        f"📝 {html.escape(str(change))}"
    )
    await notify_admins(bot, text)
    await send_channel_log(bot, "SERVICE", text)


async def notify_users_price_change(bot: Bot, service_id: int, service_name: str,
                                    old_sell, new_sell):
    """Service price changes go only to admins and logs, never to users."""
    direction = "UP" if Decimal(str(new_sell)) > Decimal(str(old_sell)) else "DOWN"
    text = (
        f"💰 <b>SERVICE PRICE {direction}</b>\n\n"
        f"🆔 Service: <b>#{service_id}</b>\n"
        f"🛍️ <b>{html.escape(str(service_name))}</b>\n"
        f"💵 Old Selling Rate: <b>₹{old_sell}</b>\n"
        f"💵 New Selling Rate: <b>₹{new_sell}</b>"
    )
    await notify_admins(bot, text)
    await send_channel_log(bot, "SERVICE", text)

async def notify_user_order_status_change(bot: Bot, order_id: int, user_id: int, service_name: str, status: str, refund_amount: Decimal = Decimal("0.00")):
    refund_line = f"\n💰 Refund: <b>₹{refund_amount}</b>" if refund_amount and refund_amount > 0 else ""
    text = (
        f"{get_status_emoji(status)} <b>ORDER STATUS UPDATED</b>\n\n"
        f"🆔 Order: <b>#{order_id}</b>\n"
        f"🛍️ Service: <b>{html.escape(str(service_name))}</b>\n"
        f"📊 Status: <b>{html.escape(str(status))}</b>{refund_line}"
    )
    await send_user_notification(bot, int(user_id), text)
    await notify_admins(bot, text + f"\n👤 User: <code>{user_id}</code>")
    await send_channel_log(bot, "ORDER", text + f"\n👤 User: <code>{user_id}</code>")


async def notify_users_discount_change(bot: Bot, discount, net):
    """Discount changes are admin/log-only; never notify users."""
    text = (
        "⚠️ <b>DISCOUNT CHANGED</b>\n\n"
        f"🎁 New Discount: <b>{discount}%</b>\n"
        f"📈 Net Profit: <b>{net}%</b>"
    )
    await notify_admins(bot, text)
    await send_channel_log(bot, "SERVICE", text)

async def notify_user_deposit_event(bot: Bot, user_id: int, title: str, deposit_id: int, amount, extra: str = ""):
    text = (
        f"{title}\n\n"
        f"🆔 Deposit: <b>#{deposit_id}</b>\n"
        f"💰 Amount: <b>₹{amount}</b>"
        f"{extra}"
    )
    await send_user_notification(bot, int(user_id), text)
    return text


async def deposit_expiry_worker(bot: Bot):
    while True:
        try:
            await asyncio.sleep(30)
            now_ts = time.time()
            async with get_db() as db:
                rows = await (await db.execute(
                    "SELECT * FROM deposits WHERE status = 'PENDING' AND method = 'UPI_AUTO' AND expires_at IS NOT NULL AND expires_at < ?",
                    (now_ts,)
                )).fetchall()
                for dep in rows:
                    if dep["qr_msg_id"]:
                        try:
                            await bot.delete_message(chat_id=dep["user_id"], message_id=dep["qr_msg_id"])
                        except Exception:
                            pass
                    await db.execute(
                        "UPDATE deposits SET status = 'EXPIRED' WHERE id = ? AND status = 'PENDING'",
                        (dep["id"],)
                    )
                    expiry_text = (
                        "⏰ <b>DEPOSIT QR EXPIRED</b>\n\n"
                        f"🆔 Deposit: <code>#{dep['id']}</code>\n"
                        f"👤 User: <code>{dep['user_id']}</code>\n"
                        f"💰 Amount: <b>₹{dep['amount']}</b>"
                    )
                    await send_user_notification(
                        bot, int(dep["user_id"]),
                        "⏰ <b>DEPOSIT QR EXPIRED</b>\n\n"
                        f"🆔 Deposit: <b>#{dep['id']}</b>\n"
                        f"💰 Amount: <b>₹{dep['amount']}</b>\n\n"
                        "Your payment QR has expired. Please create a new deposit."
                    )
                    await notify_admins(bot, expiry_text)
                    await send_channel_log(bot, "PAYMENT", expiry_text)
                await db.commit()
        except asyncio.CancelledError:
            break
        except Exception as e:
            logger.error(f"Deposit expiry worker error: {e}", exc_info=True)


# ---------------------------------------------------------------------------
# GLOBAL ERROR HANDLER
# ---------------------------------------------------------------------------
@router.error()
async def global_bot_error_handler(event: ErrorEvent):
    exc = event.exception
    if _is_not_modified(exc):
        logger.debug(f"Ignored no-op edit: {exc}")
        return
    logger.error(f"Global Router Exception: {exc}", exc_info=True)


MINIAPP_HTML = r'''<!DOCTYPE html>
<html lang="en" class="light">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no, viewport-fit=cover">
<meta name="theme-color" content="#f5f7fc">
<title>SMM Panel</title>
<script src="https://telegram.org/js/telegram-web-app.js"></script>
<link rel="preconnect" href="https://cdn.tailwindcss.com" crossorigin>
<script src="https://cdn.tailwindcss.com"></script>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="dns-prefetch" href="https://quickchart.io">
<link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;600;700;800&family=JetBrains+Mono:wght@500;700&display=swap" rel="stylesheet" media="print" onload="this.media='all'">
<noscript><link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;600;700;800&family=JetBrains+Mono:wght@500;700&display=swap" rel="stylesheet"></noscript>
<script>
tailwind.config = { darkMode: 'class', theme: { extend: {
  fontFamily: { sans: ['"Plus Jakarta Sans"', 'system-ui', 'sans-serif'], mono: ['"JetBrains Mono"', 'monospace'] },
  colors: { ink: '#f5f7fc', panel: '#232834', brand: { 400: '#a78bfa', 500: '#8b5cf6', 600: '#7c3aed' } },
  keyframes: {
    float: { '0%,100%': { transform: 'translate3d(0,0,0) scale(1)' }, '50%': { transform: 'translate3d(30px,-40px,0) scale(1.15)' } },
    rise: { from: { opacity: 0, transform: 'translateY(14px)' }, to: { opacity: 1, transform: 'translateY(0)' } },
    shimmer: { '100%': { transform: 'translateX(100%)' } },
    pop: { '0%': { transform: 'scale(.6)', opacity: 0 }, '70%': { transform: 'scale(1.08)' }, '100%': { transform: 'scale(1)', opacity: 1 } },
    sheet: { from: { transform: 'translateY(100%)' }, to: { transform: 'translateY(0)' } },
    spin2: { to: { transform: 'rotate(360deg)' } }
  },
  animation: { float: 'float 14s ease-in-out infinite', rise: 'rise .45s cubic-bezier(.2,.8,.2,1) both', pop: 'pop .5s cubic-bezier(.2,.8,.2,1) both', sheet: 'sheet .35s cubic-bezier(.2,.8,.2,1) both' }
} } }
</script>
<style>
  :root { --bg:#f5f7fc; --txt:#0f172a; --muted:#64748b; --line:#e6eaf2; --acc:#7c3aed; }
  html, body { height:100%; background:var(--bg); -webkit-tap-highlight-color:transparent; overscroll-behavior:none; }
  body { font-family:'Plus Jakarta Sans',system-ui,sans-serif; color:var(--txt); -webkit-font-smoothing:antialiased; }
  body::before { content:''; position:fixed; inset:0; z-index:0; pointer-events:none;
    background:radial-gradient(60% 40% at 92% -5%,rgba(167,139,250,.30),transparent 70%),radial-gradient(50% 35% at -10% 12%,rgba(56,189,248,.18),transparent 70%),radial-gradient(60% 40% at 50% 112%,rgba(244,114,182,.16),transparent 70%); }
  ::-webkit-scrollbar { width:0; height:0; }
  .glass { background:#fff; border:1px solid var(--line); box-shadow:0 1px 2px rgba(15,23,42,.04),0 12px 28px -14px rgba(76,29,149,.16); }
  .glass-strong { background:rgba(255,255,255,.88); -webkit-backdrop-filter:blur(18px); backdrop-filter:blur(18px); border:1px solid var(--line); box-shadow:0 1px 2px rgba(15,23,42,.04),0 14px 34px -14px rgba(76,29,149,.22); }
  .grad-text { background:linear-gradient(120deg,#6d28d9,#c026d3 60%,#e11d74); -webkit-background-clip:text; background-clip:text; color:transparent; }
  .grad-btn { background:linear-gradient(135deg,#7c5cff,#6d3df5 55%,#9333ea); color:#fff; border:none; box-shadow:0 10px 22px -8px rgba(109,61,245,.6),inset 0 1px 0 rgba(255,255,255,.28); transition:transform .15s,box-shadow .15s,filter .15s; }
  .grad-btn:active { transform:scale(.97); filter:brightness(.95); box-shadow:0 4px 10px -4px rgba(109,61,245,.6); }
  .btn-soft { background:#fff; border:1px solid var(--line); color:#1e293b; box-shadow:0 1px 2px rgba(15,23,42,.05),0 6px 14px -8px rgba(15,23,42,.18); transition:transform .15s,box-shadow .15s; }
  .btn-soft:active { transform:scale(.97); background:#f8fafc; }
  .card-hover { transition:transform .15s ease,box-shadow .15s ease; }
  .card-hover:active { transform:scale(.982); }
  .field { background:#fff; border:1px solid #dfe5ef; color:var(--txt); box-shadow:0 1px 2px rgba(15,23,42,.04); transition:border-color .2s,box-shadow .2s; }
  .field:focus { outline:none; border-color:#8b5cf6; box-shadow:0 0 0 4px rgba(139,92,246,.16); }
  .field::placeholder { color:#94a3b8; }
  .chip-on { background:#f3efff !important; color:#6d28d9 !important; border:1px solid #d9ccff !important; }
  .toggle { width:50px; height:28px; border-radius:999px; background:#dbe1ec; position:relative; transition:all .25s; flex:none; }
  .toggle::after { content:''; position:absolute; top:3px; left:3px; width:22px; height:22px; border-radius:50%; background:#fff; box-shadow:0 2px 5px rgba(15,23,42,.3); transition:transform .25s; }
  .toggle.on { background:linear-gradient(135deg,#8b5cf6,#6d3df5); }
  .toggle.on::after { transform:translateX(22px); }
  [style*="background:rgba(255,255,255,.05)"],[style*="background:rgba(255,255,255,.06)"],[style*="background:rgba(255,255,255,.07)"],[style*="background:rgba(255,255,255,.08)"] { background:#f6f8fc !important; border:1px solid #ebeff6 !important; }
  /* gradient hero cards keep white text */
  [style*="gradient(135deg,#"] { color:#fff; box-shadow:0 18px 36px -16px rgba(109,40,217,.55); }
  [style*="gradient(135deg,#"] [class*="text-white"], [style*="gradient(135deg,#"] .ico { color:rgba(255,255,255,.92) !important; }
  /* light-theme remap of dark utility classes */
  .text-white { color:#0f172a !important; }
  .text-white\/80,.text-white\/75,.text-white\/70,.text-white\/65,.text-white\/60 { color:#475569 !important; }
  .text-white\/55,.text-white\/50 { color:#64748b !important; }
  .text-white\/45,.text-white\/40,.text-white\/35,.text-white\/30 { color:#94a3b8 !important; }
  .grad-btn.grad-btn, .grad-btn [class*="text-white"], .grad-btn .ico { color:#fff !important; }
  .text-rose-300 { color:#be123c !important; } .text-rose-200 { color:#9f1239 !important; }
  .text-emerald-300 { color:#047857 !important; } .text-sky-300 { color:#0369a1 !important; }
  .text-amber-300 { color:#b45309 !important; } .text-amber-200 { color:#92400e !important; }
  .text-orange-300 { color:#c2410c !important; } .text-brand-400 { color:#7c3aed !important; }
  .bg-white\/8,.bg-white\/10,.bg-white\/15 { background:#eef1f7 !important; }
  .bg-white\/20 { background:#cbd5e1 !important; }
  .bg-black\/70 { background:rgba(15,23,42,.38) !important; }
  [class*="border-white"] { border-color:#e6eaf2 !important; }
  .skeleton { position:relative; overflow:hidden; background:#e9edf5; border-radius:18px; }
  .skeleton::after { content:''; position:absolute; inset:0; transform:translateX(-100%); background:linear-gradient(90deg,transparent,rgba(255,255,255,.8),transparent); animation:shimmer 1.4s infinite; }
  .safe-bottom { padding-bottom:max(env(safe-area-inset-bottom),10px); }
  .spinner { width:18px; height:18px; border-radius:50%; border:2.5px solid rgba(124,58,237,.25); border-top-color:#7c3aed; animation:spin2 .7s linear infinite; display:inline-block; }
  .grad-btn .spinner { border-color:rgba(255,255,255,.4); border-top-color:#fff; }
  input, textarea, select { font-family:inherit; }
  .tab-ico { transition:all .25s; }
  .tab-active { color:#6d28d9 !important; }
  .tab-active .tab-ico { background:linear-gradient(135deg,#ede9fe,#f5f3ff); color:#6d28d9; box-shadow:inset 0 0 0 1px #ddd6fe,0 6px 14px -8px rgba(109,40,217,.5); }
  .stagger > * { animation:rise .45s cubic-bezier(.2,.8,.2,1) both; }
  .stagger > *:nth-child(2){animation-delay:.04s} .stagger > *:nth-child(3){animation-delay:.08s}
  .stagger > *:nth-child(4){animation-delay:.12s} .stagger > *:nth-child(5){animation-delay:.16s}
  .stagger > *:nth-child(6){animation-delay:.2s} .stagger > *:nth-child(n+7){animation-delay:.24s}
  .line-clamp-2 { display:-webkit-box; -webkit-line-clamp:2; -webkit-box-orient:vertical; overflow:hidden; }
  #sheet { box-shadow:0 -14px 50px rgba(15,23,42,.22); }
  /* icons */
  .ico { width:1em; height:1em; display:inline-block; vertical-align:-.14em; flex:none; }
  .sdot { display:inline-block; width:.55em; height:.55em; border-radius:50%; vertical-align:.02em; }
  .ico-tile { width:44px; height:44px; border-radius:14px; display:flex; align-items:center; justify-content:center; font-size:22px; color:#6d28d9; background:linear-gradient(135deg,#f3eeff,#e8e0ff); box-shadow:inset 0 0 0 1px rgba(139,92,246,.16); }
  .big-ico { width:76px; height:76px; border-radius:26px; display:flex; align-items:center; justify-content:center; font-size:36px; color:#7c3aed; background:linear-gradient(135deg,#f3eeff,#fde7f3); box-shadow:inset 0 0 0 1px rgba(139,92,246,.14),0 14px 28px -16px rgba(124,58,237,.5); }
  .toast-x { background:#fff; color:#0f172a; border:1px solid var(--line); border-left:4px solid var(--tc); box-shadow:0 16px 36px -12px rgba(15,23,42,.3); }
  .toast-ic { color:var(--tc); font-size:18px; line-height:1; margin-top:1px; }
</style>
</head>
<body class="text-slate-900">

<!-- Blocked screen (outside Telegram) -->
<div id="blocked" class="hidden fixed inset-0 z-[100] bg-ink flex items-center justify-center p-8 text-center">
  <div class="max-w-xs animate-pop">
    <div class="mx-auto w-24 h-24 rounded-3xl grad-btn flex items-center justify-center text-5xl mb-6">🔒</div>
    <h1 class="text-2xl font-extrabold mb-2">Open in Telegram</h1>
    <p class="text-white/60 text-sm leading-relaxed" id="blockedMsg">This app only works inside Telegram. Open our bot and tap the app button.</p>
  </div>
</div>

<div id="app" class="relative z-10 max-w-xl mx-auto min-h-full flex flex-col">
  <header id="topbar" class="sticky top-0 z-30 px-4 pt-3 pb-2 hidden">
    <div class="glass-strong rounded-2xl px-3 py-2.5 flex items-center gap-3">
      <button id="backBtn" class="hidden w-9 h-9 rounded-xl btn-soft flex items-center justify-center text-lg" onclick="goBack()">‹</button>
      <div class="flex-1 min-w-0">
        <div id="topTitle" class="font-extrabold text-[15px] truncate grad-text">SMM Panel</div>
        <div id="topSub" class="text-[11px] text-white/45 truncate"></div>
      </div>
      <button onclick="go('funds')" class="rounded-xl px-3 py-1.5 btn-soft flex items-center gap-2 active:scale-95 transition">
        <span class="text-sm">👛</span>
        <span id="balPill" class="font-mono text-[13px] font-bold">—</span>
        <span class="w-6 h-6 rounded-lg grad-btn text-[14px] leading-6 text-center font-bold">+</span>
      </button>
    </div>
  </header>

  <main id="view" class="flex-1 px-4 pb-32 pt-2"></main>
</div>

<!-- Bottom nav -->
<nav id="nav" class="hidden fixed bottom-0 inset-x-0 z-40 safe-bottom px-3">
  <div class="max-w-xl mx-auto glass-strong rounded-[28px] px-2 pt-2.5 pb-2 flex justify-around"></div>
</nav>

<!-- Sheet -->
<div id="sheetWrap" class="hidden fixed inset-0 z-50">
  <div class="absolute inset-0 bg-black/70 backdrop-blur-sm" onclick="closeSheet()"></div>
  <div id="sheet" class="absolute bottom-0 inset-x-0 max-w-xl mx-auto glass-strong rounded-t-[32px] p-5 safe-bottom animate-sheet max-h-[90vh] overflow-y-auto">
    <div class="w-11 h-1.5 rounded-full bg-white/20 mx-auto mb-4"></div>
    <div id="sheetBody"></div>
  </div>
</div>

<!-- Toast -->
<div id="toastHost" class="fixed top-3 inset-x-0 z-[90] flex flex-col items-center gap-2 pointer-events-none px-4"></div>

<script>
/* ===================== core ===================== */
const tg = window.Telegram && Telegram.WebApp;
const S = { me: null, boot: null, stack: [], route: 'home', params: {}, admin: false, cache: {} };
const $ = (s, r = document) => r.querySelector(s);
const esc = s => String(s ?? '').replace(/[&<>"']/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));
const view = () => $('#view');

function haptic(t = 'light') { try { tg.HapticFeedback.impactOccurred(t); } catch (e) {} }
function notifyH(t = 'success') { try { tg.HapticFeedback.notificationOccurred(t); } catch (e) {} }

function toast(msg, kind = 'info') {
  const colors = { info: '#7c3aed', ok: '#059669', err: '#e11d48' };
  const ico = { info: '✨', ok: '✅', err: '⚠️' };
  const el = document.createElement('div');
  el.className = 'animate-pop pointer-events-auto max-w-sm w-full rounded-2xl px-4 py-3 text-sm font-semibold flex gap-2.5 items-start toast-x';
  el.style.setProperty('--tc', colors[kind]);
  el.innerHTML = `<span class="toast-ic">${ico[kind]}</span><span class="flex-1">${esc(msg)}</span>`;
  $('#toastHost').appendChild(el);
  if (kind === 'err') notifyH('error'); else if (kind === 'ok') notifyH('success');
  setTimeout(() => { el.style.transition = 'all .3s'; el.style.opacity = 0; el.style.transform = 'translateY(-10px)'; setTimeout(() => el.remove(), 300); }, 3200);
}

async function api(path, opts = {}) {
  const init = tg ? tg.initData : '';
  const o = { method: opts.method || 'GET', headers: { 'X-Init-Data': init } };
  if (opts.body !== undefined) { o.method = 'POST'; o.headers['Content-Type'] = 'application/json'; o.body = JSON.stringify(opts.body); }
  let r;
  try { r = await fetch(path, o); } catch (e) { throw new Error('Network error. Check your connection.'); }
  let j = {};
  try { j = await r.json(); } catch (e) {}
  if (r.status === 401) { showBlocked(); throw new Error('Open inside Telegram'); }
  if (r.status === 403 && j.code === 'banned') { showBlocked('Your account is suspended. Please contact support.'); throw new Error(j.error); }
  if (!r.ok || j.ok === false) { const e = new Error(j.error || 'Something went wrong'); e.code = j.code; e.data = j; throw e; }
  return j;
}

function showBlocked(msg) { if (msg) $('#blockedMsg').textContent = msg; $('#blocked').classList.remove('hidden'); $('#blocked').classList.add('flex'); }

/* ===================== money ===================== */
function money(inr, forceCur) {
  const cur = forceCur || (S.me && S.me.currency) || 'INR';
  const n = parseFloat(inr || 0);
  if (cur === 'USD') { const r = parseFloat(S.boot.usd_rate || 0); return '$' + (r > 0 ? n / r : n).toFixed(2); }
  return '₹' + n.toLocaleString('en-IN', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
}
const num = n => Number(n || 0).toLocaleString('en-IN');
const statusMeta = {
  'Pending': ['🟡', 'bg-amber-50 text-amber-700 border-amber-200'],
  'Processing': ['🔵', 'bg-sky-50 text-sky-700 border-sky-200'],
  'In Progress': ['🔵', 'bg-sky-50 text-sky-700 border-sky-200'],
  'Completed': ['🟢', 'bg-emerald-50 text-emerald-700 border-emerald-200'],
  'Partial': ['🟠', 'bg-orange-50 text-orange-700 border-orange-200'],
  'Cancelled': ['🔴', 'bg-rose-50 text-rose-700 border-rose-200'],
  'Failed': ['❌', 'bg-rose-50 text-rose-700 border-rose-200'],
  'PENDING': ['🟡', 'bg-amber-50 text-amber-700 border-amber-200'],
  'APPROVED': ['🟢', 'bg-emerald-50 text-emerald-700 border-emerald-200'],
  'REJECTED': ['🔴', 'bg-rose-50 text-rose-700 border-rose-200'],
  'EXPIRED': ['⏰', 'bg-slate-100 text-slate-600 border-slate-200'],
  'CANCELLED': ['🔴', 'bg-rose-50 text-rose-700 border-rose-200'],
  'FAILED': ['❌', 'bg-rose-50 text-rose-700 border-rose-200'],
  'DUPLICATE': ['⚠️', 'bg-orange-50 text-orange-700 border-orange-200'],
};
const badge = st => { const m = statusMeta[st] || ['⚪', 'bg-slate-100 text-slate-600 border-slate-200']; return `<span class="inline-flex items-center gap-1 text-[11px] font-bold px-2.5 py-1 rounded-full border ${m[1]}">${m[0]} ${esc(st)}</span>`; };

const skel = (n = 4, h = 'h-20') => `<div class="space-y-3">${Array.from({ length: n }, () => `<div class="skeleton ${h}"></div>`).join('')}</div>`;
const empty = (ico, t, s = '') => `<div class="text-center py-16 animate-rise"><div class="big-ico mx-auto mb-3">${ico}</div><div class="font-bold text-lg">${t}</div><div class="text-white/50 text-sm mt-1">${s}</div></div>`;
const btn = (label, onclick, cls = 'grad-btn', extra = '') => `<button ${extra} onclick="${onclick}" class="${cls} w-full rounded-2xl py-3.5 font-bold text-[15px] transition active:scale-[.98] disabled:opacity-50">${label}</button>`;

async function withBtn(el, fn) {
  if (el.dataset.busy) return; el.dataset.busy = 1;
  const html = el.innerHTML; el.innerHTML = '<span class="spinner"></span>'; el.disabled = true;
  try { return await fn(); } finally { el.innerHTML = html; el.disabled = false; delete el.dataset.busy; }
}

/* ===================== navigation ===================== */
const TABS = [
  { id: 'home', ico: '🏠', label: 'Home' },
  { id: 'services', ico: '🛍️', label: 'Services' },
  { id: 'orders', ico: '📦', label: 'Orders' },
  { id: 'funds', ico: '💰', label: 'Funds' },
  { id: 'profile', ico: '👤', label: 'Me' },
];
const ROOTS = ['home', 'services', 'orders', 'funds', 'profile', 'admin'];

function renderNav() {
  const tabs = S.me.is_admin ? [...TABS.slice(0, 4), { id: 'admin', ico: '🛡️', label: 'Admin' }, TABS[4]] : TABS;
  const root = S.route.startsWith('admin') ? 'admin' : (S.route.startsWith('track') ? 'orders' : (S.route.startsWith('svc') || S.route.startsWith('order') || S.route.startsWith('cat') ? 'services' : S.route));
  $('#nav > div').innerHTML = tabs.map(t => `
    <button onclick="go('${t.id}')" class="flex-1 flex flex-col items-center gap-1 py-1 ${root === t.id ? 'tab-active' : 'text-white/45'} transition">
      <span class="tab-ico w-11 h-10 rounded-2xl flex items-center justify-center text-[19px]">${t.ico}</span>
      <span class="text-[10px] font-bold tracking-wide">${t.label}</span>
    </button>`).join('');
}

function go(route, params = {}, push = true) {
  haptic();
  if (push && !(ROOTS.includes(route) && ROOTS.includes(S.route))) S.stack.push({ route: S.route, params: S.params });
  if (ROOTS.includes(route)) S.stack = [];
  S.route = route; S.params = params;
  render();
  window.scrollTo({ top: 0, behavior: 'smooth' });
}
function goBack() {
  haptic();
  const p = S.stack.pop();
  if (p) { S.route = p.route; S.params = p.params; render(); } else go('home', {}, false);
}

function setTop(title, sub = '', back = false) {
  $('#topTitle').textContent = title; $('#topSub').textContent = sub;
  $('#backBtn').classList.toggle('hidden', !back);
  if (tg) { try { back ? tg.BackButton.show() : tg.BackButton.hide(); } catch (e) {} }
}

function render() {
  renderNav();
  $('#balPill').textContent = money(S.me.balance);
  const r = S.route;
  const map = { home: vHome, services: vServices, cat: vCatServices, svc: vService, orders: vOrders, track: vTrack, funds: vFunds,
    dep_auto: vDepAuto, dep_manual: vDepManual, dep_usdt: vDepUsdt, profile: vProfile, history: vHistory, gift: vGift, support: vSupport,
    admin: vAdmin, admin_orders: vAdminOrders, admin_order: vAdminOrder, admin_users: vAdminUsers, admin_user: vAdminUser,
    admin_deposits: vAdminDeposits, admin_services: vAdminServices, admin_gifts: vAdminGifts, admin_settings: vAdminSettings,
    admin_providers: vAdminProviders, admin_broadcast: vAdminBroadcast, admin_categories: vAdminCategories,
    admin_add_service: vAdminAddService, admin_prov_services: vAdminProvServices };
  (map[r] || vHome)();
}

async function refreshMe() {
  try { const j = await api('/api/bootstrap'); S.boot = j; S.me = j.me; $('#balPill').textContent = money(S.me.balance); } catch (e) {}
}

function sheet(html) { $('#sheetBody').innerHTML = html; $('#sheetWrap').classList.remove('hidden'); haptic(); }
function closeSheet() { $('#sheetWrap').classList.add('hidden'); }

function maintenanceBanner() {
  const m = S.boot.maintenance;
  if (!(m.full || m.order || m.deposit || m.service)) return '';
  return `<div class="glass rounded-2xl p-3.5 mb-4 flex gap-3 items-center border-amber-400/30"><span class="text-2xl">🛠️</span><div class="text-sm"><b>Maintenance</b><div class="text-white/60 text-xs">Some features are temporarily unavailable.</div></div></div>`;
}

/* ===================== HOME ===================== */
async function vHome() {
  setTop(S.boot.bot_name || 'SMM Panel', 'Premium social growth', false);
  const m = S.me, cats = S.boot.categories;
  const icons = ['✨', '🚀', '💎', '🔥', '⚡', '🌟', '🎯', '🏆'];
  view().innerHTML = `
  <div class="stagger space-y-4">
    ${maintenanceBanner()}
    <div class="relative overflow-hidden rounded-[28px] p-5" style="background:linear-gradient(135deg,#6d28d9 0%,#a21caf 55%,#db2777 100%)">
      <div class="absolute -right-10 -top-10 w-44 h-44 rounded-full bg-white/15 blur-2xl"></div>
      <div class="absolute -left-8 -bottom-12 w-40 h-40 rounded-full bg-cyan-300/20 blur-2xl"></div>
      <div class="relative">
        <div class="text-white/75 text-xs font-semibold">Welcome back,</div>
        <div class="text-2xl font-extrabold leading-tight">${esc(m.first_name)} <span class="inline-block animate-pop">👋</span></div>
        <div class="mt-5 flex items-end justify-between">
          <div><div class="text-white/70 text-[11px] font-semibold uppercase tracking-widest">Wallet balance</div>
          <div class="text-[34px] font-extrabold font-mono leading-none mt-1">${money(m.balance)}</div></div>
          <button onclick="go('funds')" class="bg-white text-violet-700 font-extrabold text-sm rounded-2xl px-4 py-2.5 shadow-xl active:scale-95 transition">+ Add Funds</button>
        </div>
      </div>
    </div>
    <div class="grid grid-cols-4 gap-2.5">
      ${[['🛍️', 'Order', "go('services')"], ['📦', 'Orders', "go('orders')"], ['🎁', 'Gift', "go('gift')"], ['💬', 'Support', "go('support')"]].map(a => `
      <button onclick="${a[2]}" class="glass rounded-2xl py-3.5 flex flex-col items-center gap-1.5 card-hover"><span class="ico-tile">${a[0]}</span><span class="text-[11px] font-bold text-white/80">${a[1]}</span></button>`).join('')}
    </div>
    ${S.boot.discount.on ? `<div class="rounded-2xl p-4 flex items-center gap-3" style="background:linear-gradient(135deg,rgba(16,185,129,.25),rgba(34,211,238,.18));border:1px solid rgba(52,211,153,.35)"><span class="text-3xl">🎁</span><div><div class="font-extrabold">${esc(S.boot.discount.pct)}% OFF everything</div><div class="text-xs text-white/65">Discount is applied automatically to all services.</div></div></div>` : ''}
    <div>
      <div class="flex items-center justify-between mb-3"><h2 class="font-extrabold text-lg">Categories</h2><button onclick="go('services')" class="text-xs font-bold text-brand-400">See all →</button></div>
      <div class="grid grid-cols-2 gap-3">
        ${cats.length ? cats.slice(0, 6).map((c, i) => `
        <button onclick="go('cat',{id:${c.id},name:'${esc(c.name).replace(/'/g, '')}'})" class="glass rounded-2xl p-4 text-left card-hover relative overflow-hidden">
          <div class="absolute -right-4 -bottom-4 text-6xl opacity-10">${esc(c.icon || icons[i % 8])}</div>
          <div class="ico-tile mb-3">${esc(c.icon || icons[i % 8])}</div>
          <div class="font-bold text-sm leading-tight line-clamp-2">${esc(c.name)}</div>
          <div class="text-[11px] text-white/45 mt-1">${c.cnt} services</div>
        </button>`).join('') : empty('🛍️', 'No categories yet', 'Check back soon.')}
      </div>
    </div>
    <div id="recent"></div>
  </div>`;
  try {
    const j = await api('/api/orders?page=1');
    if (j.orders.length) $('#recent').innerHTML = `
      <div class="flex items-center justify-between mb-3"><h2 class="font-extrabold text-lg">Recent orders</h2><button onclick="go('orders')" class="text-xs font-bold text-brand-400">View all →</button></div>
      <div class="space-y-2.5">${j.orders.slice(0, 3).map(orderRow).join('')}</div>`;
  } catch (e) {}
}

/* ===================== SERVICES ===================== */
function vServices() {
  setTop('Services', 'Pick a category or search', false);
  const icons = ['✨', '🚀', '💎', '🔥', '⚡', '🌟', '🎯', '🏆'];
  const cats = S.boot.categories;
  view().innerHTML = `
  <div class="space-y-4">
    ${maintenanceBanner()}
    <div class="relative"><span class="absolute left-4 top-1/2 -translate-y-1/2 text-white/40">🔎</span>
      <input id="q" oninput="debSearch()" placeholder="Search all services…" class="field w-full rounded-2xl pl-11 pr-4 py-3.5 text-sm text-white placeholder-white/35"></div>
    <div id="svcOut" class="stagger grid grid-cols-2 gap-3">
      ${cats.length ? cats.map((c, i) => `
      <button onclick="go('cat',{id:${c.id},name:'${esc(c.name).replace(/'/g, '')}'})" class="glass rounded-2xl p-4 text-left card-hover relative overflow-hidden">
        <div class="absolute -right-4 -bottom-4 text-6xl opacity-10">${esc(c.icon || icons[i % 8])}</div>
        <div class="ico-tile mb-3">${esc(c.icon || icons[i % 8])}</div>
        <div class="font-bold text-sm leading-tight line-clamp-2">${esc(c.name)}</div>
        <div class="text-[11px] text-white/45 mt-1">${c.cnt} services</div>
      </button>`).join('') : empty('🛍️', 'No categories yet')}
    </div>
  </div>`;
}
let _deb; function debSearch() { clearTimeout(_deb); _deb = setTimeout(runSearch, 320); }
async function runSearch() {
  const q = $('#q').value.trim(); const out = $('#svcOut');
  if (!q) { vServices(); return; }
  out.className = 'space-y-3'; out.innerHTML = skel(4);
  try { const j = await api('/api/services?q=' + encodeURIComponent(q)); out.innerHTML = j.services.length ? j.services.map(svcCard).join('') : empty('🔍', 'No results', 'Try another keyword.'); }
  catch (e) { out.innerHTML = empty('🛠️', esc(e.message)); }
}
function svcCard(s) {
  return `<button onclick="go('svc',{id:${s.id}})" class="glass rounded-2xl p-4 text-left w-full card-hover block animate-rise">
    <div class="flex gap-3 items-start"><div class="flex-1 min-w-0">
      <div class="text-[11px] font-mono text-white/35 mb-0.5">#${s.id}</div>
      <div class="font-bold text-sm leading-snug line-clamp-2">${esc(s.name)}</div>
      <div class="flex flex-wrap gap-1.5 mt-2.5">
        <span class="text-[10px] font-bold px-2 py-1 rounded-lg bg-white/8 text-white/60" style="background:rgba(255,255,255,.07)">${num(s.min)} – ${num(s.max)}</span>
        <span class="text-[10px] font-bold px-2 py-1 rounded-lg ${s.refill ? 'bg-emerald-400/15 text-emerald-300' : 'text-white/40'}" ${s.refill ? '' : 'style="background:rgba(255,255,255,.05)"'}>${s.refill ? '♻️ Refill' : 'No refill'}</span>
      </div></div>
      <div class="text-right shrink-0">
        ${s.orig_rate ? `<div class="text-[10px] line-through text-white/35 font-mono">${money(s.orig_rate)}</div>` : ''}
        <div class="font-mono font-extrabold text-[15px] grad-text">${money(s.rate)}</div>
        <div class="text-[10px] text-white/40">per 1000</div>
      </div></div></button>`;
}
async function vCatServices() {
  setTop(S.params.name || 'Category', 'Choose a service', true);
  view().innerHTML = skel(5);
  try {
    const j = await api('/api/services?category_id=' + S.params.id);
    view().innerHTML = `<div class="space-y-3 stagger">${maintenanceBanner()}${j.services.length ? j.services.map(svcCard).join('') : empty('📭', 'No services here', 'This category is empty right now.')}</div>`;
  } catch (e) { view().innerHTML = empty('🛠️', esc(e.message)); }
}

/* ===================== SERVICE / ORDER ===================== */
async function vService() {
  setTop('New Order', 'Fill in the details', true);
  view().innerHTML = skel(3, 'h-40');
  let s;
  try { s = (await api('/api/service/' + S.params.id)).service; } catch (e) { view().innerHTML = empty('🛠️', esc(e.message)); return; }
  window._svc = s;
  view().innerHTML = `
  <div class="space-y-4 stagger">
    <div class="glass rounded-3xl p-5">
      <div class="text-[11px] font-mono text-white/35">SERVICE #${s.id}</div>
      <div class="font-extrabold text-lg leading-snug mt-1">${esc(s.name)}</div>
      <div class="grid grid-cols-3 gap-2 mt-4">
        <div class="rounded-2xl p-3 text-center" style="background:rgba(255,255,255,.05)"><div class="text-[10px] text-white/45 font-bold">PRICE/1K</div>
          ${s.orig_rate ? `<div class="text-[10px] line-through text-white/35 font-mono">${money(s.orig_rate)}</div>` : ''}<div class="font-mono font-extrabold text-sm grad-text">${money(s.rate)}</div></div>
        <div class="rounded-2xl p-3 text-center" style="background:rgba(255,255,255,.05)"><div class="text-[10px] text-white/45 font-bold">MIN</div><div class="font-mono font-extrabold text-sm">${num(s.min)}</div></div>
        <div class="rounded-2xl p-3 text-center" style="background:rgba(255,255,255,.05)"><div class="text-[10px] text-white/45 font-bold">MAX</div><div class="font-mono font-extrabold text-sm">${num(s.max)}</div></div>
      </div>
      <div class="flex gap-2 mt-3 flex-wrap">
        <span class="text-[11px] font-bold px-3 py-1.5 rounded-full ${s.refill ? 'bg-emerald-400/15 text-emerald-300' : 'bg-white/8 text-white/50'}">♻️ ${esc(s.refill_text)}</span>
        <span class="text-[11px] font-bold px-3 py-1.5 rounded-full ${s.cancel ? 'bg-sky-400/15 text-sky-300' : 'bg-white/8 text-white/50'}">${s.cancel ? '✅ Cancel supported' : '🚫 No cancel'}</span>
      </div>
      <div class="mt-4 text-[13px] text-white/60 leading-relaxed whitespace-pre-line">${esc(s.description || 'High quality, fast delivery.')}</div>
    </div>
    <div class="glass rounded-3xl p-5 space-y-4">
      <div><label class="text-xs font-bold text-white/60 ml-1">🔗 Link</label>
        <input id="oLink" type="url" inputmode="url" placeholder="https://…" class="field w-full rounded-2xl px-4 py-3.5 mt-1.5 text-sm"></div>
      <div><label class="text-xs font-bold text-white/60 ml-1">🔢 Quantity</label>
        <input id="oQty" type="number" inputmode="numeric" oninput="calcCost()" placeholder="${s.min} – ${s.max}" class="field w-full rounded-2xl px-4 py-3.5 mt-1.5 text-sm font-mono"></div>
      <div class="rounded-2xl p-4 flex items-center justify-between" style="background:linear-gradient(135deg,rgba(124,58,237,.22),rgba(192,38,211,.16));border:1px solid rgba(167,139,250,.25)">
        <div><div class="text-[11px] text-white/55 font-bold">TOTAL</div><div id="oTotal" class="font-mono font-extrabold text-2xl">${money(0)}</div></div>
        <div class="text-right"><div class="text-[11px] text-white/55 font-bold">YOUR BALANCE</div><div class="font-mono font-bold text-sm">${money(S.me.balance)}</div></div>
      </div>
      <button id="oBtn" onclick="placeOrder(this)" class="grad-btn w-full rounded-2xl py-4 font-extrabold text-[15px] active:scale-[.98] transition">Place Order 🚀</button>
    </div>
  </div>`;
}
function calcCost() {
  const q = parseInt($('#oQty').value || 0), s = window._svc;
  $('#oTotal').textContent = money(q > 0 ? (q / 1000) * parseFloat(s.rate) : 0);
}
async function placeOrder(el) {
  const s = window._svc, link = $('#oLink').value.trim(), qty = parseInt($('#oQty').value || 0);
  if (!/^(https?|tg):\/\//i.test(link)) return toast('Enter a valid link starting with http:// or https://', 'err');
  if (!qty || qty < s.min || qty > s.max) return toast(`Quantity must be between ${num(s.min)} and ${num(s.max)}`, 'err');
  const cost = (qty / 1000) * parseFloat(s.rate);
  sheet(`<div class="text-center">
    <div class="big-ico mx-auto mb-2">📦</div><div class="font-extrabold text-xl mb-4">Confirm order</div>
    <div class="rounded-2xl p-4 text-left space-y-2 text-sm mb-4" style="background:rgba(255,255,255,.05)">
      <div class="flex justify-between gap-3"><span class="text-white/50">Service</span><b class="text-right">${esc(s.name)}</b></div>
      <div class="flex justify-between gap-3"><span class="text-white/50">Link</span><b class="text-right break-all font-mono text-xs">${esc(link)}</b></div>
      <div class="flex justify-between"><span class="text-white/50">Quantity</span><b class="font-mono">${num(qty)}</b></div>
      <div class="flex justify-between"><span class="text-white/50">Total</span><b class="font-mono grad-text text-base">${money(cost)}</b></div>
    </div>
    <div class="grid grid-cols-2 gap-3">${btn('Cancel', 'closeSheet()', 'btn-soft')}
      <button onclick="confirmOrder(this)" class="grad-btn rounded-2xl py-3.5 font-bold">Confirm ✓</button></div></div>`);
  window._pending = { service_id: s.id, link, quantity: qty };
}
async function confirmOrder(el) {
  await withBtn(el, async () => {
    try {
      const j = await api('/api/order', { body: window._pending });
      closeSheet(); await refreshMe(); notifyH('success');
      sheet(`<div class="text-center py-2"><div class="big-ico mx-auto animate-pop mb-2">🎉</div><div class="font-extrabold text-xl">Order placed!</div>
        <div class="text-white/60 text-sm mt-1">Order <b class="font-mono">#${j.order_id}</b> · ${j.status}<br>${j.note ? esc(j.note) : ''}</div>
        <div class="grid grid-cols-2 gap-3 mt-5">${btn('Track order', `closeSheet();go('track',{id:${j.order_id}})`)}${btn('New order', "closeSheet();go('services')", 'btn-soft')}</div></div>`);
    } catch (e) {
      closeSheet();
      if (e.code === 'balance' || /balance/i.test(e.message)) sheet(`<div class="text-center"><div class="big-ico mx-auto mb-2">👛</div><div class="font-extrabold text-xl">Insufficient balance</div><div class="text-white/60 text-sm mt-1">Add funds to continue with this order.</div><div class="mt-5">${btn('Add funds', "closeSheet();go('funds')")}</div></div>`);
      else toast(e.message, 'err');
    }
  });
}

/* ===================== ORDERS ===================== */
function orderRow(o) {
  return `<button onclick="go('track',{id:${o.id}})" class="glass rounded-2xl p-4 text-left w-full card-hover block animate-rise">
    <div class="flex items-start justify-between gap-3"><div class="min-w-0"><div class="text-[11px] font-mono text-white/35">#${o.id} · ${esc(o.created_at)}</div>
      <div class="font-bold text-sm mt-0.5 truncate">${esc(o.service_name)}</div>
      <div class="text-[11px] text-white/40 truncate font-mono mt-0.5">${esc(o.link)}</div></div>${badge(o.status)}</div>
    <div class="flex justify-between mt-3 text-xs"><span class="text-white/50">Qty <b class="font-mono text-white/80">${num(o.quantity)}</b></span><b class="font-mono grad-text">${money(o.charge)}</b></div></button>`;
}
async function vOrders() {
  setTop('My Orders', 'Track everything', false);
  const f = S.params.filter || 'all';
  const filters = [['all', 'All'], ['active', 'Active'], ['completed', 'Done'], ['partial', 'Partial'], ['cancelled', 'Cancelled']];
  view().innerHTML = `
    <div class="flex gap-2 overflow-x-auto pb-3 -mx-1 px-1">${filters.map(x => `<button onclick="S.params={filter:'${x[0]}'};vOrders()" class="shrink-0 px-4 py-2 rounded-full text-xs font-bold border border-white/10 ${f === x[0] ? 'chip-on' : 'btn-soft text-white/70'}">${x[1]}</button>`).join('')}</div>
    <div id="olist">${skel(4)}</div>`;
  loadOrders(f, 1);
}
async function loadOrders(f, page) {
  const box = $('#olist');
  try {
    const j = await api(`/api/orders?status=${f}&page=${page}`);
    if (page === 1 && !j.orders.length) { box.innerHTML = empty('📦', 'No orders yet', 'Your orders will appear here.'); return; }
    if (page === 1) box.innerHTML = '<div id="orows" class="space-y-3 stagger"></div><div id="omore" class="mt-4"></div>';
    $('#orows').insertAdjacentHTML('beforeend', j.orders.map(orderRow).join(''));
    $('#omore').innerHTML = page < j.pages ? btn('Load more', `loadOrders('${f}',${page + 1})`, 'btn-soft') : '';
  } catch (e) { box.innerHTML = empty('🛠️', esc(e.message)); }
}
async function vTrack() {
  setTop('Order #' + S.params.id, 'Live tracking', true);
  view().innerHTML = skel(3, 'h-32');
  try { const j = await api('/api/order/' + S.params.id); drawTrack(j.order); } catch (e) { view().innerHTML = empty('🛠️', esc(e.message)); }
}
function drawTrack(o) {
  const pct = o.quantity ? Math.min(100, Math.round(o.delivered / o.quantity * 100)) : 0;
  view().innerHTML = `<div class="space-y-4 stagger">
    <div class="glass rounded-3xl p-5">
      <div class="flex justify-between items-start gap-3"><div><div class="text-[11px] font-mono text-white/35">ORDER #${o.id}</div><div class="font-extrabold text-lg leading-snug">${esc(o.service_name)}</div></div>${badge(o.status)}</div>
      <div class="mt-4"><div class="flex justify-between text-xs mb-1.5"><span class="text-white/55 font-semibold">Delivery progress</span><b class="font-mono">${pct}%</b></div>
        <div class="h-3 rounded-full overflow-hidden" style="background:rgba(255,255,255,.08)"><div class="h-full rounded-full transition-all duration-700" style="width:${pct}%;background:linear-gradient(90deg,#8b5cf6,#d946ef,#22d3ee)"></div></div></div>
      <div class="grid grid-cols-3 gap-2 mt-4 text-center">
        <div class="rounded-2xl p-3" style="background:rgba(255,255,255,.05)"><div class="text-[10px] text-white/45 font-bold">START</div><div class="font-mono font-bold text-sm">${num(o.start_count)}</div></div>
        <div class="rounded-2xl p-3" style="background:rgba(255,255,255,.05)"><div class="text-[10px] text-white/45 font-bold">DELIVERED</div><div class="font-mono font-bold text-sm text-emerald-300">${num(o.delivered)}</div></div>
        <div class="rounded-2xl p-3" style="background:rgba(255,255,255,.05)"><div class="text-[10px] text-white/45 font-bold">REMAINS</div><div class="font-mono font-bold text-sm text-amber-300">${num(o.remains)}</div></div></div>
    </div>
    <div class="glass rounded-3xl p-5 space-y-3 text-sm">
      <div class="flex justify-between gap-4"><span class="text-white/50">Link</span><span class="font-mono text-xs break-all text-right">${esc(o.link)}</span></div>
      <div class="flex justify-between"><span class="text-white/50">Quantity</span><b class="font-mono">${num(o.quantity)}</b></div>
      <div class="flex justify-between"><span class="text-white/50">Charge</span><b class="font-mono grad-text">${money(o.charge)}</b></div>
      ${parseFloat(o.refunded_amount) > 0 ? `<div class="flex justify-between"><span class="text-white/50">Refunded</span><b class="font-mono text-emerald-300">${money(o.refunded_amount)}</b></div>` : ''}
      <div class="flex justify-between"><span class="text-white/50">Refill</span><b>${o.refill ? (o.refill_days ? o.refill_days + ' days' : 'Lifetime') : 'Not available'}${o.refill_status !== 'None' ? ' · ' + esc(o.refill_status) : ''}</b></div>
      <div class="flex justify-between"><span class="text-white/50">Created</span><b class="font-mono text-xs">${esc(o.created_at)}</b></div>
      <div class="flex justify-between"><span class="text-white/50">Updated</span><b class="font-mono text-xs">${esc(o.updated_at)}</b></div>
    </div>
    <div class="grid grid-cols-2 gap-3">
      <button onclick="syncOrder(this,${o.id})" class="grad-btn rounded-2xl py-3.5 font-bold">🔄 Refresh</button>
      ${o.refill && ['Completed', 'Partial'].includes(o.status) ? `<button onclick="refillOrder(this,${o.id})" class="btn-soft rounded-2xl py-3.5 font-bold">♻️ Refill</button>` : `<button onclick="go('support')" class="btn-soft rounded-2xl py-3.5 font-bold">💬 Support</button>`}
    </div></div>`;
}
async function syncOrder(el, id) { await withBtn(el, async () => { try { const j = await api(`/api/order/${id}/sync`, { body: {} }); drawTrack(j.order); await refreshMe(); toast('Status: ' + j.order.status, 'ok'); } catch (e) { toast(e.message, 'err'); } }); }
async function refillOrder(el, id) { await withBtn(el, async () => { try { await api(`/api/order/${id}/refill`, { body: {} }); toast('Refill requested ♻️', 'ok'); vTrack(); } catch (e) { toast(e.message, 'err'); } }); }

/* ===================== FUNDS ===================== */
function vFunds() {
  setTop('Add Funds', 'Top up your wallet', false);
  const p = S.boot.pay, usd = S.me.currency === 'USD', L = S.boot.limits;
  const opt = (ico, title, sub, onclick, grad, off) => `
  <button ${off ? 'disabled' : ''} onclick="${onclick}" class="glass rounded-3xl p-4 text-left w-full card-hover flex items-center gap-4 ${off ? 'opacity-40' : ''}">
    <div class="w-14 h-14 rounded-2xl flex items-center justify-center text-2xl shrink-0" style="background:${grad}">${ico}</div>
    <div class="flex-1"><div class="font-extrabold">${title}</div><div class="text-xs text-white/55 mt-0.5">${sub}</div></div><span class="text-white/30 text-xl">›</span></button>`;
  view().innerHTML = `<div class="space-y-3 stagger">
    ${maintenanceBanner()}
    <div class="glass rounded-3xl p-5 flex items-center justify-between"><div><div class="text-[11px] text-white/50 font-bold tracking-widest">CURRENT BALANCE</div><div class="font-mono font-extrabold text-3xl mt-1">${money(S.me.balance)}</div></div><div class="text-4xl">👛</div></div>
    ${usd ? `<div class="rounded-2xl p-3.5 text-sm text-amber-200" style="background:rgba(251,191,36,.12);border:1px solid rgba(251,191,36,.3)">USD accounts support crypto payments only.</div>` : ''}
    ${opt('⚡', 'UPI · Instant', `Auto-verified · ₹${L.min} – ₹${L.max}`, "go('dep_auto')", 'linear-gradient(135deg,#059669,#10b981)', !p.upi_auto || usd)}
    ${opt('🏦', 'UPI · Manual', 'Pay & submit UTR · admin approval', "go('dep_manual')", 'linear-gradient(135deg,#2563eb,#38bdf8)', !p.upi_manual || usd)}
    ${opt('🪙', 'Crypto · USDT BEP-20', `${L.usdt_min} – ${L.usdt_max} USDT · admin approval`, "go('dep_usdt')", 'linear-gradient(135deg,#d97706,#facc15)', !p.usdt)}
    ${opt('🎁', 'Redeem gift card', 'Have a code? Add instantly', "go('gift')", 'linear-gradient(135deg,#c026d3,#ec4899)', false)}
    ${btn('🧾 Deposit & wallet history', "go('history')", 'btn-soft')}
  </div>`;
}
const quickAmts = [100, 250, 500, 1000, 2000, 5000];
function amountPicker(id) { return `<div class="grid grid-cols-3 gap-2 mt-3">${quickAmts.map(a => `<button onclick="document.getElementById('${id}').value=${a};haptic()" class="btn-soft rounded-xl py-2.5 text-sm font-bold font-mono">₹${num(a)}</button>`).join('')}</div>`; }

function vDepAuto() {
  setTop('UPI · Instant', 'Auto verified', true);
  const L = S.boot.limits;
  view().innerHTML = `<div class="glass rounded-3xl p-5 animate-rise">
    <div class="big-ico mx-auto mb-2">⚡</div><div class="font-extrabold text-xl">Enter amount</div><div class="text-xs text-white/50 mt-1">Min ₹${L.min} · Max ₹${L.max}</div>
    <div class="relative mt-4"><span class="absolute left-4 top-1/2 -translate-y-1/2 font-mono font-bold text-white/50 text-lg">₹</span>
      <input id="dAmt" type="number" inputmode="numeric" step="1" min="1" placeholder="0" class="field w-full rounded-2xl pl-10 pr-4 py-4 text-2xl font-mono font-bold"></div>
    ${amountPicker('dAmt')}
    <div class="mt-5">${btn('Generate QR', 'startAuto(this)', 'grad-btn')}</div></div>`;
}
let _autoTimer = null;
async function startAuto(el) {
  const amount = $('#dAmt').value;
  await withBtn(el, async () => {
    try { const j = await api('/api/deposit/upi_auto', { body: { amount } }); showQr(j.deposit); }
    catch (e) { toast(e.message, 'err'); }
  });
}
function qrHtml(d) {
  const src = d.qr || d.qr_fallback || '';
  if (!src) return '<span class="text-black text-xs">QR unavailable — use the UPI ID</span>';
  return `<img src="${esc(src)}" width="240" height="240" decoding="async" fetchpriority="high" alt="UPI QR" data-fb="${esc(d.qr_fallback || '')}" onerror="qrFail(this)" class="w-full h-full object-contain rounded-xl">`;
}
function qrFail(img) {
  const fb = img.dataset.fb;
  if (fb && !img.dataset.f) { img.dataset.f = 1; img.src = fb; return; }
  img.outerHTML = '<span class="text-black text-xs">QR unavailable — use the UPI ID</span>';
}
function showQr(d) {
  setTop('Scan & Pay', 'Deposit #' + d.id, true);
  view().innerHTML = `<div class="space-y-4 animate-rise">
    <div class="glass rounded-3xl p-5 text-center">
      <div class="text-xs text-white/55 font-bold">SCAN, ENTER &amp; PAY</div><div class="font-mono font-extrabold text-4xl grad-text mt-1">₹${esc(d.amount)}</div>
      <div class="bg-white rounded-3xl p-3 mx-auto mt-4 w-60 h-60 flex items-center justify-center shadow-2xl shadow-violet-500/30">${qrHtml(d)}</div>
      <div class="text-[11px] text-white/45 mt-3 px-2">Scan with any UPI app, enter ₹${esc(d.amount)} and pay. The exact amount you pay is credited.</div>
      <div class="mt-3 inline-flex items-center gap-2 rounded-full px-4 py-2 text-sm font-mono font-bold" style="background:var(--bg);box-shadow:var(--inset-sm);color:#fcd34d">⏱ <span id="cd">--:--</span></div>
    </div>
    <div class="glass rounded-3xl p-5 space-y-3 text-sm">
      <div class="flex justify-between items-center gap-3"><span class="text-white/50">Payee</span><b>${esc(d.payee)}</b></div>
      <div class="flex justify-between items-center gap-3"><span class="text-white/50">UPI ID</span><button onclick="copyTxt('${esc(d.upi_id)}')" class="font-mono font-bold text-brand-400 break-all text-right">${esc(d.upi_id)} 📋</button></div>
      <div class="flex justify-between items-center gap-3"><span class="text-white/50">Transaction ref</span><button onclick="copyTxt('${esc(d.reference)}')" class="font-mono font-bold text-xs text-brand-400">${esc(d.reference)} 📋</button></div>
    </div>
    <a href="${esc(d.upi_link)}" class="block text-center btn-soft rounded-2xl py-3.5 font-bold text-sm">📱 Open UPI app</a>
    <div class="grid grid-cols-2 gap-3"><button id="chkBtn" onclick="checkAuto(this,${d.id})" class="grad-btn rounded-2xl py-3.5 font-bold">✅ Check payment</button>
      <button onclick="cancelDep(${d.id})" class="btn-soft rounded-2xl py-3.5 font-bold text-rose-300">Cancel</button></div>
    <div class="text-[11px] text-white/40 text-center">Pay using the QR above, then tap “Check payment”.</div></div>`;
  clearInterval(_autoTimer);
  const end = d.expires_at * 1000;
  _autoTimer = setInterval(() => {
    const el = $('#cd'); if (!el) return clearInterval(_autoTimer);
    const s = Math.max(0, Math.floor((end - Date.now()) / 1000));
    el.textContent = String(Math.floor(s / 60)).padStart(2, '0') + ':' + String(s % 60).padStart(2, '0');
    if (!s) { clearInterval(_autoTimer); el.textContent = 'Expired'; }
  }, 1000);
}
async function checkAuto(el, id) {
  await withBtn(el, async () => {
    try {
      const j = await api('/api/deposit/upi_auto/check', { body: { id } });
      if (j.state === 'approved') { clearInterval(_autoTimer); await refreshMe(); notifyH('success'); toast(j.message, 'ok'); celebrate('Payment successful!', j.message); }
      else toast(j.message, (j.state === 'pending' || j.state === 'unpaid') ? 'info' : 'err');
      if (['expired', 'duplicate', 'mismatch', 'failed'].includes(j.state)) { clearInterval(_autoTimer); }
    } catch (e) { toast(e.message, 'err'); }
  });
}
async function cancelDep(id) { try { await api('/api/deposit/cancel', { body: { id } }); clearInterval(_autoTimer); toast('Deposit cancelled', 'info'); go('funds'); } catch (e) { toast(e.message, 'err'); } }
function celebrate(t, s) {
  view().innerHTML = `<div class="text-center pt-16 animate-pop"><div class="big-ico mx-auto mb-3">🎉</div><div class="font-extrabold text-2xl grad-text">${esc(t)}</div><div class="text-white/60 text-sm mt-2 px-6">${esc(s)}</div>
    <div class="font-mono font-extrabold text-4xl mt-5">${money(S.me.balance)}</div><div class="grid grid-cols-2 gap-3 mt-8 px-2">${btn('Browse services', "go('services')")}${btn('Home', "go('home')", 'btn-soft')}</div></div>`;
}
function copyTxt(t) {
  try { navigator.clipboard.writeText(t); } catch (e) { const i = document.createElement('input'); i.value = t; document.body.appendChild(i); i.select(); document.execCommand('copy'); i.remove(); }
  toast('Copied', 'ok'); haptic();
}

function vDepManual() {
  setTop('UPI · Manual', 'Admin approval', true);
  const L = S.boot.limits;
  view().innerHTML = `<div class="glass rounded-3xl p-5 animate-rise"><div class="big-ico mx-auto mb-2">🏦</div><div class="font-extrabold text-xl">Enter amount</div><div class="text-xs text-white/50 mt-1">Min ₹${L.min} · Max ₹${L.max}</div>
    <div class="relative mt-4"><span class="absolute left-4 top-1/2 -translate-y-1/2 font-mono font-bold text-white/50 text-lg">₹</span>
    <input id="mAmt" type="number" inputmode="numeric" step="1" min="1" placeholder="0" class="field w-full rounded-2xl pl-10 pr-4 py-4 text-2xl font-mono font-bold"></div>${amountPicker('mAmt')}
    <div class="mt-5">${btn('Continue', 'startManual(this)')}</div></div>`;
}
async function startManual(el) {
  const amount = $('#mAmt').value;
  await withBtn(el, async () => {
    try {
      const d = (await api('/api/deposit/upi_manual', { body: { amount } })).deposit;
      setTop('Pay & Submit UTR', 'Deposit #' + d.id, true);
      view().innerHTML = `<div class="space-y-4 animate-rise">
        <div class="glass rounded-3xl p-5 text-center"><div class="text-xs text-white/55 font-bold">PAY EXACTLY</div><div class="font-mono font-extrabold text-4xl grad-text mt-1">₹${esc(d.amount)}</div>
        <div class="bg-white rounded-3xl p-3 mx-auto mt-4 w-56 h-56 flex items-center justify-center shadow-2xl shadow-sky-500/30">${qrHtml(d)}</div></div>
        <div class="glass rounded-3xl p-5 space-y-3 text-sm"><div class="flex justify-between gap-3"><span class="text-white/50">UPI ID</span><button onclick="copyTxt('${esc(d.upi_id)}')" class="font-mono font-bold text-brand-400 break-all text-right">${esc(d.upi_id)} 📋</button></div>
        <div class="flex justify-between"><span class="text-white/50">Payee</span><b>${esc(d.payee)}</b></div></div>
        <div class="glass rounded-3xl p-5"><label class="text-xs font-bold text-white/60 ml-1">After paying, enter your UTR / Ref No.</label>
          <input id="mUtr" inputmode="numeric" placeholder="e.g. 412345678901" class="field w-full rounded-2xl px-4 py-3.5 mt-1.5 font-mono">
          <div class="mt-4">${btn('Submit for approval', `submitUtr(this,${d.id})`)}</div></div>
        <button onclick="cancelDep(${d.id})" class="w-full text-center text-rose-300 text-sm font-bold py-2">Cancel deposit</button></div>`;
    } catch (e) { toast(e.message, 'err'); }
  });
}
async function submitUtr(el, id) {
  const utr = $('#mUtr').value.trim();
  await withBtn(el, async () => { try { await api('/api/deposit/upi_manual/utr', { body: { id, utr } }); celebrate('Request submitted', 'An admin will verify and credit your wallet shortly.'); } catch (e) { toast(e.message, 'err'); } });
}

async function vDepUsdt() {
  setTop('Crypto · USDT', 'BEP-20 network', true);
  view().innerHTML = skel(2, 'h-40');
  try {
    const i = await api('/api/deposit/usdt/info');
    view().innerHTML = `<div class="space-y-4 animate-rise">
      <div class="glass rounded-3xl p-5"><div class="flex items-center gap-3"><div class="w-12 h-12 rounded-2xl flex items-center justify-center text-2xl" style="background:linear-gradient(135deg,#d97706,#facc15)">🪙</div><div><div class="font-extrabold">Send USDT (BEP-20)</div><div class="text-xs text-white/50">1 USDT ≈ ₹${esc(parseFloat(i.rate).toFixed(2))}</div></div></div>
        <div class="rounded-2xl p-3.5 mt-4 font-mono text-[12px] break-all" style="background:rgba(255,255,255,.06)">${esc(i.address)}</div>
        <button onclick="copyTxt('${esc(i.address)}')" class="btn-soft w-full rounded-xl py-2.5 mt-2 text-sm font-bold">📋 Copy address</button>
        <div class="text-[11px] text-amber-300 mt-3">⚠️ Only BEP-20 (BSC). Min ${esc(i.min)} · Max ${esc(i.max)} USDT.</div></div>
      <div class="glass rounded-3xl p-5"><label class="text-xs font-bold text-white/60 ml-1">USDT amount you sent</label>
        <input id="uAmt" type="number" inputmode="decimal" oninput="usdtPrev(${i.rate})" placeholder="0.00" class="field w-full rounded-2xl px-4 py-3.5 mt-1.5 font-mono text-lg">
        <div class="text-xs text-white/50 mt-2">You will receive ≈ <b id="uPrev" class="font-mono grad-text">₹0.00</b></div>
        <label class="text-xs font-bold text-white/60 ml-1 mt-4 block">Transaction hash (TXID)</label>
        <input id="uTx" placeholder="0x…" class="field w-full rounded-2xl px-4 py-3.5 mt-1.5 font-mono text-xs">
        <div class="mt-4">${btn('Submit for approval', 'submitUsdt(this)')}</div></div></div>`;
  } catch (e) { view().innerHTML = empty('🛠️', esc(e.message)); }
}
function usdtPrev(r) { $('#uPrev').textContent = '₹' + ((parseFloat($('#uAmt').value || 0)) * r).toFixed(2); }
async function submitUsdt(el) {
  const amount = $('#uAmt').value, txid = $('#uTx').value.trim();
  await withBtn(el, async () => {
    try {
      const d = (await api('/api/deposit/usdt', { body: { amount } })).deposit;
      await api('/api/deposit/usdt/txid', { body: { id: d.id, txid } });
      celebrate('USDT submitted', `Credit ₹${d.inr} after admin verification.`);
    } catch (e) { toast(e.message, 'err'); }
  });
}

/* ===================== HISTORY / GIFT / SUPPORT ===================== */
async function vHistory() {
  setTop('History', 'Deposits & wallet', true);
  view().innerHTML = skel(5, 'h-16');
  try {
    const [d, t] = await Promise.all([api('/api/deposits'), api('/api/transactions')]);
    view().innerHTML = `<div class="space-y-5">
      <div><h3 class="font-extrabold mb-2">Deposits</h3><div class="space-y-2 stagger">${d.deposits.length ? d.deposits.map(x => `
        <div class="glass rounded-2xl p-3.5 flex items-center justify-between gap-3"><div><div class="font-mono font-bold">₹${esc(x.amount)} <span class="text-[10px] text-white/40">${esc(x.method.replace('_', ' '))}</span></div><div class="text-[11px] text-white/40 font-mono">#${x.id} · ${esc(x.created_at)}</div></div>${badge(x.status)}</div>`).join('') : '<div class="text-sm text-white/40">No deposits yet.</div>'}</div></div>
      <div><h3 class="font-extrabold mb-2">Wallet transactions</h3><div class="space-y-2 stagger">${t.transactions.length ? t.transactions.map(x => {
        const pos = parseFloat(x.amount) >= 0; return `<div class="glass rounded-2xl p-3.5 flex items-center justify-between gap-3"><div class="min-w-0"><div class="font-bold text-sm truncate">${esc(x.type)}</div><div class="text-[11px] text-white/40 truncate">${esc(x.description || '')}</div><div class="text-[10px] text-white/30 font-mono">${esc(x.created_at)}</div></div><div class="font-mono font-extrabold ${pos ? 'text-emerald-300' : 'text-rose-300'}">${pos ? '+' : ''}${money(x.amount)}</div></div>`; }).join('') : '<div class="text-sm text-white/40">No transactions yet.</div>'}</div></div></div>`;
  } catch (e) { view().innerHTML = empty('🛠️', esc(e.message)); }
}
function vGift() {
  setTop('Gift Card', 'Redeem a code', true);
  view().innerHTML = `<div class="glass rounded-3xl p-6 text-center animate-rise"><div class="big-ico mx-auto mb-2">🎁</div><div class="font-extrabold text-xl">Redeem gift card</div>
    <input id="gCode" placeholder="ENTER CODE" class="field w-full rounded-2xl px-4 py-4 mt-5 text-center font-mono font-bold tracking-widest uppercase">
    <div class="mt-4">${btn('Redeem 🎉', 'redeemGift(this)')}</div></div>`;
}
async function redeemGift(el) {
  const code = $('#gCode').value.trim();
  await withBtn(el, async () => { try { const j = await api('/api/gift', { body: { code } }); await refreshMe(); celebrate('Gift redeemed!', money(j.amount) + ' added to your wallet.'); } catch (e) { toast(e.message, 'err'); } });
}
function vSupport() {
  setTop('Support', 'We are here to help', true);
  const s = S.boot.support, u = String(s.username || '').replace('@', '');
  view().innerHTML = `<div class="space-y-3 stagger"><div class="glass rounded-3xl p-6 text-center"><div class="big-ico mx-auto mb-2">💬</div><div class="font-extrabold text-xl">Need help?</div><div class="text-sm text-white/55 mt-1">Hours: 10 AM – 10 PM IST</div></div>
    <button onclick="openLink('https://t.me/${esc(u)}')" class="glass rounded-2xl p-4 w-full text-left card-hover flex items-center gap-3"><span class="text-2xl">📞</span><div><div class="font-bold">Contact support</div><div class="text-xs text-white/50">${esc(s.username)}</div></div></button>
    <button onclick="openLink('${esc(s.channel)}')" class="glass rounded-2xl p-4 w-full text-left card-hover flex items-center gap-3"><span class="text-2xl">📢</span><div><div class="font-bold">Updates channel</div><div class="text-xs text-white/50 break-all">${esc(s.channel)}</div></div></button></div>`;
}
function openLink(u) { try { /t\.me\//.test(u) ? tg.openTelegramLink(u) : tg.openLink(u); } catch (e) { window.open(u, '_blank'); } }

/* ===================== PROFILE ===================== */
async function vProfile() {
  setTop('Profile', 'Your account', false);
  view().innerHTML = skel(3, 'h-32');
  try {
    const j = await api('/api/profile'), p = j.profile, st = j.stats, r = j.referral;
    view().innerHTML = `<div class="space-y-4 stagger">
      <div class="glass rounded-3xl p-5 flex items-center gap-4"><div class="w-16 h-16 rounded-2xl grad-btn flex items-center justify-center text-3xl font-extrabold">${esc((p.first_name || 'U')[0].toUpperCase())}</div>
        <div class="min-w-0"><div class="font-extrabold text-lg truncate">${esc(p.first_name)}</div><div class="text-xs text-white/50 font-mono">ID ${p.id}${p.username ? ' · @' + esc(p.username) : ''}</div><div class="text-[11px] text-white/35">Joined ${esc(p.joined)}</div></div></div>
      <div class="grid grid-cols-2 gap-3">
        <div class="glass rounded-2xl p-4"><div class="text-[11px] text-white/50 font-bold">BALANCE</div><div class="font-mono font-extrabold text-lg mt-1">${money(p.balance)}</div></div>
        <div class="glass rounded-2xl p-4"><div class="text-[11px] text-white/50 font-bold">SPENT</div><div class="font-mono font-extrabold text-lg mt-1">${money(p.total_spent)}</div></div>
        <div class="glass rounded-2xl p-4"><div class="text-[11px] text-white/50 font-bold">DEPOSITED</div><div class="font-mono font-extrabold text-lg mt-1">${money(p.total_deposited)}</div></div>
        <div class="glass rounded-2xl p-4"><div class="text-[11px] text-white/50 font-bold">REFERRAL EARNED</div><div class="font-mono font-extrabold text-lg mt-1 text-emerald-300">${money(p.referral_earnings)}</div></div></div>
      <div class="glass rounded-3xl p-5"><div class="font-extrabold mb-3">📊 My stats</div><div class="grid grid-cols-5 gap-2 text-center">
        ${[['Total', st.total, ''], ['Done', st.completed, 'text-emerald-300'], ['Active', st.active, 'text-sky-300'], ['Partial', st.partial, 'text-orange-300'], ['Failed', st.cancelled, 'text-rose-300']].map(x => `<div class="rounded-xl py-2.5" style="background:rgba(255,255,255,.05)"><div class="font-mono font-extrabold ${x[2]}">${x[1]}</div><div class="text-[9px] text-white/45 font-bold">${x[0]}</div></div>`).join('')}</div></div>
      <div class="glass rounded-3xl p-5"><div class="font-extrabold">🤝 Refer & Earn</div><div class="text-xs text-white/55 mt-1">${r.enabled ? `Earn <b>${esc(r.percent)}%</b> on every deposit your friends make.` : 'Referral program is currently disabled.'} · ${r.count} referrals</div>
        <div class="rounded-xl p-3 mt-3 font-mono text-[11px] break-all" style="background:rgba(255,255,255,.06)">${esc(r.link)}</div>
        <div class="grid grid-cols-2 gap-2 mt-3"><button onclick="copyTxt('${esc(r.link)}')" class="btn-soft rounded-xl py-2.5 text-sm font-bold">📋 Copy</button>
          <button onclick="openLink('https://t.me/share/url?url=${encodeURIComponent(r.link)}&text=${encodeURIComponent('Join this amazing SMM bot!')}')" class="grad-btn rounded-xl py-2.5 text-sm font-bold">📤 Share</button></div></div>
      <div class="glass rounded-3xl p-5 flex items-center justify-between"><div><div class="font-extrabold">💱 Currency</div><div class="text-xs text-white/50">Display prices in</div></div>
        <div class="flex rounded-xl overflow-hidden border border-white/10">${['INR', 'USD'].map(c => `<button onclick="setCur('${c}')" class="px-4 py-2 text-sm font-bold ${p.currency === c ? 'chip-on' : 'text-white/60'}">${c}</button>`).join('')}</div></div>
      <div class="grid grid-cols-2 gap-3">${btn('🎁 Gift card', "go('gift')", 'btn-soft')}${btn('🧾 History', "go('history')", 'btn-soft')}</div>
    </div>`;
  } catch (e) { view().innerHTML = empty('🛠️', esc(e.message)); }
}
async function setCur(c) { try { await api('/api/currency', { body: { currency: c } }); S.me.currency = c; $('#balPill').textContent = money(S.me.balance); toast('Currency: ' + c, 'ok'); vProfile(); } catch (e) { toast(e.message, 'err'); } }

/* ===================== ADMIN ===================== */
function adminGuard() { if (!S.me.is_admin) { go('home'); return false; } return true; }
const A_TILES = [['📦', 'Orders', 'admin_orders'], ['👥', 'Users', 'admin_users'], ['💳', 'Deposits', 'admin_deposits'], ['🛍️', 'Services', 'admin_services'],
  ['➕', 'Add service', 'admin_add_service'], ['🏷️', 'Categories', 'admin_categories'], ['🔌', 'Providers', 'admin_providers'], ['🎁', 'Gift cards', 'admin_gifts'],
  ['📣', 'Broadcast', 'admin_broadcast'], ['⚙️', 'Settings', 'admin_settings']];
async function vAdmin() {
  if (!adminGuard()) return; setTop('Admin Panel', 'Control center', false);
  view().innerHTML = skel(3, 'h-32');
  try {
    const d = (await api('/api/admin/dashboard')).dashboard;
    const stat = (l, v, c = '') => `<div class="glass rounded-2xl p-3.5"><div class="text-[10px] text-white/45 font-bold tracking-wide">${l}</div><div class="font-mono font-extrabold text-lg mt-0.5 ${c}">${v}</div></div>`;
    view().innerHTML = `<div class="space-y-4 stagger">
      <div class="rounded-[28px] p-5 relative overflow-hidden" style="background:linear-gradient(135deg,#0f172a,#312e81 55%,#6d28d9)"><div class="absolute -right-8 -top-8 w-36 h-36 rounded-full bg-fuchsia-400/25 blur-2xl"></div>
        <div class="text-white/60 text-[11px] font-bold tracking-widest">REVENUE</div><div class="font-mono font-extrabold text-3xl mt-1">₹${num(Math.round(d.revenue))}</div>
        <div class="flex gap-5 mt-4 text-xs"><div><div class="text-white/50">Today orders</div><b class="font-mono">${d.today_orders}</b></div><div><div class="text-white/50">Today revenue</div><b class="font-mono">₹${num(Math.round(d.today_revenue))}</b></div><div><div class="text-white/50">New users</div><b class="font-mono">${d.today_users}</b></div></div></div>
      ${d.pending_deposits ? `<button onclick="go('admin_deposits',{status:'PENDING'})" class="w-full rounded-2xl p-4 flex items-center gap-3 text-left" style="background:linear-gradient(135deg,rgba(251,191,36,.2),rgba(249,115,22,.15));border:1px solid rgba(251,191,36,.35)"><span class="text-2xl">⏳</span><div class="flex-1"><b>${d.pending_deposits} deposit(s) awaiting approval</b><div class="text-xs text-white/60">Tap to review</div></div><span>›</span></button>` : ''}
      <div class="grid grid-cols-3 gap-2.5">${stat('USERS', num(d.users))}${stat('ORDERS', num(d.orders))}${stat('DEPOSITS', '₹' + num(Math.round(d.deposits)), 'text-emerald-300')}
        ${stat('PENDING', d.pending, 'text-amber-300')}${stat('PROCESSING', d.processing, 'text-sky-300')}${stat('COMPLETED', d.completed, 'text-emerald-300')}
        ${stat('PARTIAL', d.partial, 'text-orange-300')}${stat('CANCELLED', d.cancelled + d.failed, 'text-rose-300')}${stat('REFUNDS', '₹' + num(Math.round(d.refunds)))}
        ${stat('WALLETS', '₹' + num(Math.round(d.wallets)))}${stat('SERVICES', d.services)}${stat('PROVIDERS', d.providers)}</div>
      <div class="grid grid-cols-2 gap-3">${A_TILES.map(t => `<button onclick="go('${t[2]}')" class="glass rounded-2xl p-4 flex items-center gap-3 card-hover"><span class="text-2xl">${t[0]}</span><span class="font-bold text-sm">${t[1]}</span></button>`).join('')}</div></div>`;
  } catch (e) { view().innerHTML = empty('🛠️', esc(e.message)); }
}
function pager(fnName, page, pages) { return pages > 1 ? `<div class="flex items-center justify-between mt-4"><button ${page <= 1 ? 'disabled' : ''} onclick="${fnName}(${page - 1})" class="btn-soft rounded-xl px-4 py-2.5 font-bold text-sm disabled:opacity-30">‹ Prev</button><span class="text-xs text-white/50 font-mono">${page} / ${pages}</span><button ${page >= pages ? 'disabled' : ''} onclick="${fnName}(${page + 1})" class="btn-soft rounded-xl px-4 py-2.5 font-bold text-sm disabled:opacity-30">Next ›</button></div>` : ''; }

/* --- admin orders --- */
function vAdminOrders() { if (!adminGuard()) return; setTop('Orders', 'All orders', true); S.ao = S.ao || { f: 'all', q: '' }; drawAO(1); }
async function drawAO(page) {
  const f = S.ao.f, filters = [['all', 'All'], ['active', 'Active'], ['completed', 'Done'], ['partial', 'Partial'], ['cancelled', 'Cancelled']];
  view().innerHTML = `<div class="space-y-3"><input id="aoq" value="${esc(S.ao.q)}" placeholder="Search order ID / user ID / link…" onkeydown="if(event.key==='Enter'){S.ao.q=this.value;drawAO(1)}" class="field w-full rounded-2xl px-4 py-3 text-sm">
    <div class="flex gap-2 overflow-x-auto pb-1">${filters.map(x => `<button onclick="S.ao.f='${x[0]}';drawAO(1)" class="shrink-0 px-4 py-2 rounded-full text-xs font-bold border border-white/10 ${f === x[0] ? 'chip-on' : 'btn-soft text-white/70'}">${x[1]}</button>`).join('')}</div>
    <div id="aol">${skel(4)}</div></div>`;
  try {
    const j = await api(`/api/admin/orders?status=${f}&page=${page}&q=${encodeURIComponent(S.ao.q)}`);
    $('#aol').innerHTML = j.orders.length ? `<div class="space-y-2.5 stagger">${j.orders.map(o => `<button onclick="go('admin_order',{id:${o.id}})" class="glass rounded-2xl p-3.5 text-left w-full card-hover block"><div class="flex justify-between gap-2"><div class="min-w-0"><div class="text-[11px] font-mono text-white/35">#${o.id} · 👤 ${o.user_id}</div><div class="font-bold text-sm truncate">${esc(o.service_name)}</div></div>${badge(o.status)}</div><div class="flex justify-between text-xs mt-2"><span class="text-white/50">Qty <b class="font-mono">${num(o.quantity)}</b></span><b class="font-mono grad-text">₹${esc(o.charge)}</b></div></button>`).join('')}</div>${pager('drawAO', page, j.pages)}` : empty('📭', 'No orders');
  } catch (e) { $('#aol').innerHTML = empty('🛠️', esc(e.message)); }
}
async function vAdminOrder() {
  if (!adminGuard()) return; setTop('Order #' + S.params.id, 'Manage', true); view().innerHTML = skel(3, 'h-32');
  try {
    const o = (await api('/api/admin/order/' + S.params.id)).order;
    view().innerHTML = `<div class="space-y-4 stagger">
      <div class="glass rounded-3xl p-5 space-y-2.5 text-sm"><div class="flex justify-between items-start gap-3"><div class="font-extrabold">${esc(o.service_name)}</div>${badge(o.status)}</div>
        <div class="flex justify-between"><span class="text-white/50">User</span><button onclick="go('admin_user',{id:${o.user_id}})" class="font-mono font-bold text-brand-400">${o.user_id} ›</button></div>
        <div class="flex justify-between gap-4"><span class="text-white/50">Link</span><span class="font-mono text-xs break-all text-right">${esc(o.link)}</span></div>
        <div class="flex justify-between"><span class="text-white/50">Provider order</span><b class="font-mono">${esc(o.provider_order_id || '—')}</b></div>
        <div class="flex justify-between"><span class="text-white/50">Quantity / Remains</span><b class="font-mono">${num(o.quantity)} / ${num(o.remains)}</b></div>
        <div class="flex justify-between"><span class="text-white/50">Charge</span><b class="font-mono grad-text">₹${esc(o.charge)}</b></div>
        <div class="flex justify-between"><span class="text-white/50">Refund</span><b class="font-mono">${esc(o.refund_state)} · ₹${esc(o.refunded_amount)}</b></div>
        <div class="flex justify-between"><span class="text-white/50">Refill</span><b>${esc(o.refill_status)}</b></div></div>
      <div class="glass rounded-3xl p-5"><div class="font-bold mb-3 text-sm">Set status</div><div class="grid grid-cols-3 gap-2">${['Pending', 'Processing', 'In Progress', 'Completed', 'Partial'].map(s => `<button onclick="aoStatus(${o.id},'${s}')" class="btn-soft rounded-xl py-2.5 text-xs font-bold">${s}</button>`).join('')}</div></div>
      <div class="grid grid-cols-2 gap-3"><button onclick="aoAct(this,${o.id},'sync')" class="grad-btn rounded-2xl py-3.5 font-bold">🔄 Sync</button><button onclick="aoAct(this,${o.id},'refill')" class="btn-soft rounded-2xl py-3.5 font-bold">♻️ Refill</button>
        <button onclick="aoAct(this,${o.id},'refill_cancel')" class="btn-soft rounded-2xl py-3.5 font-bold text-xs">Cancel refill</button><button onclick="aoCancel(${o.id},'${o.charge}','${o.refunded_amount}')" class="rounded-2xl py-3.5 font-bold text-rose-300" style="background:rgba(244,63,94,.14);border:1px solid rgba(244,63,94,.3)">🚫 Cancel</button></div></div>`;
  } catch (e) { view().innerHTML = empty('🛠️', esc(e.message)); }
}
async function aoStatus(id, status) { try { await api(`/api/admin/order/${id}/status`, { body: { status } }); toast('Status updated', 'ok'); vAdminOrder(); } catch (e) { toast(e.message, 'err'); } }
async function aoAct(el, id, a) { await withBtn(el, async () => { try { await api(`/api/admin/order/${id}/${a}`, { body: {} }); toast('Done', 'ok'); vAdminOrder(); } catch (e) { toast(e.message, 'err'); } }); }
function aoCancel(id, charge, refunded) {
  sheet(`<div class="font-extrabold text-xl mb-1">Cancel order #${id}</div><div class="text-xs text-white/50 mb-4">Refundable: ₹${(parseFloat(charge) - parseFloat(refunded)).toFixed(2)}</div>
  <div class="space-y-2.5"><button onclick="aoDoCancel(${id},'full')" class="grad-btn w-full rounded-2xl py-3.5 font-bold">Cancel + full refund</button>
  <button onclick="aoDoCancel(${id},'none')" class="btn-soft w-full rounded-2xl py-3.5 font-bold">Cancel without refund</button>
  <div class="flex gap-2"><input id="prAmt" type="number" inputmode="decimal" placeholder="Partial refund ₹" class="field flex-1 rounded-2xl px-4 py-3 text-sm font-mono"><button onclick="aoDoCancel(${id},'partial')" class="btn-soft rounded-2xl px-5 font-bold">Refund</button></div></div>`);
}
async function aoDoCancel(id, mode) { try { await api(`/api/admin/order/${id}/cancel`, { body: { mode, amount: mode === 'partial' ? $('#prAmt').value : '' } }); closeSheet(); toast('Done', 'ok'); vAdminOrder(); } catch (e) { toast(e.message, 'err'); } }

/* --- admin users --- */
function vAdminUsers() { if (!adminGuard()) return; setTop('Users', 'Search & manage', true); S.auq = S.auq || ''; drawAU(1); }
async function drawAU(page) {
  view().innerHTML = `<div class="space-y-3"><input value="${esc(S.auq)}" placeholder="User ID / @username / name" onkeydown="if(event.key==='Enter'){S.auq=this.value;drawAU(1)}" class="field w-full rounded-2xl px-4 py-3 text-sm"><div id="aul">${skel(4)}</div></div>`;
  try {
    const j = await api(`/api/admin/users?page=${page}&q=${encodeURIComponent(S.auq)}`);
    $('#aul').innerHTML = j.users.length ? `<div class="text-xs text-white/40 mb-2">${j.total} users</div><div class="space-y-2.5 stagger">${j.users.map(u => `<button onclick="go('admin_user',{id:${u.id}})" class="glass rounded-2xl p-3.5 text-left w-full card-hover flex items-center gap-3"><div class="w-11 h-11 rounded-xl grad-btn flex items-center justify-center font-extrabold">${esc((u.first_name || 'U')[0].toUpperCase())}</div><div class="flex-1 min-w-0"><div class="font-bold text-sm truncate">${esc(u.first_name)} ${u.banned ? '🚫' : ''}</div><div class="text-[11px] text-white/40 font-mono">${u.id}${u.username ? ' · @' + esc(u.username) : ''}</div></div><b class="font-mono text-sm">₹${esc(u.balance)}</b></button>`).join('')}</div>${pager('drawAU', page, j.pages)}` : empty('🔍', 'No users found');
  } catch (e) { $('#aul').innerHTML = empty('🛠️', esc(e.message)); }
}
async function vAdminUser() {
  if (!adminGuard()) return; setTop('User ' + S.params.id, 'Profile', true); view().innerHTML = skel(3, 'h-32');
  try {
    const j = await api('/api/admin/user/' + S.params.id), u = j.user;
    view().innerHTML = `<div class="space-y-4 stagger">
      <div class="glass rounded-3xl p-5"><div class="flex items-center gap-4"><div class="w-14 h-14 rounded-2xl grad-btn flex items-center justify-center text-2xl font-extrabold">${esc((u.first_name || 'U')[0].toUpperCase())}</div><div><div class="font-extrabold text-lg">${esc(u.first_name)} ${u.banned ? '🚫' : ''}</div><div class="text-xs text-white/50 font-mono">${u.id}${u.username ? ' · @' + esc(u.username) : ''}</div></div></div>
        <div class="grid grid-cols-2 gap-2 mt-4 text-center">${[['Balance', '₹' + u.balance], ['Spent', '₹' + u.total_spent], ['Deposited', '₹' + u.total_deposited], ['Orders', u.orders], ['Referrals', u.referrals], ['Ref. earned', '₹' + u.referral_earnings]].map(x => `<div class="rounded-xl p-2.5" style="background:rgba(255,255,255,.05)"><div class="text-[10px] text-white/45 font-bold">${x[0]}</div><div class="font-mono font-bold text-sm">${esc(x[1])}</div></div>`).join('')}</div></div>
      <div class="grid grid-cols-3 gap-2.5"><button onclick="auBal(${u.id})" class="grad-btn rounded-2xl py-3 font-bold text-xs">💰 Balance</button><button onclick="auMsg(${u.id})" class="btn-soft rounded-2xl py-3 font-bold text-xs">✉️ Message</button>
        <button onclick="auBan(${u.id})" class="rounded-2xl py-3 font-bold text-xs ${u.banned ? 'btn-soft' : 'text-rose-300'}" ${u.banned ? '' : 'style="background:rgba(244,63,94,.14);border:1px solid rgba(244,63,94,.3)"'}>${u.banned ? '✅ Unban' : '🚫 Ban'}</button></div>
      <div><div class="font-extrabold mb-2 text-sm">Recent orders</div><div class="space-y-2">${j.recent_orders.length ? j.recent_orders.map(o => `<button onclick="go('admin_order',{id:${o.id}})" class="glass rounded-xl p-3 w-full text-left flex justify-between items-center gap-2"><div class="min-w-0"><div class="text-[11px] font-mono text-white/35">#${o.id}</div><div class="text-sm font-bold truncate">${esc(o.service_name)}</div></div>${badge(o.status)}</button>`).join('') : '<div class="text-sm text-white/40">None</div>'}</div></div>
      <div><div class="font-extrabold mb-2 text-sm">Recent transactions</div><div class="space-y-2">${j.recent_tx.map(t => `<div class="glass rounded-xl p-3 flex justify-between gap-2 text-sm"><div class="min-w-0"><div class="font-bold truncate">${esc(t.type)}</div><div class="text-[10px] text-white/35">${esc(t.created_at)}</div></div><b class="font-mono ${parseFloat(t.amount) >= 0 ? 'text-emerald-300' : 'text-rose-300'}">₹${esc(t.amount)}</b></div>`).join('') || '<div class="text-sm text-white/40">None</div>'}</div></div></div>`;
  } catch (e) { view().innerHTML = empty('🛠️', esc(e.message)); }
}
function auBal(id) {
  sheet(`<div class="font-extrabold text-xl mb-4">Adjust balance</div><div class="space-y-3">
    <div class="grid grid-cols-2 gap-2"><button id="bAdd" onclick="window._ba='add';this.classList.add('chip-on');$('#bDed').classList.remove('chip-on')" class="btn-soft chip-on rounded-xl py-2.5 font-bold">➕ Add</button><button id="bDed" onclick="window._ba='deduct';this.classList.add('chip-on');$('#bAdd').classList.remove('chip-on')" class="btn-soft rounded-xl py-2.5 font-bold">➖ Deduct</button></div>
    <input id="bAmt" type="number" inputmode="decimal" placeholder="Amount ₹" class="field w-full rounded-2xl px-4 py-3.5 font-mono"><input id="bWhy" placeholder="Reason" class="field w-full rounded-2xl px-4 py-3.5 text-sm">
    <button onclick="auBalGo(this,${id})" class="grad-btn w-full rounded-2xl py-3.5 font-bold">Apply</button></div>`); window._ba = 'add';
}
async function auBalGo(el, id) { await withBtn(el, async () => { try { await api(`/api/admin/user/${id}/balance`, { body: { action: window._ba, amount: $('#bAmt').value, reason: $('#bWhy').value } }); closeSheet(); toast('Balance updated', 'ok'); vAdminUser(); } catch (e) { toast(e.message, 'err'); } }); }
function auMsg(id) { sheet(`<div class="font-extrabold text-xl mb-4">Message user</div><textarea id="mTxt" rows="4" placeholder="Write a message…" class="field w-full rounded-2xl px-4 py-3 text-sm"></textarea><div class="mt-3"><button onclick="auMsgGo(this,${id})" class="grad-btn w-full rounded-2xl py-3.5 font-bold">Send ✉️</button></div>`); }
async function auMsgGo(el, id) { await withBtn(el, async () => { try { await api(`/api/admin/user/${id}/message`, { body: { text: $('#mTxt').value } }); closeSheet(); toast('Sent', 'ok'); } catch (e) { toast(e.message, 'err'); } }); }
async function auBan(id) { try { const j = await api(`/api/admin/user/${id}/ban`, { body: {} }); toast(j.banned ? 'User banned' : 'User unbanned', 'ok'); vAdminUser(); } catch (e) { toast(e.message, 'err'); } }

/* --- admin deposits --- */
function vAdminDeposits() { if (!adminGuard()) return; setTop('Deposits', 'Approve & review', true); S.adf = S.params.status || S.adf || ''; drawAD(1); }
async function drawAD(page) {
  const f = S.adf, fl = [['', 'All'], ['PENDING', 'Pending'], ['APPROVED', 'Approved'], ['REJECTED', 'Rejected'], ['EXPIRED', 'Expired']];
  view().innerHTML = `<div class="space-y-3"><div class="flex gap-2 overflow-x-auto pb-1">${fl.map(x => `<button onclick="S.adf='${x[0]}';drawAD(1)" class="shrink-0 px-4 py-2 rounded-full text-xs font-bold border border-white/10 ${f === x[0] ? 'chip-on' : 'btn-soft text-white/70'}">${x[1]}</button>`).join('')}</div><div id="adl">${skel(4)}</div></div>`;
  try {
    const j = await api(`/api/admin/deposits?page=${page}&status=${f}`);
    $('#adl').innerHTML = j.deposits.length ? `<div class="space-y-2.5 stagger">${j.deposits.map(d => `<div class="glass rounded-2xl p-3.5"><div class="flex justify-between items-start gap-2"><div><div class="font-mono font-extrabold">₹${esc(d.amount)} ${d.crypto_amount ? `<span class="text-[10px] text-white/40">(${esc(d.crypto_amount)} USDT)</span>` : ''}</div><div class="text-[11px] text-white/40 font-mono">#${d.id} · 👤 ${d.user_id} · ${esc(d.method)}</div>${d.utr ? `<div class="text-[11px] font-mono text-white/60 break-all mt-0.5">UTR: ${esc(d.utr)}</div>` : ''}<div class="text-[10px] text-white/30">${esc(d.created_at)}</div></div>${badge(d.status)}</div>
      ${d.status === 'PENDING' && d.method !== 'UPI_AUTO' ? `<div class="grid grid-cols-2 gap-2 mt-3"><button onclick="depAct(${d.id},'approve')" class="grad-btn rounded-xl py-2.5 text-sm font-bold">✅ Approve</button><button onclick="depAct(${d.id},'reject')" class="rounded-xl py-2.5 text-sm font-bold text-rose-300" style="background:rgba(244,63,94,.14);border:1px solid rgba(244,63,94,.3)">❌ Reject</button></div>` : ''}</div>`).join('')}</div>${pager('drawAD', page, j.pages)}` : empty('💳', 'No deposits');
  } catch (e) { $('#adl').innerHTML = empty('🛠️', esc(e.message)); }
}
async function depAct(id, a) { try { await api(`/api/admin/deposit/${id}/${a}`, { body: {} }); toast(a === 'approve' ? 'Deposit approved' : 'Deposit rejected', 'ok'); drawAD(1); } catch (e) { toast(e.message, 'err'); } }

/* --- admin services --- */
function vAdminServices() { if (!adminGuard()) return; setTop('Services', 'Prices & status', true); S.asq = S.asq || ''; drawAS(1); }
async function drawAS(page) {
  view().innerHTML = `<div class="space-y-3"><input value="${esc(S.asq)}" placeholder="Search service ID / name" onkeydown="if(event.key==='Enter'){S.asq=this.value;drawAS(1)}" class="field w-full rounded-2xl px-4 py-3 text-sm"><div id="asl">${skel(4)}</div>
    <div class="grid grid-cols-2 gap-3">${btn('➕ Add service', "go('admin_add_service')")}${btn('🏷️ Categories', "go('admin_categories')", 'btn-soft')}</div>${btn('♻️ Recalculate prices', 'recalc(this)', 'btn-soft')}</div>`;
  try {
    const j = await api(`/api/admin/services?page=${page}&q=${encodeURIComponent(S.asq)}`);
    $('#asl').innerHTML = j.services.length ? `<div class="space-y-2.5 stagger">${j.services.map(s => `<div class="glass rounded-2xl p-3.5"><div class="flex justify-between gap-2"><div class="min-w-0"><div class="text-[11px] font-mono text-white/35">#${s.id} · ${esc(s.category)}</div><div class="font-bold text-sm line-clamp-2">${esc(s.name)}</div></div><button onclick="svcToggle(${s.id})" class="toggle ${s.active ? 'on' : ''}"></button></div>
      <div class="flex justify-between items-center mt-2.5"><div class="text-xs text-white/50">Cost <b class="font-mono">₹${esc(s.provider_rate)}</b> · Sell <b class="font-mono grad-text">₹${esc(s.rate)}</b></div><div class="flex gap-2"><button onclick="svcPrice(${s.id},'${esc(s.rate)}')" class="btn-soft rounded-lg px-3 py-1.5 text-xs font-bold">✏️ Price</button><button onclick="svcDelete(${s.id})" class="rounded-lg px-3 py-1.5 text-xs font-bold text-rose-300" style="background:rgba(244,63,94,.14);border:1px solid rgba(244,63,94,.3)">🗑</button></div></div></div>`).join('')}</div>${pager('drawAS', page, j.pages)}` : empty('🛍️', 'No services');
  } catch (e) { $('#asl').innerHTML = empty('🛠️', esc(e.message)); }
}
async function svcToggle(id) { try { await api(`/api/admin/service/${id}/toggle`, { body: {} }); drawAS(1); } catch (e) { toast(e.message, 'err'); } }
function svcPrice(id, cur) { sheet(`<div class="font-extrabold text-xl mb-4">Set price / 1000</div><input id="spv" type="number" inputmode="decimal" value="${esc(cur)}" class="field w-full rounded-2xl px-4 py-3.5 font-mono text-lg"><div class="mt-3"><button onclick="svcPriceGo(this,${id})" class="grad-btn w-full rounded-2xl py-3.5 font-bold">Save</button></div>`); }
async function svcPriceGo(el, id) { await withBtn(el, async () => { try { await api(`/api/admin/service/${id}/price`, { body: { rate: $('#spv').value } }); closeSheet(); toast('Price updated', 'ok'); drawAS(1); } catch (e) { toast(e.message, 'err'); } }); }
async function recalc(el) { await withBtn(el, async () => { try { const j = await api('/api/admin/recalc', { body: {} }); toast(`Updated ${j.updated}, skipped ${j.skipped}`, 'ok'); } catch (e) { toast(e.message, 'err'); } }); }

/* --- admin gifts --- */
async function vAdminGifts() {
  if (!adminGuard()) return; setTop('Gift Cards', 'Create & manage', true); view().innerHTML = skel(3);
  try {
    const j = await api('/api/admin/gifts');
    view().innerHTML = `<div class="space-y-3"><div class="glass rounded-3xl p-4 space-y-2.5"><div class="font-extrabold text-sm">Create gift card</div>
      <div class="grid grid-cols-3 gap-2"><input id="gcCode" placeholder="CODE" class="field rounded-xl px-3 py-3 text-xs font-mono uppercase col-span-3"><input id="gcAmt" type="number" placeholder="₹ amount" class="field rounded-xl px-3 py-3 text-sm font-mono col-span-2"><input id="gcUses" type="number" placeholder="Uses" value="1" class="field rounded-xl px-3 py-3 text-sm font-mono"></div>${btn('Create 🎁', 'gcCreate(this)')}</div>
      <div class="space-y-2 stagger">${j.gifts.map(g => `<div class="glass rounded-2xl p-3.5 flex items-center justify-between gap-2"><div><div class="font-mono font-bold">${esc(g.code)}</div><div class="text-[11px] text-white/45">₹${esc(g.amount)} · ${g.used}/${g.max_uses} used</div></div><div class="flex items-center gap-3"><button onclick="gcToggle('${esc(g.code)}')" class="toggle ${g.active ? 'on' : ''}"></button><button onclick="gcDel('${esc(g.code)}')" class="text-rose-300 text-lg">🗑</button></div></div>`).join('') || empty('🎁', 'No gift cards')}</div></div>`;
  } catch (e) { view().innerHTML = empty('🛠️', esc(e.message)); }
}
async function gcCreate(el) { await withBtn(el, async () => { try { await api('/api/admin/gift', { body: { code: $('#gcCode').value, amount: $('#gcAmt').value, uses: $('#gcUses').value } }); toast('Created', 'ok'); vAdminGifts(); } catch (e) { toast(e.message, 'err'); } }); }
async function gcToggle(c) { try { await api(`/api/admin/gift/${encodeURIComponent(c)}/toggle`, { body: {} }); vAdminGifts(); } catch (e) { toast(e.message, 'err'); } }
async function gcDel(c) { if (!confirm('Delete this gift card?')) return; try { await api(`/api/admin/gift/${encodeURIComponent(c)}/delete`, { body: {} }); vAdminGifts(); } catch (e) { toast(e.message, 'err'); } }

/* --- admin providers --- */
async function vAdminProviders() {
  if (!adminGuard()) return; setTop('Providers', 'API connections', true); view().innerHTML = skel(3);
  try {
    const j = await api('/api/admin/providers');
    window._provs = j.providers;
    view().innerHTML = `<div class="space-y-3">${btn('➕ Add provider', 'provAddSheet()')}<div class="space-y-3 stagger">${j.providers.map(p => `<div class="glass rounded-2xl p-4"><div class="flex justify-between items-center"><div class="font-extrabold">🔌 ${esc(p.name)} <span class="text-[11px] font-mono text-white/35">#${p.id}</span></div><button onclick="provToggle(${p.id})" class="toggle ${p.active ? 'on' : ''}"></button></div>
      <div class="text-[11px] text-white/40 font-mono break-all mt-1">${esc(p.url)}</div><div class="text-[11px] text-white/40 font-mono">${esc(p.key)}</div>
      <div class="flex items-center justify-between mt-3"><div id="pb${p.id}" class="font-mono font-bold text-sm text-white/60">—</div><button onclick="provBal(${p.id})" class="btn-soft rounded-lg px-3 py-1.5 text-xs font-bold">💰 Balance</button></div>
      <div class="grid grid-cols-2 gap-2 mt-3"><button onclick="go('admin_prov_services',{id:${p.id},name:'${esc(p.name).replace(/'/g, '')}'})" class="grad-btn rounded-xl py-2.5 text-xs font-bold">📥 Add services</button><button onclick="provDelete(${p.id})" class="rounded-xl py-2.5 text-xs font-bold text-rose-300" style="background:rgba(244,63,94,.14);border:1px solid rgba(244,63,94,.3)">🗑 Delete</button></div></div>`).join('') || empty('🔌', 'No providers yet', 'Add your first provider above.')}</div></div>`;
  } catch (e) { view().innerHTML = empty('🛠️', esc(e.message)); }
}
async function provBal(id) { const el = $('#pb' + id); el.innerHTML = '<span class="spinner"></span>'; try { const j = await api(`/api/admin/provider/${id}/balance`); el.textContent = j.balance + ' ' + j.currency; el.className = 'font-mono font-bold text-sm text-emerald-300'; } catch (e) { el.textContent = e.message; el.className = 'text-xs text-rose-300'; } }
async function provToggle(id) { try { await api(`/api/admin/provider/${id}/toggle`, { body: {} }); vAdminProviders(); } catch (e) { toast(e.message, 'err'); } }

/* --- admin broadcast --- */
function vAdminBroadcast() {
  if (!adminGuard()) return; setTop('Broadcast', 'Message all users', true);
  view().innerHTML = `<div class="glass rounded-3xl p-5 animate-rise"><div class="big-ico mx-auto mb-2">📣</div><div class="font-extrabold text-xl">Send to everyone</div><div class="text-xs text-white/50 mt-1">HTML tags like &lt;b&gt; &lt;i&gt; are supported.</div>
    <textarea id="bcTxt" rows="6" placeholder="Write your announcement…" class="field w-full rounded-2xl px-4 py-3 mt-4 text-sm"></textarea><div class="mt-4">${btn('Send broadcast 🚀', 'bcSend(this)')}</div></div>`;
}
async function bcSend(el) { if (!confirm('Send this message to all users?')) return; await withBtn(el, async () => { try { const j = await api('/api/admin/broadcast', { body: { text: $('#bcTxt').value } }); toast(`Broadcasting to ${j.total} users…`, 'ok'); } catch (e) { toast(e.message, 'err'); } }); }

/* --- admin settings --- */
const TOGGLE_LABELS = { MAINTENANCE_MODE: '🛠️ Full maintenance', MAINT_NEW_ORDER: '📦 Maintenance · new orders', MAINT_ADD_BALANCE: '💰 Maintenance · add balance', MAINT_SERVICE: '🛍️ Maintenance · services',
  PAY_UPI_AUTO_ENABLED: '⚡ UPI 1 (auto)', PAY_UPI_MANUAL_ENABLED: '🏦 UPI 2 (manual)', PAY_USDT_ENABLED: '🪙 USDT payments', REFERRAL_ENABLED: '🤝 Referral program', LOG_CHANNEL_ENABLED: '📢 Log channel' };
async function vAdminSettings() {
  if (!adminGuard()) return; setTop('Settings', 'Store configuration', true); view().innerHTML = skel(4);
  try {
    const j = await api('/api/admin/settings'), s = j.settings, cats = (await api('/api/admin/categories')).categories;
    view().innerHTML = `<div class="space-y-4 stagger">
      <div class="glass rounded-3xl p-4 space-y-3.5">${j.toggles.map(k => `<div class="flex items-center justify-between gap-3"><span class="text-sm font-semibold">${TOGGLE_LABELS[k] || k}</span><button onclick="setTog('${k}',${s[k] === '1' ? 0 : 1})" class="toggle ${s[k] === '1' ? 'on' : ''}"></button></div>`).join('')}</div>
      <div class="glass rounded-3xl p-4 space-y-3"><div class="font-extrabold text-sm">🎯 Pricing & discount</div>
        <div class="grid grid-cols-2 gap-2"><div><label class="text-[10px] text-white/50 font-bold">MARKUP %</label><input id="dMk" type="number" value="${esc(s.GLOBAL_MARKUP_PERCENT)}" class="field w-full rounded-xl px-3 py-2.5 font-mono text-sm mt-1"></div><div><label class="text-[10px] text-white/50 font-bold">DISCOUNT %</label><input id="dDs" type="number" value="${esc(s.GLOBAL_DISCOUNT_PERCENT)}" class="field w-full rounded-xl px-3 py-2.5 font-mono text-sm mt-1"></div></div>
        <div class="flex items-center justify-between"><span class="text-sm font-semibold">Discount active</span><button id="dOn" onclick="this.classList.toggle('on')" class="toggle ${s.DISCOUNT_ENABLED === '1' ? 'on' : ''}"></button></div>${btn('Save pricing', 'saveDisc(this)')}</div>
      <div class="glass rounded-3xl p-4 space-y-3"><div class="font-extrabold text-sm">⚙️ Values</div>${j.values.map(k => `<div><label class="text-[10px] text-white/50 font-bold">${k}</label><div class="flex gap-2 mt-1"><input id="sv_${k}" value="${esc(s[k])}" class="field flex-1 rounded-xl px-3 py-2.5 text-xs font-mono"><button onclick="saveVal('${k}')" class="btn-soft rounded-xl px-4 text-xs font-bold">Save</button></div></div>`).join('')}</div>
      <div class="glass rounded-3xl p-4 space-y-3"><div class="font-extrabold text-sm">🏷️ Categories</div>${cats.map(c => `<div class="flex items-center justify-between gap-3"><span class="text-sm">${esc(c.icon)} ${esc(c.name)} <span class="text-[10px] text-white/35">${c.count}</span></span><button onclick="catTog(${c.id})" class="toggle ${c.active ? 'on' : ''}"></button></div>`).join('') || '<div class="text-sm text-white/40">No categories</div>'}</div></div>`;
  } catch (e) { view().innerHTML = empty('🛠️', esc(e.message)); }
}
async function setTog(k, v) { try { await api('/api/admin/setting', { body: { key: k, value: String(v) } }); await refreshMe(); vAdminSettings(); } catch (e) { toast(e.message, 'err'); } }
async function saveVal(k) { try { await api('/api/admin/setting', { body: { key: k, value: $('#sv_' + k).value } }); toast('Saved', 'ok'); await refreshMe(); } catch (e) { toast(e.message, 'err'); } }
async function saveDisc(el) { await withBtn(el, async () => { try { await api('/api/admin/discount', { body: { markup: $('#dMk').value, discount: $('#dDs').value, enabled: $('#dOn').classList.contains('on') } }); toast('Saved — run “Recalculate prices” in Services to apply', 'ok'); await refreshMe(); } catch (e) { toast(e.message, 'err'); } }); }
async function catTog(id) { try { await api(`/api/admin/category/${id}/toggle`, { body: {} }); await refreshMe(); vAdminSettings(); } catch (e) { toast(e.message, 'err'); } }


/* --- admin: add provider / services / categories --- */
function provAddSheet() {
  sheet(`<div class="font-extrabold text-xl mb-1">Add provider</div><div class="text-xs text-white/50 mb-4">Connect an SMM panel API (v2 format).</div>
    <div class="space-y-3"><input id="pvName" placeholder="Provider name" class="field w-full rounded-2xl px-4 py-3.5 text-sm">
    <input id="pvUrl" type="url" placeholder="https://panel.com/api/v2" class="field w-full rounded-2xl px-4 py-3.5 text-sm font-mono">
    <input id="pvKey" placeholder="API key" class="field w-full rounded-2xl px-4 py-3.5 text-sm font-mono">
    <button onclick="provAddGo(this)" class="grad-btn w-full rounded-2xl py-3.5 font-bold">Add provider</button></div>`);
}
async function provAddGo(el) {
  await withBtn(el, async () => {
    try {
      const j = await api('/api/admin/provider', { body: { name: $('#pvName').value, url: $('#pvUrl').value, key: $('#pvKey').value } });
      closeSheet();
      toast(j.check.ok ? `Provider added · balance ${j.check.balance} ${j.check.currency}` : 'Provider added, but the API check failed: ' + (j.check.error || 'unknown'), j.check.ok ? 'ok' : 'info');
      vAdminProviders();
    } catch (e) { toast(e.message, 'err'); }
  });
}
function provDelete(id) {
  sheet(`<div class="text-center"><div class="big-ico mx-auto mb-2">🗑️</div><div class="font-extrabold text-xl">Delete provider #${id}?</div><div class="text-white/55 text-sm mt-1">All services from this provider will be removed from the catalog. Past orders stay intact.</div>
    <div class="grid grid-cols-2 gap-3 mt-5">${btn('Keep', 'closeSheet()', 'btn-soft')}<button onclick="provDeleteGo(this,${id})" class="rounded-2xl py-3.5 font-bold text-rose-200" style="background:rgba(244,63,94,.3);border:1px solid rgba(244,63,94,.5)">Delete</button></div></div>`);
}
async function provDeleteGo(el, id) { await withBtn(el, async () => { try { await api(`/api/admin/provider/${id}/delete`, { body: {} }); closeSheet(); toast('Provider deleted', 'ok'); vAdminProviders(); } catch (e) { toast(e.message, 'err'); } }); }

function svcDelete(id) {
  sheet(`<div class="text-center"><div class="big-ico mx-auto mb-2">🗑️</div><div class="font-extrabold text-xl">Delete service #${id}?</div><div class="text-white/55 text-sm mt-1">It will be removed from the catalog. Past orders stay intact.</div>
    <div class="grid grid-cols-2 gap-3 mt-5">${btn('Keep', 'closeSheet()', 'btn-soft')}<button onclick="svcDeleteGo(this,${id})" class="rounded-2xl py-3.5 font-bold text-rose-200" style="background:rgba(244,63,94,.3);border:1px solid rgba(244,63,94,.5)">Delete</button></div></div>`);
}
async function svcDeleteGo(el, id) { await withBtn(el, async () => { try { await api(`/api/admin/service/${id}/delete`, { body: {} }); closeSheet(); toast('Service deleted', 'ok'); drawAS(1); } catch (e) { toast(e.message, 'err'); } }); }

async function vAdminAddService() {
  if (!adminGuard()) return; setTop('Add Service', 'Choose a provider', true); view().innerHTML = skel(3);
  try {
    const j = await api('/api/admin/providers');
    view().innerHTML = `<div class="space-y-3 stagger"><div class="glass rounded-3xl p-5 text-center"><div class="big-ico mx-auto mb-2">➕</div><div class="font-extrabold text-lg">Pick a provider</div><div class="text-xs text-white/50 mt-1">Browse its catalog and add services with your own pricing.</div></div>
      ${j.providers.map(p => `<button onclick="go('admin_prov_services',{id:${p.id},name:'${esc(p.name).replace(/'/g, '')}'})" class="glass rounded-2xl p-4 w-full text-left card-hover flex items-center gap-3 ${p.active ? '' : 'opacity-50'}"><span class="text-2xl">🔌</span><div class="flex-1"><div class="font-bold">${esc(p.name)}</div><div class="text-[11px] text-white/40 font-mono">#${p.id}${p.active ? '' : ' · disabled'}</div></div><span class="text-white/30 text-xl">›</span></button>`).join('')}
      ${btn('🔌 Add new provider', 'provAddSheet()', 'btn-soft')}</div>`;
    if (!j.providers.length) view().innerHTML = `<div>${empty('🔌', 'No providers yet', 'Add a provider first.')}${btn('➕ Add provider', 'provAddSheet()')}</div>`;
  } catch (e) { view().innerHTML = empty('🛠️', esc(e.message)); }
}

const PS = { sel: new Set(), items: {}, q: '', page: 1, pages: 1 };
function vAdminProvServices() {
  if (!adminGuard()) return; setTop(S.params.name || 'Provider', 'Select services to add', true);
  PS.sel = new Set(); PS.items = {}; PS.q = ''; PS.page = 1;
  view().innerHTML = `<div class="space-y-3"><div class="flex gap-2"><input id="psq" placeholder="Search name / ID / category" onkeydown="if(event.key==='Enter'){PS.q=this.value;psLoad(1)}" class="field flex-1 rounded-2xl px-4 py-3 text-sm"><button onclick="psLoad(1,true)" class="btn-soft rounded-2xl px-4 font-bold" title="Refresh">🔄</button></div>
    <div id="psl">${skel(5)}</div><div id="psb" class="fixed bottom-24 inset-x-0 px-4 z-40 pointer-events-none"></div></div>`;
  psLoad(1);
}
async function psLoad(page, refresh) {
  PS.q = ($('#psq') && $('#psq').value) || PS.q; PS.page = page;
  const box = $('#psl'); if (page === 1) box.innerHTML = skel(5);
  try {
    const j = await api(`/api/admin/provider/${S.params.id}/services?page=${page}&q=${encodeURIComponent(PS.q)}${refresh ? '&refresh=1' : ''}`);
    PS.pages = j.pages;
    j.items.forEach(i => PS.items[i.sid] = i);
    const rows = j.items.map(i => `<button onclick="psToggle('${esc(i.sid)}')" id="ps_${esc(i.sid)}" class="glass rounded-2xl p-3.5 text-left w-full block ${i.exists ? 'opacity-50' : ''}">
      <div class="flex gap-3 items-start"><div class="w-6 h-6 rounded-lg border border-white/20 flex items-center justify-center text-xs shrink-0 mt-0.5 ${PS.sel.has(i.sid) ? 'chip-on' : ''}" data-ck>${PS.sel.has(i.sid) ? '✓' : ''}</div>
      <div class="flex-1 min-w-0"><div class="text-[11px] font-mono text-white/35">ID ${esc(i.sid)}${i.category ? ' · ' + esc(i.category) : ''}</div><div class="font-bold text-sm leading-snug line-clamp-2">${esc(i.name)}</div>
      <div class="flex gap-2 mt-1.5 text-[11px]"><span class="font-mono grad-text font-bold">₹${esc(i.rate)}/1K</span><span class="text-white/40 font-mono">${num(i.min)}–${num(i.max)}</span>${i.exists ? '<span class="text-emerald-300 font-bold">✓ added #' + i.exists + '</span>' : ''}</div></div></div></button>`).join('');
    if (page === 1) box.innerHTML = `<div class="text-xs text-white/40 mb-2">${j.total} services found · tap to select</div><div id="psrows" class="space-y-2.5"></div><div id="psmore" class="mt-3"></div>`;
    $('#psrows').insertAdjacentHTML('beforeend', rows || (page === 1 ? empty('🔍', 'No services found') : ''));
    $('#psmore').innerHTML = page < j.pages ? btn('Load more', `psLoad(${page + 1})`, 'btn-soft') : '';
    psBar();
  } catch (e) { box.innerHTML = empty('🛠️', esc(e.message)); }
}
function psToggle(sid) {
  if (PS.items[sid] && PS.items[sid].exists) return toast('This service is already in your catalog', 'info');
  haptic(); PS.sel.has(sid) ? PS.sel.delete(sid) : PS.sel.add(sid);
  const el = document.getElementById('ps_' + sid);
  if (el) { const ck = el.querySelector('[data-ck]'); const on = PS.sel.has(sid); ck.textContent = on ? '✓' : ''; ck.classList.toggle('chip-on', on); }
  psBar();
}
function psBar() {
  const n = PS.sel.size;
  $('#psb').innerHTML = n ? `<div class="max-w-xl mx-auto pointer-events-auto animate-rise"><button onclick="psAddSheet()" class="grad-btn w-full rounded-2xl py-4 font-extrabold shadow-2xl">Add ${n} selected service${n > 1 ? 's' : ''} →</button></div>` : '';
}
async function psAddSheet() {
  const ids = [...PS.sel]; if (!ids.length) return;
  let cats = [];
  try { cats = (await api('/api/admin/categories')).categories.filter(c => c.active); } catch (e) { return toast(e.message, 'err'); }
  const one = ids.length === 1 ? PS.items[ids[0]] : null;
  sheet(`<div class="font-extrabold text-xl mb-1">Add ${ids.length} service${ids.length > 1 ? 's' : ''}</div><div class="text-xs text-white/50 mb-4">Choose a category and your pricing.</div>
    <div class="space-y-3">
      ${cats.length ? `<select id="asCat" class="field w-full rounded-2xl px-4 py-3.5 text-sm bg-transparent">${cats.map(c => `<option value="${c.id}" style="color:#000">${esc(c.icon)} ${esc(c.name)}</option>`).join('')}</select>` :
        `<div class="rounded-2xl p-3.5 text-sm text-amber-200" style="background:rgba(251,191,36,.12);border:1px solid rgba(251,191,36,.3)">No active category yet. Create one first.</div>${btn('🏷️ Create category', "closeSheet();go('admin_categories')", 'btn-soft')}`}
      <div class="grid grid-cols-2 gap-2"><button id="pmP" onclick="pmSet('percent')" class="btn-soft chip-on rounded-xl py-2.5 font-bold text-sm">📈 Markup %</button><button id="pmF" onclick="pmSet('fixed')" class="btn-soft rounded-xl py-2.5 font-bold text-sm">💵 Fixed rate</button></div>
      <input id="asVal" type="number" inputmode="decimal" value="20" oninput="pmPrev()" class="field w-full rounded-2xl px-4 py-3.5 font-mono text-lg">
      ${one ? `<div id="asPrev" class="text-xs text-white/55"></div><input id="asName" value="${esc(one.name)}" class="field w-full rounded-2xl px-4 py-3 text-sm"><textarea id="asDesc" rows="2" placeholder="Description (optional — provider description is used if empty)" class="field w-full rounded-2xl px-4 py-3 text-sm"></textarea>` : ''}
      ${cats.length ? `<button onclick="psAddGo(this)" class="grad-btn w-full rounded-2xl py-3.5 font-bold">Add to catalog</button>` : ''}</div>`);
  window._pm = 'percent'; pmPrev();
}
function pmSet(m) {
  window._pm = m; $('#pmP').classList.toggle('chip-on', m === 'percent'); $('#pmF').classList.toggle('chip-on', m === 'fixed');
  $('#asVal').value = m === 'percent' ? '20' : ''; $('#asVal').placeholder = m === 'percent' ? 'Markup %' : 'Selling rate ₹ per 1000'; pmPrev();
}
function pmPrev() {
  const el = $('#asPrev'); if (!el) return; const it = PS.items[[...PS.sel][0]]; const v = parseFloat($('#asVal').value || 0); const r = parseFloat(it.rate || 0);
  const sell = window._pm === 'fixed' ? v : r * (1 + v / 100);
  el.innerHTML = `Provider ₹${r.toFixed(2)} → you sell at <b class="font-mono grad-text">₹${(sell || 0).toFixed(2)}</b> / 1000`;
}
async function psAddGo(el) {
  const ids = [...PS.sel], one = ids.length === 1;
  await withBtn(el, async () => {
    try {
      const j = await api('/api/admin/services/add', { body: { provider_id: S.params.id, provider_service_ids: ids, category_id: $('#asCat').value, mode: window._pm, value: $('#asVal').value,
        name: one ? $('#asName').value : '', description: one ? $('#asDesc').value : null } });
      closeSheet(); toast(`${j.added.length} service(s) added` + (j.skipped.length ? ` · ${j.skipped.length} skipped` : ''), 'ok');
      PS.sel = new Set(); vAdminProvServices();
    } catch (e) { toast(e.message, 'err'); }
  });
}

async function vAdminCategories() {
  if (!adminGuard()) return; setTop('Categories', 'Organise your catalog', true); view().innerHTML = skel(4);
  try {
    const j = await api('/api/admin/categories'); window._cats = j.categories;
    view().innerHTML = `<div class="space-y-3"><div class="glass rounded-3xl p-4 space-y-2.5"><div class="font-extrabold text-sm">➕ New category</div>
      <div class="flex gap-2"><input id="ncIcon" value="📁" maxlength="4" class="field w-16 rounded-xl px-2 py-3 text-center text-xl"><input id="ncName" placeholder="Category name" class="field flex-1 rounded-xl px-4 py-3 text-sm"></div>${btn('Create category', 'catAddGo(this)')}</div>
      <div class="space-y-2.5 stagger">${j.categories.map(c => `<div class="glass rounded-2xl p-3.5 flex items-center gap-3"><div class="text-2xl w-10 text-center">${esc(c.icon)}</div><div class="flex-1 min-w-0"><div class="font-bold text-sm truncate">${esc(c.name)}</div><div class="text-[11px] text-white/40">${c.count} services · #${c.id}</div></div>
        <button onclick="catEdit(${c.id})" class="btn-soft rounded-lg w-9 h-9">✏️</button><button onclick="catTog2(${c.id})" class="toggle ${c.active ? 'on' : ''}"></button><button onclick="catDelete(${c.id})" class="text-rose-300 text-lg w-8">🗑</button></div>`).join('') || empty('🏷️', 'No categories yet', 'Create your first one above.')}</div></div>`;
  } catch (e) { view().innerHTML = empty('🛠️', esc(e.message)); }
}
async function catAddGo(el) { await withBtn(el, async () => { try { await api('/api/admin/category', { body: { name: $('#ncName').value, icon: $('#ncIcon').value } }); await refreshMe(); toast('Category created', 'ok'); vAdminCategories(); } catch (e) { toast(e.message, 'err'); } }); }
async function catTog2(id) { try { await api(`/api/admin/category/${id}/toggle`, { body: {} }); await refreshMe(); vAdminCategories(); } catch (e) { toast(e.message, 'err'); } }
function catEdit(id) {
  const c = window._cats.find(x => x.id === id);
  sheet(`<div class="font-extrabold text-xl mb-4">Edit category</div><div class="flex gap-2"><input id="ceIcon" value="${esc(c.icon)}" maxlength="4" class="field w-16 rounded-xl px-2 py-3 text-center text-xl"><input id="ceName" value="${esc(c.name)}" class="field flex-1 rounded-xl px-4 py-3 text-sm"></div><div class="mt-4"><button onclick="catEditGo(this,${id})" class="grad-btn w-full rounded-2xl py-3.5 font-bold">Save</button></div>`);
}
async function catEditGo(el, id) { await withBtn(el, async () => { try { await api(`/api/admin/category/${id}/edit`, { body: { name: $('#ceName').value, icon: $('#ceIcon').value } }); closeSheet(); await refreshMe(); toast('Saved', 'ok'); vAdminCategories(); } catch (e) { toast(e.message, 'err'); } }); }
function catDelete(id) {
  const c = window._cats.find(x => x.id === id);
  sheet(`<div class="text-center"><div class="big-ico mx-auto mb-2">🗑️</div><div class="font-extrabold text-xl">Delete “${esc(c.name)}”?</div><div class="text-white/55 text-sm mt-1">${c.count ? `This category has <b>${c.count}</b> service(s). Deleting it will remove them too.` : 'This category is empty.'}</div>
    <div class="grid grid-cols-2 gap-3 mt-5">${btn('Keep', 'closeSheet()', 'btn-soft')}<button onclick="catDeleteGo(this,${id},${c.count})" class="rounded-2xl py-3.5 font-bold text-rose-200" style="background:rgba(244,63,94,.3);border:1px solid rgba(244,63,94,.5)">Delete</button></div></div>`);
}
async function catDeleteGo(el, id, count) { await withBtn(el, async () => { try { await api(`/api/admin/category/${id}/delete`, { body: { with_services: count > 0 } }); closeSheet(); await refreshMe(); toast('Category deleted', 'ok'); vAdminCategories(); } catch (e) { toast(e.message, 'err'); } }); }

/* ===================== boot ===================== */
async function boot() {
  if (!tg || !tg.initData) { showBlocked(); return; }
  try {
    tg.ready(); tg.expand();
    try { tg.setHeaderColor('#f5f7fc'); tg.setBackgroundColor('#f5f7fc'); tg.disableVerticalSwipes && tg.disableVerticalSwipes(); } catch (e) {}
    tg.BackButton.onClick(goBack);
  } catch (e) {}
  try {
    const j = await api('/api/bootstrap');
    S.boot = j; S.me = j.me;
    $('#topbar').classList.remove('hidden'); $('#nav').classList.remove('hidden');
    if (j.maintenance.full) {
      $('#nav').classList.add('hidden');
      view().innerHTML = `<div class="text-center pt-24 animate-pop"><div class="big-ico mx-auto mb-4">🛠️</div><div class="font-extrabold text-2xl">Under maintenance</div><div class="text-white/55 text-sm mt-2 px-8">We'll be back very soon. Please check again shortly.</div></div>`;
      setTop(j.bot_name || 'SMM Panel', 'Maintenance', false); return;
    }
    const sp = (tg.initDataUnsafe && tg.initDataUnsafe.start_param) || '';
    render();
  } catch (e) {
    if (!$('#blocked').classList.contains('flex')) view().innerHTML = empty('📡', 'Could not load', esc(e.message));
  }
}
boot();
</script>
<script>
(function(){
const P={"home": "<path d=\"M3 10.5 12 3l9 7.5V20a1 1 0 0 1-1 1h-5v-6H9v6H4a1 1 0 0 1-1-1z\"/>", "bag": "<path d=\"M6 2 3 6v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2V6l-3-4z\"/><path d=\"M3 6h18\"/><path d=\"M16 10a4 4 0 0 1-8 0\"/>", "box": "<path d=\"m7.5 4.27 9 5.15\"/><path d=\"M21 8a2 2 0 0 0-1-1.73l-7-4a2 2 0 0 0-2 0l-7 4A2 2 0 0 0 3 8v8a2 2 0 0 0 1 1.73l7 4a2 2 0 0 0 2 0l7-4A2 2 0 0 0 21 16Z\"/><path d=\"m3.3 7 8.7 5 8.7-5\"/><path d=\"M12 22V12\"/>", "wallet": "<path d=\"M19 7V4a1 1 0 0 0-1-1H5a2 2 0 0 0 0 4h15a1 1 0 0 1 1 1v4h-3a2 2 0 0 0 0 4h3a1 1 0 0 0 1-1v-2a1 1 0 0 0-1-1\"/><path d=\"M3 5v14a2 2 0 0 0 2 2h15a1 1 0 0 0 1-1v-4\"/>", "user": "<path d=\"M19 21v-2a4 4 0 0 0-4-4H9a4 4 0 0 0-4 4v2\"/><circle cx=\"12\" cy=\"7\" r=\"4\"/>", "users": "<path d=\"M16 21v-2a4 4 0 0 0-4-4H6a4 4 0 0 0-4 4v2\"/><circle cx=\"9\" cy=\"7\" r=\"4\"/><path d=\"M22 21v-2a4 4 0 0 0-3-3.87\"/><path d=\"M16 3.13a4 4 0 0 1 0 7.75\"/>", "shield": "<path d=\"M20 13c0 5-3.5 7.5-7.66 8.95a1 1 0 0 1-.67-.01C7.5 20.5 4 18 4 13V6a1 1 0 0 1 1-1c2 0 4.5-1.2 6.24-2.72a1.17 1.17 0 0 1 1.52 0C14.51 3.81 17 5 19 5a1 1 0 0 1 1 1z\"/>", "gift": "<rect x=\"3\" y=\"8\" width=\"18\" height=\"4\" rx=\"1\"/><path d=\"M12 8v13\"/><path d=\"M19 12v7a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2v-7\"/><path d=\"M7.5 8a2.5 2.5 0 0 1 0-5A4.8 8 0 0 1 12 8a4.8 8 0 0 1 4.5-5 2.5 2.5 0 0 1 0 5\"/>", "chat": "<path d=\"M7.9 20A9 9 0 1 0 4 16.1L2 22Z\"/>", "search": "<circle cx=\"11\" cy=\"11\" r=\"8\"/><path d=\"m21 21-4.3-4.3\"/>", "sparkles": "<path d=\"M9.937 15.5A2 2 0 0 0 8.5 14.063l-6.135-1.582a.5.5 0 0 1 0-.962L8.5 9.936A2 2 0 0 0 9.937 8.5l1.582-6.135a.5.5 0 0 1 .963 0L14.063 8.5A2 2 0 0 0 15.5 9.937l6.135 1.581a.5.5 0 0 1 0 .964L15.5 14.063a2 2 0 0 0-1.437 1.437l-1.582 6.135a.5.5 0 0 1-.963 0z\"/><path d=\"M20 3v4\"/><path d=\"M22 5h-4\"/><path d=\"M4 17v2\"/><path d=\"M5 18H3\"/>", "rocket": "<path d=\"M4.5 16.5c-1.5 1.26-2 5-2 5s3.74-.5 5-2c.71-.84.7-2.13-.09-2.91a2.18 2.18 0 0 0-2.91-.09z\"/><path d=\"m12 15-3-3a22 22 0 0 1 2-3.95A12.88 12.88 0 0 1 22 2c0 2.72-.78 7.5-6 11a22.35 22.35 0 0 1-4 2z\"/><path d=\"M9 12H4s.55-3.03 2-4c1.62-1.08 5 0 5 0\"/><path d=\"M12 15v5s3.03-.55 4-2c1.08-1.62 0-5 0-5\"/>", "gem": "<path d=\"M6 3h12l4 6-10 13L2 9Z\"/><path d=\"M11 3 8 9l4 13 4-13-3-6\"/><path d=\"M2 9h20\"/>", "flame": "<path d=\"M8.5 14.5A2.5 2.5 0 0 0 11 12c0-1.38-.5-2-1-3-1.07-2.14-.22-4.05 2-6 .5 2.5 2 4.9 4 6.5 2 1.6 3 3.5 3 5.5a7 7 0 1 1-14 0c0-1.15.43-2.3 1-3a2.5 2.5 0 0 0 2.5 2.5z\"/>", "zap": "<path d=\"M4 14a1 1 0 0 1-.78-1.63l9.9-10.2a.5.5 0 0 1 .86.46l-1.92 6.02A1 1 0 0 0 13 10h7a1 1 0 0 1 .78 1.63l-9.9 10.2a.5.5 0 0 1-.86-.46l1.92-6.02A1 1 0 0 0 11 14z\"/>", "star": "<polygon points=\"12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2\"/>", "target": "<circle cx=\"12\" cy=\"12\" r=\"10\"/><circle cx=\"12\" cy=\"12\" r=\"6\"/><circle cx=\"12\" cy=\"12\" r=\"2\"/>", "trophy": "<path d=\"M6 9H4.5a2.5 2.5 0 0 1 0-5H6\"/><path d=\"M18 9h1.5a2.5 2.5 0 0 0 0-5H18\"/><path d=\"M4 22h16\"/><path d=\"M10 14.66V17c0 .55-.47.98-1.21 1.21C7.85 18.75 7 20.24 7 22\"/><path d=\"M14 14.66V17c0 .55.47.98 1.21 1.21C16.15 18.75 17 20.24 17 22\"/><path d=\"M18 2H6v7a6 6 0 0 0 12 0V2Z\"/>", "lock": "<rect x=\"3\" y=\"11\" width=\"18\" height=\"11\" rx=\"2\"/><path d=\"M7 11V7a5 5 0 0 1 10 0v4\"/>", "checkc": "<path d=\"M21.801 10A10 10 0 1 1 17 3.335\"/><path d=\"m9 11 3 3L22 4\"/>", "check": "<path d=\"M20 6 9 17l-5-5\"/>", "xc": "<circle cx=\"12\" cy=\"12\" r=\"10\"/><path d=\"m15 9-6 6\"/><path d=\"m9 9 6 6\"/>", "ban": "<circle cx=\"12\" cy=\"12\" r=\"10\"/><path d=\"m4.9 4.9 14.2 14.2\"/>", "alert": "<path d=\"m21.73 18-8-14a2 2 0 0 0-3.48 0l-8 14A2 2 0 0 0 4 21h16a2 2 0 0 0 1.73-3\"/><path d=\"M12 9v4\"/><path d=\"M12 17h.01\"/>", "clock": "<circle cx=\"12\" cy=\"12\" r=\"10\"/><path d=\"M12 6v6l4 2\"/>", "wrench": "<path d=\"M14.7 6.3a1 1 0 0 0 0 1.4l1.6 1.6a1 1 0 0 0 1.4 0l3.77-3.77a6 6 0 0 1-7.94 7.94l-6.91 6.91a2.12 2.12 0 0 1-3-3l6.91-6.91a6 6 0 0 1 7.94-7.94z\"/>", "plug": "<path d=\"M12 22v-5\"/><path d=\"M9 8V2\"/><path d=\"M15 8V2\"/><path d=\"M18 8v5a4 4 0 0 1-4 4h-4a4 4 0 0 1-4-4V8Z\"/>", "plus": "<path d=\"M5 12h14\"/><path d=\"M12 5v14\"/>", "minus": "<path d=\"M5 12h14\"/>", "refresh": "<path d=\"M3 12a9 9 0 0 1 9-9 9.75 9.75 0 0 1 6.74 2.74L21 8\"/><path d=\"M21 3v5h-5\"/><path d=\"M21 12a9 9 0 0 1-9 9 9.75 9.75 0 0 1-6.74-2.74L3 16\"/><path d=\"M8 16H3v5\"/>", "list": "<rect x=\"8\" y=\"2\" width=\"8\" height=\"4\" rx=\"1\"/><path d=\"M16 4h2a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V6a2 2 0 0 1 2-2h2\"/><path d=\"M12 11h4\"/><path d=\"M12 16h4\"/><path d=\"M8 11h.01\"/><path d=\"M8 16h.01\"/>", "tag": "<path d=\"M12.586 2.586A2 2 0 0 0 11.172 2H4a2 2 0 0 0-2 2v7.172a2 2 0 0 0 .586 1.414l8.704 8.704a2.426 2.426 0 0 0 3.42 0l6.58-6.58a2.426 2.426 0 0 0 0-3.42z\"/><circle cx=\"7.5\" cy=\"7.5\" r=\".5\" fill=\"currentColor\"/>", "trash": "<path d=\"M3 6h18\"/><path d=\"M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6\"/><path d=\"M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2\"/>", "party": "<path d=\"M5.8 11.3 2 22l10.7-3.79\"/><path d=\"M4 3h.01\"/><path d=\"M22 8h.01\"/><path d=\"M15 2h.01\"/><path d=\"M22 20h.01\"/><path d=\"m22 2-2.24.75a2.9 2.9 0 0 0-1.96 3.12c.1.86-.57 1.63-1.45 1.63h-.38c-.86 0-1.6.6-1.76 1.44L14 10\"/><path d=\"m22 13-.82-.33c-.86-.34-1.82.2-1.98 1.11c-.11.7-.72 1.22-1.43 1.22H17\"/><path d=\"m11 2 .33.82c.34.86-.2 1.82-1.11 1.98C9.52 4.9 9 5.52 9 6.23V7\"/><path d=\"M11 13c1.93 1.93 2.83 4.17 2 5-.83.83-3.07-.07-5-2-1.93-1.93-2.83-4.17-2-5 .83-.83 3.07.07 5 2Z\"/>", "bank": "<line x1=\"3\" x2=\"21\" y1=\"22\" y2=\"22\"/><line x1=\"6\" x2=\"6\" y1=\"18\" y2=\"11\"/><line x1=\"10\" x2=\"10\" y1=\"18\" y2=\"11\"/><line x1=\"14\" x2=\"14\" y1=\"18\" y2=\"11\"/><line x1=\"18\" x2=\"18\" y1=\"18\" y2=\"11\"/><polygon points=\"12 2 20 7 4 7\"/>", "coins": "<circle cx=\"8\" cy=\"8\" r=\"6\"/><path d=\"M18.09 10.37A6 6 0 1 1 10.34 18\"/><path d=\"M7 6h1v4\"/><path d=\"m16.71 13.88.7.71-2.82 2.82\"/>", "card": "<rect width=\"20\" height=\"14\" x=\"2\" y=\"5\" rx=\"2\"/><line x1=\"2\" x2=\"22\" y1=\"10\" y2=\"10\"/>", "inbox": "<polyline points=\"22 12 16 12 14 15 10 15 8 12 2 12\"/><path d=\"M5.45 5.11 2 12v6a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2v-6l-3.45-6.89A2 2 0 0 0 16.76 4H7.24a2 2 0 0 0-1.79 1.11z\"/>", "receipt": "<path d=\"M4 2v20l2-1 2 1 2-1 2 1 2-1 2 1 2-1 2 1V2l-2 1-2-1-2 1-2-1-2 1-2-1-2 1Z\"/><path d=\"M16 8h-6a2 2 0 1 0 0 4h4a2 2 0 1 1 0 4H8\"/><path d=\"M12 17.5v-11\"/>", "mega": "<path d=\"m3 11 18-5v12L3 14v-3z\"/><path d=\"M11.6 16.8a3 3 0 1 1-5.8-1.6\"/>", "handshake": "<path d=\"m11 17 2 2a1 1 0 1 0 3-3\"/><path d=\"m14 14 2.5 2.5a1 1 0 1 0 3-3l-3.88-3.88a3 3 0 0 0-4.24 0l-.88.88a1 1 0 1 1-3-3l2.81-2.81a5.79 5.79 0 0 1 7.06-.87l.47.28a2 2 0 0 0 1.42.25L21 4\"/><path d=\"m21 3 1 11h-2\"/><path d=\"M3 3 2 14l6.5 6.5a1 1 0 1 0 3-3\"/><path d=\"M3 4h8\"/>", "gear": "<path d=\"M12.22 2h-.44a2 2 0 0 0-2 2v.18a2 2 0 0 1-1 1.73l-.43.25a2 2 0 0 1-2 0l-.15-.08a2 2 0 0 0-2.73.73l-.22.38a2 2 0 0 0 .73 2.73l.15.1a2 2 0 0 1 1 1.72v.51a2 2 0 0 1-1 1.74l-.15.09a2 2 0 0 0-.73 2.73l.22.38a2 2 0 0 0 2.73.73l.15-.08a2 2 0 0 1 2 0l.43.25a2 2 0 0 1 1 1.73V20a2 2 0 0 0 2 2h.44a2 2 0 0 0 2-2v-.18a2 2 0 0 1 1-1.73l.43-.25a2 2 0 0 1 2 0l.15.08a2 2 0 0 0 2.73-.73l.22-.39a2 2 0 0 0-.73-2.73l-.15-.08a2 2 0 0 1-1-1.74v-.5a2 2 0 0 1 1-1.74l.15-.09a2 2 0 0 0 .73-2.73l-.22-.38a2 2 0 0 0-2.73-.73l-.15.08a2 2 0 0 1-2 0l-.43-.25a2 2 0 0 1-1-1.73V4a2 2 0 0 0-2-2z\"/><circle cx=\"12\" cy=\"12\" r=\"3\"/>", "mail": "<rect width=\"20\" height=\"16\" x=\"2\" y=\"4\" rx=\"2\"/><path d=\"m22 7-8.97 5.7a1.94 1.94 0 0 1-2.06 0L2 7\"/>", "pencil": "<path d=\"M21.174 6.812a1 1 0 0 0-3.986-3.987L3.842 16.174a2 2 0 0 0-.5.83l-1.321 4.352a.5.5 0 0 0 .623.622l4.353-1.32a2 2 0 0 0 .83-.497z\"/><path d=\"m15 5 4 4\"/>", "hand": "<path d=\"M18 11V6a2 2 0 0 0-4 0v5\"/><path d=\"M14 10V4a2 2 0 0 0-4 0v2\"/><path d=\"M10 10.5V6a2 2 0 0 0-4 0v8\"/><path d=\"M18 8a2 2 0 1 1 4 0v6a8 8 0 0 1-8 8h-2c-2.8 0-4.5-.86-5.99-2.34l-3.6-3.6a2 2 0 0 1 2.83-2.82L7 15\"/>", "link": "<path d=\"M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71\"/><path d=\"M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71\"/>", "hash": "<line x1=\"4\" x2=\"20\" y1=\"9\" y2=\"9\"/><line x1=\"4\" x2=\"20\" y1=\"15\" y2=\"15\"/><line x1=\"10\" x2=\"8\" y1=\"3\" y2=\"21\"/><line x1=\"16\" x2=\"14\" y1=\"3\" y2=\"21\"/>", "phone": "<rect width=\"14\" height=\"20\" x=\"5\" y=\"2\" rx=\"2\" ry=\"2\"/><path d=\"M12 18h.01\"/>", "call": "<path d=\"M22 16.92v3a2 2 0 0 1-2.18 2 19.79 19.79 0 0 1-8.63-3.07 19.5 19.5 0 0 1-6-6 19.79 19.79 0 0 1-3.07-8.67A2 2 0 0 1 4.11 2h3a2 2 0 0 1 2 1.72c.127.96.361 1.903.7 2.81a2 2 0 0 1-.45 2.11L8.09 9.91a16 16 0 0 0 6 6l1.27-1.27a2 2 0 0 1 2.11-.45c.907.339 1.85.573 2.81.7A2 2 0 0 1 22 16.92z\"/>", "bars": "<line x1=\"12\" x2=\"12\" y1=\"20\" y2=\"10\"/><line x1=\"18\" x2=\"18\" y1=\"20\" y2=\"4\"/><line x1=\"6\" x2=\"6\" y1=\"20\" y2=\"16\"/>", "trend": "<polyline points=\"22 7 13.5 15.5 8.5 10.5 2 17\"/><polyline points=\"16 7 22 7 22 13\"/>", "upload": "<path d=\"M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4\"/><polyline points=\"17 8 12 3 7 8\"/><line x1=\"12\" x2=\"12\" y1=\"3\" y2=\"15\"/>", "download": "<path d=\"M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4\"/><polyline points=\"7 10 12 15 17 10\"/><line x1=\"12\" x2=\"12\" y1=\"15\" y2=\"3\"/>", "folder": "<path d=\"M20 20a2 2 0 0 0 2-2V8a2 2 0 0 0-2-2h-7.9a2 2 0 0 1-1.69-.9L9.6 3.9A2 2 0 0 0 7.93 3H4a2 2 0 0 0-2 2v13a2 2 0 0 0 2 2Z\"/>", "radio": "<path d=\"M4.9 19.1C1 15.2 1 8.8 4.9 4.9\"/><path d=\"M7.8 16.2c-2.3-2.3-2.3-6.1 0-8.5\"/><circle cx=\"12\" cy=\"12\" r=\"2\"/><path d=\"M16.2 7.8c2.3 2.3 2.3 6.1 0 8.5\"/><path d=\"M19.1 4.9C23 8.8 23 15.1 19.1 19\"/>", "camera": "<path d=\"M14.5 4h-5L7 7H4a2 2 0 0 0-2 2v9a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2V9a2 2 0 0 0-2-2h-3z\"/><circle cx=\"12\" cy=\"13\" r=\"3\"/>", "play": "<polygon points=\"6 3 20 12 6 21 6 3\"/>", "send": "<path d=\"M14.536 21.686a.5.5 0 0 0 .937-.024l6.5-19a.496.496 0 0 0-.635-.635l-19 6.5a.5.5 0 0 0-.024.937l7.93 3.18a2 2 0 0 1 1.112 1.11z\"/><path d=\"m21.854 2.147-10.94 10.939\"/>", "heart": "<path d=\"M19 14c1.49-1.46 3-3.21 3-5.5A5.5 5.5 0 0 0 16.5 3c-1.76 0-3 .5-4.5 2-1.5-1.5-2.74-2-4.5-2A5.5 5.5 0 0 0 2 8.5c0 2.3 1.5 4.05 3 5.5l7 7Z\"/>", "eye": "<path d=\"M2.062 12.348a1 1 0 0 1 0-.696 10.75 10.75 0 0 1 19.876 0 1 1 0 0 1 0 .696 10.75 10.75 0 0 1-19.876 0\"/><circle cx=\"12\" cy=\"12\" r=\"3\"/>", "thumb": "<path d=\"M7 10v12\"/><path d=\"M15 5.88 14 10h5.83a2 2 0 0 1 1.92 2.56l-2.33 8A2 2 0 0 1 17.5 22H4a2 2 0 0 1-2-2v-8a2 2 0 0 1 2-2h2.76a2 2 0 0 0 1.79-1.11L12 2a3.13 3.13 0 0 1 3 3.88Z\"/>", "music": "<path d=\"M9 18V5l12-2v13\"/><circle cx=\"6\" cy=\"18\" r=\"3\"/><circle cx=\"18\" cy=\"16\" r=\"3\"/>"};
const A={"🏠": "home", "🛍": "bag", "📦": "box", "💰": "wallet", "👛": "wallet", "💵": "coins", "💱": "coins", "🪙": "coins", "👤": "user", "👥": "users", "🛡": "shield", "🎁": "gift", "💬": "chat", "🔎": "search", "🔍": "search", "✨": "sparkles", "🚀": "rocket", "💎": "gem", "🔥": "flame", "⚡": "zap", "🌟": "star", "⭐": "star", "🎯": "target", "🏆": "trophy", "🔒": "lock", "✅": "checkc", "✓": "check", "✔": "check", "❌": "xc", "🚫": "ban", "⚠": "alert", "⏰": "clock", "⏳": "clock", "⏱": "clock", "🛠": "wrench", "🔌": "plug", "➕": "plus", "➖": "minus", "♻": "refresh", "🔄": "refresh", "📋": "list", "🏷": "tag", "🗑": "trash", "🎉": "party", "🏦": "bank", "💳": "card", "📭": "inbox", "🧾": "receipt", "📢": "mega", "📣": "mega", "🤝": "handshake", "⚙": "gear", "✉": "mail", "✏": "pencil", "👋": "hand", "🔗": "link", "🔢": "hash", "📱": "phone", "📞": "call", "📊": "bars", "📈": "trend", "📤": "upload", "📥": "download", "📁": "folder", "📡": "radio", "📸": "camera", "▶": "play", "✈": "send", "❤": "heart", "👁": "eye", "👍": "thumb", "🎵": "music"};
const DOT={'🔴':'#ef4444','🟡':'#f59e0b','🔵':'#0ea5e9','🟢':'#10b981','🟠':'#f97316','⚪':'#cbd5e1'};
const RE=/(?:\p{Extended_Pictographic}\uFE0F?(?:\u200D\p{Extended_Pictographic}\uFE0F?)*)|\u2713/gu;
const RT=/\p{Extended_Pictographic}|\u2713/u;
const SKIP=/^(OPTION|TEXTAREA|SCRIPT|STYLE|TITLE)$/;
function mk(e){
  const k=e.replace(/\uFE0F/g,'');
  if(DOT[k]){const d=document.createElement('span');d.className='sdot';d.style.background=DOT[k];return d;}
  const t=document.createElement('template');
  t.innerHTML='<svg class="ico" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">'+P[A[k]||'sparkles']+'</svg>';
  return t.content.firstChild;
}
function conv(n){
  const t=n.nodeValue; if(!t||!RT.test(t))return;
  const p=n.parentNode; if(!p)return;
  if(SKIP.test(p.nodeName)){ if(p.nodeName==='OPTION') n.nodeValue=t.replace(RE,'').trim(); return; }
  const f=document.createDocumentFragment(); let last=0,m; RE.lastIndex=0;
  while((m=RE.exec(t))){ if(m.index>last)f.appendChild(document.createTextNode(t.slice(last,m.index))); f.appendChild(mk(m[0])); last=m.index+m[0].length; }
  if(last<t.length)f.appendChild(document.createTextNode(t.slice(last)));
  p.replaceChild(f,n);
}
function scan(r){
  if(r.nodeType===3){conv(r);return;}
  const w=document.createTreeWalker(r,NodeFilter.SHOW_TEXT),a=[];
  while(w.nextNode())a.push(w.currentNode);
  a.forEach(conv);
}
scan(document.body);
new MutationObserver(ms=>{for(const m of ms){
  if(m.type==='characterData')scan(m.target);
  else m.addedNodes.forEach(n=>{if(n.nodeType===1||n.nodeType===3)scan(n);});
}}).observe(document.body,{childList:true,subtree:true,characterData:true});
})();
</script>
</body>
</html>
'''


# ===========================================================================
# 🌐 TELEGRAM MINI APP — web server + JSON API (shares the bot's database)
# ===========================================================================
# * Starts together with the bot (same process, same SQLite file).
# * Every /api/* call must carry Telegram's signed initData; it is verified
#   with the bot token, so users are logged in automatically as their own
#   Telegram ID and the API refuses anything opened outside Telegram.
# * /api/admin/* additionally requires the caller to be a bot admin.
# ===========================================================================
import base64
import hashlib
import hmac
from urllib.parse import parse_qsl
from aiohttp import web

WEBAPP_BOT: Optional[Bot] = None
WEBAPP_REFILL_LOCKS: Dict[int, asyncio.Lock] = {}


def _jdefault(o):
    if isinstance(o, Decimal):
        return str(o)
    if isinstance(o, (bytes, bytearray)):
        return o.decode("utf-8", "ignore")
    return str(o)


def jresp(data: Any, status: int = 200) -> web.Response:
    body = json.dumps(data, default=_jdefault, separators=(",", ":"))
    resp = web.Response(text=body, status=status, content_type="application/json", charset="utf-8",
                        headers={"Cache-Control": "no-store"})
    if len(body) > 1024:
        resp.enable_compression()
    return resp


def jerr(message: str, status: int = 400, **extra) -> web.Response:
    return jresp({"ok": False, "error": message, **extra}, status=status)


def row_dict(row) -> Dict[str, Any]:
    return {k: row[k] for k in row.keys()} if row is not None else {}


def validate_init_data(init_data: str, max_age: int = 86400) -> Optional[Dict[str, Any]]:
    """Verify Telegram WebApp initData signature (HMAC-SHA256 with the bot token)."""
    try:
        if not init_data or not BOT_TOKEN:
            return None
        data = dict(parse_qsl(init_data, keep_blank_values=True))
        recv_hash = data.pop("hash", None)
        if not recv_hash:
            return None
        check = "\n".join(f"{k}={v}" for k, v in sorted(data.items()))
        secret = hmac.new(b"WebAppData", BOT_TOKEN.encode(), hashlib.sha256).digest()
        calc = hmac.new(secret, check.encode(), hashlib.sha256).hexdigest()
        if not hmac.compare_digest(calc, recv_hash):
            return None
        if time.time() - int(data.get("auth_date", "0")) > max_age:
            return None
        user = json.loads(data.get("user", "{}"))
        if not user.get("id"):
            return None
        return {"user": user, "start_param": data.get("start_param", "")}
    except Exception:
        return None


async def ensure_webapp_user(tg: Dict[str, Any], start_param: str = ""):
    uid = int(tg["id"])
    first_name = tg.get("first_name") or "User"
    username = tg.get("username") or ""
    referrer_id = None
    if start_param and str(start_param).startswith("ref_"):
        try:
            cand = int(str(start_param)[4:])
            if cand != uid:
                referrer_id = cand
        except ValueError:
            pass
    is_new = False
    async with get_db() as db:
        u = await (await db.execute("SELECT * FROM users WHERE user_id = ?", (uid,))).fetchone()
        if not u:
            is_new = True
            await db.execute(
                "INSERT OR IGNORE INTO users (user_id, first_name, username, referrer_id) VALUES (?, ?, ?, ?)",
                (uid, first_name, username, referrer_id))
            await db.commit()
            u = await (await db.execute("SELECT * FROM users WHERE user_id = ?", (uid,))).fetchone()
        elif (u["first_name"] or "") != first_name or (u["username"] or "") != username:
            await db.execute("UPDATE users SET first_name = ?, username = ? WHERE user_id = ?",
                             (first_name, username, uid))
            await db.commit()
            u = dict(u)
            u["first_name"], u["username"] = first_name, username
    if is_new and WEBAPP_BOT is not None:
        asyncio.create_task(notify_new_user_to_admins(WEBAPP_BOT, uid, first_name, username, referrer_id))
    return u


async def webapp_maintenance(uid: int, is_adm: bool, feature: str = "FULL") -> bool:
    if is_adm:
        return False
    full = await get_setting("MAINTENANCE_MODE", "0")
    feat = "0"
    if feature == "ORDER":
        feat = await get_setting("MAINT_NEW_ORDER", "0")
    elif feature == "DEPOSIT":
        feat = await get_setting("MAINT_ADD_BALANCE", "0")
    elif feature == "SERVICE":
        feat = await get_setting("MAINT_SERVICE", "0")
    return _flag_on(full) or _flag_on(feat)


MAINT_MSG = "🛠️ This feature is temporarily under maintenance. Please check back shortly."


@web.middleware
async def webapp_cors_middleware(request: web.Request, handler):
    """Allow the configured Vercel frontend to call this API, including preflight."""
    origin = request.headers.get("Origin", "")
    configured = os.getenv("WEBAPP_ALLOWED_ORIGINS", os.getenv("WEBAPP_URL", WEBAPP_URL))
    allowed = {x.strip().rstrip("/") for x in str(configured or "").split(",") if x.strip()}
    headers = {}
    if origin and origin.rstrip("/") in allowed:
        headers = {
            "Access-Control-Allow-Origin": origin,
            "Vary": "Origin",
            "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
            "Access-Control-Allow-Headers": "Content-Type, X-Init-Data",
            "Access-Control-Max-Age": "600",
        }
    if request.method == "OPTIONS":
        return web.Response(status=204, headers=headers)
    response = await handler(request)
    for key, value in headers.items():
        response.headers[key] = value
    return response


@web.middleware
async def webapp_auth_middleware(request: web.Request, handler):
    path = request.path
    if not path.startswith("/api/"):
        return await handler(request)
    info = validate_init_data(request.headers.get("X-Init-Data", ""))
    if not info:
        return jerr("Please open this app from inside Telegram.", 401, code="not_telegram")
    tg = info["user"]
    uid = int(tg["id"])
    if not await security_rate_limit(-uid, limit=90, window=10):
        return jerr("Too many requests. Please wait a moment.", 429)
    try:
        row = await ensure_webapp_user(tg, info.get("start_param", ""))
        adm = await is_admin(uid)
        if row["is_banned"] and not adm:
            return jerr("Your account is suspended.", 403, code="banned")
        request["uid"] = uid
        request["tg"] = tg
        request["is_admin"] = adm
        if path.startswith("/api/admin/") and not adm:
            return jerr("Admins only.", 403)
        return await handler(request)
    except web.HTTPException:
        raise
    except Exception as e:
        logger.error("Mini app API error on %s: %s", path, e, exc_info=True)
        return jerr("Server error. Please try again.", 500)


async def body_of(request: web.Request) -> Dict[str, Any]:
    try:
        data = await request.json()
        return data if isinstance(data, dict) else {}
    except Exception:
        return {}


def _int(v, default=0) -> int:
    try:
        return int(v)
    except Exception:
        return default


def _refill_plain(avail, days, duration) -> str:
    if not int(avail or 0):
        return "No refill"
    d = str(duration or "").strip()
    if d:
        return f"Refill · {d}"
    n = _int(days)
    return f"Refill · {n} days" if n > 0 else "Refill · Lifetime"


async def _discount_info() -> Dict[str, Any]:
    on = await get_setting("DISCOUNT_ENABLED", "0") == "1"
    try:
        pct = Decimal(await get_setting("GLOBAL_DISCOUNT_PERCENT", "0.00"))
    except Exception:
        pct = Decimal("0")
    return {"on": bool(on and pct > 0), "pct": pct}


def _svc_json(s, disc: Dict[str, Any]) -> Dict[str, Any]:
    out = {
        "id": s["id"], "category_id": s["category_id"], "name": s["name"],
        "min": s["min_quantity"], "max": s["max_quantity"], "rate": str(s["selling_rate"]),
        "description": s["description"] or "", "refill": bool(s["refill_available"]),
        "refill_text": _refill_plain(s["refill_available"], s["refill_days"], s["refill_duration"]),
        "cancel": bool(s["cancel_available"]), "orig_rate": None,
    }
    if disc["on"]:
        try:
            out["orig_rate"] = str(quantize_currency(
                Decimal(str(s["selling_rate"])) / (Decimal("1") - disc["pct"] / Decimal("100"))))
        except Exception:
            pass
    return out


# ------------------------------- bootstrap --------------------------------
async def api_bootstrap(request: web.Request):
    uid = request["uid"]
    adm = request["is_admin"]
    async with get_db() as db:
        u = await (await db.execute("SELECT * FROM users WHERE user_id = ?", (uid,))).fetchone()
        cats = await (await db.execute(
            """SELECT c.id, c.name, c.icon,
                      (SELECT COUNT(*) FROM services s WHERE s.category_id=c.id AND s.is_active=1) AS cnt
               FROM categories c WHERE c.is_active=1 ORDER BY c.display_order ASC, c.id ASC""")).fetchall()
    disc = await _discount_info()
    rate = await get_live_usdt_inr_rate()
    pay = {
        "upi_auto": await payment_method_enabled("UPI_AUTO"),
        "upi_manual": await payment_method_enabled("UPI_MANUAL"),
        "usdt": await payment_method_enabled("USDT_BEP20"),
    }
    return jresp({
        "ok": True,
        "me": {
            "id": uid, "first_name": u["first_name"] or "User", "username": u["username"] or "",
            "balance": u["balance"], "total_spent": u["total_spent"], "total_deposited": u["total_deposited"],
            "currency": u["currency"] or "INR", "is_admin": adm,
        },
        "usd_rate": str(rate),
        "bot_name": BOT_NAME,
        "bot_username": _BOT_INFO.get("username") or "",
        "maintenance": {
            "full": _flag_on(await get_setting("MAINTENANCE_MODE", "0")) and not adm,
            "order": _flag_on(await get_setting("MAINT_NEW_ORDER", "0")) and not adm,
            "deposit": _flag_on(await get_setting("MAINT_ADD_BALANCE", "0")) and not adm,
            "service": _flag_on(await get_setting("MAINT_SERVICE", "0")) and not adm,
        },
        "pay": pay,
        "limits": {
            "min": await get_setting("MIN_DEPOSIT", "1.00"), "max": await get_setting("MAX_DEPOSIT", "50000.00"),
            "usdt_min": await get_setting("USDT_MIN_DEPOSIT", "1.00"),
            "usdt_max": await get_setting("USDT_MAX_DEPOSIT", "10000.00"),
        },
        "support": {"username": await get_setting("SUPPORT_USERNAME", "@telegram"),
                    "channel": await get_setting("SUPPORT_CHANNEL", "https://t.me/telegram")},
        "discount": {"on": disc["on"], "pct": str(disc["pct"])},
        "categories": [row_dict(c) for c in cats],
    })


# ------------------------------- catalog ----------------------------------
async def api_services(request: web.Request):
    if await webapp_maintenance(request["uid"], request["is_admin"], "SERVICE"):
        return jerr(MAINT_MSG, 503, code="maintenance")
    cat = _int(request.query.get("category_id"))
    q = (request.query.get("q") or "").strip()[:60]
    sql = ("SELECT s.* FROM services s JOIN categories c ON c.id = s.category_id "
           "WHERE s.is_active = 1 AND c.is_active = 1")
    args: List[Any] = []
    if cat:
        sql += " AND s.category_id = ?"
        args.append(cat)
    if q:
        sql += " AND (s.name LIKE ? OR CAST(s.id AS TEXT) = ?)"
        args += [f"%{q}%", q]
    sql += " ORDER BY s.id ASC LIMIT 300"
    async with get_db() as db:
        rows = await (await db.execute(sql, args)).fetchall()
    disc = await _discount_info()
    return jresp({"ok": True, "services": [_svc_json(s, disc) for s in rows]})


async def api_service(request: web.Request):
    sid = _int(request.match_info["sid"])
    async with get_db() as db:
        s = await (await db.execute("SELECT * FROM services WHERE id = ? AND is_active = 1", (sid,))).fetchone()
    if not s:
        return jerr("Service unavailable.", 404)
    return jresp({"ok": True, "service": _svc_json(s, await _discount_info())})


# -------------------------------- orders ----------------------------------
ORDER_SELECT = """
    SELECT o.*, COALESCE(o.service_name_snapshot, s.name, 'Service') AS service_name,
           COALESCE(o.service_refill_snapshot, s.refill_available, 0) AS refill_ok,
           COALESCE(o.service_cancel_snapshot, s.cancel_available, 0) AS cancel_ok
    FROM orders o LEFT JOIN services s ON o.service_id = s.id
"""


def _order_json(o) -> Dict[str, Any]:
    qty = _int(o["quantity"])
    rem = _int(o["remains"])
    return {
        "id": o["id"], "user_id": o["user_id"], "service_name": o["service_name"], "link": o["link"],
        "quantity": qty, "charge": o["charge"], "status": o["status"],
        "start_count": _int(o["start_count"]), "remains": rem, "delivered": max(0, qty - rem),
        "refill": bool(o["refill_ok"]), "refill_days": _int(o["refill_days"]),
        "refill_status": o["refill_status"] or "None", "cancel": bool(o["cancel_ok"]),
        "refund_state": o["refund_state"], "refunded_amount": o["refunded_amount"],
        "created_at": str(o["created_at"])[:16], "updated_at": str(o["updated_at"])[:16],
    }


STATUS_GROUPS = {
    "active": ("Pending", "Processing", "In Progress"),
    "completed": ("Completed",),
    "partial": ("Partial",),
    "cancelled": ("Cancelled", "Failed"),
}


async def place_order_core(uid: int, srv_id: int, link: str, qty: int) -> Dict[str, Any]:
    """Same wallet-safe flow as the bot's order confirmation."""
    async with get_db() as db:
        s = await (await db.execute("SELECT * FROM services WHERE id = ?", (srv_id,))).fetchone()
    if not s or not s["is_active"]:
        return {"ok": False, "error": "Service is currently unavailable."}
    if qty < s["min_quantity"] or qty > s["max_quantity"]:
        return {"ok": False, "error": f"Quantity must be between {s['min_quantity']:,} and {s['max_quantity']:,}."}
    cost = quantize_currency((Decimal(qty) / Decimal("1000")) * Decimal(str(s["selling_rate"])))

    async with get_db() as db:
        await db.execute("BEGIN IMMEDIATE")
        try:
            u = await (await db.execute("SELECT balance FROM users WHERE user_id = ?", (uid,))).fetchone()
            if not u or Decimal(str(u["balance"])) < cost:
                await db.rollback()
                return {"ok": False, "error": "Insufficient balance. Please add funds first.", "code": "balance"}
            deducted = await modify_wallet_atomic(db, uid, -cost, "Order Deduction",
                                                  description=f"Order for {s['name']}")
            if not deducted:
                await db.rollback()
                return {"ok": False, "error": "Balance deduction error."}
            await db.commit()
        except Exception as e:
            await db.rollback()
            logger.error(f"Mini app order deduction error: {e}", exc_info=True)
            return {"ok": False, "error": "Order failed. Please try again."}

    discount_applied = "0.00"
    original_rate = str(s["selling_rate"])
    if await get_setting("DISCOUNT_ENABLED", "0") == "1":
        discount_applied = await get_setting("GLOBAL_DISCOUNT_PERCENT", "0.00")
        try:
            disc = Decimal(discount_applied)
            if disc > 0:
                original_rate = str(quantize_currency(
                    Decimal(str(s["selling_rate"])) / (Decimal("1") - disc / Decimal("100"))))
        except Exception:
            pass

    prov_res = await ProviderAPIClient.create_order(s["provider_id"], s["provider_service_id"], link, qty)
    prov_order_id = prov_res.get("order") if isinstance(prov_res, dict) else None

    insert_cols = (
        "user_id, service_id, provider_id, {prov}link, quantity, charge, status, refill_available, refill_days, "
        "service_name_snapshot, service_description_snapshot, service_category_snapshot, "
        "service_provider_snapshot, provider_service_id_snapshot, service_rate_snapshot, "
        "service_min_snapshot, service_max_snapshot, service_refill_snapshot, "
        "service_refill_days_snapshot, service_cancel_snapshot, discount_applied_percent, original_rate_snapshot")

    def snap_args():
        return (s["refill_available"], s["refill_days"], s["name"], s["description"] or "",
                str(s["category_id"] or ""), str(s["provider_id"] or ""), str(s["provider_service_id"] or ""),
                str(s["selling_rate"] or s["provider_rate"] or ""), int(s["min_quantity"] or 0),
                int(s["max_quantity"] or 0), int(s["refill_available"] or 0), int(s["refill_days"] or 0),
                int(s["cancel_available"] or 0), str(discount_applied), str(original_rate))

    async with get_db() as db:
        try:
            if prov_order_id:
                cur = await db.execute(
                    f"INSERT INTO orders ({insert_cols.format(prov='provider_order_id, ')}) "
                    f"VALUES (?, ?, ?, ?, ?, ?, ?, 'Processing', {', '.join(['?'] * 15)})",
                    (uid, srv_id, s["provider_id"], str(prov_order_id), link, qty, str(cost)) + snap_args())
                oid = cur.lastrowid
                await db.commit()
                if WEBAPP_BOT is not None:
                    await notify_admins(WEBAPP_BOT, f"📦 <b>NEW ORDER #{oid}</b> (Mini App)\n👤 User: <code>{uid}</code>\n🛍️ {html.escape(s['name'])}\n🔢 {qty:,}\n💰 ₹{cost}\n🕒 {get_ist_time()}")
                    await send_channel_log(WEBAPP_BOT, "ORDER", f"📦 <b>New Order #{oid}</b> (Mini App)\n• User: <code>{uid}</code>\n• Service: <b>{html.escape(s['name'])}</b>\n• Quantity: {qty:,}\n• Amount: ₹{cost}")
                return {"ok": True, "order_id": oid, "status": "Processing", "charge": str(cost)}
            err_msg = prov_res.get("error", "Provider API rejection") if isinstance(prov_res, dict) else str(prov_res)
            if str(err_msg) == "PROVIDER_TIMEOUT":
                cur = await db.execute(
                    f"INSERT INTO orders ({insert_cols.format(prov='')}) "
                    f"VALUES (?, ?, ?, ?, ?, ?, 'Pending', {', '.join(['?'] * 15)})",
                    (uid, srv_id, s["provider_id"], link, qty, str(cost)) + snap_args())
                oid = cur.lastrowid
                await db.commit()
                return {"ok": True, "order_id": oid, "status": "Pending", "charge": str(cost),
                        "note": "Provider did not respond in time. Your balance is kept reserved while the order is reconciled."}
            await modify_wallet_atomic(db, uid, cost, "Full Refund",
                                       description=f"Auto-Refund: Order creation rejected ({err_msg})")
            await db.commit()
            return {"ok": False, "error": f"Order failed: {err_msg}. Amount refunded to your wallet."}
        except Exception as db_err:
            logger.error(f"Mini app DB insert after provider success failed: {db_err}", exc_info=True)
            if WEBAPP_BOT is not None:
                await notify_admins(WEBAPP_BOT, f"🚨 <b>CRITICAL ORDER RECONCILIATION</b> (Mini App)\n\n👤 User: <code>{uid}</code>\n🛍️ Service: <code>{srv_id}</code>\n🔢 Quantity: <b>{qty:,}</b>\n💰 Charged: <b>₹{cost}</b>\n⚠️ Provider accepted/DB insert may be inconsistent.")
            return {"ok": False, "error": "Order reconciliation required. Support has been alerted; your balance was not auto-refunded."}


async def api_place_order(request: web.Request):
    uid = request["uid"]
    if await webapp_maintenance(uid, request["is_admin"], "ORDER"):
        return jerr(MAINT_MSG, 503, code="maintenance")
    b = await body_of(request)
    link = str(b.get("link") or "").strip()
    if not (link.startswith("http://") or link.startswith("https://") or link.startswith("tg://")):
        return jerr("Invalid link. It must start with http:// or https://")
    srv_id, qty = _int(b.get("service_id")), _int(b.get("quantity"))
    if srv_id <= 0 or qty <= 0:
        return jerr("Invalid service or quantity.")
    lock = await get_order_confirm_lock(uid)
    if lock.locked():
        return jerr("⏳ Your previous order is still processing.", 429)
    async with lock:
        res = await place_order_core(uid, srv_id, link, qty)
    return jresp(res, 200 if res.get("ok") else 400)


async def api_orders(request: web.Request):
    uid = request["uid"]
    page = max(1, _int(request.query.get("page"), 1))
    group = (request.query.get("status") or "all").lower()
    per = 15
    where, args = "WHERE o.user_id = ?", [uid]
    if group in STATUS_GROUPS:
        sts = STATUS_GROUPS[group]
        where += f" AND o.status IN ({','.join('?' * len(sts))})"
        args += list(sts)
    async with get_db() as db:
        total = (await (await db.execute(f"SELECT COUNT(*) FROM orders o {where}", args)).fetchone())[0]
        rows = await (await db.execute(
            f"{ORDER_SELECT} {where} ORDER BY o.id DESC LIMIT ? OFFSET ?", args + [per, (page - 1) * per])).fetchall()
    return jresp({"ok": True, "orders": [_order_json(o) for o in rows], "page": page,
                  "pages": max(1, (total + per - 1) // per), "total": total})


async def api_order(request: web.Request):
    oid = _int(request.match_info["oid"])
    async with get_db() as db:
        o = await (await db.execute(f"{ORDER_SELECT} WHERE o.id = ? AND o.user_id = ?", (oid, request["uid"]))).fetchone()
    if not o:
        return jerr("Order not found.", 404)
    return jresp({"ok": True, "order": _order_json(o)})


def _map_provider_status(raw: str) -> str:
    raw = (raw or "").strip().lower()
    return {"pending": "Pending", "in progress": "Processing", "processing": "Processing",
            "completed": "Completed", "partial": "Partial", "canceled": "Cancelled",
            "cancelled": "Cancelled", "failed": "Failed"}.get(raw, "Processing")


async def sync_order_with_provider(o) -> Optional[str]:
    """Pull live status from the provider, apply automatic refunds, notify like the bot."""
    if not o or not o["provider_order_id"]:
        return None
    res = await ProviderAPIClient.get_order_status(o["provider_id"], o["provider_order_id"])
    if not (isinstance(res, dict) and "status" in res):
        return None
    remains = _int(res.get("remains"))
    start_count = _int(res.get("start_count"))
    mapped = _map_provider_status(str(res.get("status", "")))
    notify_refund = Decimal("0.00")
    async with get_db() as db:
        cur_o = await (await db.execute("SELECT * FROM orders WHERE id = ?", (o["id"],))).fetchone()
        if mapped == "Partial" and cur_o["refund_state"] == "NOT_REFUNDED":
            ordered_qty = Decimal(str(cur_o["quantity"]))
            charge_amt = Decimal(str(cur_o["charge"]))
            refund_val = quantize_currency((Decimal(remains) / ordered_qty) * charge_amt) if ordered_qty > 0 else Decimal("0.00")
            if refund_val > 0:
                notify_refund = refund_val
                await modify_wallet_atomic(db, cur_o["user_id"], refund_val, "Partial Refund",
                                           reference_id=str(cur_o["id"]),
                                           description=f"Partial refund for order #{cur_o['id']}")
                await db.execute("INSERT INTO refunds (order_id, user_id, amount, reason) VALUES (?, ?, ?, 'Partial Order Unfinished Quantity')",
                                 (cur_o["id"], cur_o["user_id"], str(refund_val)))
                await db.execute("UPDATE orders SET refund_state = 'PARTIALLY_REFUNDED', refunded_amount = ? WHERE id = ?",
                                 (str(refund_val), cur_o["id"]))
        elif mapped in ("Cancelled", "Failed") and cur_o["refund_state"] == "NOT_REFUNDED":
            charge_amt = Decimal(str(cur_o["charge"]))
            notify_refund = charge_amt
            await modify_wallet_atomic(db, cur_o["user_id"], charge_amt, "Full Refund",
                                       reference_id=str(cur_o["id"]),
                                       description=f"Full refund for {mapped.lower()} order #{cur_o['id']}")
            await db.execute("INSERT INTO refunds (order_id, user_id, amount, reason) VALUES (?, ?, ?, ?)",
                             (cur_o["id"], cur_o["user_id"], str(charge_amt), f"{mapped} Order by Provider"))
            await db.execute("UPDATE orders SET refund_state = 'FULLY_REFUNDED', refunded_amount = ? WHERE id = ?",
                             (str(charge_amt), cur_o["id"]))
        await db.execute("UPDATE orders SET status = ?, start_count = ?, remains = ?, updated_at = CURRENT_TIMESTAMP WHERE id = ?",
                         (mapped, start_count, remains, cur_o["id"]))
        await db.commit()
    if cur_o["status"] != mapped and WEBAPP_BOT is not None:
        await notify_user_order_status_change(WEBAPP_BOT, cur_o["id"], cur_o["user_id"],
                                              cur_o["service_name_snapshot"] or "Service", mapped, notify_refund)
    return mapped


async def api_order_sync(request: web.Request):
    oid = _int(request.match_info["oid"])
    async with get_db() as db:
        o = await (await db.execute("SELECT * FROM orders WHERE id = ? AND user_id = ?", (oid, request["uid"]))).fetchone()
    if not o or not o["provider_order_id"]:
        return jerr("Unable to sync with provider.", 400)
    mapped = await sync_order_with_provider(o)
    if mapped is None:
        return jerr("Status check timed out. Try again.", 504)
    async with get_db() as db:
        o2 = await (await db.execute(f"{ORDER_SELECT} WHERE o.id = ?", (oid,))).fetchone()
    return jresp({"ok": True, "order": _order_json(o2)})


async def refill_core(o, uid_for_row: int, enforce_rules: bool = True) -> Dict[str, Any]:
    oid = o["id"]
    if enforce_rules:
        if not o["refill_available"]:
            return {"ok": False, "error": "This service does not support refills."}
        if o["status"] not in ("Completed", "Partial"):
            return {"ok": False, "error": f"Refills only allowed for completed orders. Current: {o['status']}"}
    async with get_db() as db:
        claimed = await db.execute(
            "UPDATE orders SET refill_status='Processing' WHERE id=? AND refill_status NOT IN ('Pending','Processing')", (oid,))
        await db.commit()
        if claimed.rowcount != 1:
            return {"ok": False, "error": "A refill request is already pending or being processed."}
    if enforce_rules:
        try:
            created_dt = datetime.strptime(str(o["created_at"])[:19], "%Y-%m-%d %H:%M:%S").replace(tzinfo=timezone.utc)
            days = _int(o["refill_days"])
            if days > 0 and datetime.now(timezone.utc) - created_dt > timedelta(days=days):
                async with get_db() as db:
                    await db.execute("UPDATE orders SET refill_status='None' WHERE id=? AND refill_status='Processing'", (oid,))
                    await db.commit()
                return {"ok": False, "error": f"Refill period expired. Valid within {days} days."}
        except Exception:
            pass
    res = await ProviderAPIClient.request_refill(o["provider_id"], o["provider_order_id"])
    ok = isinstance(res, dict) and (res.get("refill") or res.get("status") == "success" or not res.get("error"))
    if ok:
        pref = str(res.get("refill", ""))
        async with get_db() as db:
            await db.execute("INSERT INTO refills (order_id, user_id, provider_id, provider_order_id, provider_refill_id, status) VALUES (?, ?, ?, ?, ?, 'Pending')",
                             (oid, o["user_id"], o["provider_id"], o["provider_order_id"], pref))
            await db.execute("UPDATE orders SET refill_status = 'Pending', provider_refill_id = ? WHERE id = ?", (pref, oid))
            await db.commit()
        if WEBAPP_BOT is not None:
            await send_channel_log(WEBAPP_BOT, "REFILL", f"♻️ <b>Refill Requested:</b> Order #{oid}\n• User: <code>{o['user_id']}</code>")
        return {"ok": True}
    err = res.get("error", "Provider rejected refill.") if isinstance(res, dict) else str(res)
    async with get_db() as db:
        await db.execute("UPDATE orders SET refill_status='None' WHERE id=? AND refill_status='Processing'", (oid,))
        await db.commit()
    return {"ok": False, "error": str(err)}


async def api_order_refill(request: web.Request):
    oid = _int(request.match_info["oid"])
    uid = request["uid"]
    async with get_db() as db:
        o = await (await db.execute("SELECT * FROM orders WHERE id = ? AND user_id = ?", (oid, uid))).fetchone()
    if not o:
        return jerr("Order not found.", 404)
    lock = WEBAPP_REFILL_LOCKS.setdefault(oid, asyncio.Lock())
    if lock.locked():
        return jerr("Refill already in progress.", 429)
    async with lock:
        res = await refill_core(o, uid)
    return jresp(res, 200 if res.get("ok") else 400)


# -------------------------------- profile ---------------------------------
async def api_profile(request: web.Request):
    uid = request["uid"]
    async with get_db() as db:
        u = await (await db.execute("SELECT * FROM users WHERE user_id = ?", (uid,))).fetchone()
        async def cnt(sql, *a):
            return (await (await db.execute(sql, a)).fetchone())[0]
        stats = {
            "total": await cnt("SELECT COUNT(*) FROM orders WHERE user_id = ?", uid),
            "completed": await cnt("SELECT COUNT(*) FROM orders WHERE user_id = ? AND status = 'Completed'", uid),
            "active": await cnt("SELECT COUNT(*) FROM orders WHERE user_id = ? AND status IN ('Pending','Processing','In Progress')", uid),
            "partial": await cnt("SELECT COUNT(*) FROM orders WHERE user_id = ? AND status = 'Partial'", uid),
            "cancelled": await cnt("SELECT COUNT(*) FROM orders WHERE user_id = ? AND status IN ('Cancelled','Failed')", uid),
        }
        refs = await cnt("SELECT COUNT(*) FROM users WHERE referrer_id = ?", uid)
    uname = _BOT_INFO.get("username") or "Bot"
    return jresp({"ok": True, "profile": {
        "id": uid, "first_name": u["first_name"] or "User", "username": u["username"] or "",
        "balance": u["balance"], "total_spent": u["total_spent"], "total_deposited": u["total_deposited"],
        "referral_earnings": u["referral_earnings"], "currency": u["currency"] or "INR",
        "joined": str(u["created_at"])[:10],
    }, "stats": stats, "referral": {
        "count": refs, "link": f"https://t.me/{uname}?start=ref_{uid}",
        "percent": await get_setting("REFERRAL_PERCENT", "5.00"),
        "enabled": await get_setting("REFERRAL_ENABLED", "1") == "1",
    }})


async def api_set_currency(request: web.Request):
    cur = str((await body_of(request)).get("currency") or "").upper()
    if cur not in ("INR", "USD"):
        return jerr("Invalid currency.")
    async with get_db() as db:
        await db.execute("UPDATE users SET currency = ? WHERE user_id = ?", (cur, request["uid"]))
        await db.commit()
    return jresp({"ok": True, "currency": cur})


async def api_transactions(request: web.Request):
    async with get_db() as db:
        rows = await (await db.execute(
            "SELECT id, type, amount, balance_after, description, created_at FROM transactions WHERE user_id = ? ORDER BY id DESC LIMIT 60",
            (request["uid"],))).fetchall()
    return jresp({"ok": True, "transactions": [
        {**row_dict(r), "created_at": str(r["created_at"])[:16]} for r in rows]})


async def api_redeem_gift(request: web.Request):
    uid = request["uid"]
    code = str((await body_of(request)).get("code") or "").strip().upper()
    if not code or len(code) > 50:
        return jerr("Invalid gift card code.")
    async with get_db() as db:
        await db.execute("BEGIN IMMEDIATE")
        try:
            gc = await (await db.execute("SELECT * FROM gift_cards WHERE UPPER(code) = ? AND is_active = 1", (code,))).fetchone()
            if not gc:
                await db.rollback()
                return jerr("Invalid or inactive gift card code.")
            if await (await db.execute("SELECT id FROM gift_card_redemptions WHERE UPPER(code) = ? AND user_id = ?", (code, uid))).fetchone():
                await db.rollback()
                return jerr("You have already redeemed this gift card.")
            if gc["used_count"] >= gc["max_uses"]:
                await db.rollback()
                return jerr("This gift card has reached its limit.")
            amt = Decimal(str(gc["amount"]))
            await db.execute("INSERT INTO gift_card_redemptions (code, user_id, amount) VALUES (?, ?, ?)", (code, uid, str(amt)))
            await db.execute("UPDATE gift_cards SET used_count = used_count + 1 WHERE UPPER(code) = ?", (code,))
            ok = await modify_wallet_atomic(db, uid, amt, "Gift Card Redemption", reference_id=code, description=f"Redeemed code {code}")
            if not ok:
                await db.rollback()
                return jerr("Failed to add balance. Try again.")
            await db.commit()
        except Exception as e:
            await db.rollback()
            logger.error(f"Mini app gift card error: {e}", exc_info=True)
            return jerr("System error. Try again.", 500)
    if WEBAPP_BOT is not None:
        await send_channel_log(WEBAPP_BOT, "PAYMENT", f"🎁 <b>Gift Card Redeemed</b> (Mini App)\n• User: <code>{uid}</code>\n• Code: <code>{code}</code>\n• Amount: ₹{amt}")
    return jresp({"ok": True, "amount": str(amt)})


# -------------------------------- deposits --------------------------------
def _deposit_json(d) -> Dict[str, Any]:
    return {"id": d["id"], "method": d["method"], "amount": d["amount"], "crypto_amount": d["crypto_amount"],
            "utr": d["utr"] or "", "reference": d["reference"] or "", "status": d["status"],
            "created_at": str(d["created_at"])[:16], "expires_at": d["expires_at"]}


async def api_deposits(request: web.Request):
    async with get_db() as db:
        rows = await (await db.execute("SELECT * FROM deposits WHERE user_id = ? ORDER BY id DESC LIMIT 40", (request["uid"],))).fetchall()
    return jresp({"ok": True, "deposits": [_deposit_json(d) for d in rows]})


def _make_qr_data_uri(text: str) -> Optional[str]:
    try:
        return "data:image/svg+xml;base64," + base64.b64encode(make_qr_svg(text).encode()).decode()
    except Exception as e:
        logger.error(f"Mini app QR (svg) generation failed: {e}", exc_info=True)
    try:
        return "data:image/png;base64," + base64.b64encode(make_qr_png_bytes(text, 8, 3)).decode()
    except Exception as e:
        logger.error(f"Mini app QR (png) generation failed: {e}", exc_info=True)
        return None


_BG_TASKS: set = set()


def _spawn_bg(coro) -> None:
    """Run slow side-effects (admin/log-channel messages) without delaying the API response."""
    async def runner():
        try:
            await coro
        except Exception as e:
            logger.warning("Background task failed: %s", e)
    t = asyncio.create_task(runner())
    _BG_TASKS.add(t)
    t.add_done_callback(_BG_TASKS.discard)


def _qr_fallback_url(link: str) -> str:
    return f"{QR_BASE}?text={quote(link, safe='')}&size=500&margin=2"


async def _parse_inr_amount(raw) -> Union[Decimal, str]:
    raw_s = str(raw if raw is not None else "").strip()
    if not re.fullmatch(r"[0-9]{1,9}", raw_s) or int(raw_s) <= 0:
        return "Invalid amount. Enter a positive whole number (e.g. 100)."
    try:
        amt = quantize_currency(raw_s)
        if amt <= 0:
            raise ValueError
    except Exception:
        return "Invalid amount. Enter a positive whole number (e.g. 100)."
    mn, mx = Decimal(await get_setting("MIN_DEPOSIT", "1.00")), Decimal(await get_setting("MAX_DEPOSIT", "50000.00"))
    if amt < mn or amt > mx:
        return f"Amount must be between ₹{mn} and ₹{mx}."
    return amt


async def api_dep_upi_auto(request: web.Request):
    uid = request["uid"]
    if await webapp_maintenance(uid, request["is_admin"], "DEPOSIT"):
        return jerr(MAINT_MSG, 503, code="maintenance")
    if not await payment_method_enabled("UPI_AUTO"):
        return jerr("UPI 1 is disabled by admin.")
    if await get_user_currency(uid) == "USD":
        return jerr("USD accounts support crypto payments only.")
    amt = await _parse_inr_amount((await body_of(request)).get("amount"))
    if isinstance(amt, str):
        return jerr(amt)
    ref = await new_upi_tr()
    expires_at = time.time() + DEPOSIT_EXPIRY_SECONDS
    async with get_db() as db:
        cur = await db.execute("INSERT INTO deposits (user_id, method, amount, status, reference, expires_at, purpose) VALUES (?, 'UPI_AUTO', ?, 'PENDING', ?, ?, ?)",
                               (uid, str(amt), ref, expires_at, ref))
        dep_id = cur.lastrowid
        await db.commit()
    upi_id_val, upi_name_val = await get_auto_upi()
    upi_link = build_upi_pay_link(upi_id_val, upi_name_val, ref)
    qr_uri = await asyncio.to_thread(_make_qr_data_uri, upi_link)
    if WEBAPP_BOT is not None:
        async def _notify():
            await notify_admins(WEBAPP_BOT, f"🟢 <b>NEW UPI 1 DEPOSIT</b> (Mini App)\n👤 <code>{uid}</code>\n💰 ₹{amt}\n🆔 #{dep_id}\n🔖 <code>{ref}</code>")
            await send_channel_log(WEBAPP_BOT, "PAYMENT", f"🟢 <b>New UPI 1 Deposit</b> (Mini App)\n• User: <code>{uid}</code>\n• Amount: ₹{amt}\n• Deposit: #{dep_id}")
        _spawn_bg(_notify())
    return jresp({"ok": True, "deposit": {
        "id": dep_id, "amount": str(amt), "reference": ref, "upi_id": upi_id_val, "payee": upi_name_val,
        "upi_link": upi_link, "qr": qr_uri,
        "qr_fallback": _qr_fallback_url(upi_link), "expires_at": expires_at,
        "seconds_left": DEPOSIT_EXPIRY_SECONDS}})


async def api_dep_upi_check(request: web.Request):
    uid = request["uid"]
    dep_id = _int((await body_of(request)).get("id"))
    res = await process_upi_auto_check(dep_id, uid, WEBAPP_BOT)
    st = res["state"]
    if st == "notfound":
        return jerr("Deposit not found.", 404)
    if st == "success":
        return jresp({"ok": True, "state": "approved", "amount": str(res["amount"]), "balance": res["balance"], "utr": res["utr"],
                      "message": f"Payment successful! ₹{res['amount']} added to your wallet."})
    if st == "already":
        return jresp({"ok": True, "state": "approved", "amount": str(res["dep"]["amount"]), "message": "Payment already credited."})
    if st == "expired":
        return jresp({"ok": True, "state": "expired", "message": "QR expired. Please start a new deposit."})
    if st == "unpaid":
        return jresp({"ok": True, "state": "unpaid", "message": UNPAID_MESSAGE})
    if st == "pending":
        return jresp({"ok": True, "state": "pending", "message": "Your payment is still being confirmed. Wait a moment, then click Check Payment."})
    if st == "duplicate":
        return jresp({"ok": True, "state": "duplicate", "message": "This payment was already used. Balance not credited."})
    if st == "error":
        return jresp({"ok": True, "state": "error", "message": res.get("message") or "Could not verify right now. Please try again."})
    if st == "noref":
        return jresp({"ok": True, "state": "failed", "message": "Reference missing. Please start again."})
    if st == "closed":
        return jresp({"ok": True, "state": "failed", "message": f"Deposit status: {res.get('status')}"})
    return jresp({"ok": True, "state": "error", "message": res.get("message") or "System error. Contact support."})


async def api_dep_cancel(request: web.Request):
    uid = request["uid"]
    dep_id = _int((await body_of(request)).get("id"))
    async with get_db() as db:
        dep = await (await db.execute("SELECT * FROM deposits WHERE id = ? AND user_id = ? AND status = 'PENDING'", (dep_id, uid))).fetchone()
        await db.execute("UPDATE deposits SET status = 'CANCELLED' WHERE id = ? AND user_id = ? AND status = 'PENDING'", (dep_id, uid))
        await db.commit()
    if dep and WEBAPP_BOT is not None:
        text = await notify_user_deposit_event(WEBAPP_BOT, uid, "❌ <b>DEPOSIT CANCELLED</b>", dep_id, dep["amount"], "\n\nYour deposit request was cancelled.")
        await notify_admins(WEBAPP_BOT, text + f"\n👤 User: <code>{uid}</code>")
        await send_channel_log(WEBAPP_BOT, "PAYMENT", text + f"\n👤 User: <code>{uid}</code>")
    return jresp({"ok": True})


async def api_dep_manual(request: web.Request):
    uid = request["uid"]
    if await webapp_maintenance(uid, request["is_admin"], "DEPOSIT"):
        return jerr(MAINT_MSG, 503, code="maintenance")
    if not await payment_method_enabled("UPI_MANUAL"):
        return jerr("UPI 2 is disabled by admin.")
    if await get_user_currency(uid) == "USD":
        return jerr("USD accounts support crypto payments only.")
    amt = await _parse_inr_amount((await body_of(request)).get("amount"))
    if isinstance(amt, str):
        return jerr(amt)
    ref = generate_ref()
    async with get_db() as db:
        cur = await db.execute("INSERT INTO deposits (user_id, method, amount, status, reference, purpose) VALUES (?, 'UPI_MANUAL', ?, 'PENDING', ?, 'MANUAL_UPI')", (uid, str(amt), ref))
        dep_id = cur.lastrowid
        await db.commit()
    upi_id_val = await get_setting("UPI_ID", UPI_ID)
    upi_name_val = await get_setting("UPI_NAME", BOT_NAME)
    upi_link = (f"upi://pay?pa={quote(str(upi_id_val), safe='')}&pn={quote(str(upi_name_val), safe='')}"
                f"&tn={quote(str(ref), safe='')}&cu=INR&am={quote(format(amt, 'f'), safe='')}")
    qr_uri = await asyncio.to_thread(_make_qr_data_uri, upi_link)
    return jresp({"ok": True, "deposit": {"id": dep_id, "amount": str(amt), "reference": ref, "upi_id": upi_id_val,
                                          "payee": upi_name_val, "upi_link": upi_link, "qr": qr_uri,
                                          "qr_fallback": _qr_fallback_url(upi_link)}})


async def api_dep_manual_utr(request: web.Request):
    uid = request["uid"]
    b = await body_of(request)
    dep_id, utr = _int(b.get("id")), str(b.get("utr") or "").strip()
    if not (utr.isdigit() and 4 <= len(utr) <= 30):
        return jerr("Invalid UTR. Numeric 4-30 digits.")
    async with get_db() as db:
        dep = await (await db.execute("SELECT * FROM deposits WHERE id = ? AND user_id = ? AND method = 'UPI_MANUAL'", (dep_id, uid))).fetchone()
        if not dep or dep["status"] != "PENDING":
            return jerr("Deposit is no longer pending.")
        if await (await db.execute("SELECT id FROM deposits WHERE utr = ? AND status IN ('PENDING','APPROVED') LIMIT 1", (utr,))).fetchone():
            return jerr("This UTR is already submitted. Use a different UTR.")
        await db.execute("UPDATE deposits SET utr = ? WHERE id = ? AND status = 'PENDING'", (utr, dep_id))
        await db.commit()
    await _notify_admin_deposit_request(uid, dep, "🟡 <b>NEW UPI 2 DEPOSIT</b> (Mini App)", f"🔢 UTR: <code>{utr}</code>\n", dep_id)
    return jresp({"ok": True})


async def _notify_admin_deposit_request(uid: int, dep, title: str, extra: str, dep_id: int):
    if WEBAPP_BOT is None:
        return
    await send_channel_log(WEBAPP_BOT, "PAYMENT", f"{title}\n• User: <code>{uid}</code>\n• Amount: ₹{dep['amount']}\n• Deposit: #{dep_id}")
    kb = InlineKeyboardMarkup(inline_keyboard=[[
        InlineKeyboardButton(text="✅ Approve", callback_data=f"amanual_approve_{dep_id}"),
        InlineKeyboardButton(text="❌ Reject", callback_data=f"amanual_reject_{dep_id}")]])
    await notify_admins(WEBAPP_BOT, f"{title}\n\n👤 User: <code>{uid}</code>\n💰 Amount: <b>₹{dep['amount']}</b>\n{extra}🆔 Deposit: #{dep_id}", reply_markup=kb)


async def api_dep_usdt_info(request: web.Request):
    if not await payment_method_enabled("USDT_BEP20"):
        return jerr("USDT payment is disabled by admin.")
    rate = await get_live_usdt_inr_rate(force=True)
    return jresp({"ok": True, "rate": str(rate), "min": await get_setting("USDT_MIN_DEPOSIT", "1.00"),
                  "max": await get_setting("USDT_MAX_DEPOSIT", "10000.00"),
                  "address": await get_setting("USDT_BEP20_ADDRESS", DEFAULT_SETTINGS["USDT_BEP20_ADDRESS"])})


async def api_dep_usdt(request: web.Request):
    uid = request["uid"]
    if await webapp_maintenance(uid, request["is_admin"], "DEPOSIT"):
        return jerr(MAINT_MSG, 503, code="maintenance")
    if not await payment_method_enabled("USDT_BEP20"):
        return jerr("USDT payment is disabled by admin.")
    try:
        crypto = Decimal(str((await body_of(request)).get("amount")).strip()).quantize(Decimal("0.000001"), rounding=ROUND_HALF_UP)
        if crypto <= 0:
            raise ValueError
    except Exception:
        return jerr("Invalid USDT amount. Example: 5.00")
    mn, mx = Decimal(await get_setting("USDT_MIN_DEPOSIT", "1.00")), Decimal(await get_setting("USDT_MAX_DEPOSIT", "10000.00"))
    if crypto < mn or crypto > mx:
        return jerr(f"USDT amount must be between {mn} and {mx}.")
    rate = await get_live_usdt_inr_rate(force=True)
    inr = quantize_currency(crypto * rate)
    async with get_db() as db:
        cur = await db.execute("INSERT INTO deposits (user_id, method, amount, crypto_amount, status, purpose) VALUES (?, 'USDT_BEP20', ?, ?, 'PENDING', 'USDT_BEP20')", (uid, str(inr), str(crypto)))
        dep_id = cur.lastrowid
        await db.commit()
    return jresp({"ok": True, "deposit": {"id": dep_id, "usdt": str(crypto), "inr": str(inr), "rate": str(rate),
                                          "address": await get_setting("USDT_BEP20_ADDRESS", DEFAULT_SETTINGS["USDT_BEP20_ADDRESS"])}})


async def api_dep_usdt_txid(request: web.Request):
    uid = request["uid"]
    b = await body_of(request)
    dep_id, txid = _int(b.get("id")), str(b.get("txid") or "").strip()
    if len(txid) < 10 or len(txid) > 120:
        return jerr("Invalid TXID. Send the transaction hash.")
    async with get_db() as db:
        dep = await (await db.execute("SELECT * FROM deposits WHERE id = ? AND user_id = ? AND method = 'USDT_BEP20'", (dep_id, uid))).fetchone()
        if not dep or dep["status"] != "PENDING":
            return jerr("Deposit not found or already processed.")
        if await (await db.execute("SELECT id FROM deposits WHERE utr = ? AND status IN ('PENDING','APPROVED') LIMIT 1", (txid,))).fetchone():
            return jerr("This TXID is already submitted.")
        await db.execute("UPDATE deposits SET utr = ? WHERE id = ? AND status = 'PENDING'", (txid, dep_id))
        await db.commit()
    await _notify_admin_deposit_request(uid, dep, "🔵 <b>NEW USDT DEPOSIT</b> (Mini App)",
                                        f"💰 USDT: <b>{dep['crypto_amount']}</b>\n🔗 TXID: <code>{txid}</code>\n", dep_id)
    return jresp({"ok": True})


# ---------------------------------- admin ---------------------------------
async def api_admin_dashboard(request: web.Request):
    async with get_db() as db:
        async def one(sql, *a):
            return (await (await db.execute(sql, a)).fetchone())[0]
        d = {
            "users": await one("SELECT COUNT(*) FROM users"),
            "banned": await one("SELECT COUNT(*) FROM users WHERE is_banned = 1"),
            "orders": await one("SELECT COUNT(*) FROM orders"),
            "pending": await one("SELECT COUNT(*) FROM orders WHERE status = 'Pending'"),
            "processing": await one("SELECT COUNT(*) FROM orders WHERE status IN ('Processing','In Progress')"),
            "completed": await one("SELECT COUNT(*) FROM orders WHERE status = 'Completed'"),
            "partial": await one("SELECT COUNT(*) FROM orders WHERE status = 'Partial'"),
            "cancelled": await one("SELECT COUNT(*) FROM orders WHERE status = 'Cancelled'"),
            "failed": await one("SELECT COUNT(*) FROM orders WHERE status = 'Failed'"),
            "deposits": await one("SELECT COALESCE(SUM(CAST(amount AS REAL)),0) FROM deposits WHERE status = 'APPROVED'"),
            "refunds": await one("SELECT COALESCE(SUM(CAST(amount AS REAL)),0) FROM refunds"),
            "revenue": await one("SELECT COALESCE(SUM(CAST(charge AS REAL)),0) FROM orders"),
            "services": await one("SELECT COUNT(*) FROM services WHERE is_active = 1"),
            "providers": await one("SELECT COUNT(*) FROM providers WHERE status = 1"),
            "pending_deposits": await one("SELECT COUNT(*) FROM deposits WHERE status = 'PENDING' AND method IN ('UPI_MANUAL','USDT_BEP20')"),
            "wallets": await one("SELECT COALESCE(SUM(CAST(balance AS REAL)),0) FROM users"),
            "today_orders": await one("SELECT COUNT(*) FROM orders WHERE date(created_at) = date('now')"),
            "today_revenue": await one("SELECT COALESCE(SUM(CAST(charge AS REAL)),0) FROM orders WHERE date(created_at) = date('now')"),
            "today_users": await one("SELECT COUNT(*) FROM users WHERE date(created_at) = date('now')"),
        }
    return jresp({"ok": True, "dashboard": d})


async def api_admin_orders(request: web.Request):
    page = max(1, _int(request.query.get("page"), 1))
    group = (request.query.get("status") or "all").lower()
    q = (request.query.get("q") or "").strip()
    per = 15
    where, args = "WHERE 1=1", []
    if group in STATUS_GROUPS:
        sts = STATUS_GROUPS[group]
        where += f" AND o.status IN ({','.join('?' * len(sts))})"
        args += list(sts)
    if q.isdigit():
        where += " AND (o.id = ? OR o.user_id = ? OR o.provider_order_id = ?)"
        args += [int(q), int(q), q]
    elif q:
        where += " AND o.link LIKE ?"
        args.append(f"%{q}%")
    async with get_db() as db:
        total = (await (await db.execute(f"SELECT COUNT(*) FROM orders o {where}", args)).fetchone())[0]
        rows = await (await db.execute(f"{ORDER_SELECT} {where} ORDER BY o.id DESC LIMIT ? OFFSET ?", args + [per, (page - 1) * per])).fetchall()
    return jresp({"ok": True, "orders": [_order_json(o) for o in rows], "page": page,
                  "pages": max(1, (total + per - 1) // per), "total": total})


async def _admin_order_row(oid: int):
    async with get_db() as db:
        return await (await db.execute(f"{ORDER_SELECT} WHERE o.id = ?", (oid,))).fetchone()


async def api_admin_order(request: web.Request):
    o = await _admin_order_row(_int(request.match_info["oid"]))
    if not o:
        return jerr("Order not found.", 404)
    d = _order_json(o)
    d["provider_order_id"] = o["provider_order_id"] or ""
    return jresp({"ok": True, "order": d})


async def api_admin_order_status(request: web.Request):
    oid = _int(request.match_info["oid"])
    new_status = str((await body_of(request)).get("status") or "")
    if new_status not in ("Pending", "Processing", "In Progress", "Completed", "Partial"):
        return jerr("Use the Cancel button for refund options.")
    async with get_db() as db:
        old = await (await db.execute("SELECT user_id, status, service_name_snapshot FROM orders WHERE id = ?", (oid,))).fetchone()
        if not old:
            return jerr("Order not found.", 404)
        await db.execute("UPDATE orders SET status = ?, updated_at = CURRENT_TIMESTAMP WHERE id = ?", (new_status, oid))
        await db.commit()
    if old["status"] != new_status and WEBAPP_BOT is not None:
        await notify_user_order_status_change(WEBAPP_BOT, oid, old["user_id"], old["service_name_snapshot"] or "Service", new_status, Decimal("0.00"))
        await send_channel_log(WEBAPP_BOT, "ORDER", f"🔄 <b>Order #{oid}</b> (Mini App)\n• User: <code>{old['user_id']}</code>\n• {html.escape(old['status'])} → {html.escape(new_status)}\n• Admin: <code>{request['uid']}</code>")
    await log_admin_event(request["uid"], "SET_ORDER_STATUS", str(oid), f"Set to {new_status}")
    return jresp({"ok": True})


async def api_admin_order_cancel(request: web.Request):
    oid = _int(request.match_info["oid"])
    b = await body_of(request)
    mode = str(b.get("mode") or "")
    async with get_db() as db:
        o = await (await db.execute("SELECT * FROM orders WHERE id = ?", (oid,))).fetchone()
        if not o:
            return jerr("Order not found.", 404)
        charge, already = Decimal(str(o["charge"])), Decimal(str(o["refunded_amount"]))
        if mode == "none":
            await db.execute("UPDATE orders SET status = 'Cancelled', updated_at = CURRENT_TIMESTAMP WHERE id = ?", (oid,))
            await db.commit()
            await log_admin_event(request["uid"], "CANCEL_NO_REFUND", str(oid), "No refund")
            return jresp({"ok": True})
        if mode == "full":
            if o["refund_state"] == "FULLY_REFUNDED":
                return jerr("Already fully refunded.")
            net = charge - already
            if net > 0:
                await modify_wallet_atomic(db, o["user_id"], net, "Admin Full Refund", reference_id=str(oid), description=f"Full refund for order #{oid}")
                await db.execute("INSERT INTO refunds (order_id, user_id, amount, reason) VALUES (?, ?, ?, 'Admin Full Refund')", (oid, o["user_id"], str(net)))
            await db.execute("UPDATE orders SET status = 'Cancelled', refund_state = 'FULLY_REFUNDED', refunded_amount = ?, updated_at = CURRENT_TIMESTAMP WHERE id = ?", (str(charge), oid))
            await db.commit()
            await log_admin_event(request["uid"], "CANCEL_FULL_REFUND", str(oid), f"Refunded ₹{net}")
            return jresp({"ok": True, "refunded": str(net)})
        if mode == "partial":
            try:
                amt = quantize_currency(str(b.get("amount")).strip())
                if amt <= 0:
                    raise ValueError
            except Exception:
                return jerr("Invalid numeric refund amount.")
            max_ref = charge - already
            if amt > max_ref:
                return jerr(f"Cannot exceed remaining refundable (₹{max_ref}).")
            new_total = already + amt
            new_state = "FULLY_REFUNDED" if new_total >= charge else "PARTIALLY_REFUNDED"
            await modify_wallet_atomic(db, o["user_id"], amt, "Admin Partial Refund", reference_id=str(oid), description=f"Admin partial refund #{oid}")
            await db.execute("INSERT INTO refunds (order_id, user_id, amount, reason) VALUES (?, ?, ?, 'Admin Partial Refund')", (oid, o["user_id"], str(amt)))
            await db.execute("UPDATE orders SET refund_state = ?, refunded_amount = ?, updated_at = CURRENT_TIMESTAMP WHERE id = ?", (new_state, str(new_total), oid))
            await db.commit()
            await log_admin_event(request["uid"], "PARTIAL_REFUND", str(oid), f"₹{amt}")
            return jresp({"ok": True, "refunded": str(amt)})
    return jerr("Invalid cancel mode.")


async def api_admin_order_sync(request: web.Request):
    oid = _int(request.match_info["oid"])
    o = await _admin_order_row(oid)
    if not o or not o["provider_order_id"]:
        return jerr("Unable to sync order.")
    if await sync_order_with_provider(o) is None:
        return jerr("Provider sync failed.", 504)
    return jresp({"ok": True})


async def api_admin_order_refill(request: web.Request):
    oid = _int(request.match_info["oid"])
    async with get_db() as db:
        o = await (await db.execute("SELECT * FROM orders WHERE id = ?", (oid,))).fetchone()
    if not o or not o["provider_order_id"]:
        return jerr("Cannot refill.")
    res = await refill_core(o, o["user_id"], enforce_rules=False)
    return jresp(res, 200 if res.get("ok") else 400)


async def api_admin_order_refill_cancel(request: web.Request):
    oid = _int(request.match_info["oid"])
    async with get_db() as db:
        await db.execute("UPDATE orders SET refill_status = 'Cancelled' WHERE id = ?", (oid,))
        await db.execute("UPDATE refills SET status = 'Cancelled' WHERE order_id = ?", (oid,))
        await db.commit()
    return jresp({"ok": True})


async def api_admin_users(request: web.Request):
    page = max(1, _int(request.query.get("page"), 1))
    q = (request.query.get("q") or "").strip().lstrip("@")
    per = 15
    where, args = "WHERE 1=1", []
    if q.isdigit():
        where += " AND user_id = ?"
        args.append(int(q))
    elif q:
        where += " AND (username LIKE ? OR first_name LIKE ?)"
        args += [f"%{q}%", f"%{q}%"]
    async with get_db() as db:
        total = (await (await db.execute(f"SELECT COUNT(*) FROM users {where}", args)).fetchone())[0]
        rows = await (await db.execute(f"SELECT * FROM users {where} ORDER BY created_at DESC LIMIT ? OFFSET ?", args + [per, (page - 1) * per])).fetchall()
    return jresp({"ok": True, "users": [{
        "id": r["user_id"], "first_name": r["first_name"] or "User", "username": r["username"] or "",
        "balance": r["balance"], "total_spent": r["total_spent"], "banned": bool(r["is_banned"]),
        "joined": str(r["created_at"])[:10]} for r in rows], "page": page,
        "pages": max(1, (total + per - 1) // per), "total": total})


async def api_admin_user(request: web.Request):
    uid = _int(request.match_info["uid"])
    async with get_db() as db:
        u = await (await db.execute("SELECT * FROM users WHERE user_id = ?", (uid,))).fetchone()
        if not u:
            return jerr("User not found.", 404)
        oc = (await (await db.execute("SELECT COUNT(*) FROM orders WHERE user_id = ?", (uid,))).fetchone())[0]
        refs = (await (await db.execute("SELECT COUNT(*) FROM users WHERE referrer_id = ?", (uid,))).fetchone())[0]
        orders = await (await db.execute(f"{ORDER_SELECT} WHERE o.user_id = ? ORDER BY o.id DESC LIMIT 10", (uid,))).fetchall()
        txs = await (await db.execute("SELECT type, amount, balance_after, description, created_at FROM transactions WHERE user_id = ? ORDER BY id DESC LIMIT 10", (uid,))).fetchall()
    return jresp({"ok": True, "user": {
        "id": u["user_id"], "first_name": u["first_name"] or "User", "username": u["username"] or "",
        "balance": u["balance"], "total_spent": u["total_spent"], "total_deposited": u["total_deposited"],
        "referral_earnings": u["referral_earnings"], "referrals": refs, "orders": oc,
        "banned": bool(u["is_banned"]), "currency": u["currency"], "joined": str(u["created_at"])[:16]},
        "recent_orders": [_order_json(o) for o in orders],
        "recent_tx": [{**row_dict(t), "created_at": str(t["created_at"])[:16]} for t in txs]})


async def api_admin_user_ban(request: web.Request):
    uid = _int(request.match_info["uid"])
    if await is_admin(uid):
        return jerr("Cannot ban an admin.")
    async with get_db() as db:
        u = await (await db.execute("SELECT is_banned FROM users WHERE user_id = ?", (uid,))).fetchone()
        if not u:
            return jerr("User not found.", 404)
        nv = 0 if u["is_banned"] else 1
        await db.execute("UPDATE users SET is_banned = ? WHERE user_id = ?", (nv, uid))
        await db.commit()
    await log_admin_event(request["uid"], "BAN_TOGGLE", str(uid), "Banned" if nv else "Unbanned")
    return jresp({"ok": True, "banned": bool(nv)})


async def api_admin_user_balance(request: web.Request):
    uid = _int(request.match_info["uid"])
    b = await body_of(request)
    act = str(b.get("action") or "")
    reason = str(b.get("reason") or "").strip()[:200]
    try:
        amt = quantize_currency(str(b.get("amount")).strip())
        if amt <= 0 or act not in ("add", "deduct"):
            raise ValueError
    except Exception:
        return jerr("Enter a valid amount.")
    delta = amt if act == "add" else -amt
    async with get_db() as db:
        ok = await modify_wallet_atomic(db, uid, delta, "Admin Balance Adjustment", description=f"Admin #{request['uid']} Adj: {reason}")
        await db.commit()
    if not ok:
        return jerr("Adjustment failed (balance cannot become negative).")
    await log_admin_event(request["uid"], "BAL_ADJ", str(uid), f"{act.upper()} ₹{amt} | {reason}")
    return jresp({"ok": True})


async def api_admin_user_message(request: web.Request):
    uid = _int(request.match_info["uid"])
    text = str((await body_of(request)).get("text") or "").strip()
    if not text:
        return jerr("Message is empty.")
    try:
        await WEBAPP_BOT.send_message(uid, f"📩 <b>Support Message:</b>\n\n{html.escape(text)}", parse_mode=ParseMode.HTML)
    except Exception as e:
        return jerr(f"Failed to deliver: {e}")
    return jresp({"ok": True})


async def api_admin_deposits(request: web.Request):
    page = max(1, _int(request.query.get("page"), 1))
    st = (request.query.get("status") or "").upper()
    per = 15
    where, args = "WHERE 1=1", []
    if st in ("PENDING", "APPROVED", "REJECTED", "EXPIRED", "FAILED", "CANCELLED", "DUPLICATE"):
        where += " AND status = ?"
        args.append(st)
    async with get_db() as db:
        total = (await (await db.execute(f"SELECT COUNT(*) FROM deposits {where}", args)).fetchone())[0]
        rows = await (await db.execute(f"SELECT * FROM deposits {where} ORDER BY id DESC LIMIT ? OFFSET ?", args + [per, (page - 1) * per])).fetchall()
    return jresp({"ok": True, "deposits": [{**_deposit_json(d), "user_id": d["user_id"]} for d in rows],
                  "page": page, "pages": max(1, (total + per - 1) // per), "total": total})


async def api_admin_deposit_approve(request: web.Request):
    did = _int(request.match_info["did"])
    try:
        async with get_db() as db:
            await db.execute("BEGIN IMMEDIATE")
            dep = await (await db.execute("SELECT * FROM deposits WHERE id = ? AND status = 'PENDING'", (did,))).fetchone()
            if not dep:
                await db.rollback()
                return jerr("Deposit is not pending.")
            amt = quantize_currency(dep["amount"])
            upd = await db.execute("UPDATE deposits SET status = 'APPROVED', processed_at = CURRENT_TIMESTAMP WHERE id = ? AND status = 'PENDING'", (did,))
            if upd.rowcount != 1:
                await db.rollback()
                return jerr("Deposit state changed.")
            ok = await modify_wallet_atomic(db, dep["user_id"], amt, "Deposit", reference_id=dep["utr"] or dep["reference"] or str(did), description=f"Manual deposit #{did} approved by admin")
            if not ok:
                await db.rollback()
                return jerr("Crediting failed.")
            await credit_referral_bonus(db, dep["user_id"], amt)
            await db.commit()
    except Exception as e:
        logger.error(f"Mini app approve error: {e}", exc_info=True)
        return jerr("System error.", 500)
    text = await notify_user_deposit_event(WEBAPP_BOT, int(dep["user_id"]), "✅ <b>DEPOSIT APPROVED</b>", did, amt, "\n\nYour balance has been credited successfully.")
    await notify_admins(WEBAPP_BOT, text + f"\n👤 User: <code>{dep['user_id']}</code>")
    await log_admin_event(request["uid"], "APPROVE_DEPOSIT", str(did), f"₹{amt}")
    await send_channel_log(WEBAPP_BOT, "PAYMENT", f"✅ <b>Deposit Approved #{did}</b>\n• User: <code>{dep['user_id']}</code>\n• Amount: ₹{amt}\n• Admin: <code>{request['uid']}</code>")
    return jresp({"ok": True})


async def api_admin_deposit_reject(request: web.Request):
    did = _int(request.match_info["did"])
    async with get_db() as db:
        dep = await (await db.execute("SELECT * FROM deposits WHERE id = ? AND status = 'PENDING'", (did,))).fetchone()
        if not dep:
            return jerr("Deposit is not pending.")
        await db.execute("UPDATE deposits SET status = 'REJECTED', processed_at = CURRENT_TIMESTAMP WHERE id = ? AND status = 'PENDING'", (did,))
        await db.commit()
    text = await notify_user_deposit_event(WEBAPP_BOT, int(dep["user_id"]), "❌ <b>DEPOSIT REJECTED</b>", did, dep["amount"], "\n\nYour deposit was rejected by admin. Please contact support if needed.")
    await notify_admins(WEBAPP_BOT, text + f"\n👤 User: <code>{dep['user_id']}</code>")
    await log_admin_event(request["uid"], "REJECT_DEPOSIT", str(did), "Rejected")
    await send_channel_log(WEBAPP_BOT, "PAYMENT", f"❌ <b>Deposit Rejected #{did}</b>\n• User: <code>{dep['user_id']}</code>\n• Amount: ₹{dep['amount']}\n• Admin: <code>{request['uid']}</code>")
    return jresp({"ok": True})


async def api_admin_services(request: web.Request):
    page = max(1, _int(request.query.get("page"), 1))
    q = (request.query.get("q") or "").strip()
    per = 20
    where, args = "WHERE 1=1", []
    if q.isdigit():
        where += " AND (s.id = ? OR s.name LIKE ?)"
        args += [int(q), f"%{q}%"]
    elif q:
        where += " AND s.name LIKE ?"
        args.append(f"%{q}%")
    async with get_db() as db:
        total = (await (await db.execute(f"SELECT COUNT(*) FROM services s {where}", args)).fetchone())[0]
        rows = await (await db.execute(
            f"SELECT s.*, c.name AS category_name FROM services s LEFT JOIN categories c ON c.id = s.category_id {where} ORDER BY s.id DESC LIMIT ? OFFSET ?",
            args + [per, (page - 1) * per])).fetchall()
    return jresp({"ok": True, "services": [{
        "id": s["id"], "name": s["name"], "category": s["category_name"] or "—", "rate": s["selling_rate"],
        "provider_rate": s["provider_rate"], "active": bool(s["is_active"]), "min": s["min_quantity"], "max": s["max_quantity"],
        "profit_type": s["profit_type"]} for s in rows], "page": page, "pages": max(1, (total + per - 1) // per), "total": total})


async def api_admin_service_toggle(request: web.Request):
    sid = _int(request.match_info["sid"])
    async with get_db() as db:
        s = await (await db.execute("SELECT name, is_active FROM services WHERE id = ?", (sid,))).fetchone()
        if not s:
            return jerr("Service not found.", 404)
        nv = 0 if s["is_active"] else 1
        await db.execute("UPDATE services SET is_active = ? WHERE id = ?", (nv, sid))
        await db.commit()
    await notify_service_change(WEBAPP_BOT, sid, s["name"], "Enabled." if nv else "Disabled.")
    await log_admin_event(request["uid"], "TOGGLE_SERVICE", str(sid), "ON" if nv else "OFF")
    return jresp({"ok": True, "active": bool(nv)})


async def api_admin_service_price(request: web.Request):
    sid = _int(request.match_info["sid"])
    try:
        val = quantize_currency(str((await body_of(request)).get("rate")).strip())
        if val <= 0:
            raise ValueError
    except Exception:
        return jerr("Enter a valid price.")
    async with get_db() as db:
        before = await (await db.execute("SELECT name, selling_rate FROM services WHERE id = ?", (sid,))).fetchone()
        if not before:
            return jerr("Service not found.", 404)
        await db.execute("UPDATE services SET selling_rate=?, profit_type='FIXED', profit_value='0' WHERE id=?", (str(val), sid))
        await db.commit()
    await notify_users_price_change(WEBAPP_BOT, sid, before["name"], before["selling_rate"], str(val))
    await log_admin_event(request["uid"], "EDIT_SERVICE", str(sid), "price")
    return jresp({"ok": True})


async def api_admin_categories(request: web.Request):
    async with get_db() as db:
        rows = await (await db.execute("SELECT c.*, (SELECT COUNT(*) FROM services s WHERE s.category_id=c.id) AS cnt FROM categories c ORDER BY c.display_order, c.id")).fetchall()
    return jresp({"ok": True, "categories": [{"id": c["id"], "name": c["name"], "icon": c["icon"], "active": bool(c["is_active"]), "count": c["cnt"]} for c in rows]})


async def api_admin_category_toggle(request: web.Request):
    cid = _int(request.match_info["cid"])
    async with get_db() as db:
        c = await (await db.execute("SELECT is_active FROM categories WHERE id = ?", (cid,))).fetchone()
        if not c:
            return jerr("Category not found.", 404)
        nv = 0 if c["is_active"] else 1
        await db.execute("UPDATE categories SET is_active = ? WHERE id = ?", (nv, cid))
        await db.commit()
    await log_admin_event(request["uid"], "TOGGLE_CATEGORY", str(cid), "ON" if nv else "OFF")
    return jresp({"ok": True, "active": bool(nv)})


async def api_admin_gifts(request: web.Request):
    async with get_db() as db:
        rows = await (await db.execute("SELECT * FROM gift_cards ORDER BY created_at DESC LIMIT 100")).fetchall()
    return jresp({"ok": True, "gifts": [{"code": g["code"], "amount": g["amount"], "max_uses": g["max_uses"], "used": g["used_count"], "active": bool(g["is_active"])} for g in rows]})


async def api_admin_gift_create(request: web.Request):
    b = await body_of(request)
    code = str(b.get("code") or "").strip().upper()
    try:
        amt = quantize_currency(str(b.get("amount")).strip())
        uses = int(b.get("uses"))
        if amt <= 0 or uses < 1 or not code or len(code) > 50:
            raise ValueError
    except Exception:
        return jerr("Enter a valid code, amount and uses.")
    async with get_db() as db:
        if await (await db.execute("SELECT 1 FROM gift_cards WHERE UPPER(code) = ?", (code,))).fetchone():
            return jerr("Code already exists.")
        await db.execute("INSERT INTO gift_cards (code, amount, max_uses) VALUES (?, ?, ?)", (code, str(amt), uses))
        await db.commit()
    await log_admin_event(request["uid"], "ADD_GIFT", code, f"₹{amt} x{uses}")
    return jresp({"ok": True})


async def api_admin_gift_toggle(request: web.Request):
    code = request.match_info["code"].upper()
    async with get_db() as db:
        g = await (await db.execute("SELECT is_active FROM gift_cards WHERE UPPER(code) = ?", (code,))).fetchone()
        if not g:
            return jerr("Gift card not found.", 404)
        nv = 0 if g["is_active"] else 1
        await db.execute("UPDATE gift_cards SET is_active = ? WHERE UPPER(code) = ?", (nv, code))
        await db.commit()
    return jresp({"ok": True, "active": bool(nv)})


async def api_admin_gift_delete(request: web.Request):
    code = request.match_info["code"].upper()
    async with get_db() as db:
        await db.execute("DELETE FROM gift_cards WHERE UPPER(code) = ?", (code,))
        await db.commit()
    await log_admin_event(request["uid"], "DELETE_GIFT", code, "")
    return jresp({"ok": True})


SETTING_TOGGLES = ["MAINTENANCE_MODE", "MAINT_NEW_ORDER", "MAINT_ADD_BALANCE", "MAINT_SERVICE",
                   "PAY_UPI_AUTO_ENABLED", "PAY_UPI_MANUAL_ENABLED", "PAY_USDT_ENABLED",
                   "REFERRAL_ENABLED", "LOG_CHANNEL_ENABLED"]
SETTING_VALUES = ["MIN_DEPOSIT", "MAX_DEPOSIT", "USDT_MIN_DEPOSIT", "USDT_MAX_DEPOSIT", "REFERRAL_PERCENT",
                  "AUTO_UPI_ID", "AUTO_UPI_NAME", "AUTO_PAY_MID", "AUTO_PAY_API_URL", "UPI_ID", "UPI_NAME", "USDT_BEP20_ADDRESS", "SUPPORT_USERNAME", "SUPPORT_CHANNEL", "LOG_CHANNEL_ID"]


async def api_admin_settings(request: web.Request):
    out = {}
    for k in SETTING_TOGGLES + SETTING_VALUES + ["GLOBAL_MARKUP_PERCENT", "GLOBAL_DISCOUNT_PERCENT", "DISCOUNT_ENABLED"]:
        out[k] = await get_setting(k, DEFAULT_SETTINGS.get(k, ""))
    return jresp({"ok": True, "settings": out, "toggles": SETTING_TOGGLES, "values": SETTING_VALUES})


async def api_admin_setting_set(request: web.Request):
    b = await body_of(request)
    key, value = str(b.get("key") or ""), str(b.get("value") if b.get("value") is not None else "").strip()
    if key in SETTING_TOGGLES:
        value = "1" if value in ("1", "true", "True") else "0"
    elif key in SETTING_VALUES:
        if key in ("MIN_DEPOSIT", "MAX_DEPOSIT", "USDT_MIN_DEPOSIT", "USDT_MAX_DEPOSIT", "REFERRAL_PERCENT"):
            try:
                if Decimal(value) < 0:
                    raise ValueError
            except Exception:
                return jerr("Enter a valid number.")
        if not value and key not in ("LOG_CHANNEL_ID",):
            return jerr("Value cannot be empty.")
    else:
        return jerr("This setting cannot be changed here.")
    await set_setting(key, value)
    await log_admin_event(request["uid"], "SET_SETTING", key, value if "PASSWORD" not in key else "***")
    return jresp({"ok": True})


async def api_admin_discount(request: web.Request):
    b = await body_of(request)
    try:
        disc = Decimal(str(b.get("discount")).strip())
        markup = Decimal(str(b.get("markup")).strip())
        if disc < 0 or disc >= 100 or markup < 0:
            raise ValueError
    except Exception:
        return jerr("Enter valid discount (0-99) and markup percentages.")
    await set_setting("GLOBAL_DISCOUNT_PERCENT", str(disc))
    await set_setting("GLOBAL_MARKUP_PERCENT", str(markup))
    await set_setting("DISCOUNT_ENABLED", "1" if b.get("enabled") else "0")
    net = markup - disc
    await notify_users_discount_change(WEBAPP_BOT, disc, net)
    await log_admin_event(request["uid"], "SET_DISCOUNT", f"{disc}%", f"markup {markup}%")
    return jresp({"ok": True, "net": str(net)})


async def api_admin_recalc(request: web.Request):
    updated = skipped = 0
    async with get_db() as db:
        services = await (await db.execute("SELECT * FROM services")).fetchall()
        for s in services:
            try:
                if str(s["profit_type"]).upper() == "FIXED":
                    skipped += 1
                    continue
                new_sell = await calculate_service_selling_rate(s, Decimal(str(s["provider_rate"])))
                if new_sell != Decimal(str(s["selling_rate"])):
                    await db.execute("UPDATE services SET selling_rate=? WHERE id=?", (str(new_sell), s["id"]))
                    updated += 1
                else:
                    skipped += 1
            except Exception:
                skipped += 1
        await db.commit()
    await log_admin_event(request["uid"], "RECALC_PRICES", "ALL", f"Updated={updated}, Skipped={skipped}")
    return jresp({"ok": True, "updated": updated, "skipped": skipped})


async def api_admin_providers(request: web.Request):
    async with get_db() as db:
        rows = await (await db.execute("SELECT id, name, api_url, api_key, status FROM providers ORDER BY id")).fetchall()
    return jresp({"ok": True, "providers": [{"id": p["id"], "name": p["name"], "url": p["api_url"], "key": mask_api_key(p["api_key"]), "active": bool(p["status"])} for p in rows]})


async def api_admin_provider_balance(request: web.Request):
    res = await ProviderAPIClient.get_balance(_int(request.match_info["pid"]))
    if isinstance(res, dict) and "balance" in res:
        return jresp({"ok": True, "balance": str(res.get("balance")), "currency": str(res.get("currency", ""))})
    return jerr(str((res or {}).get("error", "Could not fetch balance.")) if isinstance(res, dict) else "Could not fetch balance.")


async def api_admin_provider_toggle(request: web.Request):
    pid = _int(request.match_info["pid"])
    async with get_db() as db:
        p = await (await db.execute("SELECT status FROM providers WHERE id = ?", (pid,))).fetchone()
        if not p:
            return jerr("Provider not found.", 404)
        nv = 0 if p["status"] else 1
        await db.execute("UPDATE providers SET status = ? WHERE id = ?", (nv, pid))
        await db.commit()
    await log_admin_event(request["uid"], "TOGGLE_PROVIDER", str(pid), "ON" if nv else "OFF")
    return jresp({"ok": True, "active": bool(nv)})


async def api_admin_broadcast(request: web.Request):
    text = str((await body_of(request)).get("text") or "").strip()
    if not text:
        return jerr("Message is empty.")
    async with get_db() as db:
        rows = await (await db.execute("SELECT user_id FROM users WHERE is_banned = 0")).fetchall()
    ids = [r["user_id"] for r in rows]
    admin_id = request["uid"]

    async def run():
        sent = failed = 0
        for i in ids:
            try:
                await WEBAPP_BOT.send_message(i, text, parse_mode=ParseMode.HTML)
                sent += 1
            except Exception:
                failed += 1
            await asyncio.sleep(0.05)
        await notify_admins(WEBAPP_BOT, f"✅ <b>Broadcast Complete</b> (Mini App)\n\n👥 Total: {len(ids)}\n✅ Sent: {sent}\n❌ Failed: {failed}")

    asyncio.create_task(run())
    await log_admin_event(admin_id, "BROADCAST", "ALL", f"{len(ids)} users")
    return jresp({"ok": True, "total": len(ids)})


# ------------------------- admin: providers / services / categories --------
PROVIDER_CATALOG_CACHE: Dict[int, tuple] = {}


async def _provider_catalog(pid: int, force: bool = False):
    cached = PROVIDER_CATALOG_CACHE.get(pid)
    if cached and not force and time.time() - cached[0] < 180:
        return cached[1], None
    try:
        res = await ProviderAPIClient.fetch_services_list(pid)
    except Exception as e:
        logger.error(f"Mini app provider catalog fetch failed: {e}", exc_info=True)
        return None, "Provider fetch failed. Check the provider URL / API key."
    if not res.get("success"):
        return None, str(res.get("error", "Provider fetch failed."))
    PROVIDER_CATALOG_CACHE[pid] = (time.time(), res.get("services") or [])
    return PROVIDER_CATALOG_CACHE[pid][1], None


async def api_admin_provider_add(request: web.Request):
    b = await body_of(request)
    name = str(b.get("name") or "").strip()
    url = str(b.get("url") or "").strip()
    key = str(b.get("key") or "").strip()
    if not name or len(name) > 80:
        return jerr("Provider name is required (1-80 characters).")
    if not url.startswith("https://") or len(url) > 500:
        return jerr("Invalid API URL. It must start with https://")
    if not key or len(key) > 300:
        return jerr("API key cannot be empty.")
    async with get_db() as db:
        cur = await db.execute("INSERT INTO providers (name, api_url, api_key, status) VALUES (?, ?, ?, 1)", (name, url, key))
        pid = cur.lastrowid
        await db.commit()
    await log_admin_event(request["uid"], "ADD_PROVIDER", str(pid), name)
    check: Dict[str, Any] = {"ok": False}
    try:
        res = await ProviderAPIClient.get_balance(pid)
        if isinstance(res, dict) and "balance" in res:
            check = {"ok": True, "balance": str(res.get("balance")), "currency": str(res.get("currency", ""))}
        else:
            check = {"ok": False, "error": str((res or {}).get("error", "Could not verify API")) if isinstance(res, dict) else "Could not verify API"}
    except Exception as e:
        check = {"ok": False, "error": str(e)}
    if WEBAPP_BOT is not None:
        await notify_admins(WEBAPP_BOT, f"🔌 <b>PROVIDER ADDED</b> (Mini App)\n\n🆔 ID: <code>{pid}</code>\n🔌 {html.escape(name)}\n👤 Admin: <code>{request['uid']}</code>")
    return jresp({"ok": True, "id": pid, "check": check})


async def api_admin_provider_delete(request: web.Request):
    pid = _int(request.match_info["pid"])
    async with get_db() as db:
        p = await (await db.execute("SELECT name FROM providers WHERE id = ?", (pid,))).fetchone()
        if not p:
            return jerr("Provider not found.", 404)
        await db.execute("DELETE FROM services WHERE provider_id = ?", (pid,))
        await db.execute("DELETE FROM providers WHERE id = ?", (pid,))
        await db.commit()
    PROVIDER_CATALOG_CACHE.pop(pid, None)
    await log_admin_event(request["uid"], "DELETE_PROVIDER", str(pid), p["name"])
    return jresp({"ok": True})


async def api_admin_provider_services(request: web.Request):
    pid = _int(request.match_info["pid"])
    page = max(1, _int(request.query.get("page"), 1))
    q = (request.query.get("q") or "").strip().lower()
    force = request.query.get("refresh") == "1"
    per = 25
    items, err = await _provider_catalog(pid, force)
    if items is None:
        return jerr(err or "Provider fetch failed.", 502)
    async with get_db() as db:
        rows = await (await db.execute("SELECT id, provider_service_id FROM services WHERE provider_id = ?", (pid,))).fetchall()
    existing = {str(r["provider_service_id"]): r["id"] for r in rows}
    out = []
    for x in items:
        if not isinstance(x, dict):
            continue
        sid = _p_sid(x)
        if sid is None:
            continue
        name = str(x.get("name") or x.get("service_name") or x.get("title") or f"Service {sid}")
        cat = str(x.get("category") or "")
        if q and q not in name.lower() and q not in str(sid) and q not in cat.lower():
            continue
        out.append({"sid": str(sid), "name": name, "category": cat, "rate": str(x.get("rate", x.get("price", "0"))),
                    "min": _int(x.get("min", x.get("minimum", 0))), "max": _int(x.get("max", x.get("maximum", 0))),
                    "exists": existing.get(str(sid))})
    total = len(out)
    start = (page - 1) * per
    return jresp({"ok": True, "items": out[start:start + per], "page": page,
                  "pages": max(1, (total + per - 1) // per), "total": total})


async def api_admin_services_add(request: web.Request):
    b = await body_of(request)
    pid, cid = _int(b.get("provider_id")), _int(b.get("category_id"))
    ids = [str(i).strip() for i in (b.get("provider_service_ids") or []) if str(i).strip()]
    mode = str(b.get("mode") or "percent").lower()
    if not ids or len(ids) > 200:
        return jerr("Select between 1 and 200 services.")
    try:
        value = Decimal(str(b.get("value")).strip())
        if value < 0 or (mode == "fixed" and value <= 0):
            raise ValueError
    except Exception:
        return jerr("Enter a valid " + ("selling rate." if mode == "fixed" else "markup percent."))
    if mode not in ("percent", "fixed"):
        return jerr("Invalid pricing mode.")
    async with get_db() as db:
        prov = await (await db.execute("SELECT id FROM providers WHERE id = ?", (pid,))).fetchone()
        cat = await (await db.execute("SELECT id FROM categories WHERE id = ?", (cid,))).fetchone()
    if not prov:
        return jerr("Provider not found.", 404)
    if not cat:
        return jerr("Select a valid category.")
    items, err = await _provider_catalog(pid)
    if items is None:
        return jerr(err or "Provider fetch failed.", 502)
    by_id = {str(_p_sid(x)): x for x in items if isinstance(x, dict) and _p_sid(x) is not None}
    single = len(ids) == 1
    name_over = str(b.get("name") or "").strip()[:200] if single else ""
    desc_over = b.get("description")
    added, skipped = [], []
    async with get_db() as db:
        for sid in ids:
            item = by_id.get(sid)
            if not item:
                skipped.append({"sid": sid, "reason": "Not found in provider list"})
                continue
            try:
                prate = quantize_currency(item.get("rate", item.get("price")))
                min_q = int(item.get("min", item.get("minimum", 0)) or 0)
                max_q = int(item.get("max", item.get("maximum", 0)) or 0)
                if prate <= 0 or min_q < 1 or max_q < min_q:
                    raise ValueError
            except Exception:
                skipped.append({"sid": sid, "reason": "Invalid rate/min/max from provider"})
                continue
            if await (await db.execute("SELECT id FROM services WHERE provider_id = ? AND provider_service_id = ? LIMIT 1", (pid, int(sid)))).fetchone():
                skipped.append({"sid": sid, "reason": "Already added"})
                continue
            if mode == "fixed":
                sell, ptype, pval = quantize_currency(value), "FIXED", "0"
            else:
                sell = quantize_currency(prate * (Decimal("1") + value / Decimal("100")))
                ptype, pval = "PERCENT", str(value)
            name = name_over or str(item.get("name") or item.get("service_name") or item.get("title") or f"Service {sid}").strip()
            desc = str(desc_over).strip() if (single and desc_over is not None and str(desc_over).strip()) else _p_desc(item)
            refill_available, refill_days, refill_duration = _p_refill(item)
            cancel_available = _p_cancel(item)
            cur = await db.execute(
                """INSERT INTO services(category_id,provider_id,provider_service_id,name,min_quantity,max_quantity,provider_rate,profit_type,profit_value,selling_rate,description,refill_available,refill_days,refill_duration,cancel_available,is_active)
                   VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,1)""",
                (cid, pid, int(sid), name, min_q, max_q, str(prate), ptype, pval, str(sell), desc,
                 refill_available, refill_days, refill_duration, cancel_available))
            added.append({"id": cur.lastrowid, "name": name})
        await db.commit()
    if added:
        await log_admin_event(request["uid"], "ADD_SERVICE", ",".join(str(a["id"]) for a in added[:20]), f"{len(added)} added (Mini App)")
        if WEBAPP_BOT is not None:
            await send_channel_log(WEBAPP_BOT, "SERVICE", f"➕ <b>{len(added)} Service(s) Added</b> (Mini App)\n• Admin: <code>{request['uid']}</code>")
            await notify_admins(WEBAPP_BOT, "➕ <b>NEW SERVICE(S) ADDED</b> (Mini App)\n\n" + "\n".join(f"🆔 <b>#{a['id']}</b> {html.escape(a['name'])}" for a in added[:15]) + f"\n\n👤 Admin: <code>{request['uid']}</code>")
    if not added:
        return jerr("; ".join(f"{s['sid']}: {s['reason']}" for s in skipped[:3]) or "Nothing was added.", 400, skipped=skipped)
    return jresp({"ok": True, "added": added, "skipped": skipped})


async def api_admin_service_delete(request: web.Request):
    sid = _int(request.match_info["sid"])
    async with get_db() as db:
        s = await (await db.execute("SELECT id, name FROM services WHERE id = ?", (sid,))).fetchone()
        if not s:
            return jerr("Service not found.", 404)
        await db.execute("DELETE FROM services WHERE id = ?", (sid,))
        await db.commit()
    await log_admin_event(request["uid"], "DELETE_SERVICE", str(sid), s["name"])
    if WEBAPP_BOT is not None:
        await send_channel_log(WEBAPP_BOT, "SERVICE", f"🗑️ <b>Service Deleted #{sid}</b> (Mini App)\n• {html.escape(s['name'])}\n• Admin: <code>{request['uid']}</code>")
    return jresp({"ok": True})


async def api_admin_category_add(request: web.Request):
    b = await body_of(request)
    name = str(b.get("name") or "").strip()
    icon = str(b.get("icon") or "").strip()[:8] or "📁"
    if not name or len(name) > 80:
        return jerr("Category name is required (1-80 characters).")
    async with get_db() as db:
        if await (await db.execute("SELECT 1 FROM categories WHERE LOWER(name) = LOWER(?)", (name,))).fetchone():
            return jerr("A category with this name already exists.")
        mx = (await (await db.execute("SELECT COALESCE(MAX(display_order),0) FROM categories")).fetchone())[0] or 0
        cur = await db.execute("INSERT INTO categories(name, icon, display_order, is_active) VALUES(?, ?, ?, 1)", (name, icon, int(mx) + 1))
        cid = cur.lastrowid
        await db.commit()
    await log_admin_event(request["uid"], "ADD_CATEGORY", str(cid), name)
    return jresp({"ok": True, "id": cid})


async def api_admin_category_edit(request: web.Request):
    cid = _int(request.match_info["cid"])
    b = await body_of(request)
    name = str(b.get("name") or "").strip()
    icon = str(b.get("icon") or "").strip()[:8] or "📁"
    if not name or len(name) > 80:
        return jerr("Category name is required (1-80 characters).")
    async with get_db() as db:
        if not await (await db.execute("SELECT 1 FROM categories WHERE id = ?", (cid,))).fetchone():
            return jerr("Category not found.", 404)
        await db.execute("UPDATE categories SET name = ?, icon = ? WHERE id = ?", (name, icon, cid))
        await db.commit()
    await log_admin_event(request["uid"], "EDIT_CATEGORY", str(cid), name)
    return jresp({"ok": True})


async def api_admin_category_delete(request: web.Request):
    cid = _int(request.match_info["cid"])
    with_services = bool((await body_of(request)).get("with_services"))
    async with get_db() as db:
        c = await (await db.execute("SELECT id, name FROM categories WHERE id = ?", (cid,))).fetchone()
        if not c:
            return jerr("Category not found.", 404)
        cnt = (await (await db.execute("SELECT COUNT(*) FROM services WHERE category_id = ?", (cid,))).fetchone())[0]
        if cnt and not with_services:
            return jerr(f"This category has {cnt} service(s).", 409, count=cnt)
        if cnt:
            await db.execute("DELETE FROM services WHERE category_id = ?", (cid,))
        await db.execute("DELETE FROM categories WHERE id = ?", (cid,))
        await db.commit()
    await log_admin_event(request["uid"], "DELETE_CATEGORY", str(cid), c["name"])
    return jresp({"ok": True})


# ------------------------------- web server -------------------------------
import gzip as _gzip

_INDEX_RAW = MINIAPP_HTML.encode("utf-8")
_INDEX_GZ = _gzip.compress(_INDEX_RAW, 9)
_INDEX_ETAG = '"' + hashlib.sha1(_INDEX_RAW).hexdigest()[:16] + '"'


async def serve_index(request: web.Request) -> web.Response:
    headers = {"Cache-Control": "no-cache", "ETag": _INDEX_ETAG, "X-Content-Type-Options": "nosniff",
               "Vary": "Accept-Encoding"}
    if request.headers.get("If-None-Match") == _INDEX_ETAG:
        return web.Response(status=304, headers=headers)
    if "gzip" in request.headers.get("Accept-Encoding", ""):
        headers["Content-Encoding"] = "gzip"
        return web.Response(body=_INDEX_GZ, content_type="text/html", charset="utf-8", headers=headers)
    return web.Response(body=_INDEX_RAW, content_type="text/html", charset="utf-8", headers=headers)


async def serve_health(request: web.Request) -> web.Response:
    return web.json_response({"ok": True})


async def serve_options(request: web.Request) -> web.Response:
    return web.Response(status=204)


def build_webapp() -> web.Application:
    app = web.Application(middlewares=[webapp_cors_middleware, webapp_auth_middleware], client_max_size=1024 * 1024)
    r = app.router
    r.add_get("/", serve_index)
    r.add_get("/healthz", serve_health)
    r.add_route("OPTIONS", "/{tail:.*}", serve_options)
    # user
    r.add_get("/api/bootstrap", api_bootstrap)
    r.add_get("/api/services", api_services)
    r.add_get("/api/service/{sid}", api_service)
    r.add_post("/api/order", api_place_order)
    r.add_get("/api/orders", api_orders)
    r.add_get("/api/order/{oid}", api_order)
    r.add_post("/api/order/{oid}/sync", api_order_sync)
    r.add_post("/api/order/{oid}/refill", api_order_refill)
    r.add_get("/api/profile", api_profile)
    r.add_post("/api/currency", api_set_currency)
    r.add_get("/api/transactions", api_transactions)
    r.add_post("/api/gift", api_redeem_gift)
    r.add_get("/api/deposits", api_deposits)
    r.add_post("/api/deposit/upi_auto", api_dep_upi_auto)
    r.add_post("/api/deposit/upi_auto/check", api_dep_upi_check)
    r.add_post("/api/deposit/cancel", api_dep_cancel)
    r.add_post("/api/deposit/upi_manual", api_dep_manual)
    r.add_post("/api/deposit/upi_manual/utr", api_dep_manual_utr)
    r.add_get("/api/deposit/usdt/info", api_dep_usdt_info)
    r.add_post("/api/deposit/usdt", api_dep_usdt)
    r.add_post("/api/deposit/usdt/txid", api_dep_usdt_txid)
    # admin
    r.add_get("/api/admin/dashboard", api_admin_dashboard)
    r.add_get("/api/admin/orders", api_admin_orders)
    r.add_get("/api/admin/order/{oid}", api_admin_order)
    r.add_post("/api/admin/order/{oid}/status", api_admin_order_status)
    r.add_post("/api/admin/order/{oid}/cancel", api_admin_order_cancel)
    r.add_post("/api/admin/order/{oid}/sync", api_admin_order_sync)
    r.add_post("/api/admin/order/{oid}/refill", api_admin_order_refill)
    r.add_post("/api/admin/order/{oid}/refill_cancel", api_admin_order_refill_cancel)
    r.add_get("/api/admin/users", api_admin_users)
    r.add_get("/api/admin/user/{uid}", api_admin_user)
    r.add_post("/api/admin/user/{uid}/ban", api_admin_user_ban)
    r.add_post("/api/admin/user/{uid}/balance", api_admin_user_balance)
    r.add_post("/api/admin/user/{uid}/message", api_admin_user_message)
    r.add_get("/api/admin/deposits", api_admin_deposits)
    r.add_post("/api/admin/deposit/{did}/approve", api_admin_deposit_approve)
    r.add_post("/api/admin/deposit/{did}/reject", api_admin_deposit_reject)
    r.add_get("/api/admin/services", api_admin_services)
    r.add_post("/api/admin/service/{sid}/toggle", api_admin_service_toggle)
    r.add_post("/api/admin/service/{sid}/price", api_admin_service_price)
    r.add_get("/api/admin/categories", api_admin_categories)
    r.add_post("/api/admin/category/{cid}/toggle", api_admin_category_toggle)
    r.add_get("/api/admin/gifts", api_admin_gifts)
    r.add_post("/api/admin/gift", api_admin_gift_create)
    r.add_post("/api/admin/gift/{code}/toggle", api_admin_gift_toggle)
    r.add_post("/api/admin/gift/{code}/delete", api_admin_gift_delete)
    r.add_get("/api/admin/settings", api_admin_settings)
    r.add_post("/api/admin/setting", api_admin_setting_set)
    r.add_post("/api/admin/discount", api_admin_discount)
    r.add_post("/api/admin/recalc", api_admin_recalc)
    r.add_get("/api/admin/providers", api_admin_providers)
    r.add_get("/api/admin/provider/{pid}/balance", api_admin_provider_balance)
    r.add_post("/api/admin/provider/{pid}/toggle", api_admin_provider_toggle)
    r.add_post("/api/admin/broadcast", api_admin_broadcast)
    r.add_post("/api/admin/provider", api_admin_provider_add)
    r.add_post("/api/admin/provider/{pid}/delete", api_admin_provider_delete)
    r.add_get("/api/admin/provider/{pid}/services", api_admin_provider_services)
    r.add_post("/api/admin/services/add", api_admin_services_add)
    r.add_post("/api/admin/service/{sid}/delete", api_admin_service_delete)
    r.add_post("/api/admin/category", api_admin_category_add)
    r.add_post("/api/admin/category/{cid}/edit", api_admin_category_edit)
    r.add_post("/api/admin/category/{cid}/delete", api_admin_category_delete)
    return app


async def start_webapp_server() -> web.AppRunner:
    runner = web.AppRunner(build_webapp(), access_log=None)
    await runner.setup()
    await web.TCPSite(runner, WEBAPP_HOST, WEBAPP_PORT, backlog=1024).start()
    logger.info("Mini App web server listening on %s:%s", WEBAPP_HOST, WEBAPP_PORT)
    return runner


async def setup_webapp_menu_button(bot: Bot) -> None:
    """Point the bot's menu button at the Mini App (needs a public HTTPS URL)."""
    global WEBAPP_PUBLIC_URL
    url = (await get_setting("WEBAPP_URL", WEBAPP_URL) or WEBAPP_URL or "").strip()
    WEBAPP_PUBLIC_URL = url
    if not url.lower().startswith("https://"):
        logger.warning("WEBAPP_URL is not set to a public https:// URL — the Mini App server is running locally, "
                       "but Telegram buttons are disabled until WEBAPP_URL is configured.")
        return
    try:
        from telethon.tl import functions as tl_functions
        await bot.client(tl_functions.bots.SetBotMenuButtonRequest(
            user_id=tl_types.InputUserEmpty(),
            button=tl_types.BotMenuButton(text="🚀 Open App", url=url)))
        logger.info("Mini App menu button set: %s", url)
    except Exception as e:
        logger.warning("Could not set Mini App menu button: %s", e)


# ---------------------------------------------------------------------------
# 🚀 MAIN ENTRYPOINT
# ---------------------------------------------------------------------------
async def main():
    await init_db()
    if not BOT_TOKEN:
        raise RuntimeError("BOT_TOKEN is missing. Put your bot token in BOT_TOKEN at the top/config section.")
    if not API_ID or not API_HASH:
        raise RuntimeError("API_ID and API_HASH are missing. Get them from https://my.telegram.org and set them in the config section.")

    client = TelegramClient(SESSION_NAME, API_ID, API_HASH)
    await client.start(bot_token=BOT_TOKEN)
    bot = Bot(client)
    dp = Dispatcher(storage=MemoryStorage())

    dp.message.middleware(BanCheckMiddleware())
    dp.callback_query.middleware(BanCheckMiddleware())
    dp.message.middleware(HighSecurityMiddleware())
    dp.callback_query.middleware(HighSecurityMiddleware())

    dp.include_router(router)
    dp.attach(bot)
    global WEBAPP_BOT
    WEBAPP_BOT = bot

    # Detect the actual Telegram bot name automatically.
    global BOT_NAME, WELCOME_TEXT, TERMS_TEXT
    try:
        me = await bot.get_me()
        _BOT_INFO["username"] = getattr(me, "username", None)
        BOT_NAME = me.first_name or (f"@{me.username}" if me.username else BOT_NAME)
        WELCOME_TEXT = build_welcome_text()
        TERMS_TEXT = build_terms_text()
        logger.info("Detected bot name: %s", BOT_NAME)
    except Exception as e:
        logger.warning("Could not detect bot name: %s", e)

    webapp_runner = None
    try:
        webapp_runner = await start_webapp_server()
        await setup_webapp_menu_button(bot)
    except Exception as e:
        logger.error("Mini App server failed to start: %s", e, exc_info=True)

    sync_task = asyncio.create_task(background_sync_daemon(bot))
    monitor_task = asyncio.create_task(background_provider_service_monitor(bot))
    expiry_task = asyncio.create_task(deposit_expiry_worker(bot))
    live_rate_task = asyncio.create_task(live_usdt_rate_worker(bot))

    logger.info("SMM Bot + Mini App running (Telethon)...")

    try:
        await client.run_until_disconnected()
    finally:
        sync_task.cancel()
        monitor_task.cancel()
        expiry_task.cancel()
        live_rate_task.cancel()
        if webapp_runner is not None:
            try:
                await webapp_runner.cleanup()
            except Exception:
                pass
        try:
            await client.disconnect()
        except Exception:
            pass


if __name__ == "__main__":
    try:
        asyncio.run(main())
    except (KeyboardInterrupt, SystemExit):
        logger.info("Bot stopped.")
