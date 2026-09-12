                      
"""ROCKET SHELL 2.2.0 — single-file edition.

Unified Windows / WSL command line.
- Real subprocess to cmd.exe / powershell.exe / wsl.exe
- Auto/Windows/Linux modes, multi-distro WSL
- Linux side supports direct / bash -lc / sh -c invocation
- Two-way path bridge: C:\\... <-> /mnt/c/...
- Relative path resolution in both environments
- Chains: ; && ||  and pipes | |&  (routed to active shell)
- Env expansion: $VAR, ${VAR}, %VAR%, session env via export
- Session env is forwarded to subprocesses (os.environ visible)
- Streaming output, timeouts, Ctrl+C handling
- Persistent history, aliases, bookmarks, config in ~/.rocket/
- Persistent readline history for Ctrl+R / arrow-up
- Tab completion, ANSI colors (auto/always/never)
- GUI with colored output (tagged Text widget)
- CLI (default) and Tkinter GUI (--gui)
- Embedded selftest (--selftest)

No third-party dependencies.
"""
from __future__ import annotations

import argparse
import atexit
import json
import os
import platform
import posixpath
import queue
import re
import shutil
import subprocess
import sys
import threading
import time
from dataclasses import dataclass
from pathlib import Path
from typing import Callable, Dict, List, Optional, Tuple

VERSION = "2.2.0"

ROCKET_DIR = Path.home() / ".rocket"
CONFIG_FILE = ROCKET_DIR / "config.json"
HISTORY_FILE = ROCKET_DIR / "history.json"
ALIASES_FILE = ROCKET_DIR / "aliases.json"
BOOKMARKS_FILE = ROCKET_DIR / "bookmarks.json"
CLI_HISTORY_FILE = ROCKET_DIR / "cli_history"


                                                                            
        
                                                                            

class C:
    RESET = "\033[0m"
    BOLD = "\033[1m"
    DIM = "\033[2m"
    RED = "\033[31m"
    GREEN = "\033[32m"
    YELLOW = "\033[33m"
    BLUE = "\033[34m"
    MAGENTA = "\033[35m"
    CYAN = "\033[36m"
    GREY = "\033[90m"


def _enable_windows_ansi() -> bool:
    if platform.system().lower() != "windows":
        return True
    try:
        import ctypes
        kernel32 = ctypes.windll.kernel32
        handle = kernel32.GetStdHandle(-11)
        mode = ctypes.c_uint32()
        if not kernel32.GetConsoleMode(handle, ctypes.byref(mode)):
            return False
        ENABLE_VIRTUAL_TERMINAL_PROCESSING = 0x0004
        return bool(kernel32.SetConsoleMode(
            handle, mode.value | ENABLE_VIRTUAL_TERMINAL_PROCESSING))
    except Exception:
        return False


ANSI_ENABLED = _enable_windows_ansi()


def color(text: str, code: str) -> str:
    if not ANSI_ENABLED:
        return text
    return f"{code}{text}{C.RESET}"


def apply_color_mode(mode: str) -> None:
    global ANSI_ENABLED
    if mode == "never":
        ANSI_ENABLED = False
    elif mode == "always":
        ANSI_ENABLED = True
    else:
        ANSI_ENABLED = _enable_windows_ansi()


                                                                            
             
                                                                            

def _ensure_dir() -> None:
    try:
        ROCKET_DIR.mkdir(parents=True, exist_ok=True)
    except OSError:
        pass


def _load_json(path: Path, default):
    try:
        if path.exists():
            return json.loads(path.read_text(encoding="utf-8"))
    except (OSError, json.JSONDecodeError):
        pass
    return default


def _save_json(path: Path, data) -> None:
    _ensure_dir()
    try:
        path.write_text(json.dumps(data, indent=2, ensure_ascii=False),
                        encoding="utf-8")
    except OSError:
        pass


DEFAULT_CONFIG = {
    "windows_shell": "cmd",                             
    "linux_command": "wsl",                                         
    "linux_shell": "direct",                                
    "default_distro": "",
    "mode": "auto",
    "show_exit_code": False,
    "show_timing": False,
    "timeout": 0,
    "color": "auto",
}


def _env_overrides(cfg: Dict) -> Dict:
    if os.environ.get("ROCKET_MODE"):
        cfg["mode"] = os.environ["ROCKET_MODE"].lower()
    if os.environ.get("ROCKET_WINDOWS_SHELL"):
        cfg["windows_shell"] = os.environ["ROCKET_WINDOWS_SHELL"].lower()
    if os.environ.get("ROCKET_LINUX_COMMAND"):
        cfg["linux_command"] = os.environ["ROCKET_LINUX_COMMAND"]
    if os.environ.get("ROCKET_LINUX_SHELL"):
        cfg["linux_shell"] = os.environ["ROCKET_LINUX_SHELL"].lower()
    if os.environ.get("ROCKET_DISTRO"):
        cfg["default_distro"] = os.environ["ROCKET_DISTRO"]
    if os.environ.get("ROCKET_TIMEOUT"):
        try:
            cfg["timeout"] = int(os.environ["ROCKET_TIMEOUT"])
        except ValueError:
            pass
    if os.environ.get("ROCKET_NO_COLOR"):
        cfg["color"] = "never"
    return cfg


class Config:
    def __init__(self) -> None:
        self.data = dict(DEFAULT_CONFIG)
        loaded = _load_json(CONFIG_FILE, {})
        if isinstance(loaded, dict):
            self.data.update(loaded)
        self.data = _env_overrides(self.data)

    def __getitem__(self, k):
        return self.data.get(k)

    def __setitem__(self, k, v):
        self.data[k] = v

    def save(self) -> None:
        _save_json(CONFIG_FILE, self.data)


class History:
    def __init__(self, max_size: int = 1000) -> None:
        self.max_size = max_size
        loaded = _load_json(HISTORY_FILE, [])
        self._items: List[str] = list(loaded) if isinstance(loaded, list) else []

    def add(self, cmd: str) -> None:
        cmd = cmd.strip()
        if not cmd:
            return
        if self._items and self._items[-1] == cmd:
            return
        self._items.append(cmd)
        if len(self._items) > self.max_size:
            self._items = self._items[-self.max_size:]
        self.save()

    def all(self) -> List[str]:
        return list(self._items)

    def get(self, index: int) -> Optional[str]:
        if 1 <= index <= len(self._items):
            return self._items[index - 1]
        return None

    def clear(self) -> None:
        self._items.clear()
        self.save()

    def save(self) -> None:
        _save_json(HISTORY_FILE, self._items)


class Aliases:
    def __init__(self) -> None:
        loaded = _load_json(ALIASES_FILE, {})
        self._map: Dict[str, str] = dict(loaded) if isinstance(loaded, dict) else {}

    def set(self, name: str, value: str) -> None:
        if name:
            self._map[name] = value
            _save_json(ALIASES_FILE, self._map)

    def unset(self, name: str) -> None:
        if name in self._map:
            del self._map[name]
            _save_json(ALIASES_FILE, self._map)

    def get(self, name: str) -> Optional[str]:
        return self._map.get(name)

    def all(self) -> Dict[str, str]:
        return dict(self._map)

    def expand(self, cmd: str) -> str:
        if not cmd.strip():
            return cmd
        parts = cmd.split(maxsplit=1)
        head = parts[0]
        if head in self._map:
            rest = parts[1] if len(parts) > 1 else ""
            expanded = self._map[head]
            return (expanded + (" " + rest if rest else "")).strip()
        return cmd


class Bookmarks:
    def __init__(self) -> None:
        loaded = _load_json(BOOKMARKS_FILE, {})
        self._map: Dict[str, str] = dict(loaded) if isinstance(loaded, dict) else {}

    def add(self, name: str, path: str) -> None:
        self._map[name] = path
        _save_json(BOOKMARKS_FILE, self._map)

    def get(self, name: str) -> Optional[str]:
        return self._map.get(name)

    def rm(self, name: str) -> None:
        if name in self._map:
            del self._map[name]
            _save_json(BOOKMARKS_FILE, self._map)

    def all(self) -> Dict[str, str]:
        return dict(self._map)


                                                                            
       
                                                                            

_WIN_DRIVE = re.compile(r"^([A-Za-z]):(?:[\\/](.*))?$")
_WSL_MNT = re.compile(r"^/mnt/([A-Za-z])(?:/(.*))?$")


def is_windows_path(p: str) -> bool:
    if not p:
        return False
    if _WIN_DRIVE.match(p):
        return True
    if p.startswith("\\\\") or p.startswith("//"):
        return len(p) > 2
    return False


def is_wsl_path(p: str) -> bool:
    if not p:
        return False
    return p.startswith("/") and not p.startswith("//")


def win_to_wsl(p: str) -> str:
    if not p:
        return p
    p = p.replace("\\", "/")
    m = _WIN_DRIVE.match(p)
    if m:
        drive = m.group(1).lower()
        rest = m.group(2)
        return f"/mnt/{drive}/{rest}" if rest else f"/mnt/{drive}"
    return p


def wsl_to_win(p: str) -> str:
    if not p:
        return p
    m = _WSL_MNT.match(p)
    if m:
        drive = m.group(1).upper()
        rest = (m.group(2) or "").replace("/", "\\")
        return f"{drive}:\\{rest}" if rest else f"{drive}:\\"
    return p


def resolve_linux_path(target: str, cwd: str, home: str) -> str:
    if target is None:
        target = ""
    target = target.strip()
    if target == "" or target == "~":
        return ""
    if target.startswith("~/"):
        base = home or "/"
        return posixpath.normpath(base.rstrip("/") + "/" + target[2:])
    if target.startswith("/"):
        return posixpath.normpath(target)
    base = cwd if cwd else (home or "/")
    return posixpath.normpath(base.rstrip("/") + "/" + target)


def resolve_windows_path(target: str, cwd: str) -> str:
    if not target:
        target = os.path.expanduser("~")
    else:
        target = os.path.expanduser(target)
    if os.path.isabs(target):
        return os.path.normpath(target)
    base = cwd or os.getcwd()
    return os.path.normpath(os.path.join(base, target))


                                                                            
               
                                                                            

_WIN_VAR = re.compile(r"%([A-Za-z_][A-Za-z0-9_]*)%")
_NIX_VAR = re.compile(r"\$(?:\{([A-Za-z_][A-Za-z0-9_]*)\}|([A-Za-z_][A-Za-z0-9_]*))")


def expand_env(s: str, extra: Optional[Dict[str, str]] = None) -> str:
    env = dict(os.environ)
    if extra:
        env.update(extra)

    def repl_win(m):
        return env.get(m.group(1), m.group(0))

    def repl_nix(m):
        name = m.group(1) or m.group(2)
        return env.get(name, m.group(0))

    s = _WIN_VAR.sub(repl_win, s)
    s = _NIX_VAR.sub(repl_nix, s)
    return s


                                                                            
        
                                                                            

@dataclass
class ParsedCommand:
    raw: str
    mode_override: Optional[str]
    argv: List[str]


def _tokenize(line: str) -> List[str]:
    out: List[str] = []
    cur: List[str] = []
    quote: Optional[str] = None
    escape = False
    for ch in line:
        if escape:
            cur.append(ch)
            escape = False
            continue
        if ch == "\\" and quote != "'":
            escape = True
            continue
        if quote:
            if ch == quote:
                quote = None
            else:
                cur.append(ch)
            continue
        if ch in ('"', "'"):
            quote = ch
            continue
        if ch.isspace():
            if cur:
                out.append("".join(cur))
                cur = []
            continue
        cur.append(ch)
    if cur:
        out.append("".join(cur))
    return out


def split_chain(line: str) -> List[Tuple[str, str]]:
    segments: List[Tuple[str, str]] = []
    buf: List[str] = []
    quote: Optional[str] = None
    escape = False
    op = ""
    i = 0
    n = len(line)
    while i < n:
        ch = line[i]
        if escape:
            buf.append(ch)
            escape = False
            i += 1
            continue
        if ch == "\\" and quote != "'":
            buf.append(ch)
            escape = True
            i += 1
            continue
        if quote:
            buf.append(ch)
            if ch == quote:
                quote = None
            i += 1
            continue
        if ch in ('"', "'"):
            quote = ch
            buf.append(ch)
            i += 1
            continue
        if line.startswith("&&", i):
            segments.append((op, "".join(buf).strip()))
            buf = []
            op = "&&"
            i += 2
            continue
        if line.startswith("||", i):
            segments.append((op, "".join(buf).strip()))
            buf = []
            op = "||"
            i += 2
            continue
        if ch == ";":
            segments.append((op, "".join(buf).strip()))
            buf = []
            op = ";"
            i += 1
            continue
        buf.append(ch)
        i += 1
    if buf:
        segments.append((op, "".join(buf).strip()))
    return segments


def has_toplevel_pipe(s: str) -> bool:
    """True if s has an unquoted top-level pipe.

    Recognized: `|` and `|&` (bash stderr-pipe).
    NOT a pipe: `||` (chain OR).
    """
    quote: Optional[str] = None
    escape = False
    i = 0
    n = len(s)
    while i < n:
        ch = s[i]
        if escape:
            escape = False
            i += 1
            continue
        if ch == "\\" and quote != "'":
            escape = True
            i += 1
            continue
        if quote:
            if ch == quote:
                quote = None
            i += 1
            continue
        if ch in ('"', "'"):
            quote = ch
            i += 1
            continue
        if ch == "|":
            if i + 1 < n and s[i + 1] == "|":
                i += 2
                continue
            return True
        i += 1
    return False


def parse(line: str) -> Optional[ParsedCommand]:
    original = line
    line = line.strip()
    if not line:
        return None
    mode_override: Optional[str] = None
    tokens = line.split()
    if len(tokens) >= 2 and tokens[0].lower() == "run":
        target = tokens[1].lower()
        if target in ("win", "windows", "cmd"):
            mode_override = "windows"
            rest = line.split(None, 2)
            line = rest[2] if len(rest) > 2 else ""
        elif target in ("linux", "wsl", "sh", "bash"):
            mode_override = "linux"
            rest = line.split(None, 2)
            line = rest[2] if len(rest) > 2 else ""
        else:
            return ParsedCommand(raw=original, mode_override=None, argv=tokens)
    argv = _tokenize(line) if line else []
    if not argv and mode_override is None:
        return None
    return ParsedCommand(raw=original, mode_override=mode_override, argv=argv)


def strip_run_prefix(s: str) -> Tuple[Optional[str], str]:
    tokens = s.split(None, 2)
    if len(tokens) >= 2 and tokens[0].lower() == "run":
        t = tokens[1].lower()
        if t in ("win", "windows", "cmd"):
            return "windows", (tokens[2] if len(tokens) > 2 else "")
        if t in ("linux", "wsl", "sh", "bash"):
            return "linux", (tokens[2] if len(tokens) > 2 else "")
    return None, s


                                                                            
        
                                                                            

LINUX_COMMANDS = {
    "ls", "grep", "sed", "awk", "uname", "pwd", "cat", "chmod", "chown",
    "find", "which", "ps", "top", "df", "du", "tar", "gzip",
    "curl", "wget", "ssh", "scp", "touch", "rm", "cp", "mv", "ln",
    "mkdir", "rmdir", "env", "export", "man", "less", "more", "head",
    "tail", "wc", "sort", "uniq", "tr", "cut", "xargs", "tee", "sudo",
    "kill", "killall", "systemctl", "service", "nano", "vim", "emacs",
    "bash", "sh", "zsh", "echo",
}

WINDOWS_COMMANDS = {
    "dir", "ipconfig", "tasklist", "cls", "systeminfo", "taskkill",
    "tracert", "netstat", "nslookup", "where", "type", "copy", "move",
    "del", "ren", "md", "rd", "attrib", "findstr", "icacls", "takeown",
    "sfc", "chkdsk", "sc", "reg", "wmic", "powershell", "cmd", "start",
    "path", "echo",
}


def detect_mode(argv: List[str], current_mode: str) -> str:
    if not argv:
        return current_mode
    head = argv[0].lower()
    base = head
    for ext in (".exe", ".cmd", ".bat", ".ps1"):
        if base.endswith(ext):
            base = base[: -len(ext)]
            break
    if base in WINDOWS_COMMANDS:
        return "windows"
    if base in LINUX_COMMANDS:
        return "linux"
    return current_mode


                                                                            
             
                                                                            

def is_windows() -> bool:
    return platform.system().lower().startswith("win")


def _utf8_env_linux() -> Dict[str, str]:
    env = dict(os.environ)
    env["LANG"] = "C.UTF-8"
    env["LC_ALL"] = "C.UTF-8"
    env["LANGUAGE"] = "C"
    return env


_wsl_cache: Dict[str, object] = {"value": None, "time": 0.0}
_WSL_CACHE_TTL = 30.0

_distros_cache: Dict[str, object] = {"value": None, "time": 0.0}
_DISTROS_CACHE_TTL = 10.0


def _check_wsl() -> bool:
    if shutil.which("wsl") is None:
        return False
    try:
        p = subprocess.run(
            ["wsl", "--status"], capture_output=True, text=True,
            encoding="utf-8", errors="replace",
            env=_utf8_env_linux(), timeout=6,
        )
        if p.returncode == 0:
            return True
    except (OSError, subprocess.SubprocessError):
        pass
    try:
        p = subprocess.run(
            ["wsl", "-l", "-q"], capture_output=True, text=True,
            encoding="utf-8", errors="replace",
            env=_utf8_env_linux(), timeout=6,
        )
        return p.returncode == 0
    except (OSError, subprocess.SubprocessError):
        return False


def is_wsl_available(force: bool = False) -> bool:
    now = time.time()
    cached = _wsl_cache["value"]
    if not force and cached is not None and now - float(_wsl_cache["time"]) < _WSL_CACHE_TTL:
        return bool(cached)
    result = _check_wsl()
    _wsl_cache["value"] = result
    _wsl_cache["time"] = now
    return result


def invalidate_wsl_cache() -> None:
    _wsl_cache["value"] = None
    _wsl_cache["time"] = 0.0
    _distros_cache["value"] = None
    _distros_cache["time"] = 0.0


def list_wsl_distros(force: bool = False) -> List[str]:
    now = time.time()
    cached = _distros_cache["value"]
    if not force and cached is not None and now - float(_distros_cache["time"]) < _DISTROS_CACHE_TTL:
        return list(cached)                          
    try:
        p = subprocess.run(
            ["wsl", "-l", "-q"], capture_output=True, text=True,
            encoding="utf-8", errors="replace",
            env=_utf8_env_linux(), timeout=6,
        )
    except (OSError, subprocess.SubprocessError):
        invalidate_wsl_cache()
        return []
    if p.returncode != 0:
        invalidate_wsl_cache()
        return []
    raw = (p.stdout or "").replace("\x00", "")
    result = [line.strip() for line in raw.splitlines() if line.strip()]
    _distros_cache["value"] = result
    _distros_cache["time"] = now
    return result


def get_sys_info() -> Dict[str, str]:
    return {
        "os": platform.system(),
        "os_release": platform.release(),
        "os_version": platform.version(),
        "machine": platform.machine(),
        "processor": platform.processor(),
        "python": sys.version.split()[0],
        "hostname": platform.node(),
    }


                                                                            
         
                                                                            

PrintFn = Callable[[str], None]

                                                                           

_CMD_QUOTE_CHARS = set(' \t"|&<>^()')


def _cmd_quote(s: str) -> str:
    """Quote an argv item for `cmd /c`.

    - Empty -> ""
    - No metachars -> bare
    - Else -> "...", internal `"` becomes `""` (cmd's own escape inside quotes)
    """
    if s == "":
        return '""'
    if not any(c in _CMD_QUOTE_CHARS for c in s):
        return s
    return '"' + s.replace('"', '""') + '"'


                                                                           

_PS_META = set(" \t\n\"'`&|;<>(){}[]")
_PS_DOLLAR = re.compile(r"\$[A-Za-z_0-9{(?^]")


def _ps_quote(s: str) -> str:
    """Quote an argv item for `powershell -Command "<joined>"`.

    - Empty -> ''
    - No metachars -> bare (so `$env:PATH`, `$_`, `$?` expand naturally)
    - Metachars + `$` -> "..." (so expansion still happens, `"` and `` ` `` escaped)
    - Metachars without `$` -> '...' (literal)
    """
    if s == "":
        return "''"
    if not any(c in _PS_META for c in s):
        return s
    if _PS_DOLLAR.search(s):
        escaped = s.replace("`", "``").replace('"', '`"')
        return f'"{escaped}"'
    return "'" + s.replace("'", "''") + "'"


                                                                           

_SH_SAFE = re.compile(r"^[A-Za-z0-9_\-./=:@%,+]+$")


def _sh_quote(s: str) -> str:
    if s == "":
        return "''"
    if _SH_SAFE.match(s):
        return s
    return "'" + s.replace("'", "'\\''") + "'"


def _build_windows_cmd(argv: List[str], shell: str) -> List[str]:
    if shell == "powershell":
        ps_args = " ".join(_ps_quote(a) for a in argv)
        ps = "[Console]::OutputEncoding=[System.Text.Encoding]::UTF8; " + ps_args
        return ["powershell", "-NoLogo", "-NoProfile", "-Command", ps]
    joined = " ".join(_cmd_quote(a) for a in argv)
    return ["cmd", "/d", "/c", f"chcp 65001 >nul & {joined}"]


def _build_linux_cmd(argv: List[str], cwd: str, wsl_exe: str, distro: str,
                     linux_shell: str) -> List[str]:
    base = [wsl_exe]
    if distro:
        base += ["-d", distro]
    if cwd and cwd.startswith("/"):
        base += ["--cd", cwd]
    if linux_shell == "bash":
        joined = " ".join(_sh_quote(a) for a in argv)
        base += ["bash", "-lc", joined]
    elif linux_shell == "sh":
        joined = " ".join(_sh_quote(a) for a in argv)
        base += ["sh", "-c", joined]
    else:
        base += argv
    return base


def _stream_run(cmd: List[str], cwd: str, env: Optional[Dict[str, str]],
                timeout: int, on_line: PrintFn,
                on_stderr: PrintFn) -> int:
    kwargs: Dict = {}
    if cwd:
        kwargs["cwd"] = cwd
    if env is not None:
        kwargs["env"] = env
    try:
        proc = subprocess.Popen(
            cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE,
            text=True, encoding="utf-8", errors="replace",
            bufsize=1, **kwargs,
        )
    except FileNotFoundError as e:
        on_stderr(f"Command not found: {e}")
        return 1
    except OSError as e:
        on_stderr(f"OS error: {e}")
        return 1

    def reader(stream, emit):
        try:
            for line in iter(stream.readline, ""):
                if line == "":
                    break
                emit(line.rstrip("\n"))
        finally:
            try:
                stream.close()
            except Exception:
                pass

    t_out = threading.Thread(target=reader, args=(proc.stdout, on_line), daemon=True)
    t_err = threading.Thread(target=reader, args=(proc.stderr, on_stderr), daemon=True)
    t_out.start()
    t_err.start()

    try:
        code = proc.wait(timeout=timeout if timeout > 0 else None)
    except subprocess.TimeoutExpired:
        try:
            proc.kill()
        except Exception:
            pass
        on_stderr(f"[timeout after {timeout}s]")
        code = 124
    except KeyboardInterrupt:
        try:
            proc.kill()
        except Exception:
            pass
        try:
            proc.wait(timeout=2)
        except Exception:
            pass
        t_out.join(timeout=1)
        t_err.join(timeout=1)
        raise
    t_out.join(timeout=2)
    t_err.join(timeout=2)
    return code


def run_windows(argv: List[str], cwd: str, shell: str,
                timeout: int, out: PrintFn, err: PrintFn,
                env: Optional[Dict[str, str]] = None) -> int:
    if not argv:
        return 0
    return _stream_run(_build_windows_cmd(argv, shell), cwd, env, timeout, out, err)


def run_linux(argv: List[str], cwd: str, wsl_exe: str, distro: str,
              linux_shell: str, timeout: int, out: PrintFn, err: PrintFn,
              env: Optional[Dict[str, str]] = None) -> int:
    if not argv:
        return 0
    if env is None:
        env = _utf8_env_linux()
    cmd = _build_linux_cmd(argv, cwd, wsl_exe, distro, linux_shell)
    return _stream_run(cmd, "", env, timeout, out, err)


def run_windows_raw(raw: str, cwd: str, shell: str, timeout: int,
                    out: PrintFn, err: PrintFn,
                    env: Optional[Dict[str, str]] = None) -> int:
    if shell == "powershell":
        ps = "[Console]::OutputEncoding=[System.Text.Encoding]::UTF8; " + raw
        cmd = ["powershell", "-NoLogo", "-NoProfile", "-Command", ps]
    else:
        cmd = ["cmd", "/d", "/c", f"chcp 65001 >nul & {raw}"]
    return _stream_run(cmd, cwd, env, timeout, out, err)


def run_linux_raw(raw: str, cwd: str, wsl_exe: str, distro: str,
                  timeout: int, out: PrintFn, err: PrintFn,
                  env: Optional[Dict[str, str]] = None) -> int:
    base = [wsl_exe]
    if distro:
        base += ["-d", distro]
    if cwd and cwd.startswith("/"):
        base += ["--cd", cwd]
    base += ["bash", "-lc", raw]
    if env is None:
        env = _utf8_env_linux()
    return _stream_run(base, "", env, timeout, out, err)


                                                                            
      
                                                                            

HELP_TEXT = f"""ROCKET SHELL {VERSION} — built-in commands

  Core
    help                        Show this help
    version                     Show version
    status                      Mode, OS, WSL, Python, shells, paths
    sysinfo                     System information
    mode [auto|windows|linux]   Show or change mode
    win / linux                 Shortcut to switch mode
    shell [status|cmd|powershell|direct|bash|sh]
                                Show/set active shell per mode
    run win <cmd...>            Force Windows execution
    run linux <cmd...>          Force Linux execution
    timeout <sec> <cmd...>      Run with a timeout (0 = no timeout)
    time on|off                 Toggle timing display
    exitcode on|off             Toggle exit code display

  Filesystem
    cd <path>                   Change directory (relative / ~ supported)
    pwd                         Show current directory
    path                        Show both cwd + bridge
    bridge <path>               Convert path win <-> wsl
    w2l <path>                  Windows path -> WSL path
    l2w <path>                  WSL path -> Windows path
    drives                      List Windows drives as /mnt/*
    bm add <name> [path]        Add bookmark (default: current cwd)
    bm <name>                   cd to bookmark
    bm list                     List bookmarks
    bm rm <name>                Remove bookmark

  History & aliases
    history                     Show history
    history run <n>             Repeat entry #n
    history clear               Clear history
    alias                       List aliases
    alias name=value            Set alias
    alias name                  Show one alias
    unalias name                Remove alias

  Session environment
    export NAME=value           Set a session env var (visible to subprocesses)
    unset NAME                  Remove a session env var
    env                         List session env vars

  WSL & config
    wsl                         List distros
    wsl -d <distro>             Use specific distro for this session
    wsl -d default              Reset to default distro
    config                      Show config
    config set <key> <value>    Change config value
    config save                 Persist config

  Misc
    clear / cls                 Clear screen
    rocket                      Banner
    exit / quit                 Exit

  Chains:  <cmd1> ; <cmd2>     run both
           <cmd1> && <cmd2>    cmd2 if cmd1 ok
           <cmd1> || <cmd2>    cmd2 if cmd1 failed
  Pipes:   <cmd1> | <cmd2>     routed to active shell
           <cmd1> |& <cmd2>    bash stderr-pipe (linux mode)
  Env:     $VAR  ${{VAR}}  %VAR%  expanded before run
"""


                                                                            
       
                                                                            

class Shell:
    VERSION = VERSION

    def __init__(self, timeout: int = 0,
                 color_mode: Optional[str] = None,
                 extra_env: Optional[Dict[str, str]] = None) -> None:
        self.config = Config()
        if color_mode:
            self.config["color"] = color_mode
        apply_color_mode(str(self.config["color"]))
        self.mode = str(self.config["mode"]).lower()
        if self.mode not in ("auto", "windows", "linux"):
            self.mode = "auto"
        self.windows_cwd = os.getcwd() if is_windows() else ""
        self.linux_cwd = ""
        self.linux_home: Optional[str] = None
        self.history = History()
        self.aliases = Aliases()
        self.bookmarks = Bookmarks()
        self.session_env: Dict[str, str] = dict(extra_env or {})
        self.running = True
        self.show_exit_code = bool(self.config["show_exit_code"])
        self.show_timing = bool(self.config["show_timing"])
        self.timeout = timeout or int(self.config["timeout"] or 0)
        self.distro = str(self.config["default_distro"] or "")
        self._interrupted = False
        self.on_clear: Optional[Callable[[], None]] = None

                                                                           

    def print(self, text: str = "") -> None:
        print(text)

    def info(self, text: str) -> None:
        print(color(text, C.GREY))

    def warn(self, text: str) -> None:
        print(color(text, C.YELLOW))

    def error(self, text: str) -> None:
        print(color(text, C.RED))

    def prompt(self) -> str:
        if self.mode == "windows":
            p = "win"
        elif self.mode == "linux":
            p = "linux"
        else:
            p = "rocket"
        cwd = self.linux_cwd if self.mode == "linux" else self.windows_cwd
        tail = f" {C.DIM}{cwd}{C.RESET}" if cwd else ""
        return f"{color(p, C.CYAN)}{color('>', C.CYAN)}{tail} "

                                                                           

    def _windows_env(self) -> Dict[str, str]:
        env = dict(os.environ)
        env.update(self.session_env)
        return env

    def _linux_env(self) -> Dict[str, str]:
        env = _utf8_env_linux()
        env.update(self.session_env)
        return env

                                                                           

    def execute(self, line: str) -> None:
        line = line.strip()
        if not line:
            return
        self.history.add(line)
        segments = split_chain(line)
        prev_code = 0
        for op, seg in segments:
            if op == "&&" and prev_code != 0:
                continue
            if op == "||" and prev_code == 0:
                continue
            if not seg:
                continue
            prev_code = self._execute_one(seg)
            if self._interrupted:
                self._interrupted = False
                break

    def _resolve_target(self, mode_override: Optional[str],
                        argv_for_detect: List[str]) -> str:
        if mode_override is not None:
            return mode_override
        if self.mode == "windows":
            return "windows"
        if self.mode == "linux":
            return "linux"
        default = "windows" if is_windows() else "linux"
        return detect_mode(argv_for_detect, default)

    def _execute_one(self, seg: str) -> int:
        expanded = expand_env(self.aliases.expand(seg), self.session_env)

        mode_override, body = strip_run_prefix(expanded)
        if has_toplevel_pipe(body):
            target = self._resolve_target(mode_override, body.split())
            return self._run_raw_pipe(body, target)

        parsed = parse(expanded)
        if parsed is None:
            return 0
        builtin_code = self._builtin(parsed)
        if builtin_code is not None:
            return builtin_code
        if not parsed.argv:
            return 0

        target = self._resolve_target(parsed.mode_override, parsed.argv)

        t0 = time.monotonic()
        try:
            if target == "linux":
                if not is_wsl_available():
                    self.error("ERROR: WSL is not available. Install WSL2 to use linux mode.")
                    return 1
                code = run_linux(
                    parsed.argv, self.linux_cwd,
                    str(self.config["linux_command"]),
                    self.distro,
                    str(self.config["linux_shell"]),
                    self.timeout, self.print, self.error,
                    env=self._linux_env(),
                )
            else:
                code = run_windows(
                    parsed.argv, self.windows_cwd,
                    str(self.config["windows_shell"]),
                    self.timeout, self.print, self.error,
                    env=self._windows_env(),
                )
        except KeyboardInterrupt:
            self._interrupted = True
            self.warn("[interrupted]")
            return 130

        elapsed = time.monotonic() - t0
        if self.show_timing:
            self.info(f"[{elapsed:.3f}s]")
        if self.show_exit_code:
            self.info(f"[exit {code}]")
        return code

    def _run_raw_pipe(self, body: str, target: str) -> int:
        t0 = time.monotonic()
        try:
            if target == "linux":
                if not is_wsl_available():
                    self.error("ERROR: WSL is not available. Install WSL2 to use linux mode.")
                    return 1
                code = run_linux_raw(
                    body, self.linux_cwd,
                    str(self.config["linux_command"]),
                    self.distro, self.timeout, self.print, self.error,
                    env=self._linux_env(),
                )
            else:
                code = run_windows_raw(
                    body, self.windows_cwd,
                    str(self.config["windows_shell"]),
                    self.timeout, self.print, self.error,
                    env=self._windows_env(),
                )
        except KeyboardInterrupt:
            self._interrupted = True
            self.warn("[interrupted]")
            return 130

        elapsed = time.monotonic() - t0
        if self.show_timing:
            self.info(f"[{elapsed:.3f}s]")
        if self.show_exit_code:
            self.info(f"[exit {code}]")
        return code

                                                                           

    def _builtin(self, parsed: ParsedCommand) -> Optional[int]:
        argv = parsed.argv
        if not argv:
            return 0
        cmd = argv[0].lower()
        args = argv[1:]

        if cmd in ("exit", "quit"):
            self.running = False
            return 0
        if cmd == "help":
            self.print(HELP_TEXT)
            return 0
        if cmd == "version":
            self.print(f"ROCKET SHELL v{self.VERSION}")
            return 0
        if cmd == "rocket":
            self.print(color("ROCKET SHELL", C.CYAN + C.BOLD)
                       + " — unified Windows/Linux command line")
            return 0
        if cmd in ("clear", "cls"):
            if self.on_clear is not None:
                self.on_clear()
            else:
                os.system("cls" if os.name == "nt" else "clear")
            return 0
        if cmd == "mode":
            if not args:
                self.print(f"Current mode: {self.mode}")
                return 0
            m = args[0].lower()
            if m == "win":
                m = "windows"
            if m not in ("auto", "windows", "linux"):
                self.warn("Usage: mode [auto|windows|linux]")
                return 1
            self.mode = m
            self.config["mode"] = m
            self.config.save()
            self.print(f"Mode set to: {m}")
            return 0
        if cmd in ("win", "windows"):
            self.mode = "windows"
            self.config["mode"] = "windows"
            self.config.save()
            self.print("Mode set to: windows")
            return 0
        if cmd == "linux":
            self.mode = "linux"
            self.config["mode"] = "linux"
            self.config.save()
            self.print("Mode set to: linux")
            return 0
        if cmd == "shell":
            return self._cmd_shell(args)
        if cmd == "timeout":
            if not args:
                self.print(f"Current timeout: {self.timeout}s (0 = disabled)")
                return 0
            try:
                self.timeout = int(args[0])
                self.print(f"Timeout set to: {self.timeout}s")
                return 0
            except ValueError:
                self.warn("Usage: timeout <seconds>")
                return 1
        if cmd == "time":
            if args and args[0].lower() in ("on", "off"):
                self.show_timing = args[0].lower() == "on"
                self.config["show_timing"] = self.show_timing
                self.config.save()
                self.print(f"Timing: {'on' if self.show_timing else 'off'}")
                return 0
            self.warn("Usage: time on|off")
            return 1
        if cmd == "exitcode":
            if args and args[0].lower() in ("on", "off"):
                self.show_exit_code = args[0].lower() == "on"
                self.config["show_exit_code"] = self.show_exit_code
                self.config.save()
                self.print(f"Exit code display: {'on' if self.show_exit_code else 'off'}")
                return 0
            self.warn("Usage: exitcode on|off")
            return 1
        if cmd == "cd":
            return self._cd(args)
        if cmd == "pwd":
            if self.mode == "linux":
                self.print(self.linux_cwd or "~")
            else:
                self.print(self.windows_cwd or os.getcwd())
            return 0
        if cmd == "path":
            self._path()
            return 0
        if cmd == "bridge":
            if not args:
                self.warn("Usage: bridge <path>")
                return 1
            p = " ".join(args)
            if is_windows_path(p):
                self.print(f"win -> wsl: {p}  =>  {win_to_wsl(p)}")
            elif is_wsl_path(p):
                self.print(f"wsl -> win: {p}  =>  {wsl_to_win(p)}")
            else:
                self.warn(f"unrecognized path: {p}")
                return 1
            return 0
        if cmd == "w2l":
            if not args:
                self.warn("Usage: w2l <windows-path>")
                return 1
            self.print(win_to_wsl(" ".join(args)))
            return 0
        if cmd == "l2w":
            if not args:
                self.warn("Usage: l2w <wsl-path>")
                return 1
            self.print(wsl_to_win(" ".join(args)))
            return 0
        if cmd == "drives":
            self._drives()
            return 0
        if cmd == "bm":
            return self._bookmarks(args)
        if cmd == "status":
            self._status()
            return 0
        if cmd == "sysinfo":
            for k, v in get_sys_info().items():
                self.print(f"{k:14}: {v}")
            return 0
        if cmd == "history":
            return self._history(args)
        if cmd == "alias":
            return self._alias(args)
        if cmd == "unalias":
            if not args:
                self.warn("Usage: unalias <name>")
                return 1
            self.aliases.unset(args[0])
            self.print(f"Alias removed: {args[0]}")
            return 0
        if cmd == "export":
            return self._export(args)
        if cmd == "unset":
            if not args:
                self.warn("Usage: unset NAME")
                return 1
            for name in args:
                self.session_env.pop(name, None)
            self.print(f"Unset: {' '.join(args)}")
            return 0
        if cmd == "env":
            if not self.session_env:
                self.info("(no session env vars)")
                return 0
            for k in sorted(self.session_env):
                self.print(f"{k}={self.session_env[k]}")
            return 0
        if cmd == "wsl":
            self._wsl(args)
            return 0
        if cmd == "config":
            return self._config(args)
        return None

                                                                           

    def _cmd_shell(self, args: List[str]) -> int:
        if not args or args[0].lower() == "status":
            self.print(f"Windows shell : {self.config['windows_shell']}")
            self.print(f"Linux shell   : {self.config['linux_shell']} "
                       f"(via {self.config['linux_command']})")
            self.print(f"Active mode   : {self.mode}")
            return 0

        target = args[0].lower()

        if target in ("cmd", "cmd.exe"):
            self.config["windows_shell"] = "cmd"
            self.config.save()
            self.print("Windows shell: cmd")
            return 0
        if target in ("powershell", "ps", "pwsh"):
            self.config["windows_shell"] = "powershell"
            self.config.save()
            self.print("Windows shell: powershell")
            return 0
        if target in ("direct", "wsl"):
            self.config["linux_shell"] = "direct"
            self.config.save()
            self.print("Linux shell: direct")
            return 0
        if target == "bash":
            self.config["linux_shell"] = "bash"
            self.config.save()
            self.print("Linux shell: bash -lc")
            return 0
        if target == "sh":
            self.config["linux_shell"] = "sh"
            self.config.save()
            self.print("Linux shell: sh -c")
            return 0

        self.warn(f"Unknown shell: {args[0]}")
        return 1

                                                                           

    def _export(self, args: List[str]) -> int:
        if not args:
            for k in sorted(self.session_env):
                self.print(f"{k}={self.session_env[k]}")
            return 0
        joined = " ".join(args)
        if "=" not in joined:
            self.warn("Usage: export NAME=value")
            return 1
        name, _, value = joined.partition("=")
        name = name.strip()
        value = value.strip().strip('"').strip("'")
        if not name:
            self.warn("Usage: export NAME=value")
            return 1
        self.session_env[name] = value
        self.print(f"export {name}={value}")
        return 0

                                                                           

    def _linux_home(self) -> str:
        if self.linux_home is not None:
            return self.linux_home
        if not is_wsl_available():
            self.linux_home = ""
            return ""
        out_buf: List[str] = []
        code = run_linux(["pwd"], "", str(self.config["linux_command"]),
                         self.distro, "direct", 6,
                         out_buf.append, lambda s: None,
                         env=self._linux_env())
        if code == 0 and out_buf:
            self.linux_home = out_buf[-1].strip()
        else:
            self.linux_home = ""
        return self.linux_home

    def _invalidate_linux_state(self) -> None:
        self.linux_home = None
        invalidate_wsl_cache()

    def _cd(self, args: List[str]) -> int:
        target = args[0] if args else ""
        if self.mode == "linux":
            return self._cd_linux(target)
        return self._cd_windows(target)

    def _cd_linux(self, target: str) -> int:
        if not is_wsl_available():
            self.error("WSL is not available. Cannot change Linux directory.")
            return 1
        if is_windows_path(target):
            bridged = win_to_wsl(target)
            self.info(f"[bridge] {target} -> {bridged}")
            target = bridged
        home = self._linux_home()
        resolved = resolve_linux_path(target, self.linux_cwd, home)
        if resolved == "":
            self.linux_cwd = ""
            self.print("~")
            return 0
        code = run_linux(["pwd"], resolved, str(self.config["linux_command"]),
                         self.distro, "direct", 6,
                         lambda s: None, lambda s: None,
                         env=self._linux_env())
        if code == 0:
            self.linux_cwd = resolved
            self.print(resolved)
            return 0
        self.error(f"cd: failed: {resolved}")
        return 1

    def _cd_windows(self, target: str) -> int:
        if is_wsl_path(target):
            bridged = wsl_to_win(target)
            self.info(f"[bridge] {target} -> {bridged}")
            target = bridged
        resolved = resolve_windows_path(target, self.windows_cwd)
        if os.path.isdir(resolved):
            self.windows_cwd = resolved
            self.print(resolved)
            return 0
        self.error(f"cd: directory not found: {resolved}")
        return 1

                                                                           

    def _path(self) -> None:
        w = self.windows_cwd or "(unset)"
        l = self.linux_cwd or "~"
        self.print(f"windows : {w}")
        self.print(f"linux   : {l}")
        if self.windows_cwd and is_windows_path(self.windows_cwd):
            self.print(f"bridged : {win_to_wsl(self.windows_cwd)}")

    def _drives(self) -> None:
        if platform.system().lower() != "windows":
            self.warn("drives: Windows host only.")
            return
        try:
            import ctypes, string
            bitmask = ctypes.windll.kernel32.GetLogicalDrives()
        except Exception as e:
            self.error(f"drives: {e}")
            return
        for i, letter in enumerate(string.ascii_uppercase):
            if bitmask & (1 << i):
                self.print(f"{letter}:\\  ->  /mnt/{letter.lower()}")

    def _bookmarks(self, args: List[str]) -> int:
        if not args or args[0].lower() == "list":
            data = self.bookmarks.all()
            if not data:
                self.info("(no bookmarks)")
                return 0
            for k, v in data.items():
                self.print(f"  {k:16} {v}")
            return 0
        sub = args[0].lower()
        if sub == "add":
            if len(args) < 2:
                self.warn("Usage: bm add <name> [path]")
                return 1
            name = args[1]
            if len(args) >= 3:
                path = " ".join(args[2:])
            else:
                path = self.linux_cwd if self.mode == "linux" else self.windows_cwd
            if not path:
                self.warn("No current directory to bookmark.")
                return 1
            self.bookmarks.add(name, path)
            self.print(f"Bookmark added: {name} -> {path}")
            return 0
        if sub == "rm":
            if len(args) < 2:
                self.warn("Usage: bm rm <name>")
                return 1
            self.bookmarks.rm(args[1])
            self.print(f"Bookmark removed: {args[1]}")
            return 0
        path = self.bookmarks.get(args[0])
        if path is None:
            self.warn(f"No bookmark: {args[0]}")
            return 1
        return self._cd([path])

    def _status(self) -> None:
        self.print(f"ROCKET SHELL version : {self.VERSION}")
        self.print(f"Current mode         : {self.mode}")
        self.print(f"Windows shell        : {self.config['windows_shell']}")
        self.print(f"Linux shell          : {self.config['linux_shell']} "
                   f"(via {self.config['linux_command']})")
        self.print(f"OS                   : {platform.system()} {platform.release()}")
        avail = is_wsl_available()
        self.print(f"WSL available        : {avail}")
        if avail:
            distros = list_wsl_distros()
            self.print(f"WSL distros          : {', '.join(distros) if distros else '(none)'}")
            if self.distro:
                self.print(f"Active distro        : {self.distro}")
        self.print(f"Python version       : {sys.version.split()[0]}")
        self.print(f"Windows cwd          : {self.windows_cwd or '(unset)'}")
        self.print(f"Linux cwd            : {self.linux_cwd or '~'}")
        self.print(f"Timeout              : {self.timeout}s (0 = disabled)")
        self.print(f"ANSI colors          : {'on' if ANSI_ENABLED else 'off'} ({self.config['color']})")
        self.print(f"Session env vars     : {len(self.session_env)}")
        self.print(f"State dir            : {ROCKET_DIR}")

    def _history(self, args: List[str]) -> int:
        if args and args[0].lower() == "run":
            if len(args) < 2:
                self.warn("Usage: history run <index>")
                return 1
            try:
                idx = int(args[1])
            except ValueError:
                self.warn("Index must be an integer.")
                return 1
            entry = self.history.get(idx)
            if entry is None:
                self.warn(f"No history entry #{idx}")
                return 1
            self.info(f"> {entry}")
            self.execute(entry)
            return 0
        if args and args[0].lower() == "clear":
            self.history.clear()
            self.print("History cleared.")
            return 0
        items = self.history.all()
        if not items:
            self.info("(history empty)")
            return 0
        for i, item in enumerate(items, 1):
            self.print(f"{i:4}  {item}")
        return 0

    def _alias(self, args: List[str]) -> int:
        if not args:
            data = self.aliases.all()
            if not data:
                self.info("(no aliases)")
                return 0
            for k, v in data.items():
                self.print(f"{k} = {v}")
            return 0
        if len(args) == 1 and "=" not in args[0]:
            val = self.aliases.get(args[0])
            if val is None:
                self.warn(f"No alias: {args[0]}")
                return 1
            self.print(f"{args[0]} = {val}")
            return 0
        joined = " ".join(args)
        if "=" not in joined:
            self.warn("Usage: alias name=value")
            return 1
        name, _, value = joined.partition("=")
        name = name.strip()
        value = value.strip().strip('"').strip("'")
        if not name or not value:
            self.warn("Usage: alias name=value")
            return 1
        self.aliases.set(name, value)
        self.print(f"Alias set: {name} = {value}")
        return 0

    def _wsl(self, args: List[str]) -> None:
        if not is_wsl_available():
            self.error("WSL is not available.")
            return
        if args and args[0] == "-d" and len(args) >= 2:
            target = args[1]
            if target == "default":
                self.distro = ""
                self.print("WSL distro: default")
            else:
                self.distro = target
                self.print(f"WSL distro: {target}")
            self._invalidate_linux_state()
            return
        if not args or args[0].lower() in ("list", "ls", "-l"):
            distros = list_wsl_distros()
            if not distros:
                self.info("(no WSL distros found)")
                return
            for d in distros:
                marker = " *" if d == self.distro else ""
                self.print(f"{d}{marker}")
            return
        self.warn(f"Unknown wsl subcommand: {args[0]}")

    def _config(self, args: List[str]) -> int:
        if not args:
            for k, v in self.config.data.items():
                self.print(f"{k:18} = {v!r}")
            return 0
        if args[0] == "save":
            self.config.save()
            self.print(f"Config saved to {CONFIG_FILE}")
            return 0
        if args[0] == "set" and len(args) >= 3:
            key = args[1]
            value = " ".join(args[2:])
            if value.lower() in ("true", "false"):
                value = value.lower() == "true"
            elif value.isdigit():
                value = int(value)
            self.config[key] = value
            self.config.save()
            if key == "color":
                apply_color_mode(str(value))
            self.print(f"Config: {key} = {value!r}")
            return 0
        self.warn("Usage: config | config save | config set <key> <value>")
        return 1


                                                                            
            
                                                                            

BUILTIN_NAMES = [
    "help", "version", "status", "sysinfo", "mode", "win", "linux", "shell",
    "run", "timeout", "time", "exitcode", "cd", "pwd", "path", "bridge",
    "w2l", "l2w", "drives", "bm", "history", "alias", "unalias",
    "export", "unset", "env", "wsl", "config", "clear", "cls", "rocket",
    "exit", "quit",
]


def setup_readline(shell: Shell) -> bool:
    readline = None
    try:
        import readline                
    except ImportError:
        try:
            import pyreadline3 as readline                
        except ImportError:
            return False

    _ensure_dir()
    try:
        if CLI_HISTORY_FILE.exists():
            readline.read_history_file(str(CLI_HISTORY_FILE))
    except Exception:
        pass
    try:
        readline.set_history_length(2000)
    except Exception:
        pass

    def completer(text: str, state: int):
        buf = readline.get_line_buffer()
        try:
            opts: List[str] = []
            if " " not in buf.lstrip():
                opts = [n for n in BUILTIN_NAMES if n.startswith(text)]
            else:
                cmd = buf.split()[0].lower()
                if cmd == "mode":
                    opts = [m for m in ("auto", "windows", "linux") if m.startswith(text)]
                elif cmd == "shell":
                    opts = [m for m in ("status", "cmd", "powershell",
                                        "direct", "bash", "sh")
                            if m.startswith(text)]
                elif cmd in ("time", "exitcode"):
                    opts = [m for m in ("on", "off") if m.startswith(text)]
                elif cmd == "bm":
                    opts = [k for k in ("add", "rm", "list") if k.startswith(text)]
                    opts += [k for k in shell.bookmarks.all() if k.startswith(text)]
                elif cmd == "wsl":
                    opts = [d for d in list_wsl_distros() if d.startswith(text)]
                    opts += [d for d in ("-d", "list") if d.startswith(text)]
                elif cmd == "cd" and text:
                    base = shell.linux_cwd if shell.mode == "linux" else shell.windows_cwd
                    try:
                        entries = os.listdir(base or ".")
                        opts = [e for e in entries if e.startswith(text)]
                    except OSError:
                        opts = []
                elif cmd in ("alias", "unset"):
                    opts = [k for k in shell.aliases.all() if k.startswith(text)]
                    opts += [k for k in shell.session_env if k.startswith(text)]
            if state < len(opts):
                return opts[state]
            return None
        except Exception:
            return None

    readline.set_completer(completer)
    readline.set_completer_delims(" \t\n")
    try:
        readline.parse_and_bind("tab: complete")
    except Exception:
        pass

    def _save():
        try:
            readline.write_history_file(str(CLI_HISTORY_FILE))
        except Exception:
            pass

    atexit.register(_save)
    return True


                                                                            
     
                                                                            

def main_cli(shell: Shell, run_cmd: Optional[str] = None) -> int:
    if run_cmd:
        shell.execute(run_cmd)
        return 0
    shell.print(color(f"ROCKET SHELL v{VERSION}", C.CYAN + C.BOLD))
    shell.info("Type 'help' for built-in commands. Type 'exit' to quit.")
    shell.info(f"State: {ROCKET_DIR}")
    shell.print(f"Current mode: {shell.mode}")
    shell.print("")
    setup_readline(shell)
    while shell.running:
        try:
            line = input(shell.prompt())
        except EOFError:
            print()
            break
        except KeyboardInterrupt:
            print()
            continue
        try:
            shell.execute(line)
        except KeyboardInterrupt:
            print()
            shell.warn("[interrupted]")
            continue
        except Exception as e:
            shell.error(f"[internal error] {e}")
    return 0


                                                                            
     
                                                                            

def main_gui(shell: Shell) -> None:
    import tkinter as tk
    from tkinter import ttk
    from tkinter.scrolledtext import ScrolledText

    class RocketGUI:
        def __init__(self, root: tk.Tk) -> None:
            self.root = root
            self.root.title(f"ROCKET SHELL {VERSION}")
            self.root.geometry("960x640")
            self.root.configure(bg="#111111")
            self.shell = shell
            self.shell.print = lambda s="": self._enqueue_tagged(s, "out")
            self.shell.info = lambda s="": self._enqueue_tagged(s, "info")
            self.shell.warn = lambda s="": self._enqueue_tagged(s, "warn")
            self.shell.error = lambda s="": self._enqueue_tagged(s, "err")
            self.shell.on_clear = self._on_clear
            self._out_q: "queue.Queue[Tuple[str, str]]" = queue.Queue()
            self._build_ui()
            self._poll_output()
            self._refresh_status()
            self._append(f"ROCKET SHELL v{VERSION}", "info")
            self._append("Type 'help' for built-in commands.", "info")

        def _build_ui(self) -> None:
            style = ttk.Style()
            try:
                style.theme_use("clam")
            except tk.TclError:
                pass
            style.configure("TCombobox",
                            fieldbackground="#222222",
                            background="#222222",
                            foreground="#dddddd")

            top = tk.Frame(self.root, bg="#111111")
            top.pack(side="top", fill="x", padx=8, pady=6)

            tk.Label(top, text="Mode:", bg="#111111", fg="#dddddd").pack(side="left")
            self.mode_var = tk.StringVar(value=self.shell.mode)
            combo = ttk.Combobox(top, textvariable=self.mode_var,
                                 values=["auto", "windows", "linux"],
                                 state="readonly", width=10)
            combo.pack(side="left", padx=6)
            combo.bind("<<ComboboxSelected>>", self._on_mode_change)

            self.wsl_label = tk.Label(top, text="WSL: ?", bg="#111111", fg="#dddddd")
            self.wsl_label.pack(side="left", padx=12)
            self.shell_label = tk.Label(top, text="shell: -", bg="#111111", fg="#dddddd")
            self.shell_label.pack(side="left", padx=12)
            self.cwd_label = tk.Label(top, text="cwd: -", bg="#111111", fg="#dddddd")
            self.cwd_label.pack(side="left", padx=12)

            tk.Button(top, text="Clear", command=self._on_clear,
                      bg="#222222", fg="#dddddd", activebackground="#333333",
                      relief="flat").pack(side="right")

            self.output = ScrolledText(self.root, wrap="word",
                                       bg="#0a0a0a", fg="#e0e0e0",
                                       insertbackground="#e0e0e0",
                                       font=("Consolas", 10), borderwidth=0)
            self.output.pack(side="top", fill="both", expand=True,
                             padx=8, pady=(0, 6))
            self.output.configure(state="disabled")
            self.output.tag_config("out", foreground="#e0e0e0")
            self.output.tag_config("info", foreground="#888888")
            self.output.tag_config("warn", foreground="#ffcc44")
            self.output.tag_config("err", foreground="#ff6666")
            self.output.tag_config("cmd", foreground="#4ea1ff")

            bottom = tk.Frame(self.root, bg="#111111")
            bottom.pack(side="bottom", fill="x", padx=8, pady=(0, 8))
            self.entry = tk.Entry(bottom, bg="#1a1a1a", fg="#e0e0e0",
                                  insertbackground="#e0e0e0", relief="flat",
                                  font=("Consolas", 11))
            self.entry.pack(side="left", fill="x", expand=True, ipady=4)
            self.entry.bind("<Return>", lambda e: self._on_run())
            tk.Button(bottom, text="Run", command=self._on_run,
                      bg="#2a6", fg="#ffffff", activebackground="#3b8",
                      relief="flat", width=8).pack(side="right", padx=(6, 0))

        def _refresh_status(self) -> None:
            avail = is_wsl_available()
            self.wsl_label.configure(text=f"WSL: {'yes' if avail else 'no'}",
                                     fg="#8f8" if avail else "#f88")

                                                             
            if self.mode_var.get() != self.shell.mode:
                self.mode_var.set(self.shell.mode)

            if self.shell.mode == "linux":
                sh = f"linux: {self.shell.config['linux_shell']}"
            elif self.shell.mode == "windows":
                sh = f"win: {self.shell.config['windows_shell']}"
            else:
                sh = (f"{self.shell.config['windows_shell']} / "
                      f"{self.shell.config['linux_shell']}")
            self.shell_label.configure(text=f"shell: {sh}")

            cwd = (self.shell.linux_cwd if self.shell.mode == "linux"
                   else self.shell.windows_cwd) or "-"
            self.cwd_label.configure(text=f"cwd: {cwd}")
            self.root.after(2000, self._refresh_status)

        def _on_mode_change(self, _event=None) -> None:
            mode = self.mode_var.get()
            self.shell.mode = mode
            self.shell.config["mode"] = mode
            self.shell.config.save()
            self._append(f"[mode set to: {mode}]", "info")

        def _on_clear(self) -> None:
            self.output.configure(state="normal")
            self.output.delete("1.0", "end")
            self.output.configure(state="disabled")

        def _on_run(self) -> None:
            cmd = self.entry.get().strip()
            if not cmd:
                return
            self.entry.delete(0, "end")
            self._append(f"{cmd}", "cmd")
            threading.Thread(target=self._run_command, args=(cmd,), daemon=True).start()

        def _run_command(self, cmd: str) -> None:
            try:
                self.shell.execute(cmd)
            except Exception as e:
                self._enqueue_tagged(f"[internal error] {e}", "err")

        def _enqueue_tagged(self, text: str = "", tag: str = "out") -> None:
            self._out_q.put((str(text), tag))

        def _append(self, text: str, tag: str = "out") -> None:
            self.output.configure(state="normal")
            self.output.insert("end", text + "\n", tag)
            self.output.see("end")
            self.output.configure(state="disabled")

        def _poll_output(self) -> None:
            try:
                while True:
                    item = self._out_q.get_nowait()
                    if isinstance(item, tuple) and len(item) == 2:
                        text, tag = item
                    else:
                        text, tag = str(item), "out"
                    self._append(text, tag)
            except queue.Empty:
                pass
            self.root.after(60, self._poll_output)

    root = tk.Tk()
    RocketGUI(root)
    root.mainloop()


                                                                            
          
                                                                            

def _run_selftest() -> int:
    checks: List[Tuple[str, bool]] = []

    def check(name: str, cond: bool) -> None:
        checks.append((name, bool(cond)))

                    
    check("parse empty -> None", parse("") is None)
    p = parse("dir /w")
    check("parse dir /w", p is not None and p.argv == ["dir", "/w"]
          and p.mode_override is None)
    p = parse('echo "hello world"')
    check("parse quoted", p is not None and p.argv == ["echo", "hello world"])
    p = parse("run win dir /w")
    check("parse run win", p is not None and p.mode_override == "windows"
          and p.argv == ["dir", "/w"])
    p = parse("run linux ls -la")
    check("parse run linux", p is not None and p.mode_override == "linux"
          and p.argv == ["ls", "-la"])

                   
    segs = split_chain("a && b || c ; d")
    ops = [op for op, _ in segs]
    check("split_chain ops", ops == ["", "&&", "||", ";"])
    check("split_chain bodies", [s for _, s in segs] == ["a", "b", "c", "d"])

                  
    check("pipe simple", has_toplevel_pipe("ls | grep x"))
    check("pipe |&", has_toplevel_pipe("ls |& grep x"))
    check("not pipe ||", not has_toplevel_pipe("a || b"))
    check("pipe in dq", not has_toplevel_pipe('echo "a | b"'))
    check("pipe in sq", not has_toplevel_pipe("echo 'a | b'"))

                   
    check("win C:\\Users", is_windows_path("C:\\Users"))
    check("win C:/Users", is_windows_path("C:/Users"))
    check("win C:", is_windows_path("C:"))
    check("not win //", not is_windows_path("//"))
    check("not win \\\\", not is_windows_path("\\\\"))
    check("win UNC", is_windows_path("//srv/share"))
    check("wsl /home", is_wsl_path("/home"))
    check("not wsl //x", not is_wsl_path("//x"))
    check("win_to_wsl C:\\Users", win_to_wsl("C:\\Users") == "/mnt/c/Users")
    check("win_to_wsl C:", win_to_wsl("C:") == "/mnt/c")
    check("wsl_to_win /mnt/d/x", wsl_to_win("/mnt/d/x") == "D:\\x")
    check("wsl_to_win /home stays", wsl_to_win("/home") == "/home")

                     
    check("resolve linux rel", resolve_linux_path("p", "/home/u", "/home/u") == "/home/u/p")
    check("resolve linux rel-home", resolve_linux_path("p", "", "/home/u") == "/home/u/p")
    check("resolve linux tilde", resolve_linux_path("~/w", "/etc", "/home/u") == "/home/u/w")
    check("resolve linux ..", resolve_linux_path("..", "/home/u/p", "/home/u") == "/home/u")
    check("resolve linux empty", resolve_linux_path("", "/etc", "/home/u") == "")

                 
    os.environ["_ROCKET_TEST_VAR"] = "42"
    check("expand $VAR", expand_env("x=$_ROCKET_TEST_VAR") == "x=42")
    check("expand ${VAR}", expand_env("x=${_ROCKET_TEST_VAR}") == "x=42")
    check("expand %VAR%", expand_env("x=%_ROCKET_TEST_VAR%") == "x=42")
    check("expand session env", expand_env("x=$FOO", {"FOO": "bar"}) == "x=bar")
    del os.environ["_ROCKET_TEST_VAR"]

                                  
    check("cmd_quote bare", _cmd_quote("dir") == "dir")
    check("cmd_quote space", _cmd_quote("hello world") == '"hello world"')
    check("cmd_quote pipe", _cmd_quote("a|b") == '"a|b"')
    check("cmd_quote amp", _cmd_quote("a&b") == '"a&b"')
    check("cmd_quote redirect", _cmd_quote("a>b") == '"a>b"')
    check("cmd_quote inner dq", _cmd_quote('a"b') == '"a""b"')
    check("cmd_quote empty", _cmd_quote("") == '""')

                                 
    check("ps bare", _ps_quote("Get-Process") == "Get-Process")
    check("ps $env bare", _ps_quote("$env:PATH") == "$env:PATH")
    check("ps $_ bare", _ps_quote("$_") == "$_")
    check("ps $_ with space", _ps_quote("$_ -eq 5") == '"$_ -eq 5"')
    check("ps $? with space", _ps_quote("$? -eq 0") == '"$? -eq 0"')
    check("ps $5 with space", _ps_quote("$5 -gt 0") == '"$5 -gt 0"')
    check("ps literal string", _ps_quote("hello world") == "'hello world'")
    check("ps expands inside dq", _ps_quote("hello $env:USER") == '"hello $env:USER"')
    check("ps escape dq", _ps_quote('a "b" $x') == '"a `"b`" $x"')
    check("ps empty", _ps_quote("") == "''")

                      
    check("sh bare", _sh_quote("ls") == "ls")
    check("sh space", _sh_quote("hello world") == "'hello world'")
    check("sh single-quote", _sh_quote("it's") == "'it'\\''s'")
    check("sh empty", _sh_quote("") == "''")

                     
    a = Aliases.__new__(Aliases)
    a._map = {"ll": "ls -la"}
    check("alias expand", a.expand("ll") == "ls -la")
    check("alias expand+args", a.expand("ll /tmp") == "ls -la /tmp")
    check("alias no match", a.expand("ls") == "ls")

                                                              
    cmd_direct = _build_linux_cmd(["ls", "-la"], "", "wsl", "", "direct")
    check("build_linux direct", cmd_direct == ["wsl", "ls", "-la"])
    cmd_bash = _build_linux_cmd(["ls", "-la"], "", "wsl", "", "bash")
    check("build_linux bash", cmd_bash == ["wsl", "bash", "-lc", "ls -la"])
    cmd_sh = _build_linux_cmd(["ls", "-la"], "", "wsl", "", "sh")
    check("build_linux sh", cmd_sh == ["wsl", "sh", "-c", "ls -la"])
    cmd_cd = _build_linux_cmd(["ls"], "/home/u", "wsl", "Ubuntu", "direct")
    check("build_linux distro+cwd",
          cmd_cd == ["wsl", "-d", "Ubuntu", "--cd", "/home/u", "ls"])

                                                 
    win_cmd = _build_windows_cmd(["echo", "a|b"], "cmd")
    check("build_win cmd quotes pipe",
          win_cmd == ["cmd", "/d", "/c", "chcp 65001 >nul & echo \"a|b\""])
    win_ps = _build_windows_cmd(["Get-Process"], "powershell")
    check("build_win ps", win_ps[0] == "powershell" and "Get-Process" in win_ps[-1])

                     
    ok = 0
    for name, passed in checks:
        mark = "PASS" if passed else "FAIL"
        print(f"[{mark}] {name}")
        if passed:
            ok += 1
    total = len(checks)
    print("")
    print(f"{ok}/{total} passed")
    return 0 if ok == total else 1


                                                                            
       
                                                                            

def _parse_env_flag(values: List[str]) -> Dict[str, str]:
    out: Dict[str, str] = {}
    for item in values or []:
        if "=" not in item:
            continue
        k, _, v = item.partition("=")
        k = k.strip()
        if k:
            out[k] = v
    return out


def main() -> int:
    ap = argparse.ArgumentParser(
        prog="rocket",
        description=f"ROCKET SHELL {VERSION} — unified Windows/Linux command line",
    )
    ap.add_argument("--gui", action="store_true", help="launch Tkinter GUI")
    ap.add_argument("-c", "--command", metavar="CMD", help="run one command and exit")
    ap.add_argument("--color", choices=["auto", "always", "never"], default=None,
                    help="ANSI color mode (default: config value, usually 'auto')")
    ap.add_argument("--no-color", action="store_true",
                    help="shortcut for --color never")
    ap.add_argument("--timeout", type=int, default=0,
                    help="default command timeout in seconds (0 = none)")
    ap.add_argument("--env", action="append", default=[], metavar="NAME=value",
                    help="set a session env var (repeatable)")
    ap.add_argument("--selftest", action="store_true",
                    help="run embedded self-tests and exit")
    ap.add_argument("--version", action="version", version=f"ROCKET SHELL {VERSION}")
    args = ap.parse_args()

    if args.selftest:
        return _run_selftest()

    color_mode = args.color
    if args.no_color:
        color_mode = "never"

    shell = Shell(
        timeout=args.timeout,
        color_mode=color_mode,
        extra_env=_parse_env_flag(args.env),
    )
    if args.gui:
        main_gui(shell)
        return 0
    return main_cli(shell, run_cmd=args.command)


if __name__ == "__main__":
    try:
        sys.exit(main())
    except KeyboardInterrupt:
        sys.exit(130)
