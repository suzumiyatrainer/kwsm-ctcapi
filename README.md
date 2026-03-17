# KwsmCTC2API データ構造・取得ガイド

本プラグインは、GitHubリポジトリの `trains.json` に生の車両データを以下の形式で出力します。

## JSON 構造

実際のデータに基づく構造です。

```json
{
  "last_update": "2026-03-17T08:45:39Z",
  "data": [
    {
      "formation": 1773737095807,
      "id": 570,
      "speed": 0,
      "notch": -8,
      "modelName": "sz5000kw_Tc",
      "isControlCar": true,
      "signal": 0,
      "driver": "",
      "passengers": [null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null],
      "pos": {
        "x": 134.406249422114,
        "y": 78.25,
        "z": 190.5
      },
      "trainStateData": [0, -8, 0, 0, 0, 2, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0],
      "name": "3028R",
      "customButton": [
        {
          "value": 10,
          "text": "快速"
        },
        {
          "value": 2,
          "text": "有田"
        },
        {
          "value": 0,
          "text": "ｶｰﾃﾝなし"
        },
        {
          "value": 2,
          "text": "前運番: 赤"
        },
        {
          "value": 1,
          "text": "後運番: 橙"
        },
        {
          "value": 0,
          "text": "null"
        },
        {
          "value": 1,
          "text": "ｻﾌﾞ灯: 急"
        },
        {
          "value": 0,
          "text": "編成十: 0"
        },
        {
          "value": 1,
          "text": "1"
        },
        {
          "value": 0,
          "text": "ATC-Sz: 有効"
        },
        {
          "value": 0,
          "text": "L/C: L"
        },
        {
          "value": 0,
          "text": "null"
        },
        {
          "value": 0,
          "text": "null"
        },
        {
          "value": 0,
          "text": "null"
        }
      ]
    }
  ]
}
```

### 1. 編成IDごとにグループ化して状態を抽出する

車両単位のデータを、`formation` IDでまとめて「1つの列車」として扱うためのロジックです。

```javascript
async function getTrainData() {
    // GitHub Pages などから JSON を取得
    const res = await fetch("https://suzumiyatrainer.github.io/kawashima-trainlist/trains.json");
    const json = await res.json();
    
    const trains = {};
    
    json.data.forEach(car => {
        const fid = car.formation;
        if (!fid) return;
        
        // 制御車 (isControlCar: true) を優先的に「列車の代表データ」として扱う
        // 制御車がない場合は最初に見つかった車両をベースにする
        if (!trains[fid] || car.isControlCar) {
            trains[fid] = {
                train_number: car.name,            // 列車番号 (例: "3028R")
                type: car.customButton?.[0]?.text || "", // 種別 (例: "快速")
                dest: car.customButton?.[1]?.text || "", // 行先 (例: "有田")
                driver: car.driver,                // 運転士名
                pos: car.pos,                      // 座標 {x, y, z}
                model: car.modelName,              // モデル名
                formation_no: car.customButton?.[8]?.text || "" // 編成番号 (例: "1")
            };
        }
    });

    return trains;
}
```

### 2. カスタムボタン (customButton) の取得

RTMのボタン設定（上からButton0, Button1...）は、`customButton` 配列のインデックス（0, 1...）にそのまま対応しています。

| ボタン | インデックス | 説明 |
| :--- | :--- | :--- |
| **Button0** | `car.customButton[0]` | 有電の場合は「種別」 |
| **Button1** | `car.customButton[1]` | 有電の場合は「行先」 |
| **ButtonN** | `car.customButton[n]` | n番目のボタン |

#### 取得コード例
```javascript
const button0Text = car.customButton?.[0]?.text;  // "快速" など
const button0Value = car.customButton?.[0]?.value; // 10 など
```

### 3. 主要な項目のマッピング表

| 項目 | 実際のJSONキー | 説明・抽出例 |
| :--- | :--- | :--- |
| **列車番号** | `name` | `car.name` |
| **種別 (Button0)** | `customButton[0].text` | `car.customButton[0].text` |
| **行先 (Button1)** | `customButton[1].text` | `car.customButton[1].text` |
| **運転士** | `driver` | `car.driver` |
| **座標** | `pos` | `car.pos.x`, `car.pos.y`, `car.pos.z` |
| **編成番号** | `customButton[8].text` | サンプルの場合 index 8 に1桁目が格納 |

## メリット
- プラグイン側でデータを削らないため、将来的に `customButton` の 3番目以降を利用したくなった場合も、プラグインの改修なしでフロントエンド側のみの変更で対応可能です。
- Java側での処理が最小限（ただの文字列転送）になるため、非常に軽量に動作します。
