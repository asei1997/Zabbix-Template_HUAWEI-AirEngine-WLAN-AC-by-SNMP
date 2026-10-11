# Zabbix-Template_HUAWEI-AirEngine-WLAN-AC-by-SNMP

[日本語](#日本語) | [English](#english)

---

## 日本語

HUAWEI製 AirEngine 無線LANコントローラ(AC)を SNMP で監視するための Zabbix テンプレートです。
AC 全体の接続端末数に加え、AC が管理している AP を自動検出し、AP ごとの状態・端末数・リソース・温度などを監視します。

### 動作要件

- Zabbix 7.0 以降
  - `walk[]` SNMP アイテム(6.4 以降)と、ディスカバリの「Disable lost resources」(7.0 以降)を使用しています。
- AC で SNMP が有効になっていること(SNMPv2c で動作確認)

### 動作確認環境

- Zabbix: 7.0.x
- AC: Huawei AirEngine 9700-M1 V200R025C00SPC100

### 監視内容

#### AC 全体

| アイテム | 内容 |
|---|---|
| `ac-users-total` | AC 全体の接続端末数 |
| `ac-users-2.4` | 2.4GHz の接続端末数 |
| `ac-users-5` | 5GHz の接続端末数 |
| `ac-users-6` | 6GHz の接続端末数 |

グラフ: `AC-Clients`

#### AP ごと(AP Discovery)

| アイテム | 内容 |
|---|---|
| `ap-state` | AP の状態(値マッピングあり) |
| `ap-clients` | AP の接続端末数(合計) |
| `ap-clients-<周波数帯>(radio0〜4)` | 無線インターフェースごとの接続端末数。名前の周波数帯は自動判定 ※AP に存在するものだけ作成 |
| `ap-cpu` / `ap-memory` | CPU / メモリ使用率 |
| `ap-temp` | 温度 |
| `ap-eth-download` / `ap-eth-upload` | 有線インターフェースの通信量 |
| `ap-model` / `ap-name` / `ap-mac` / `ap-id` / `ap-group` | 機種・名前・MAC・AP ID・AP グループ |
| `ap-latitude` / `ap-longitude` | 位置情報 |

トリガー:

- AP の状態が異常値(`{$AP_STATE_CRIT_STATUS}`)になった(正常値 `{$AP_STATE_NORMAL_STATUS}` で復旧)
- AP の温度が上限 `{$AP_TEMP_MAX_CRIT_STATUS}` 以上
- AP の温度が下限 `{$AP_TEMP_MIN_CRIT_STATUS}` 以下

グラフ: `AP-Clients` / `AP-Temperature` / `AP-Traffic`

`AP-Clients-[AP名]` は、AP に存在する radio(radio0〜4)ごとの接続端末数を積み上げ表示し、上端がすべての radio の合計になります。AP の合計端末数(`ap-clients`)も右軸に表示するため、凡例で合計の数値(最新・最小・平均・最大)を確認できます。
radio の構成(`01` / `012` / `0123` / `01234`)ごとにグラフのプロトタイプを用意し、オーバーライドで AP に合うものを 1 つだけ作成します。展開後の名前はどれも `AP-Clients-[AP名]` です。
radio が 1 つも検出されない AP(オフラインなど)や、上記以外の構成の AP ではグラフは新規作成されません(既存のグラフは残ります)。

### 無線インターフェースの自動判定

AP によって搭載している無線インターフェースの数や周波数帯が異なるため、AP Discovery では AP テーブルと radio テーブル(`hwWlanRadioInfoTable` の周波数帯 `hwWlanRadioFreqType` と接続端末数 `hwWlanRadioOnlineStaCnt`)をまとめて `walk[]` で取得し、JavaScript 前処理で AP ごとに次の LLD マクロを生成しています。

| マクロ | 内容 |
|---|---|
| `{#RADIO0}` 〜 `{#RADIO4}` | radio が存在すれば `1`、なければ `0` |
| `{#RADIO0_BAND}` 〜 `{#RADIO4_BAND}` | radio の現在の周波数帯(`2.4GHz` / `5GHz` / `6GHz`、取得できない場合は `unknown`) |
| `{#RADIO_SET}` | 存在する radio ID を連結した文字列(例: `01`、`012`)。グラフの選択に使用 |
| `{#CLIENTS_GRAPH_R01}` など | グラフ名の末尾に付けるマクロ。AP の radio 構成に合うものは空文字、合わないものは `#NOGRAPH` となり、オーバーライドで作成対象から外す |

LLD オーバーライドにより、値が `1` でない radio の端末数アイテムは作成されません(radio0〜4 に対応)。これにより、6GHz 非対応 AP で「No Such Instance currently exists at this OID」エラーが出なくなります。

radio ごとのアイテムのキーは周波数帯ではなく radio ID(`ap-clients-radio0〜4-[...]`)にしています。アイテム名には `{#RADIOn_BAND}` で実際の周波数帯を表示します(例: `ap-clients-5GHz(radio1)-[AP01]`)。
同じ周波数帯の radio が複数ある場合(radio1 と radio2 がどちらも 5GHz など)も、`5GHz(radio1)` / `5GHz(radio2)` のように区別されます。
運用中に radio の周波数帯を変更しても、キーが変わらないため履歴は途切れず、アイテム名は次回のディスカバリ(1 時間以内)で新しい周波数帯に更新されます。

### 検出されなくなった AP / radio の扱い

| 設定 | 値 | 動作 |
|---|---|---|
| Disable lost resources | 1d | 検出されなくなってから 24 時間は取得を継続し、その後無効化。再検出されれば自動で有効に戻る |
| Delete lost resources | 90d | 90 日後に削除(アイテムの履歴保存期間 90d に合わせ、過去データを保持) |

一時的な検出漏れで監視が止まらないよう、無効化まで 24 時間の猶予を設けています。
オフラインの AP は radio テーブルに現れないため、オフラインが 24 時間以上続くと radio ごとの端末数アイテムが無効になります(AP 本体のアイテムは残ります)。

### マクロ

| マクロ | 既定値 | 内容 |
|---|---|---|
| `{$AP_STATE_CRIT_STATUS}` | `4` | AP 状態の異常値(fault) |
| `{$AP_STATE_NORMAL_STATUS}` | `8` | AP 状態の正常値(normal) |
| `{$AP_TEMP_MAX_CRIT_STATUS}` | `65` | AP 温度の上限(℃) |
| `{$AP_TEMP_MIN_CRIT_STATUS}` | `-15` | AP 温度の下限(℃) |

SNMP コミュニティはホストの SNMP インターフェース設定(`{$SNMP_COMMUNITY}` などのホスト/グローバルマクロ)で指定してください。

### 導入方法

1. 「データ収集」→「テンプレート」→「インポート」から YAML をインポートします。
2. AC のホストに SNMP インターフェースを設定し、本テンプレートをリンクします。
3. CPU・メモリ・インターフェースなど AC 本体の基本監視が必要な場合は、Zabbix 公式テンプレート `Huawei VRP by SNMP` を同じホストにあわせてリンクしてください(本テンプレートは親テンプレートを持ちません)。

### 制限事項

- `ac-users-6`(`hwWlanCurAssoc6gStaNum`)は 6GHz に対応したファームウェアの AC でのみ取得できます。対応していない場合、このアイテムは取得不可になります。
- AP の radio 構成が変わった場合(機種交換など)、古いグラフが「検出されなくなった」状態で残っている間は同じ名前の新しいグラフを作成できません。古いグラフを手動で削除してください。
- AP 名が Hex-STRING で返される場合(日本語名など)のデコード処理は、実機では未検証です。
- AP 数が非常に多い環境では `walk[]` がタイムアウトする可能性があります。その場合は AP Discovery のタイムアウトを延ばしてください。

### 既存テンプレートとの関係

本テンプレートは、João Pires 氏の [template_huawei_wireless_ac](https://github.com/zabbix/community-templates/tree/main/Network_Devices/Huawei/template_huawei_wireless_ac) をもとに作成しました。主な違いは次のとおりです。

- radio の存在を自動判定し、存在する radio のアイテムだけを作成
- 6GHz(radio2)に対応
- radio の周波数帯を自動判定してアイテム名に表示
- radio3〜4 に対応し、AP ごとの端末数を radio 別の積み上げグラフで表示
- radio ごとのアイテムのキーを radio ID ベースに変更(キー互換性がないため別テンプレートとして公開)
- 親テンプレートを持たない構成(VRP の基本監視は公式テンプレートを併用)
- 検出されなくなったリソースの無効化・削除ポリシー、低温トリガーを追加

### 作者

Asei SEKIGUCHI (@asei1997)

### ライセンス

MIT License

Copyright (c) 2024 João Pires (www.noshut.pt)  
Copyright (c) 2026 Asei SEKIGUCHI (@asei1997)

---

## English

A Zabbix template for monitoring HUAWEI AirEngine wireless LAN Access Controllers (AC) via SNMP.
In addition to AC-wide client counts, it automatically discovers the APs managed by the AC and monitors per-AP status, clients, resources, temperature and more.

### Requirements

- Zabbix 7.0 or later
  - Uses the `walk[]` SNMP item (6.4+) and "Disable lost resources" in LLD (7.0+).
- SNMP enabled on the AC (tested with SNMPv2c)

### Tested with

- Zabbix: 7.0.x
- AC: Huawei AirEngine 9700-M1 V200R025C00SPC100

### What is monitored

#### AC

| Item | Description |
|---|---|
| `ac-users-total` | Total number of clients on the AC |
| `ac-users-2.4` | Number of 2.4GHz clients |
| `ac-users-5` | Number of 5GHz clients |
| `ac-users-6` | Number of 6GHz clients |

Graph: `AC-Clients`

#### Per AP (AP Discovery)

| Item | Description |
|---|---|
| `ap-state` | AP state (with value mapping) |
| `ap-clients` | Number of clients on the AP (total) |
| `ap-clients-<band>(radio0-4)` | Number of clients per radio; the band in the name is detected automatically — created only for radios present on the AP |
| `ap-cpu` / `ap-memory` | CPU / memory utilization |
| `ap-temp` | Temperature |
| `ap-eth-download` / `ap-eth-upload` | Wired interface traffic |
| `ap-model` / `ap-name` / `ap-mac` / `ap-id` / `ap-group` | Model, name, MAC, AP ID, AP group |
| `ap-latitude` / `ap-longitude` | Location |

Triggers:

- AP state is abnormal (`{$AP_STATE_CRIT_STATUS}`), recovers on normal (`{$AP_STATE_NORMAL_STATUS}`)
- AP temperature is at or above `{$AP_TEMP_MAX_CRIT_STATUS}`
- AP temperature is at or below `{$AP_TEMP_MIN_CRIT_STATUS}`

Graphs: `AP-Clients` / `AP-Temperature` / `AP-Traffic`

`AP-Clients-[AP name]` is a stacked graph of the client count of each radio present on the AP (radio0-4); its top edge is the total of all radios. The AP's total client count (`ap-clients`) is also plotted on the right axis so the total value (last/min/avg/max) appears in the legend.
A graph prototype is provided for each radio set (`01` / `012` / `0123` / `01234`) and an LLD override creates only the one matching the AP. All of them expand to the same name `AP-Clients-[AP name]`.
No new graph is created for APs with no radios discovered (e.g. offline) or with any other radio set (an existing graph is kept).

### Automatic radio detection

Since the number of radios and their bands differ between AP models, AP Discovery retrieves the AP table and the radio table (`hwWlanRadioFreqType` and `hwWlanRadioOnlineStaCnt` of `hwWlanRadioInfoTable`) together with `walk[]`, and a JavaScript preprocessing step generates the following LLD macros for each AP.

| Macro | Description |
|---|---|
| `{#RADIO0}` - `{#RADIO4}` | `1` if the radio exists, otherwise `0` |
| `{#RADIO0_BAND}` - `{#RADIO4_BAND}` | Current band of the radio (`2.4GHz` / `5GHz` / `6GHz`, or `unknown` if not available) |
| `{#RADIO_SET}` | Concatenated IDs of the radios present (e.g. `01`, `012`), used to select the graph |
| `{#CLIENTS_GRAPH_R01}` etc. | Graph name suffixes: empty for the variant matching the AP's radio set, `#NOGRAPH` for the others, which an override excludes |

LLD overrides skip creating client-count items for radios whose value is not `1` (radio0-4 are supported). This prevents "No Such Instance currently exists at this OID" errors on APs without a 6GHz radio.

Per-radio item keys are based on the radio ID (`ap-clients-radio0-4-[...]`) rather than the band. Item names show the actual band via `{#RADIOn_BAND}` (e.g. `ap-clients-5GHz(radio1)-[AP01]`).
Multiple radios on the same band (e.g. radio1 and radio2 both on 5GHz) are distinguished as `5GHz(radio1)` / `5GHz(radio2)`.
Changing a radio's band during operation does not break item history, and the item name is updated to the new band on the next discovery (within 1 hour).

### Lost APs / radios

| Setting | Value | Behavior |
|---|---|---|
| Disable lost resources | 1d | Polling continues for 24 hours after a resource is no longer discovered, then it is disabled. It is re-enabled automatically when rediscovered. |
| Delete lost resources | 90d | Deleted after 90 days (matches the 90d item history so past data is kept). |

The 24-hour grace period prevents a single missed discovery from stopping monitoring.
Offline APs do not appear in the radio table, so if an AP stays offline for more than 24 hours its per-radio client items are disabled (the AP's other items remain).

### Macros

| Macro | Default | Description |
|---|---|---|
| `{$AP_STATE_CRIT_STATUS}` | `4` | Abnormal AP state (fault) |
| `{$AP_STATE_NORMAL_STATUS}` | `8` | Normal AP state (normal) |
| `{$AP_TEMP_MAX_CRIT_STATUS}` | `65` | Upper AP temperature threshold (°C) |
| `{$AP_TEMP_MIN_CRIT_STATUS}` | `-15` | Lower AP temperature threshold (°C) |

Set the SNMP community in the host's SNMP interface (e.g. via a host or global `{$SNMP_COMMUNITY}` macro).

### Installation

1. Import the YAML from "Data collection" → "Templates" → "Import".
2. Configure an SNMP interface on the AC host and link this template.
3. For base monitoring of the AC itself (CPU, memory, interfaces, etc.), also link the official Zabbix template `Huawei VRP by SNMP` to the same host. This template has no parent template.

### Limitations

- `ac-users-6` (`hwWlanCurAssoc6gStaNum`) is only available on AC firmware that supports 6GHz; otherwise the item becomes unsupported.
- If an AP's radio set changes (e.g. model replacement), the new graph cannot be created while the old one remains as a lost resource with the same name. Delete the old graph manually.
- Decoding of AP names returned as Hex-STRING (e.g. non-ASCII names) has not been tested on real devices.
- In environments with a very large number of APs, `walk[]` may time out. Increase the timeout of AP Discovery if needed.

### Relation to the existing template

This template is based on [template_huawei_wireless_ac](https://github.com/zabbix/community-templates/tree/main/Network_Devices/Huawei/template_huawei_wireless_ac) by João Pires. Main differences:

- Detects which radios exist and creates items only for those radios
- Supports 6GHz (radio2)
- Detects each radio's band and shows it in the item name
- Supports radio3-4 and shows per-AP clients as a stacked graph by radio
- Per-radio item keys are based on radio IDs (not key-compatible, hence published as a separate template)
- No parent template (use the official template for base VRP monitoring)
- Adds a lost-resource disable/delete policy and a low-temperature trigger

### Author

Asei SEKIGUCHI (@asei1997)

### License

MIT License

Copyright (c) 2024 João Pires (www.noshut.pt)  
Copyright (c) 2026 Asei SEKIGUCHI (@asei1997)
