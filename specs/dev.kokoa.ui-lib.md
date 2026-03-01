# kokoa's Editor UI Library - 仕様書

## 概要

Unityエディタ拡張の設定UIやコンポーネントInspectorに使える、IMGUIベースのUIライブラリ。
kokoa拡張群（EditorWallpaper, KokoaToggleButtons, MeshColorTool, AnimationLabelSimplify等）の
共通UIパターンを抽出・統一し、新しい拡張でも少ないコードでリッチなUIを構築できるようにする。

## パッケージ情報

| 項目 | 値 |
|------|-----|
| パッケージ名 | `dev.kokoa.ui-lib` |
| 表示名 | kokoa's Editor UI Library |
| 名前空間 | `Kokoa.UILib` |
| 対象Unity | 2022.3+ |
| UIベース | IMGUI（EditorGUILayout / GUILayout） |
| ライセンス | MIT |
| vpmDependencies | なし |

## 設計原則

- **Clarity** - 明確な視覚階層、一貫したスペーシング
- **Consistency** - 全kokoa拡張で統一されたスタイル・カラー
- **Simplicity** - 1行で使える簡潔なAPI。既存のEditorGUILayoutの知識をそのまま活かせる
- **Feedback** - 全フィールドが変更検知をboolで返す。通知UIで状態を可視化

## クラスプレフィックス

すべてのpublicクラスに `K` プレフィックスを使用。

```
KStyle, KSection, KLayout, KField, KButton, KPrefs, KPrefsJson, KLocale, KWindow, KList, KNotice
```

---

## コンポーネント一覧

### 1. KStyle - テーマ・スタイル定義

全コンポーネントが参照する色・スペーシング・GUIStyleの一元管理。
UnityのPro(ダーク)/Personal(ライト)スキンに自動対応する。

```csharp
public static class KStyle
{
    // === カラー ===

    /// テーマアクセント色。Unityエディタに馴染むティール系をデフォルトとする。
    /// ダーク/ライトテーマで自動調整。
    public static Color AccentColor { get; }

    /// 成功状態を示す色（緑系）
    public static Color SuccessColor { get; }

    /// 警告状態を示す色（黄系）
    public static Color WarningColor { get; }

    /// エラー状態を示す色（赤系）
    public static Color ErrorColor { get; }

    /// 控えめなテキスト色（グレー系）
    public static Color MutedTextColor { get; }

    // === スペーシング ===

    /// 小さい余白（2px）
    public const float SpacingSmall = 2f;

    /// 標準余白（4px）
    public const float SpacingMedium = 4f;

    /// 大きい余白（8px）
    public const float SpacingLarge = 8f;

    // === GUIStyle ===

    /// セクション見出し用スタイル（太字、12pt）
    public static GUIStyle HeaderStyle { get; }

    /// 小見出し用スタイル（太字、通常サイズ）
    public static GUIStyle SubHeaderStyle { get; }

    /// セクションボックス用スタイル（パディング付きbox）
    public static GUIStyle BoxStyle { get; }

    // === ヘルパー ===

    /// 水平区切り線を描画する
    public static void DrawSeparator();

    /// ダークテーマかどうか
    public static bool IsDarkTheme { get; }
}
```

**設計ノート:**
- カラー値はランタイムで `EditorGUIUtility.isProSkin` を判定して自動切替
- GUIStyleは遅延初期化（初回アクセス時に生成）
- デフォルトのアクセント色は、既存拡張で使用されている `Color.cyan` にUnityエディタのトーンを合わせた色
  - ダーク: `new Color(0.4f, 0.8f, 0.9f)` 付近
  - ライト: `new Color(0.2f, 0.6f, 0.7f)` 付近

---

### 2. KSection - セクション・グループ描画

EditorWindowやInspectorで一貫したセクションレイアウトを提供する。
`Draw*` メソッド分離パターン（EditorWallpaperで採用）を促進する。

```csharp
public static class KSection
{
    /// ラベル付きセクション。見出し + コンテンツ。
    /// 前後に適切なスペーシングを自動挿入。
    public static void Draw(string label, Action drawContent);

    /// ボックスで囲まれたセクション。
    /// KStyle.BoxStyle を使用してコンテンツを視覚的にグループ化。
    public static void DrawBoxed(string label, Action drawContent);

    /// 折りたたみ可能なセクション。
    /// foldout状態をref引数で管理。閉じている間はコンテンツを描画しない。
    public static void DrawFoldout(string label, ref bool foldout, Action drawContent);

    /// トグル付き無効化セクション。
    /// enabledがfalseの間、コンテンツはグレーアウトされて操作不可になる。
    /// 戻り値: enabled値が変わった場合 true。
    public static bool DrawToggleSection(string label, ref bool enabled, Action drawContent);
}
```

**使用例:**
```csharp
// EditorWallpaper の DrawOverlaySection() に相当
KSection.DrawBoxed("オーバーレイ設定", () =>
{
    KField.ColorField("色", ref overlayColor);
    KField.Slider("透明度", ref overlayAlpha, 0f, 1f);
});

// KokoaToggleButtons の 基本設定 に相当
KSection.DrawToggleSection("トグルボタンを有効にする", ref isEnabled, () =>
{
    KField.Toggle("EditorOnlyボタンを表示", ref showEditorOnly);
    KField.Toggle("Activeボタンを表示", ref showActive);
});
```

---

### 3. KLayout - レイアウトスコープ

Begin/End ペアを `using` ステートメントで安全に管理する。
閉じ忘れによるレイアウトエラーを防止する。

```csharp
public static class KLayout
{
    /// 水平レイアウト
    public static HorizontalScope Horizontal(params GUILayoutOption[] options);

    /// 垂直レイアウト
    public static VerticalScope Vertical(params GUILayoutOption[] options);

    /// スタイル付き垂直レイアウト
    public static VerticalScope Vertical(GUIStyle style, params GUILayoutOption[] options);

    /// インデントスコープ（デフォルト1レベル）
    public static IndentScope Indent(int level = 1);

    /// 無効化スコープ
    public static DisabledScope Disabled(bool disabled);

    /// スクロールビュースコープ
    public static ScrollScope Scroll(ref Vector2 scrollPos, params GUILayoutOption[] options);

    /// 変更検知スコープ
    public static ChangeCheckScope ChangeCheck();

    // --- Scope構造体 ---
    // すべて IDisposable を実装した struct
    public struct HorizontalScope : IDisposable { ... }
    public struct VerticalScope : IDisposable { ... }
    public struct IndentScope : IDisposable { ... }
    public struct DisabledScope : IDisposable { ... }
    public struct ScrollScope : IDisposable { ... }
    public struct ChangeCheckScope : IDisposable
    {
        /// スコープ内で変更があったかどうか（Dispose後に確定）
        public bool Changed { get; }
    }
}
```

**使用例:**
```csharp
using (KLayout.Horizontal())
{
    GUILayout.Label("名前:");
    KField.TextField("", ref name);
}

using (KLayout.Disabled(!isEnabled))
{
    KField.IntSlider("オフセット", ref offset, 0, 200);
}
```

**設計ノート:**
- すべてstruct実装でGCアロケーションを回避
- `ChangeCheckScope.Changed` は `Dispose()` 実行後に値が確定する点に注意（ドキュメントで明記）

---

### 4. KField - フィールド描画

よく使うEditorGUILayoutフィールドをラップし、全て変更検知付き（bool戻り値）にする。
`BeginChangeCheck/EndChangeCheck` のボイラープレートを排除。

```csharp
public static class KField
{
    // === 基本フィールド ===
    // すべて: 変更があった場合 true を返す

    public static bool Toggle(string label, ref bool value);
    public static bool ToggleWithHelp(string label, ref bool value, string helpText);
    public static bool IntSlider(string label, ref int value, int min, int max);
    public static bool Slider(string label, ref float value, float min, float max);
    public static bool TextField(string label, ref string value);
    public static bool IntField(string label, ref int value);
    public static bool FloatField(string label, ref float value);
    public static bool Vector3Field(string label, ref Vector3 value);
    public static bool ColorField(string label, ref Color value);
    public static bool EnumPopup<T>(string label, ref T value) where T : Enum;
    public static bool ObjectField<T>(string label, ref T value, bool allowSceneObjects = true) where T : UnityEngine.Object;

    // === KPrefs連携 ===
    // KPrefsオブジェクトを直接渡すと、ラベル・値読込・描画・値保存を1行で完結

    public static bool Toggle(KPrefs<bool> pref);
    public static bool IntSlider(KPrefs<int> pref, int min, int max);
    public static bool Slider(KPrefs<float> pref, float min, float max);
    public static bool TextField(KPrefs<string> pref);
}
```

**使用例:**
```csharp
// Before (素のIMGUI)
EditorGUI.BeginChangeCheck();
showActive = EditorGUILayout.Toggle("Activeボタンを表示", showActive);
if (EditorGUI.EndChangeCheck()) { SavePreferences(); }

// After (KField)
if (KField.Toggle("Activeボタンを表示", ref showActive)) { SavePreferences(); }

// KPrefs連携なら保存も自動
KField.Toggle(showActivePref); // 1行で描画+保存
```

---

### 5. KButton - ボタン

確認ダイアログ付き、色付き等のボタンバリエーション。

```csharp
public static class KButton
{
    /// 通常のボタン
    public static bool Draw(string label, params GUILayoutOption[] options);

    /// 確認ダイアログ付きボタン。
    /// ボタンクリック後にダイアログを表示し、OKの場合のみtrueを返す。
    public static bool DrawWithConfirm(
        string label,
        string dialogTitle,
        string dialogMessage,
        string ok = "はい",
        string cancel = "いいえ");

    /// 色付きボタン。GUI.backgroundColorを一時的に変更して描画。
    public static bool DrawColored(string label, Color color, params GUILayoutOption[] options);

    /// ミニボタン（EditorStyles.miniButton使用）
    public static bool DrawMini(string label, params GUILayoutOption[] options);

    /// 横並びボタングループ。
    /// 戻り値: クリックされたボタンのインデックス（0始まり）。押されなければ -1。
    public static int DrawGroup(params string[] labels);
}
```

**使用例:**
```csharp
// KokoaToggleSettings の リセットボタン に相当
if (KButton.DrawWithConfirm("設定をリセット", "設定のリセット", "すべての設定をデフォルトに戻しますか？"))
{
    ResetToDefaults();
}

// MeshColorTool の 削除ボタン に相当
if (KButton.DrawColored("選択を削除", KStyle.ErrorColor))
{
    DeleteSelection();
}
```

---

### 6. KPrefs - 設定保存（EditorPrefs）

EditorPrefsの型安全ラッパー。宣言的に設定項目を定義し、キャッシュ付きで効率的にアクセス。

```csharp
public class KPrefs<T>
{
    /// コンストラクタ
    /// key: EditorPrefsのキー文字列
    /// label: UI表示用ラベル（KFieldとの連携で使用）
    /// defaultValue: デフォルト値
    public KPrefs(string key, string label, T defaultValue);

    public string Key { get; }
    public string Label { get; }
    public T DefaultValue { get; }

    /// 値の取得/設定。初回アクセス時にEditorPrefsから読み込みキャッシュ。
    /// setterは即座にEditorPrefsに保存。
    public T Value { get; set; }

    /// デフォルト値にリセット
    public void Reset();

    /// キャッシュを破棄（次回アクセス時に再読込）
    public void Invalidate();
}
```

**対応型:** `bool`, `int`, `float`, `string`

**使用例:**
```csharp
// KokoaToggleSettings 相当の定義
static class MyToolPrefs
{
    public static readonly KPrefs<bool> Enabled = new("MyTool_Enabled", "有効", true);
    public static readonly KPrefs<bool> ShowEditorOnly = new("MyTool_ShowEditorOnly", "EditorOnlyボタンを表示", true);
    public static readonly KPrefs<int> ButtonOffset = new("MyTool_ButtonOffset", "ボタンの右端からの距離", 80);
    public static readonly KPrefs<float> InactiveAlpha = new("MyTool_InactiveAlpha", "非アクティブボタンの透明度", 0.7f);

    public static void ResetAll()
    {
        Enabled.Reset();
        ShowEditorOnly.Reset();
        ButtonOffset.Reset();
        InactiveAlpha.Reset();
    }
}

// 使用（ロジック側）
if (MyToolPrefs.Enabled.Value) { ... }

// 使用（UI側）
KField.Toggle(MyToolPrefs.Enabled);
KField.IntSlider(MyToolPrefs.ButtonOffset, 0, 200);
```

---

### 7. KPrefsJson - 設定保存（JSONファイル）

プロジェクト単位の設定をJSONファイルで保存する。
EditorWallpaperの `ProjectSettings/EditorBackgroundSettings.json` パターンを汎用化。

```csharp
public abstract class KPrefsJson<T> where T : KPrefsJson<T>, new()
{
    /// JSONファイルの保存先パス。サブクラスで指定する。
    /// 例: "ProjectSettings/MyToolSettings.json"
    protected abstract string FilePath { get; }

    /// シングルトンインスタンス。初回アクセス時にファイルから読み込み。
    public static T Instance { get; }

    /// ファイルから設定を読み込む。ファイルがなければデフォルト値で新規作成。
    public void Load();

    /// 現在の値をJSONファイルに保存する。
    public void Save();

    /// ファイルを削除してデフォルト値にリセットする。
    public void ResetToDefault();
}
```

**使用例:**
```csharp
// EditorWallpaper 相当の定義
[Serializable]
public class WallpaperSettings : KPrefsJson<WallpaperSettings>
{
    protected override string FilePath => "ProjectSettings/EditorWallpaperSettings.json";

    public bool enabled = true;
    public float opacity = 0.8f;
    public string imagePath = "";
    public Color overlayColor = Color.black;
}

// 使用
WallpaperSettings.Instance.enabled = false;
WallpaperSettings.Instance.Save();
```

**設計ノート:**
- シリアライズに `JsonUtility` を使用（Unity標準、追加依存なし）
- ファイル変更時に `OnSettingsChanged` イベントを発火するオプション
- `ProjectSettings/` 配下に保存することでGit管理可能（プロジェクト設定として共有）

---

### 8. KLocale - 多言語対応

日本語/英語の切り替えをシンプルに実装する。
既存拡張の3つの異なるローカライゼーション方式を統一。

```csharp
public class KLocale
{
    public enum Language
    {
        Japanese,
        English
    }

    /// 現在の言語
    public Language CurrentLanguage { get; set; }

    /// 翻訳テキストを登録する。メソッドチェーン対応。
    public KLocale Add(string key, string ja, string en);

    /// 現在の言語でテキストを取得する。キーが未登録の場合はキーをそのまま返す。
    public string Get(string key);

    /// インデクサでアクセス（Get(key)と同じ）
    public string this[string key] { get; }

    /// 言語切り替えUI（ツールバー形式）を描画する。
    /// 既存拡張と同じ「日本語 | English」タブスタイル。
    /// 戻り値: 言語が変更された場合 true。
    public bool DrawLanguageSelector();
}
```

**使用例:**
```csharp
// AnimationLabelSimplify 相当の定義
private KLocale locale = new KLocale()
    .Add("title", "設定ウィンドウ", "Settings Window")
    .Add("preview", "プレビュー", "Preview")
    .Add("addRule", "新しいルールを追加", "Add New Rule")
    .Add("reset", "デフォルトに戻す", "Reset to Default")
    .Add("deleteConfirm", "このルールを削除しますか？", "Delete this rule?");

// UI描画
locale.DrawLanguageSelector();
GUILayout.Label(locale["title"], EditorStyles.boldLabel);
if (KButton.Draw(locale["addRule"])) { ... }
```

**設計ノート:**
- 言語切り替えUIは既存拡張のシアンハイライトスタイルを踏襲
- 言語設定はKLocaleインスタンス単位で管理（ウィンドウごとに独立）
- EditorPrefsへの言語設定保存はKWindow側で自動化（後述）

---

### 9. KWindow - EditorWindow基底クラス

EditorWindowを継承した基底クラス。
スクロール、ヘッダー、言語切替、フッターを自動で提供し、
サブクラスはコンテンツ描画に集中できる。

```csharp
public abstract class KWindow : EditorWindow
{
    // === サブクラスがオーバーライドするメソッド ===

    /// ヘッダーにタイトルを描画する（任意）
    /// デフォルトでは何も描画しない。
    protected virtual void DrawHeader() { }

    /// メインコンテンツを描画する（必須）。
    /// 自動的にスクロールビュー内で呼ばれる。
    protected abstract void DrawContent();

    /// フッターを描画する（任意）。
    /// デフォルトでは何も描画しない。
    protected virtual void DrawFooter() { }

    /// KLocaleインスタンスを生成する（任意）。
    /// nullを返すと言語切替機能は無効。
    protected virtual KLocale CreateLocale() => null;

    // === サブクラスから利用できるプロパティ ===

    /// KLocaleインスタンス。CreateLocale()で生成されたもの。
    protected KLocale Locale { get; }

    /// 現在のスクロール位置（読み取り専用）
    protected Vector2 ScrollPosition { get; }
}
```

**KWindowが自動で行うこと:**
1. `OnGUI()` 内で以下の順序で描画:
   - ヘッダー領域: `DrawHeader()` + 言語切替ボタン（Localeが有効な場合）
   - スクロールビュー開始
   - コンテンツ領域: `DrawContent()`
   - スクロールビュー終了
   - フッター領域: `DrawFooter()`
2. 言語設定をEditorPrefsに自動保存/復元（キー: `KWindow_{ウィンドウ型名}_Language`）

**使用例:**
```csharp
// KokoaToggleSettings をKWindowで書き直した場合
public class KokoaToggleSettingsWindow : KWindow
{
    [MenuItem("Tools/kokoaToggle")]
    static void Open()
    {
        var w = GetWindow<KokoaToggleSettingsWindow>("kokoa Toggle Settings");
        w.minSize = new Vector2(400, 300);
    }

    protected override KLocale CreateLocale() => new KLocale()
        .Add("basic", "基本設定", "Basic Settings")
        .Add("operation", "操作設定", "Operation Settings")
        .Add("display", "表示設定", "Display Settings")
        .Add("reset", "設定をリセット", "Reset Settings");

    protected override void DrawContent()
    {
        KSection.Draw(Locale["basic"], () =>
        {
            KField.Toggle(MyPrefs.Enabled);
            using (KLayout.Disabled(!MyPrefs.Enabled.Value))
            {
                KField.Toggle(MyPrefs.ShowEditorOnly);
                KField.Toggle(MyPrefs.ShowActive);
            }
        });

        KSection.Draw(Locale["operation"], () =>
        {
            KField.ToggleWithHelp(MyPrefs.RequireShift,
                Locale["shiftHelp"]);
        });

        KSection.Draw(Locale["display"], () =>
        {
            KField.IntSlider(MyPrefs.ButtonOffset, 0, 200);
            KField.Slider(MyPrefs.InactiveAlpha, 0.1f, 1f);
        });

        if (KButton.DrawWithConfirm(Locale["reset"], Locale["resetTitle"], Locale["resetMsg"]))
        {
            MyPrefs.ResetAll();
        }
    }
}
```

---

### 10. KList - リスト管理UI

項目の追加・削除・表示を管理するリストコンポーネント。
AnimationLabelSimplifyのルールリストのようなUIを簡単に構築できる。

```csharp
public static class KList
{
    /// リストUIを描画する。
    ///
    /// T: リスト要素の型
    /// items: 管理するリスト
    /// drawItem: 各要素の描画コールバック（要素, インデックス）
    /// createItem: 「追加」ボタンで生成される新しい要素のファクトリ
    /// addLabel: 追加ボタンのラベル（デフォルト: "追加"）
    /// confirmRemove: 削除前に確認ダイアログを出すか
    /// removeDialogTitle: 削除確認ダイアログのタイトル
    /// removeDialogMessage: 削除確認ダイアログのメッセージ
    /// emptyMessage: リストが空の時に表示するメッセージ
    ///
    /// 戻り値: リストに変更があった場合 true
    public static bool Draw<T>(
        IList<T> items,
        Action<T, int> drawItem,
        Func<T> createItem,
        string addLabel = "追加",
        bool confirmRemove = false,
        string removeDialogTitle = "削除確認",
        string removeDialogMessage = "この項目を削除しますか？",
        string emptyMessage = null);
}
```

**各要素の描画:**
- ボックスで囲まれた1アイテム表示
- 右上に「×」削除ボタン（KButton.DrawMini使用）
- `confirmRemove=true` の場合、削除前に確認ダイアログ

**使用例:**
```csharp
// AnimationLabelSimplify のルールリスト相当
KList.Draw(
    settings.rules,
    drawItem: (rule, index) =>
    {
        KField.Toggle("有効", ref rule.enabled);
        KField.TextField("検索文字列", ref rule.searchString);
        KField.TextField("置換文字列", ref rule.replaceString);
    },
    createItem: () => new ReplaceRule { enabled = true, searchString = "", replaceString = "" },
    addLabel: locale["addRule"],
    confirmRemove: true,
    removeDialogTitle: locale["deleteTitle"],
    removeDialogMessage: locale["deleteConfirm"]
);
```

---

### 11. KNotice - 通知バナー

HelpBoxよりリッチな通知UI。KStyleのカラーテーマに従った色分け。

```csharp
public static class KNotice
{
    /// 情報通知（アクセントカラー背景）
    public static void Info(string message);

    /// 警告通知（黄色背景）
    public static void Warning(string message);

    /// エラー通知（赤色背景）
    public static void Error(string message);

    /// 成功通知（緑色背景）
    public static void Success(string message);
}
```

**表示スタイル:**
- 左に縦のカラーバー（4px幅、各タイプの色）
- 薄い背景色
- アイコン + メッセージテキスト
- HelpBoxとは異なり、KStyleのカラーパレットに統一

**使用例:**
```csharp
if (hasUnsavedChanges)
    KNotice.Warning("未保存の変更があります");

if (operationSuccess)
    KNotice.Success("テクスチャを書き出しました");
```

---

## ファイル構成

```
dev.kokoa.ui-lib/
├── package.json
├── LICENSE
└── Editor/
    ├── dev.kokoa.ui-lib.Editor.asmdef
    ├── KStyle.cs        // テーマ・スタイル定義
    ├── KSection.cs      // セクション・グループ描画
    ├── KLayout.cs       // レイアウトスコープ
    ├── KField.cs        // フィールド描画
    ├── KButton.cs       // ボタン
    ├── KPrefs.cs        // 設定保存（EditorPrefs）
    ├── KPrefsJson.cs    // 設定保存（JSONファイル）
    ├── KLocale.cs       // 多言語対応
    ├── KWindow.cs       // EditorWindow基底クラス
    ├── KList.cs         // リスト管理UI
    └── KNotice.cs       // 通知バナー
```

---

## 既存拡張での適用例

### KokoaToggleButtons の設定画面

**Before（現在のコード: ~120行）:**
```csharp
public class KokoaToggleSettings : EditorWindow
{
    private static bool enabledToggleButtons = true;
    // ... 6つの設定変数を手動定義
    // ... EditorPrefsキーを手動定義
    // ... LoadPreferences() で手動読み込み
    // ... SavePreferences() で手動保存
    // ... OnGUI() で手動描画
    // ... ResetToDefaults() で手動リセット
}
```

**After（KWindow + KPrefs使用: ~40行）:**
```csharp
public class KokoaToggleSettingsWindow : KWindow
{
    static readonly KPrefs<bool> Enabled = new("KT_Enabled", "トグルボタンを有効にする", true);
    static readonly KPrefs<bool> ShowEditorOnly = new("KT_ShowEditorOnly", "EditorOnlyボタンを表示", true);
    static readonly KPrefs<bool> ShowActive = new("KT_ShowActive", "Activeボタンを表示", true);
    static readonly KPrefs<bool> RequireShift = new("KT_RequireShift", "Shiftキーで一括操作", true);
    static readonly KPrefs<int> ButtonOffset = new("KT_ButtonOffset", "ボタンの右端からの距離", 80);
    static readonly KPrefs<float> InactiveAlpha = new("KT_InactiveAlpha", "非アクティブボタンの透明度", 0.7f);

    [MenuItem("Tools/kokoaToggle")]
    static void Open() => GetWindow<KokoaToggleSettingsWindow>("kokoa Toggle Settings").minSize = new Vector2(400, 300);

    protected override void DrawContent()
    {
        KSection.Draw("基本設定", () =>
        {
            KField.Toggle(Enabled);
            using (KLayout.Disabled(!Enabled.Value))
            {
                KField.Toggle(ShowEditorOnly);
                KField.Toggle(ShowActive);
            }
        });
        KSection.Draw("操作設定", () =>
        {
            KField.ToggleWithHelp(RequireShift,
                "チェックを入れると、複数選択時にShiftキーを押しながらクリックした場合のみ一括操作が実行されます。");
        });
        KSection.Draw("表示設定", () =>
        {
            KField.IntSlider(ButtonOffset, 0, 200);
            KField.Slider(InactiveAlpha, 0.1f, 1f);
        });
        if (KButton.DrawWithConfirm("設定をリセット", "設定のリセット", "すべての設定をデフォルトに戻しますか？"))
        {
            Enabled.Reset(); ShowEditorOnly.Reset(); ShowActive.Reset();
            RequireShift.Reset(); ButtonOffset.Reset(); InactiveAlpha.Reset();
            Repaint();
        }
    }
}
```

---

## 将来の拡張候補（v0.2.0以降）

- **KInspector** - カスタムInspectorの基底クラス
- **KToolbar** - タブバー/ツールバーコンポーネント
- **KDialog** - モーダルダイアログビルダー
- **KProgress** - プログレスバー
- **KSearch** - 検索フィールド + フィルタリング
- **UIToolkit対応** - IMGUI版と同等のUIToolkitコンポーネント
