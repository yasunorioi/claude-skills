# Manual Merge Summarizer - Skill Definition

**Skill ID**: `manual-merge-summarizer`
**Category**: Document Processing / Analysis
**Version**: 1.0.0
**Created**: 2026-02-18

---

## Overview

複数のマニュアル抽出結果（Markdown）を統合し、既存の設定ファイル（YAML等）と照合した上で、プレゼンテーション用のポイントを抽出するパターン。

`manual-ocr-extractor` スキルで抽出した複数冊分のテキストを受け取り、重複排除・矛盾検出・設定値の網羅性チェックを行い、最終成果物（統合Markdownレポート）として出力する。

---

## Use Cases

- 複数冊の設定マニュアルを1つの参照ドキュメントに統合したい
- 既存システムの設定ファイル（YAML/JSON/TOML）とマニュアル記載値を照合したい
- プレゼン・提案資料に向けて「現行システムとの差分」や「注目ポイント」を抽出したい
- 旧システムから新システムへの設定移行時にパラメータマッピングを作成したい

### 適用実績

- ArSproutマニュアル統合（subtask_469）:
  - 1冊目 201行 + 2冊目 279行 → 統合299行の最終成果物
  - `crop_irrigation.yaml`（既存Node-RED設定）との照合
  - 2/19紹介資料向けポイント抽出
  - 出力: `/home/yasu/unipi-agri-ha/docs/arsprout_manual_summary.md`

---

## Skill Input

1. **抽出済みMarkdownファイル一覧**: `manual-ocr-extractor` の出力（複数ファイル可）
2. **既存設定ファイル（オプション）**: 照合対象のYAML/JSON/TOMLファイル
3. **出力目的**: `reference`（参照用）/ `presentation`（プレゼン用）/ `migration`（設定移行用）
4. **出力先パス**: 統合Markdownの保存先

---

## Skill Output

1. **統合Markdownレポート**: 章構成で整理されたパラメータ一覧
2. **設定ファイル照合結果**: マニュアル記載値 vs 実設定値の差分表
3. **プレゼン用ポイント**: 注目すべき特徴・変更点・推奨事項のリスト（`presentation`モード時）
4. **矛盾検出レポート**: 複数マニュアル間で値が異なるパラメータのリスト

---

## Implementation Pattern

### Step 1: 複数マニュアル抽出結果の読み込みと統合

```python
from pathlib import Path
from collections import defaultdict
import re

def load_extracts(file_paths: list[str]) -> dict[str, str]:
    """複数の抽出Markdownを読み込む"""
    extracts = {}
    for path in file_paths:
        name = Path(path).stem
        with open(path, "r", encoding="utf-8") as f:
            extracts[name] = f.read()
    return extracts


def extract_params_from_markdown(text: str) -> dict[str, dict]:
    """Markdownテーブルからパラメータを抽出"""
    params = {}
    # | パラメータ名 | 値 | 単位 | 備考 | のパターン
    table_re = re.compile(r'\|\s*([^|]{2,}?)\s*\|\s*([^|]*?)\s*\|\s*([^|]*?)\s*\|\s*([^|]*?)\s*\|')
    for m in table_re.finditer(text):
        name, value, unit, note = m.groups()
        if name and name.strip("- ") not in ("パラメータ名", "項目"):
            params[name.strip()] = {
                "value": value.strip(),
                "unit": unit.strip(),
                "note": note.strip(),
            }
    return params


def merge_extracts(extracts: dict[str, str]) -> tuple[dict, list[dict]]:
    """
    複数マニュアルを統合し、矛盾を検出する。
    Returns:
        merged_params: 統合パラメータ辞書
        conflicts: 矛盾リスト [{param, sources, values}]
    """
    all_params = {}
    conflicts = []

    for source_name, text in extracts.items():
        params = extract_params_from_markdown(text)
        for key, val in params.items():
            if key in all_params:
                existing = all_params[key]
                if existing["value"] != val["value"]:
                    conflicts.append({
                        "param": key,
                        "sources": [existing.get("_source", "unknown"), source_name],
                        "values": [existing["value"], val["value"]],
                    })
                # 後から読んだ方で上書き（より詳細なマニュアルが後）
            all_params[key] = {**val, "_source": source_name}

    return all_params, conflicts
```

### Step 2: 既存設定ファイルとの照合

```python
import yaml

def load_config_file(config_path: str) -> dict:
    """YAML設定ファイルを読み込み、フラットなキー:値辞書に変換"""
    with open(config_path, "r", encoding="utf-8") as f:
        raw = yaml.safe_load(f)

    def flatten(d: dict, prefix: str = "") -> dict:
        result = {}
        for k, v in d.items():
            full_key = f"{prefix}.{k}" if prefix else k
            if isinstance(v, dict):
                result.update(flatten(v, full_key))
            else:
                result[full_key] = v
        return result

    return flatten(raw)


def compare_with_config(manual_params: dict, config: dict) -> list[dict]:
    """
    マニュアル記載値と設定ファイルの値を照合。
    Returns: 差分リスト [{param, manual_value, config_value, status}]
    """
    diffs = []
    for param_name, param_info in manual_params.items():
        # 設定ファイルのキーを名寄せして探す（部分一致）
        matched_key = next(
            (k for k in config if param_name.lower() in k.lower()),
            None
        )
        if matched_key:
            config_val = str(config[matched_key])
            manual_val = param_info["value"]
            status = "MATCH" if config_val == manual_val else "DIFF"
            diffs.append({
                "param": param_name,
                "manual_value": manual_val,
                "config_key": matched_key,
                "config_value": config_val,
                "status": status,
            })
        else:
            diffs.append({
                "param": param_name,
                "manual_value": param_info["value"],
                "config_key": None,
                "config_value": None,
                "status": "NOT_IN_CONFIG",
            })

    return diffs
```

### Step 3: プレゼン用ポイント抽出（Claudeによる要約）

```python
import anthropic

def extract_presentation_points(
    merged_params: dict,
    conflicts: list[dict],
    diffs: list[dict],
    system_name: str,
    purpose: str,
) -> str:
    """
    統合パラメータ・矛盾・設定差分からプレゼン向けポイントをClaude AIで抽出。
    """
    client = anthropic.Anthropic()

    context = f"""
システム名: {system_name}
目的: {purpose}

## 統合パラメータ（上位）
{chr(10).join(f"- {k}: {v['value']} {v['unit']}" for k, v in list(merged_params.items())[:20])}

## マニュアル間の矛盾
{chr(10).join(f"- {c['param']}: {c['values']}" for c in conflicts[:10]) if conflicts else "矛盾なし"}

## 設定ファイルとの差分（DIFF項目）
{chr(10).join(f"- {d['param']}: マニュアル={d['manual_value']} / 現設定={d['config_value']}" for d in diffs if d['status']=='DIFF')[:10]}
"""

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": f"""{context}

上記の情報をもとに、以下を作成してください：

1. **注目パラメータ（3〜5点）**: このシステムの特徴的な設定値
2. **現設定との差分ポイント**: 重要な差分（あれば）
3. **プレゼンで強調すべき点**: 聴衆への訴求ポイント
4. **要確認事項**: 不明・矛盾していて確認が必要な点
"""
        }]
    )

    return response.content[0].text
```

### Step 4: 統合Markdownレポートの出力

```python
from datetime import datetime

def write_merged_report(
    output_path: str,
    merged_params: dict,
    conflicts: list[dict],
    diffs: list[dict],
    presentation_points: str,
    source_files: list[str],
) -> None:
    """統合レポートをMarkdownで出力"""

    lines = [
        f"# マニュアル統合レポート",
        f"",
        f"**生成日時**: {datetime.now().strftime('%Y-%m-%d %H:%M')}",
        f"**参照元**: {', '.join(source_files)}",
        f"",
        f"---",
        f"",
        f"## 1. プレゼン用ポイント",
        f"",
        presentation_points,
        f"",
        f"---",
        f"",
        f"## 2. 統合パラメータ一覧",
        f"",
        f"| パラメータ名 | 値 | 単位 | 備考 | 出典 |",
        f"|------------|-----|------|------|------|",
    ]

    for name, info in merged_params.items():
        lines.append(f"| {name} | {info['value']} | {info['unit']} | {info['note']} | {info.get('_source','')} |")

    if conflicts:
        lines += ["", "---", "", "## 3. マニュアル間の矛盾", ""]
        for c in conflicts:
            lines.append(f"- **{c['param']}**: {c['sources'][0]}={c['values'][0]} / {c['sources'][1]}={c['values'][1]}")

    if diffs:
        diff_items = [d for d in diffs if d['status'] == 'DIFF']
        if diff_items:
            lines += ["", "---", "", "## 4. 設定ファイルとの差分", ""]
            for d in diff_items:
                lines.append(f"- **{d['param']}**: マニュアル=`{d['manual_value']}` / 現設定=`{d['config_value']}`")

    with open(output_path, "w", encoding="utf-8") as f:
        f.write("\n".join(lines))

    print(f"Merged report written: {output_path} ({len(merged_params)} params)")
```

### Step 5: メインエントリポイント

```python
def run(
    extract_files: list[str],
    config_file: str | None = None,
    output_path: str = "manual_merged.md",
    system_name: str = "システム",
    purpose: str = "設定参照",
) -> None:
    # Step 1: 読み込みと統合
    extracts = load_extracts(extract_files)
    merged_params, conflicts = merge_extracts(extracts)

    # Step 2: 設定ファイルとの照合
    diffs = []
    if config_file:
        config = load_config_file(config_file)
        diffs = compare_with_config(merged_params, config)

    # Step 3: プレゼンポイント抽出
    presentation_points = extract_presentation_points(
        merged_params, conflicts, diffs, system_name, purpose
    )

    # Step 4: レポート出力
    write_merged_report(
        output_path, merged_params, conflicts, diffs,
        presentation_points, extract_files
    )


# 使用例
if __name__ == "__main__":
    run(
        extract_files=[
            "/tmp/arsprout_book1_extract.md",
            "/tmp/arsprout_book2_extract.md",
        ],
        config_file="/home/yasu/unipi-agri-ha/config/crop_irrigation.yaml",
        output_path="/home/yasu/unipi-agri-ha/docs/arsprout_manual_summary.md",
        system_name="ArSprout",
        purpose="2/19 紹介資料作成",
    )
```

---

## Notes / Limitations

- **パラメータ名寄せの精度**: 自然言語のパラメータ名は表記ゆれが多い。完全一致ではなく部分一致でマッチングしているため、誤照合が発生する可能性がある。照合結果は目視確認を推奨
- **YAML以外の設定ファイル**: TOMLやINIは別途パーサーが必要（`toml.load()` 等）
- **矛盾の自動解決は行わない**: 矛盾を検出してリスト化するのみ。解決は人間が行う
- **Claude APIコスト**: プレゼンポイント抽出のStep 3はAPI呼び出しあり。大量パラメータでは適宜チャンク分割が必要
- **前提スキル**: このスキルは `manual-ocr-extractor` の出力を入力として想定している

---

**Skill Author**: 部屋子1号提案（subtask_469）
**Last Updated**: 2026-02-18
