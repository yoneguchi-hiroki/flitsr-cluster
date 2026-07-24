# FLITSR Research Fork

このリポジトリは、元の FLITSR 実装をベースにした研究用フォークです。
元の FLITSR(https://github.com/DCallaz/flitsr)の構造や実行方法をそのまま維持しつつ、失敗テストケースの扱い方に変数情報ベースのクラスタリング結果を組み込むことで、複数バグ環境下でのバグ箇所特定精度の向上を試みています。

本フォークで行った研究の詳細は、「SBFL による欠陥限局の正確性向上に関する研究」を参照してください。

## 研究の概要

複数のバグが同時に存在する環境では、Spectrum-Based Fault Localization (SBFL) の性能が低下することが知られています。この問題に対処する手法として FLITSR が提案されていますが、異なるバグに起因する失敗テストが類似した実行経路を示す場合、FLITSR は失敗原因の違いを十分に識別できないという課題があります。

本フォークでは、失敗テストケース実行時の変数情報に基づくクラスタリング結果（[ReClues](https://doi.org/10.1145/3597503.3639179) を参考にした手法）を FLITSR の失敗テストケース削除の判断に組み込み、この課題の緩和を試みています。

## 主な変更点

元の FLITSR 実装と比較して、このリポジトリでは以下の変更を含みます。

### 1. クラスタ情報を考慮した失敗テストケースの削除（`flitsr/spectrum.py`）

`Spectrum.get_tests()` を拡張し、失敗テストケースを削除する際にクラスタ情報を考慮するようにしています。判定ロジックの実体は `compare_executing_and_cluster()` メソッドです。

- 疑惑値ランキングの上位要素を実行した失敗テストケース集合が、**単一のクラスタに完全に含まれる場合** → そのクラスタのテストは同一バグに起因すると判断し、通常の FLITSR と同様にテストケースを削除します。
- **2つ以上のクラスタにまたがって含まれる場合** → どのバグに対応する失敗かを一意に特定できないと判断し、テストケースの削除を行わず、当該要素のみを span に追加して次の候補へ進みます。
- どのクラスタにも一致しない場合 → 削除対象なしとして扱います。

### 2. FLITSR の span 構築処理への反映（`flitsr/advanced/flitsr.py`）

`Flitsr.flitsr()` は、上記のクラスタ判定を経由して失敗テストケースを削除します。

- クラスタ情報を使っても失敗テストを最後まで説明しきれない場合は、クラスタ情報を使わない通常の FLITSR の削除ロジックにフォールバックして再試行します。
- basis 構築（冗長な要素の除去）自体のロジックは元の FLITSR と同一で、変更はありません。

### 3. コマンドラインインターフェースの互換性

元の FLITSR と全く同じコマンドライン形式で実行できます。クラスタ情報を考慮した処理は標準の実行パスに組み込まれているため、追加のオプションは不要です。

## リポジトリ構成

- `main.py`: 実行のエントリーポイント
- `args.py`: コマンドライン引数の解析
- `flitsr/spectrum.py`: スペクトラム実装（クラスタ情報を考慮した失敗テスト削除を含む）
- `flitsr/advanced/flitsr.py`: FLITSR 実装（クラスタ情報を考慮した span 構築を含む）

## セットアップ

セットアップは、元の FLITSR プロジェクト（[リンク: TODO]）に準じた形で行うことを想定しています。既存の FLITSR の依存関係の導入方法や実行の流れをそのまま利用してください。元の FLITSR のセットアップに慣れている場合は、特別な環境変更がない限り、同じ手順を基本として利用してください。

追加の依存関係として、クラスタ情報ファイル（`.xls`）の読み込みに [`xlrd`](https://pypi.org/project/xlrd/) を使用しています。

## 実行方法

元の FLITSR と同じ形式で実行できます。

```bash
python -m flitsr -m ochiai <input>
```

```bash
python -m flitsr --csv <input>
```

### クラスタ情報ファイルの準備（必須）

上記のコマンドを実行する前に、失敗テストケースのクラスタリング結果を用意しておく必要があります。このリポジトリのコード自体はクラスタリングを行わないため、[ReClues](https://doi.org/10.1145/3597503.3639179) などのクラスタリングアルゴリズムで事前に計算してください。

**ファイル形式・配置場所について**

クラスタリング結果は `.xls`（Excel）形式で、次の2列を持つ必要があります。

| 列 | 内容 |
| --- | --- |
| A列 | 失敗テストケース名 |
| B列 | クラスタ ID |

例（架空のデータです。実データに差し替えてください）:

| A列（テスト名） | B列（クラスタID） |
| --- | --- |
| org.apache.commons.math.TestA | 1 |
| org.apache.commons.math.TestB | 1 |
| org.apache.commons.math.TestC | 2 |

このファイルは **`ReClues_clustering.xls`** という固定のファイル名で、実行時のカレントディレクトリのパス中に含まれる `Math-数字`（例: `Math-70`）というディレクトリ名を自動検出し、その直下に配置する必要があります（`spectrum.py` の `_load_test_clusters()` を参照）。

> **注意:** このパス自動検出ロジックは、本研究で対象とした Defects4J-mf の Math プロジェクトの実験環境（ディレクトリ名が `Math-<バージョン番号>` という命名規則）の実装です。他のプロジェクトやディレクトリ構成では、このままでは正しく動作しません。汎用化する場合は `_load_test_clusters()` を修正してください。

## 参考文献・関連リンク

- FLITSR: D. Callaghan and B. Fischer, "FLITSR: Improved Spectrum-Based Localization of Multiple Faults by Iterative Test Suite Reduction," ACM Transactions on Software Engineering and Methodology, 2025.
- ReClues: Y. Song et al., "ReClues: Representing and Indexing Failures in Parallel Debugging with Program Variables," ICSE 2024.
