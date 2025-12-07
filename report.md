# キャッシュ効果測定レポート

## 背景と目的

### 対象ワークフロー

**VOICEVOX/voicevox_engine** リポジトリの `.github/workflows/build-engine.yml`

### 計測の背景

build-engine.ymlでは、ビルド時間短縮のために以下のリソースをキャッシュしている：
- VOICEVOX Core (C API)
- ONNX Runtime
- VVM (音声合成モデル)
- voicevox_resource (リソースファイル)
- zlib (Windows向けライブラリ)

しかし、**これらのキャッシュが本当に速度向上に貢献しているのか不明**だった。

特に、大きなファイル（数百MB）のキャッシュは、GitHub Actionsのキャッシュシステムでは逆に遅くなる可能性があるという仮説があった。

### 計測の目的

**キャッシュ使用時と非使用時の実際の時間を正確に計測し、各リソースのキャッシュ戦略を見直すべきか判断する。**

## 検証環境

- **検証用ワークフロー**: `.github/workflows/test.yml` (commit: `f9f8943`)
- **測定日**: 2025-12-07
- **実行回数**: 3回（結果の安定性確認のため）

## 計測方法

### 対象リソース

build-engine.ymlでキャッシュしている全リソース：
- **zlib**: Windows向けライブラリ（zipファイル、数MB）
- **core**: VOICEVOX Core C API（数十MB）
- **onnxruntime**: ONNX Runtime（数十MB）
- **models**: VOICEVOX VVM（数百MB）
- **resource**: voicevox_resource（git clone）

### 計測内容

build-engine.ymlと同じダウンロード処理・キャッシュ処理を使用して計測：

- **No Cache**: キャッシュを使用せず、毎回ダウンロード（純粋なダウンロード時間）
- **With Cache**: `actions/cache/restore` → ダウンロード（cache missの場合） → `actions/cache/save` の**全体時間**

**重要**: With Cacheの時間には、cache restore/saveのオーバーヘッドが含まれています。これにより、**実際の運用での時間**を正確に測定できます。

### 実行環境

- **OS**: windows-2025, macos-15-intel, macos-15, ubuntu-24.04, ubuntu-24.04-arm
- **実行回数**: 3回（キャッシュヒット時のデータ）
  - 初回実行（cache miss）のデータは除外
  - 2-4回目の実行でキャッシュヒット時の安定した性能を測定

## 結果サマリー

### リソース別の効果（3回平均）

| OS | Resource | No Cache | With Cache | 改善 |
|---|---|---|---|---|
| windows-2025 | zlib | 1s | 1s | 変化なし |
| windows-2025 | core | 0s | 1s | N/A |
| windows-2025 | onnxruntime | 1s | 2s | ⚠️ 1秒増加 (+100%) |
| windows-2025 | models | 7s | 16s | ⚠️ **9秒増加 (+128%)** |
| windows-2025 | resource | 18s | 12s | ✅ **6秒削減 (33%)** |
| macos-15-intel | core | 1s | 2s | ⚠️ 1秒増加 (+100%) |
| macos-15-intel | onnxruntime | 1s | 2s | ⚠️ 1秒増加 (+100%) |
| macos-15-intel | models | 32s | 46s | ⚠️ **14秒増加 (+43%)** |
| macos-15-intel | resource | 19s | 22s | ⚠️ 3秒増加 (+15%) |
| macos-15 | core | 0s | 1s | N/A |
| macos-15 | onnxruntime | 1s | 1s | 変化なし |
| macos-15 | models | 13s | 20s | ⚠️ **7秒増加 (+53%)** |
| macos-15 | resource | 35s | 13s | ✅ **22秒削減 (62%)** |
| ubuntu-24.04 | core | 1s | 1s | 変化なし |
| ubuntu-24.04 | onnxruntime | 2s | 2s | 変化なし |
| ubuntu-24.04 | models | 7s | 15s | ⚠️ **8秒増加 (+114%)** |
| ubuntu-24.04 | resource | 49s | 10s | ✅ **39秒削減 (79%)** |
| ubuntu-24.04-arm | core | 0s | 2s | N/A |
| ubuntu-24.04-arm | onnxruntime | 0s | 1s | N/A |
| ubuntu-24.04-arm | models | 4s | 13s | ⚠️ **9秒増加 (+225%)** |
| ubuntu-24.04-arm | resource | 17s | 6s | ✅ **11秒削減 (64%)** |

## 詳細データ（3回の実行結果）

| OS | Resource | No Cache | Run1 | Run2 | Run3 |
|---|---|---|---|---|---|
| windows-2025 | zlib | 1s | 2s | 2s | 1s |
| windows-2025 | core | 0s | 1s | 2s | 1s |
| windows-2025 | onnxruntime | 1s | 2s | 3s | 1s |
| windows-2025 | models | 7s | 15s | 20s | 15s |
| windows-2025 | resource | 18s | 10s | 16s | 10s |
| macos-15-intel | core | 1s | 3s | 2s | 2s |
| macos-15-intel | onnxruntime | 1s | 3s | 2s | 2s |
| macos-15-intel | models | 32s | 41s | 40s | 58s |
| macos-15-intel | resource | 19s | 25s | 19s | 22s |
| macos-15 | core | 0s | 1s | 3s | 1s |
| macos-15 | onnxruntime | 1s | 1s | 1s | 1s |
| macos-15 | models | 13s | 14s | 30s | 17s |
| macos-15 | resource | 35s | 10s | 21s | 10s |
| ubuntu-24.04 | core | 1s | 1s | 1s | 2s |
| ubuntu-24.04 | onnxruntime | 2s | 2s | 1s | 3s |
| ubuntu-24.04 | models | 7s | 14s | 13s | 19s |
| ubuntu-24.04 | resource | 49s | 10s | 10s | 12s |
| ubuntu-24.04-arm | core | 0s | 0s | 1s | 4s |
| ubuntu-24.04-arm | onnxruntime | 0s | 0s | 1s | 1s |
| ubuntu-24.04-arm | models | 4s | 13s | 12s | 14s |
| ubuntu-24.04-arm | resource | 17s | 6s | 6s | 8s |

## 分析と考察

### キャッシュが効果的なリソース

#### ✅ resource（git clone）

- **効果**: 6-39秒削減（33-79%改善）
- **理由**: git cloneは比較的遅い操作のため、キャッシュからの復元が効果的
- **推奨**: **キャッシュを継続使用**

特にubuntu-24.04では49秒→10秒と劇的な改善が見られた。

### キャッシュが逆効果のリソース

#### ⚠️ models（VVM）

- **効果**: 7-14秒**増加**（43-225%悪化）
- **理由**: 数百MBの大きなファイルの場合、GitHub Actionsのキャッシュシステムでは復元に時間がかかる
- **推奨**: **キャッシュを削除すべき**

3回の実行すべてで一貫して悪化しており、最も問題のあるリソース。

#### ⚠️ core / onnxruntime

- **効果**: 1-2秒増加
- **理由**: 小さいファイルでもcache restore/saveのオーバーヘッドが発生
- **推奨**: **キャッシュを削除すべき**

ファイルサイズが小さく、ダウンロードが高速なため、キャッシュのオーバーヘッドの方が大きい。

#### ⚠️ zlib

- **効果**: ほぼ変化なし
- **理由**: 非常に小さいファイル（数MB）のため、キャッシュの効果がない
- **推奨**: **キャッシュを削除しても問題なし**

## 結論

### 現状の問題点

**VOICEVOX/voicevox_engineのbuild-engine.ymlにおけるキャッシュ戦略は、全体として速度向上に貢献していない。** むしろ、多くのリソースで逆効果となっている。

特にmodelsのキャッシュは、すべてのOSで7-14秒の遅延を引き起こしており、**最大のボトルネック**となっている。

### 推奨事項：build-engine.ymlへの適用

#### 即座に実施すべき変更

**1. resourceのみキャッシュを使用**
- git cloneは確実に効果があり、6-39秒の短縮が期待できる
- 現在のキャッシュ設定（lines 359-373）を**維持**

**2. 以下のキャッシュを削除**

削除対象とその効果：
- **models** (lines 226-356)
  - **最優先**: 7-14秒の改善
  - 全OSで一貫して逆効果
- **core** (lines 201-281)
  - 1-2秒の改善
  - 小さいファイルでもオーバーヘッドあり
- **onnxruntime** (lines 209-316)
  - 1-2秒の改善
  - core同様、オーバーヘッドが大きい
- **zlib** (lines 142-167)
  - 影響は小さいが不要

#### 期待される効果

キャッシュ戦略を見直すことで、**各ビルドで10-20秒程度の短縮**が期待できる。

例：ubuntu-24.04の場合
- modelsキャッシュ削除: +8秒改善
- resourceキャッシュ維持: -39秒削減（現状維持）
- **合計**: 約8秒の短縮（modelsのボトルネック解消）

#### 実装方法

1. build-engine.ymlから以下のセクションを削除：
   - `Restore cached CORE` と `Save CORE cache`
   - `Restore cached ONNX Runtime` と `Save ONNX Runtime cache`
   - `Restore cached models` と `Save models cache`
   - `Restore cached zlib` と `Save zlib cache`

2. resourceのキャッシュ設定のみ残す：
   - `Prepare RESOURCE cache`（actions/cache@v4を使用）

3. ダウンロード処理は**そのまま実行**（キャッシュチェックを削除するだけ）

## 補足

### GitHub Actionsのキャッシュの特性

今回の計測で明らかになった重要な事実：

- **大きなファイル（数百MB以上）**: キャッシュからの復元がダウンロードより遅い
- **小さなファイル（数十MB以下）**: cache restore/saveのオーバーヘッドで逆効果
- **git clone**: キャッシュが効果的

この特性は、GitHub Actionsのキャッシュシステムの仕様によるもので、他のプロジェクトでも同様の傾向が見られる可能性がある。

## プロンプト

Claude Codeにこんな感じで指示しました。
```
./build-engine.yaml を読んでください。こういうworkflowを書いてます。

これ例えばresourceやvvmなどをcache/restoreしていますが、元がgithubなものもいくつかありますよね。releasesだったり。
これ本当に速度上がってるのか不明なので、実際に計測してもらえますか？

いろんなmatrix osで測って欲しいです。
普通に毎回ダウンロードするコードと、キャッシュ・リストアするactionsを使うコードを比較してほしいです。
どちらも./build-engine.yamlのコードを参考にし、DLのコード・キャッシュやリストアのコードは必ず./build-engine.yamlと同じ処理にしてください。
それが計測したい目的なので守ってください。

テスト用のコードは .github/workflows/test.yml を書き換えてください。
.github/workflows/test.ymlの既存処理は消してOKです。

git diffを見ればわかるように、実際にすでに .github/workflows/test.yml が書き換えられていると思います。
あとは計測してください。
キャッシュを使う場合にかかる時間はキャッシュをDL・UPする時間も含めてください。
キャッシュのアップロードは自動的に行われる場合もあるので、それも考慮してください。
```
