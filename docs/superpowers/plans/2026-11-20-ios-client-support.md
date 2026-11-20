# iOS Client Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add full iOS client emulation support to hh-applicant-tool, allowing users to switch between Android and iOS client modes via `set-client` command.

**Architecture:** 
- Add `client` config key in `[general]` section of `config.toml` (`"android"` or `"ios"`)
- Create `generate_ios_useragent()` function mirroring Android implementation
- Add `IOS_WEBVIEW_USERAGENT` constant for iOS WebView capcha handling
- Modify `api_client` property to use iOS credentials when `client = "ios"`
- Modify `webview_session` to use iOS UA when in iOS mode
- Add `set-client` operation to switch clients and clear tokens
- Update `authorize.py` to use `hhios://oauthresponse` redirect scheme for iOS mode

**Tech Stack:** Python 3.12+, requests, tomllib/tomli_w for config

**Spec:** This plan (design decisions captured in grilling session)

## Global Constraints

- Default client when not specified: `"android"` (backward compatibility)
- Token file (`token.json`) must be cleared on client switch (OAuth tokens bound to client_id)
- iOS redirect scheme: `hhios://oauthresponse` (from Info.plist)
- iOS client_id: `L41G2HHQQD5JT8Q3QOR3B3MHHD2876TMGKQFA2VQ8USJC1RLR7094USQ57JL32O8`
- iOS client_secret: `KMVP77JPKV64D1CIEHV579A5BQ4P6V7NQ0TQHBDR6DK6SA8SSBQDKHPR54K48DGS`
- iOS UA format: `ru.hh.iphone/7.{minor}.{patch}, Device: {model}, iOS: {version} (UUID: {uuid})`
- iOS WebView UA: `Mozilla/5.0 (iPhone; CPU iPhone OS {major}_{minor} like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148`

## Review Focus

1. **Empty client config**: When `[general] client` is missing, default to `"android"` — existing users must not break
2. **Token mismatch after switch**: After `set-client ios`, old Android tokens are cleared; user must re-authorize
3. **OAuth redirect scheme**: iOS mode must use `hhios://oauthresponse`, not `hhandroid://oauthresponse`
4. **WebView UA consistency**: Capcha session in iOS mode must use iOS WebKit UA, not Android WebView
5. **API client credentials**: `api_client` must use `IOS_CLIENT_ID/SECRET` in iOS mode, not Android

---

### Task 1: iOS User-Agent Generator

**Files:**
- Modify: `src/hh_applicant_tool/utils/user_agent.py`
- Test: (no tests — project has no test suite)

**Interfaces:**
- Produces: `generate_ios_useragent() -> str`

- [ ] **Step 1: Add `generate_ios_useragent()` function**

Add to `user_agent.py` after `generate_android_useragent()`:

```python
def generate_ios_useragent() -> str:
    """Generates iOS App User-Agent"""
    model = "".join(
        random.choice(string.ascii_uppercase + string.digits) for _ in range(10)
    )
    minor = random.randint(100, 150)
    patch = random.randint(10000, 15000)
    ios = random.randint(16, 18)
    return (
        f"ru.hh.iphone/7.{minor}.{patch}, Device: {model}, "
        f"iOS: {ios} (UUID: {uuid.uuid4()})"
    )
```

- [ ] **Step 2: Verify syntax**

Run: `python -m py_compile src/hh_applicant_tool/utils/user_agent.py`
Expected: PASS (no output)

- [ ] **Step 3: Commit**

```bash
git add src/hh_applicant_tool/utils/user_agent.py
git commit -m "feat: add generate_ios_useragent() for iOS client emulation"
```

---

### Task 2: iOS WebView User-Agent Constant

**Files:**
- Modify: `src/hh_applicant_tool/constants.py`

**Interfaces:**
- Produces: `IOS_WEBVIEW_USERAGENT: str`

- [ ] **Step 1: Add iOS WebView UA constant**

Add after `ANDROID_WEBVIEW_USERAGENT` in `constants.py`:

```python
IOS_WEBVIEW_USERAGENT = "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148"
```

- [ ] **Step 2: Verify syntax**

Run: `python -m py_compile src/hh_applicant_tool/constants.py`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add src/hh_applicant_tool/constants.py
git commit -m "feat: add IOS_WEBVIEW_USERAGENT constant"
```

---

### Task 3: Client Config in tool.py

**Files:**
- Modify: `src/hh_applicant_tool/tool.py`

**Interfaces:**
- Consumes: `IOS_CLIENT_ID`, `IOS_CLIENT_SECRET`, `IOS_WEBVIEW_USERAGENT` from `constants`
- Consumes: `generate_ios_useragent()` from `utils.user_agent`
- Produces: `client` property on `HHApplicantTool` returning `"android"` or `"ios"`

- [ ] **Step 1: Import iOS constants and function**

In imports section (after line 48), add:
```python
from .constants import (
    ...
    IOS_CLIENT_ID,
    IOS_CLIENT_SECRET,
    IOS_WEBVIEW_USERAGENT,
    ...
)
```

And after line 80:
```python
from .utils.user_agent import generate_android_useragent, generate_ios_useragent
```

- [ ] **Step 2: Add `client` field to `BaseNamespace`**

In `BaseNamespace` class (around line 150), add:
```python
client: Literal["android", "ios"]
```

- [ ] **Step 3: Add `client` field to `HHApplicantTool` dataclass**

In `HHApplicantTool` (around line 184), add:
```python
client: Literal["android", "ios"] | None = None
```

- [ ] **Step 4: Add `--client` CLI argument**

In `_create_parser` (after line 228), add:
```python
parser.add_argument(
    "--client",
    choices=["android", "ios"],
    help="Клиент для эмуляции: android или ios. По умолчанию android",
)
```

- [ ] **Step 5: Add `client` property to `HHApplicantTool`**

After `general_config` property (around line 338), add:
```python
@property
def client(self) -> Literal["android", "ios"]:
    """Возвращает текущий клиент: 'android' или 'ios'."""
    if self._client is not None:
        return self._client
    return self.general_config.get("client", "android")

@client.setter
def client(self, value: Literal["android", "ios"]) -> None:
    self._client = value
```

Add `_client: Literal["android", "ios"] | None = None` to `HHApplicantTool.__post_init__`.

- [ ] **Step 6: Modify `api_client` property to use iOS credentials**

In `api_client` cached_property (lines 433-448), replace:
```python
client_id=ANDROID_CLIENT_ID,
client_secret=ANDROID_CLIENT_SECRET,
user_agent=generate_android_useragent(),
```

With:
```python
client_id=IOS_CLIENT_ID if self.client == "ios" else ANDROID_CLIENT_ID,
client_secret=IOS_CLIENT_SECRET if self.client == "ios" else ANDROID_CLIENT_SECRET,
user_agent=generate_ios_useragent() if self.client == "ios" else generate_android_useragent(),
```

- [ ] **Step 7: Modify `webview_session` to use iOS UA**

In `webview_session` cached_property (lines 716-720), replace:
```python
return self._create_session(
    "Android WebView", user_agent=ANDROID_WEBVIEW_USERAGENT
)
```

With:
```python
if self.client == "ios":
    return self._create_session(
        "iOS WebView", user_agent=IOS_WEBVIEW_USERAGENT
    )
return self._create_session(
    "Android WebView", user_agent=ANDROID_WEBVIEW_USERAGENT
)
```

- [ ] **Step 8: Verify syntax**

Run: `python -m py_compile src/hh_applicant_tool/tool.py`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add src/hh_applicant_tool/tool.py
git commit -m "feat: add client switching (android/ios) to tool.py"
```

---

### Task 4: iOS OAuth Redirect in authorize.py

**Files:**
- Modify: `src/hh_applicant_tool/operations/authorize.py`

**Interfaces:**
- Consumes: `HHApplicantTool.client` property
- Produces: OAuth flow using `hhios://oauthresponse` in iOS mode

- [ ] **Step 1: Import iOS constant**

At top of file (after line 16), add:
```python
from ..constants import IOS_WEBVIEW_USERAGENT
```

- [ ] **Step 2: Modify redirect URL validation for iOS mode**

In `run` method (around line 135-140), replace:
```python
if sp.scheme != "hhandroid" or sp.netloc != "oauthresponse":
    raise OperationError(
        f"Неправильный формат URI для получения кода авторизации: {location}"
    )
```

With:
```python
expected_scheme = "hhios" if self.tool.client == "ios" else "hhandroid"
if sp.scheme != expected_scheme or sp.netloc != "oauthresponse":
    raise OperationError(
        f"Неправильный формат URI для получения кода авторизации: {location}"
    )
```

- [ ] **Step 3: Verify syntax**

Run: `python -m py_compile src/hh_applicant_tool/operations/authorize.py`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add src/hh_applicant_tool/operations/authorize.py
git commit -m "feat: use hhios://oauthresponse redirect for iOS client"
```

---

### Task 5: set-client Operation

**Files:**
- Create: `src/hh_applicant_tool/operations/set_client.py`

**Interfaces:**
- Consumes: `HHApplicantTool.config`, `HHApplicantTool.token_config`
- Produces: `set-client <android|ios>` CLI command

- [ ] **Step 1: Create `set_client.py` operation**

```python
from __future__ import annotations

import argparse
import logging

from ..tool import BaseNamespace, BaseOperation

if __name__ == "hh_applicant_tool.operations.set_client":  # TYPE_CHECKING guard
    from ..tool import HHApplicantTool


logger = logging.getLogger(__name__.split(".")[0])


class Namespace(BaseNamespace):
    client: str


class Operation(BaseOperation):
    """
    Переключить клиент для эмуляции: android или ios.
    
    При переключении токены (token.json) очищаются, так как OAuth-токены
    привязаны к client_id. После переключения потребуется повторная авторизация.
    
    Примеры:
        hh-applicant-tool set-client ios
        hh-applicant-tool set-client android
    """

    def setup_parser(self, parser: argparse.ArgumentParser) -> None:
        parser.add_argument(
            "client",
            choices=["android", "ios"],
            help="Клиент для эмуляции",
        )

    def run(self, tool: HHApplicantTool, args: Namespace) -> int | None:
        config = tool.config
        config.setdefault("general", {})
        
        old_client = config.get("general", {}).get("client", "android")
        new_client = args.client
        
        if old_client == new_client:
            logger.info("Уже используется клиент: %s", new_client)
            return 0
        
        # Очищаем токены при смене клиента
        if tool.token_config.exists():
            logger.info("Очистка токенов (привязаны к client_id)...")
            tool.token_config.unlink()
        
        # Сохраняем новый клиент
        config["general"]["client"] = new_client
        config.save()
        
        print(f"✅ Клиент переключен: {old_client} → {new_client}")
        print("📝 Токены очищены. Выполните авторизацию заново:")
        print(f"   hh-applicant-tool authorize <телефон|email>")
        
        return 0
```

- [ ] **Step 2: Verify syntax**

Run: `python -m py_compile src/hh_applicant_tool/operations/set_client.py`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add src/hh_applicant_tool/operations/set_client.py
git commit -m "feat: add set-client operation to switch android/ios"
```

---

### Task 6: Integration Test (Manual)

**Files:**
- None (manual verification)

**Interfaces:**
- Consumes: All previous tasks

- [ ] **Step 1: Test default behavior (no client specified)**

Run: `python -m hh_applicant_tool config -p`
Expected: Shows config path

Run: `python -m hh_applicant_tool config general.client`
Expected: Empty or not set (defaults to android)

- [ ] **Step 2: Test set-client command**

Run: `python -m hh_applicant_tool set-client ios`
Expected: 
```
✅ Клиент переключен: android → ios
📝 Токены очищены. Выполните авторизацию заново:
   hh-applicant-tool authorize <телефон|email>
```

- [ ] **Step 3: Verify config updated**

Run: `python -m hh_applicant_tool config general.client`
Expected: `ios`

- [ ] **Step 4: Switch back to android**

Run: `python -m hh_applicant_tool set-client android`
Expected: Similar success message

Run: `python -m hh_applicant_tool config general.client`
Expected: `android`

- [ ] **Step 5: Test help**

Run: `python -m hh_applicant_tool set-client -h`
Expected: Shows help with `client` argument choices

Run: `python -m hh_applicant_tool -h`
Expected: Shows `--client` global flag

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "test: manual integration test passed"
```

---

## Review Focus (Post-Implementation)

After completing all tasks, verify:

1. **Backward compatibility**: User without `client` in config runs commands as before (android)
2. **Token clearance**: `token.json` is deleted on client switch
3. **OAuth flow**: iOS mode uses `hhios://oauthresponse` in authorize.py
4. **API credentials**: `api_client` uses iOS client_id/secret in iOS mode
5. **WebView UA**: Capcha uses iOS WebKit UA in iOS mode
