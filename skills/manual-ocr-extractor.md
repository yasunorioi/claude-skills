# Manual OCR Extractor - Skill Definition

**Skill ID**: `manual-ocr-extractor`
**Category**: Document Processing / Vision
**Version**: 1.0.0
**Created**: 2026-02-18

---

## Overview

スキャンPNG画像からClaude Visionを使って日本語マニュアルを読み取り、パラメータを構造化データとして抽出するパターン。ADF（自動原稿送り装置）で両面スキャンした際に生じるページ順序ずれ（奇数ページ→偶数ページが逆順になる）の補正処理も含む。

農業機器・産業設備の設定マニュアルのデジタル化・パラメータ抽出に特に有効。

---

## Use Cases

- 紙マニュアルをスキャンしてパラメータ一覧を自動抽出したい
- 設定値・警報閾値・制御パラメータを構造化YAML/JSON/Markdownで出力したい
- ADF両面スキャンで生成されたPNG（奇数/偶数ページが逆順）を正しい順に並べたい
- 複数冊のマニュアルを統一フォーマットで抽出したい

### 適用実績

- ArSprout農業制御システム マニュアル1冊目（補助マニュアル、28p、201行抽出）
  - PID制御パラメータ（P比例帯、I=0、D=0）
  - 警報優先順位（強風>気温上昇>温度上昇>降雨）
  - 差分値警報、風向16方位、センサー検出方法
  - 制御ノード最大8ch、時間帯設定4タイプ
- ArSprout マニュアル2冊目（22p、279行抽出）
  - PID制御（P=5, I=0, D=0）、警報動作
  - 風向16方位、風速閾値5m/s
  - センサーフィルタ60秒移動平均、片側制御N/S独立

---

## Skill Input

1. **スキャン画像**: PNG/JPEGのページ画像一式（ディレクトリまたはリスト）
2. **スキャン方式**: 片面スキャン or ADF両面スキャン（ページ順序補正要否）
3. **抽出対象**: 抽出したいパラメータカテゴリ（制御設定、警報設定、センサー設定 等）
4. **出力フォーマット**: Markdown / YAML / JSON（デフォルト: Markdown）

---

## Skill Output

1. **構造化Markdownファイル**: 章ごとにパラメータを整理した抽出結果
2. **パラメータ一覧**: テーブル形式（パラメータ名 / 値 / 単位 / 備考）
3. **読み取れなかったページのリスト**: 品質確認用

---

## Implementation Pattern

### Step 1: ページ画像の順序補正（ADF両面スキャン時）

ADF両面スキャンでは奇数ページ（表面）と偶数ページ（裏面）が分かれて出力される場合がある。
正しいページ順に並べ替える：

```python
import os
from pathlib import Path

def sort_adf_pages(scan_dir: str) -> list[str]:
    """
    ADF両面スキャンのページ順序補正。
    想定ファイル命名規則: page_001.png, page_002.png, ...
    奇数ページが表面（正順）、偶数ページが裏面（逆順）の場合。
    """
    files = sorted(Path(scan_dir).glob("*.png"))
    odd_pages = files[::2]    # 奇数インデックス: 表面（正順）
    even_pages = files[1::2]  # 偶数インデックス: 裏面（逆順に並んでいる）

    # 偶数ページを逆順に並べ直して結合
    corrected = []
    for odd, even in zip(odd_pages, reversed(even_pages)):
        corrected.append(odd)
        corrected.append(even)

    return [str(p) for p in corrected]

# 使用例
pages = sort_adf_pages("/path/to/scanned_pages")
```

### Step 2: Claude Visionによるページ読み取り

```python
import anthropic
import base64
from pathlib import Path

def extract_page(client: anthropic.Anthropic, image_path: str, page_num: int, context: str = "") -> str:
    """1ページ分の画像をClaude Visionで読み取る"""
    with open(image_path, "rb") as f:
        image_data = base64.standard_b64encode(f.read()).decode("utf-8")

    suffix = image_path.split(".")[-1].lower()
    media_type = "image/jpeg" if suffix in ("jpg", "jpeg") else "image/png"

    message = client.messages.create(
        model="claude-opus-4-6",  # Visionタスクには高性能モデルを使用
        max_tokens=4096,
        messages=[
            {
                "role": "user",
                "content": [
                    {
                        "type": "image",
                        "source": {
                            "type": "base64",
                            "media_type": media_type,
                            "data": image_data,
                        },
                    },
                    {
                        "type": "text",
                        "text": f"""このマニュアルページ（{page_num}ページ目）の内容を全て読み取り、以下の形式で出力してください：

## ページ {page_num}

### 章・節タイトル
（このページの章・節タイトルを記載）

### 本文・説明
（説明文を忠実に転記）

### パラメータ・設定値
| パラメータ名 | 値 | 単位 | 備考 |
|------------|-----|------|------|
（表や設定値はテーブル形式で抽出）

{context}

読み取れない部分は [読取不可] と記載してください。"""
                    }
                ],
            }
        ],
    )
    return message.content[0].text


def extract_manual(image_paths: list[str], output_path: str) -> None:
    """マニュアル全ページを読み取って1ファイルに統合"""
    client = anthropic.Anthropic()
    results = []

    for i, img_path in enumerate(image_paths, start=1):
        print(f"Processing page {i}/{len(image_paths)}: {img_path}")
        text = extract_page(client, img_path, i)
        results.append(text)

    with open(output_path, "w", encoding="utf-8") as f:
        f.write("\n\n---\n\n".join(results))

    print(f"Extracted {len(results)} pages → {output_path}")
```

### Step 3: 構造化パラメータの後処理

```python
import re

def postprocess_extract(raw_text: str, system_name: str) -> dict:
    """
    抽出テキストからパラメータをさらに構造化する。
    マークダウンのテーブルをパース。
    """
    params = {}

    # テーブル行を抽出: | パラメータ名 | 値 | 単位 | 備考 |
    table_pattern = re.compile(r'\|\s*([^|]+)\|\s*([^|]+)\|\s*([^|]*)\|\s*([^|]*)\|')

    for match in table_pattern.finditer(raw_text):
        name, value, unit, note = [m.strip() for m in match.groups()]
        if name and name != "パラメータ名":  # ヘッダー行をスキップ
            params[name] = {
                "value": value,
                "unit": unit,
                "note": note,
                "system": system_name,
            }

    return params
```

### Step 4: 出力ファイルの検証

```python
def validate_extract(output_path: str, expected_pages: int) -> dict:
    """抽出結果の品質チェック"""
    with open(output_path, "r", encoding="utf-8") as f:
        content = f.read()

    pages_found = len(re.findall(r'## ページ \d+', content))
    unreadable = len(re.findall(r'\[読取不可\]', content))
    tables = len(re.findall(r'\|.*\|.*\|', content))

    return {
        "pages_found": pages_found,
        "expected_pages": expected_pages,
        "unreadable_sections": unreadable,
        "table_rows": tables,
        "coverage": pages_found / expected_pages if expected_pages > 0 else 0,
    }
```

---

## Notes / Limitations

- **APIコスト**: Claude Visionは1画像あたり高コスト。28ページで数百円程度を見込む
- **画像解像度**: 300dpi以上推奨。低解像度は文字認識精度が下がる
- **日本語縦書き**: Claude Visionは縦書きも読めるが、テーブル抽出精度はやや低下する場合あり
- **ADF補正の前提**: スキャン機器によって奇数/偶数の配置が異なる場合があるため、先頭数ページで補正結果を目視確認せよ
- **読取不可箇所**: 汚れ・折れ・複雑な図表はフォールバックとして `[読取不可]` が入る。人間が後補填する
- **レート制限**: 大量ページの一括処理時は `time.sleep(1)` 等でAPI呼び出しを間引くこと

---

**Skill Author**: 部屋子2号提案（subtask_468） / 部屋子1号補完（subtask_467）
**Last Updated**: 2026-02-18
