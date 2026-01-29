# Function Scope Comparison: rdpClientConInit vs rdpLoadLayout

## Question / 質問
Are the scopes of rdpClientConInit and rdpLoadLayout the same?

rdpClientConInitとrdpLoadLayoutのスコープは同じですか？

## Answer / 答え
**No, the scopes are different.**

**いいえ、スコープは異なります。**

---

## Detailed Analysis / 詳細分析

### rdpClientConInit

**Location / 場所:**
- **File:** `module/rdpClientCon.c` (line 1794)
- **Header:** `module/rdpClientCon.h` (line 155)

**Declaration / 宣言:**
```c
extern _X_EXPORT int
rdpClientConInit(rdpPtr dev);
```

**Definition / 定義:**
```c
int
rdpClientConInit(rdpPtr dev)
{
    // ... implementation
}
```

**Scope / スコープ:**
- ✅ **Non-static** (非静的)
- ✅ **Globally visible** (グローバルに可視)
- ✅ **Exported** (`_X_EXPORT` macro makes it visible to other modules)
- ✅ **External linkage** (外部リンケージ)
- ✅ **Declared in header file** (`rdpClientCon.h`)

**Visibility / 可視性:**
This function can be called from **any file** that includes `rdpClientCon.h`. It is part of the public API of the rdpClientCon module.

この関数は`rdpClientCon.h`をインクルードする**任意のファイル**から呼び出すことができます。rdpClientConモジュールのパブリックAPIの一部です。

---

### rdpLoadLayout

**Location / 場所:**
- **File:** `xrdpkeyb/rdpKeyboard.c` (line 575)
- **Header:** None (no external declaration)

**Declaration / 宣言:**
```c
static int
rdpLoadLayout(rdpKeyboard *keyboard, struct xrdp_client_info *client_info);
```

**Definition / 定義:**
```c
static int
rdpLoadLayout(rdpKeyboard *keyboard, struct xrdp_client_info *client_info)
{
    // ... implementation
}
```

**Scope / スコープ:**
- ✅ **Static** (静的)
- ❌ **File-local only** (ファイルローカルのみ)
- ❌ **Not exported** (エクスポートされていない)
- ❌ **Internal linkage** (内部リンケージ)
- ❌ **Not declared in any header file** (ヘッダーファイルに宣言されていない)

**Visibility / 可視性:**
This function can **only** be called from within `rdpKeyboard.c`. It is a private implementation detail of the keyboard module.

この関数は`rdpKeyboard.c`の**内部からのみ**呼び出すことができます。キーボードモジュールのプライベートな実装詳細です。

---

## Comparison Table / 比較表

| Property / 特性 | rdpClientConInit | rdpLoadLayout |
|----------------|------------------|---------------|
| **Scope / スコープ** | Global / グローバル | File-local / ファイルローカル |
| **Storage class / ストレージクラス** | (none/default) | `static` |
| **Linkage / リンケージ** | External / 外部 | Internal / 内部 |
| **Visibility / 可視性** | All modules / 全モジュール | Single file / 単一ファイル |
| **Header declaration / ヘッダー宣言** | Yes (`rdpClientCon.h`) | No |
| **Export macro / エクスポートマクロ** | `_X_EXPORT` | None / なし |
| **File / ファイル** | `module/rdpClientCon.c` | `xrdpkeyb/rdpKeyboard.c` |
| **Public API / パブリックAPI** | Yes / はい | No / いいえ |

---

## Usage / 使用箇所

### rdpClientConInit - Called from external modules / 外部モジュールから呼び出される

1. **`module/rdpMain.c`** - During screen initialization
   ```c
   rdpClientConInit(dev);
   ```

2. **`xrdpdev/xrdpdev.c`** - From the device driver
   ```c
   rdpClientConInit(dev);
   ```

### rdpLoadLayout - Called only within rdpKeyboard.c / rdpKeyboard.c内部からのみ呼び出される

1. **Line 300** - In `rdpInputKeyboard()` message handler
   ```c
   rdpLoadLayout(keyboard, (struct xrdp_client_info *) param1);
   ```

2. **Line 412** - In `rdpkeybControl()` during `DEVICE_INIT`
   ```c
   rdpLoadLayout(&(dev->keyboard), NULL);
   ```

---

## Summary / まとめ

**English:**
The scopes are **completely different**:
- `rdpClientConInit` has **global scope** with external linkage - it's a public API function that can be called from other compilation units
- `rdpLoadLayout` has **file scope** with internal linkage - it's a private helper function that can only be called within `rdpKeyboard.c`

**Japanese / 日本語:**
スコープは**完全に異なります**：
- `rdpClientConInit`は**グローバルスコープ**で外部リンケージを持ちます - 他のコンパイル単位から呼び出せるパブリックAPI関数です
- `rdpLoadLayout`は**ファイルスコープ**で内部リンケージを持ちます - `rdpKeyboard.c`内部からのみ呼び出せるプライベートヘルパー関数です

---

## Design Rationale / 設計の理論的根拠

**rdpClientConInit (Public / パブリック):**
- Needs to be called during module initialization from different parts of the codebase
- Part of the client connection subsystem's public interface
- Must be accessible to the main driver and device initialization code

初期化時にコードベースの異なる部分から呼び出される必要があります。クライアント接続サブシステムのパブリックインターフェースの一部です。メインドライバーとデバイス初期化コードからアクセス可能である必要があります。

**rdpLoadLayout (Private / プライベート):**
- Implementation detail of keyboard layout loading
- Only needs to be called from within the keyboard module
- Keeping it `static` provides encapsulation and prevents external dependencies

キーボードレイアウト読み込みの実装詳細です。キーボードモジュール内からのみ呼び出される必要があります。`static`にすることでカプセル化を提供し、外部依存を防ぎます。
