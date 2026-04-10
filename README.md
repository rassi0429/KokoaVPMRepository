# kokoa's VPM Package Repository

kokoa が作った Unity / VRChat 向けパッケージのリポジトリです。

## パッケージ一覧

| パッケージ | 説明 |
|---|---|
| [VRChatUploadSound](https://github.com/rassi0429/VRChatUploadSound) | アップロード完了時に通知音を鳴らします |
| [KokoaToggleButtons](https://github.com/rassi0429/KokoaToggleButtons) | オブジェクトの有効/無効を切り替え、Editor-only に設定できる拡張 |
| [EditorWallpaper](https://github.com/rassi0429/EditorWallpaper) | Unity Editor の壁紙をカスタマイズできる拡張 |
| [MeshColorTool](https://github.com/rassi0429/MeshColorTool) | メッシュの色をワンクリックで変更できるツール |
| [MiddleClickInspectorOpener](https://github.com/rassi0429/MiddleClickInspectorOpener) | 中クリックで新しい Inspector ウィンドウを開く拡張 |
| [AnimationLabelSimplify](https://github.com/rassi0429/AnimationLabelSimplify) | Animation のラベルをシンプルにする拡張 |

## VCC への追加方法

1. [VRChat Creator Companion (VCC)](https://vcc.docs.vrchat.com/) を開く
2. Settings > Packages > Add Repository
3. 以下の URL を入力して追加

```
https://rassi0429.github.io/KokoaVPMRepository/index.json
```

または [ランディングページ](https://rassi0429.github.io/KokoaVPMRepository/) から「Add to VCC」ボタンで追加できます。

## 仕組み

- `source.json` にパッケージのリポジトリ一覧を定義
- GitHub Actions が [package-list-action](https://github.com/vrchat-community/package-list-action) を使ってリスティングをビルド
- GitHub Pages にデプロイ（毎時自動更新）
- 各パッケージのダウンロード数も自動で取得・表示

## パッケージの追加方法

`source.json` の `githubRepos` に GitHub リポジトリを追加して push するだけです。
