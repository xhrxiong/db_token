"""豆包流式对话独立服务

启动: python test.py
调用: POST http://localhost:8000/chat  {"prompt": "你好"}
响应: SSE 流式返回豆包回复
"""
from contextlib import asynccontextmanager
from fastapi import FastAPI, Body
from fastapi.responses import StreamingResponse
import uvicorn
import sys
import asyncio
import logging
import os
from playwright.async_api import async_playwright

USER_DATA_DIR = r"C:\Users\MY-PC\Desktop\备忘\rpa\1234"
TARGET_URL = "chat/completion"

def get_browser_install_path(browser_name="chrome"):
    import sys
    if sys.platform == "win32":
        import winreg

        registry_paths = {
            "chrome": r"SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\chrome.exe",
        }

        try:
            key = winreg.OpenKey(winreg.HKEY_LOCAL_MACHINE, registry_paths[browser_name])
            path, _ = winreg.QueryValueEx(key, "")
            winreg.CloseKey(key)
            return path
        except FileNotFoundError:
            try:
                key = winreg.OpenKey(winreg.HKEY_CURRENT_USER, registry_paths[browser_name])
                path, _ = winreg.QueryValueEx(key, "")
                winreg.CloseKey(key)
                return path
            except FileNotFoundError:
                return None
    else:
        paths = [
            "/usr/bin/google-chrome-stable",
            "/usr/bin/google-chrome",
        ]
        for path in paths:
            if os.path.isfile(path):
                return path

HOOK_JS = r"""
(() => {
    if (window.__hooked) return;
    window.__hooked = true;
    const orig = window.fetch;
    const dec = new TextDecoder();
    window.fetch = async function(...args) {
        const resp = await orig.apply(this, args);
        const url = typeof args[0] === 'string' ? args[0] : (args[0]?.url || '');
        if (!url.includes('TARGET_URL') || !resp.body) return resp;
        const clone = resp.clone();
        (async () => {
            const reader = clone.body.getReader();
            let buf = '';
            while (true) {
                const {done, value} = await reader.read();
                if (done) {
                    if (buf.trim()) try { __pyChunk(buf.trim()); } catch(e) {}
                    try { __pyDone(); } catch(e) {}
                    break;
                }
                buf += dec.decode(value, {stream: true});
                const lines = buf.split('\n');
                buf = lines.pop() || '';
                for (const line of lines) {
                    const t = line.trim();
                    if (t) try { __pyChunk(t); } catch(e) {}
                }
            }
        })();
        return resp;
    };
})();
""".replace("TARGET_URL", "chat/completion")

class LocalBrowserManager:
    _instances = {}
    def __new__(cls, user_data_dir, is_open=True, max_retries=3):
        # 如果这个 user_data_dir 已经有实例，直接返回
        if user_data_dir in cls._instances:
            return cls._instances[user_data_dir]
        # 否则创建新实例
        instance = super().__new__(cls)
        cls._instances[user_data_dir] = instance
        return instance

    def __init__(self,user_data_dir,is_open=True, max_retries=3):
        # 避免重复初始化
        if hasattr(self, '_initialized'):
            return
        self.user_data_dir = user_data_dir
        self.real_user_agent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.7727.56 Safari/537.36"
        self.context = None
        self.playwright = None
        self.page = None
        self.is_open = is_open
        self.max_retries = max_retries
        self.args = [
            '--no-first-run',
            '--no-default-browser-check',
            '--disable-infobars',
            '--disable-dev-shm-usage',
            '--disable-background-mode',
            '--disable-blink-features=AutomationControlled',
            '--test-type',
            '--no-sandbox',
        ]
        # if sys.platform != "win32":
        #     self.args.extend([
        #         '--window-size=1920,1080',  # ← 加这个
        #         '--window-position=0,0',  # ← 窗口从左上角开始
        #     ])
    def get_screen_size(self):
        if sys.platform!="win32":
            import tkinter as tk
            root = tk.Tk()
            root.withdraw()  # 隐藏窗口
            width = root.winfo_screenwidth()
            height = root.winfo_screenheight()
            root.destroy()
            return width, height
        elif sys.platform=="darwin":
            return 1365, 900
        else:
            return 1280,720

    def _cleanup_singleton_lock(self):
        """清理 Chrome 残留的 SingletonLock 文件，避免进程冲突"""
        lock_file = os.path.join(self.user_data_dir, "SingletonLock")
        try:
            if os.path.exists(lock_file):
                os.remove(lock_file)
                logging.info(f"已清理残留锁文件: {lock_file}")
        except Exception as e:
            logging.warning(f"清理锁文件失败: {lock_file}, {e}")

    async def _launch_browser(self):
        """启动浏览器"""
        self._cleanup_singleton_lock()
        headless = False if self.is_open else True
        self._launched_headless = headless
        screen_width, screen_height = self.get_screen_size()
        self.playwright = await async_playwright().start()

        if headless:
            self.context = await self.playwright.chromium.launch_persistent_context(
                user_data_dir=self.user_data_dir,
                executable_path=get_browser_install_path(browser_name="chrome"),
                headless=headless,
                args=self.args,
                ignore_default_args=[
                    '--enable-automation',
                ],
                no_viewport=True,
                viewport={
                    'width': screen_width,
                    'height': screen_height
                },
                user_agent=self.real_user_agent,
                locale='zh-CN',
                timeout=10000,
            )
        else:
            self.context = await self.playwright.chromium.launch_persistent_context(
                user_data_dir=self.user_data_dir,
                executable_path=get_browser_install_path(browser_name="chrome"),
                headless=headless,
                args=self.args,
                ignore_default_args=[
                    '--enable-automation',
                ],
                no_viewport=True,
                locale='zh-CN',
                viewport=None,
                timeout=10000,
            )
    async def get_context(self):
        """获取浏览器上下文，失败自动重试"""
        desired_headless = False if self.is_open else True
        if self.context and self._launched_headless is not None and self._launched_headless!=desired_headless:
            await self.close()

        if self.context:
            return self.context

        for attempt in range(self.max_retries):
            try:
                await self._launch_browser()
                return self.context
            except Exception as e:
                if attempt < self.max_retries - 1:
                    await asyncio.sleep(1)
                    await self.close()
                else:
                    raise ConnectionError(f"浏览器启动失败，已重试{self.max_retries}次")


    async def get_new_page(self):
        try:
            context = await self.get_context()
            pages = context.pages

            if pages:
                page = pages[0]
            else:
                page = await context.new_page()
            return page

        except Exception as e:
            await self.close()
            context = await self.get_context()
            pages = context.pages
            if pages:
                page = pages[0]
            else:
                page = await context.new_page()
            return page


    async def close(self):
        """关闭资源"""
        try:
            if self.context:
                await self.context.close()
            if self.playwright:
                await self.playwright.stop()
        finally:
            self.context = None
            self.playwright = None
            self._launched_headless = None
            self._cleanup_singleton_lock()


class DoubaoChat:
    """豆包流式对话浏览器管理"""

    def __init__(self, user_data_dir: str):
        self.user_data_dir = user_data_dir
        self._browser: LocalBrowserManager = None
        self._page = None
        self._ready = False
        self._active_queue: asyncio.Queue = None

    async def _on_chunk(self, data: str):
        if self._active_queue:
            await self._active_queue.put(data)

    async def _on_done(self):
        if self._active_queue:
            await self._active_queue.put(None)

    async def start(self):
        """启动浏览器 + 注入 hook + 等待登录"""
        if self._ready:
            return
        self._browser = LocalBrowserManager(self.user_data_dir, is_open=True)
        self._page = await self._browser.get_new_page()
        await self._page.expose_function("__pyChunk", self._on_chunk)
        await self._page.expose_function("__pyDone", self._on_done)
        url = "https://www.doubao.com/chat"
        if url not in (self._page.url or ""):
            await self._page.goto(url, wait_until="domcontentloaded", timeout=15000)
        await self._page.wait_for_selector("#flow_chat_sidebar .object-cover", timeout=300000)
        await self._page.evaluate(HOOK_JS)
        self._ready = True
        print("[OK] 浏览器已就绪")

    async def close(self):
        if self._browser:
            await self._browser.close()

    async def send(self, prompt: str):
        """发送对话，async generator 逐 chunk 返回 SSE 数据"""
        if not self._ready:
            yield "data: {\"error\": \"浏览器未就绪\"}\n\n"
            return
        queue = asyncio.Queue()
        self._active_queue = queue
        input_box = self._page.locator(".semi-input-textarea").first
        await input_box.fill(prompt)
        await self._page.keyboard.press("Enter")
        while True:
            data = await queue.get()
            if data is None:
                yield "data: [DONE]\n\n"
                break
            yield f"data: {data}\n\n"

# ── FastAPI ──
doubao = DoubaoChat(USER_DATA_DIR)

@asynccontextmanager
async def lifespan(app: FastAPI):
    await doubao.start()
    yield
    await doubao.close()

app = FastAPI(title="豆包流式对话服务", lifespan=lifespan)

@app.post("/chat")
async def chat(
    prompt: str = Body(..., description="提示词")
):
    return StreamingResponse(doubao.send(prompt), media_type="text/event-stream")

@app.post("/login_doubao")
async def login(

):
    browser_manager = LocalBrowserManager(USER_DATA_DIR, is_open=True)
    data = {}
    try:
        page = await browser_manager.get_new_page()
        login_url = "https://www.doubao.com/chat"
        if login_url != page.url:
            await page.goto(
                login_url,
                wait_until="domcontentloaded",
                timeout=15000
            )
        await page.wait_for_selector("#flow_chat_sidebar .object-cover", timeout=60000 * 5)
        await asyncio.sleep(3)
        avatar = await page.locator("#flow_chat_sidebar .object-cover").get_attribute('src')
        nickname = await page.locator('.text-ellipsis').inner_text()
        data["status"] = True
        data["avatar"] = avatar
        data["nickname"] = nickname
    except Exception as e:
        data["status"] = False
        data["reason"] = "登录失败"
    finally:
        await browser_manager.close()
    return data

if __name__ == "__main__":
    HOST = "0.0.0.0"
    PORT = 8000
    uvicorn.run(app, host=HOST, port=PORT)
