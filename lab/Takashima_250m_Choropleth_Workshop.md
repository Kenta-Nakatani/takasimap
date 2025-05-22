# 高島市 250mメッシュによる高齢者人口コロプレスマップ作成ワークショップ

## 目的
滋賀県高島市の**250mメッシュ単位の高齢者人口（65歳以上）**を、MapLibre GL JSを使ってインタラクティブなコロプレスマップとして可視化する。

---

## ステップ一覧

### 1. e-Statから250mメッシュポリゴン（Shapefile形式）を取得

1. e-Stat トップページにアクセス → https://www.e-stat.go.jp/
2. トップページの「地図」カードをクリック
3. 「境界データダウンロード」をクリック
4. 「統計メッシュデータ（境界データ）」の表で「2次メッシュ（250m）」を探す
5. ファイル形式「Shapefile」を選択し、「都道府県で絞込みはコチラ」をクリック
6. 「25 滋賀県」をクリックし、高島市をカバーする**4つのZIPファイル**をすべてダウンロード
7. ZIPを解凍して、`.shp`, `.dbf`, `.shx`, `.prj` ファイルを準備

---

### 2. QGISまたはPythonで4つのShapefileを統合（Merge）

#### Python (GeoPandas) での統合例：

```python
import geopandas as gpd
import pandas as pd

gdf1 = gpd.read_file("mesh1.shp")
gdf2 = gpd.read_file("mesh2.shp")
gdf3 = gpd.read_file("mesh3.shp")
gdf4 = gpd.read_file("mesh4.shp")

merged = gpd.GeoDataFrame(pd.concat([gdf1, gdf2, gdf3, gdf4], ignore_index=True))
merged.to_file("takashima_250m_merged.geojson", driver="GeoJSON")
```

---

### 3. 国勢調査の「メッシュ別人口データ」を取得

1. e-Statで「2020 国勢調査 メッシュ別人口」で検索
2. 滋賀県の250mメッシュ別人口CSVをダウンロード
3. 含まれる列の例：
   - メッシュコード（8桁）
   - 総人口
   - 高齢者人口（65歳以上）

---

### 4. メッシュコードでポリゴンと人口データを結合（Join）

#### Pythonでの結合例：

```python
df = pd.read_csv("takashima_population.csv")  # 人口データCSV
gdf = gpd.read_file("takashima_250m_merged.geojson")  # ポリゴン

joined = gdf.merge(df, left_on="MESHID", right_on="メッシュコード")
joined.to_file("takashima_250m_with_population.geojson", driver="GeoJSON")
```

---

### 5. MapLibre GL JSで可視化（コロプレスマップ）

#### index.html

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <title>高島市 高齢者人口マップ</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <script src="https://unpkg.com/maplibre-gl@latest/dist/maplibre-gl.js"></script>
  <link href="https://unpkg.com/maplibre-gl@latest/dist/maplibre-gl.css" rel="stylesheet" />
  <style>
    body { margin: 0; padding: 0; }
    #map { position: absolute; top: 0; bottom: 0; width: 100%; }
  </style>
</head>
<body>
  <div id="map"></div>
  <script src="map.js"></script>
</body>
</html>
```

#### map.js

```javascript
const map = new maplibregl.Map({
  container: 'map',
  style: 'https://demotiles.maplibre.org/style.json',
  center: [136.030, 35.300],
  zoom: 11
});

map.on('load', () => {
  map.addSource('seniors', {
    type: 'geojson',
    data: 'takashima_250m_with_population.geojson'
  });

  map.addLayer({
    id: 'senior-choropleth',
    type: 'fill',
    source: 'seniors',
    paint: {
      'fill-color': [
        'interpolate',
        ['linear'],
        ['get', '高齢者人口'],
        0, '#ffffcc',
        10, '#a1dab4',
        20, '#41b6c4',
        40, '#2c7fb8',
        60, '#253494'
      ],
      'fill-opacity': 0.7,
      'fill-outline-color': '#ffffff'
    }
  });

  map.on('click', 'senior-choropleth', (e) => {
    const props = e.features[0].properties;
    new maplibregl.Popup()
      .setLngLat(e.lngLat)
      .setHTML(`<strong>高齢者人口:</strong> ${props['高齢者人口']}人`)
      .addTo(map);
  });

  map.on('mouseenter', 'senior-choropleth', () => {
    map.getCanvas().style.cursor = 'pointer';
  });

  map.on('mouseleave', 'senior-choropleth', () => {
    map.getCanvas().style.cursor = '';
  });
});
```

---

### 6. 公開・確認

- `index.html`, `map.js`, `takashima_250m_with_population.geojson` を同じフォルダに配置
- VS Code の「Live Server」や GitHub Pages で確認・共有可能

---

