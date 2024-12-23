---
title: "人工衛星の位置をWeb地図上にリアルタイム表示する"
emoji: "🛰️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React", "MapLibre"]
published: true
---

FOSS4G Advent Calendar 2024 の 23 日目の記事です。

https://qiita.com/advent-calendar/2024/foss4g

この記事では、人工衛星の位置を Web 地図上にリアルタイム表示する React コンポーネントライブラリ react-sat-map とその実装を紹介します。

https://github.com/sankichi92/react-sat-map

## デモ

はじめに、react-sat-map がどんなものかデモページをご覧ください。
https://sankichi92.github.io/react-sat-map/

以下はスクリーンショットのため動かないですが、実際のデモページでは衛星をクリックしたり、地図を拡大・縮小したりして、衛星の名前や実際の動きを確認できます。
[![デモページのスクリーンショット](/images/2024-12-23_react-sat-map/demo_screenshot.png)](https://sankichi92.github.io/react-sat-map/)

デモで表示しているのは、直近 30 日以内に打上げられた衛星です。スクリーンショットで名前を表示している YODAKA (AE1b) は、私の所属するアークエッジ・スペースが開発した衛星で、12/9 に国際宇宙ステーションから放出されました（宣伝）。

https://arkedgespace.com/news/2024-12-10_ae1b_yodaka_onglaisat

YODAKA では、「短歌ミッション」の上の句を募集中です。

https://up-hanamaki.com/article/239/

## react-sat-map の使い方

react-sat-map のコンポーネントは、npm パッケージとして配布しています（この記事を書く直前に駆け込みでリリースしました）。

https://www.npmjs.com/package/react-sat-map

react-sat-map を使うには以下のパッケージも合わせて必要です。

- [React](https://react.dev/) [^1]
- [react-map-gl](https://visgl.github.io/react-map-gl/)
- [MapLibre GL JS](https://maplibre.org/maplibre-gl-js/docs/)

[^1]: 先日リリースされたばかりの React v19 が必要です。最新版でつくりはじめて、記事を書く直前に v18 で動作確認したら動きませんでした……。

react-sat-map のメインのコンポーネントは `SatelliteMarkers` です。`react-map-gl/maplibre` の `Map` コンポーネントの小要素として使うことができます。`satellites` prop に衛星の名前と TLE（このあと説明します）をもつオブジェクトの配列を渡してください。

以下に、YODAKA の位置を地図上に表示する例を示します。

```jsx
import { Map } from "react-map-gl/maplibre";
import { SatelliteMarkers } from "react-sat-map";
import "maplibre-gl/dist/maplibre-gl.css";
import "react-sat-map/style.css";

const satellites = [
  {
    name: "YODAKA",
    tle: {
      line1:
        "1 62295U 98067XB  24357.77132066  .00177505  00000+0  25824-2 0  9994",
      line2:
        "2 62295  51.6366 100.8197 0012760   9.6944 350.4290 15.54538666  2102",
    },
  },
];

export default function App() {
  return (
    <Map
      style={{ height: "100dvh" }}
      mapStyle="https://tile.openstreetmap.jp/styles/maptiler-basic-ja/style.json"
    >
      <SatelliteMarkers satellites={satellites} />
    </Map>
  );
}
```

ここからは、`SatelliteMarkers` コンポーネントを実装していきます。

## TLE から衛星の位置を計算する

まずは、衛星の現在位置を取得しましょう。

衛星の位置は 2 行軌道要素、TLE (Two-Line Elements) から計算できます。TLE は、人工衛星等の軌道を表すテキストフォーマットのデータで、[CelesTrak](https://celestrak.org/) や [Space-Track](https://www.space-track.org/)（要アカウント）で配布されています。上記の例の `line1` と `line2` がそれぞれ TLE の 1 行目と 2 行目にあたります。<!-- textlint-disable -->

TLE の内容はここでは説明しません[^2]が、衛星は TLE で表現される軌道上で運動を続けるので、SGP4 といったアルゴリズムによって、ある時刻にどこにいるかを求めることができます[^3]。<!-- textlint-enable -->

[^2]: TLE についてはつぎのページがわかりやすいです: https://lizard-tail.com/isana/tle/misc/what_is_tle
[^3]: 実際には外乱により軌道は刻々と変化していくので、TLE は日々更新されます。

[satellite.js](https://github.com/shashwatak/satellite-js) は、SGP4 の JavaScript 実装です。以下のようにして TLE と時刻から緯度経度を計算できます。

```javascript
import * as satellite from "satellite.js";

function twoline2geodetic(line1, line2, date = new Date()) {
  const satrec = satellite.twoline2satrec(line1, line2);
  const { position } = satellite.propagate(satrec, date);

  // 得られた position は ECI 座標系なので、緯度経度の測地座標系に変換する
  const location = satellite.eciToGeodetic(position, satellite.gstime(date));

  // ラジアンから度数法に変換する
  const longitude = satellite.degreesLong(location.longitude);
  const latitude = satellite.degreesLat(location.latitude);

  return { longitude, latitude };
}
```

## Web 地図上に衛星を表示する

衛星の位置が取得できたので、つぎは Web 地図上に表示します。

TLE と時刻を props として受け取り、`react-map-gl/maplibre` の [`Marker`](https://visgl.github.io/react-map-gl/docs/api-reference/marker) を返すコンポーネントをつくります。

```jsx
import { Marker } from "react-map-gl/maplibre";

function SatelliteMarker({ satellite, date = new Date() }) {
  const { longitude, latitude } = twoline2geodetic(
    satellite.tle.line1,
    satellite.tle.line2,
    date,
  );

  return (
    <Marker longitude={longitude} latitude={latitude}>
      🛰️
    </Marker>
  );
}
```

このコンポーネントを `react-map-gl/maplibre` の `Map` コンポーネント以下に配置すれば、衛星が地図上に表示されます。

## アニメーションでリアルタイムに位置を更新する

最後に、衛星の位置を現在時刻に合わせて更新し続けられるようにします。<!-- textlint-disable -->

`SatelliteMarkers` コンポーネントでは、[`window.requestAnimationFrame()`](https://developer.mozilla.org/ja/docs/Web/API/Window/requestAnimationFrame) を使用して、画面更新のたび `SatelliteMarker` に新しい時刻を渡すようにしています。<!-- textlint-enable -->

```jsx
export function SatelliteMarkers({ satellites }) {
  const [date, setDate] = useState(new Date(performance.timeOrigin));

  useEffect(() => {
    const animation = requestAnimationFrame((timestamp) =>
      setDate(new Date(performance.timeOrigin + timestamp));
    );
    return () => cancelAnimationFrame(animation);
  });

  return (
    <>
      {satellites.map((satellite, i) => (
        <SatelliteMarker key={i} satellite={satellite} date={date} />
      ))}
    </>
  );
}
```

---

以上が react-sat-map のエッセンスです。実際のコードはパフォーマンスや安定性、ユーザビリティを考慮してもうすこし複雑になっています。具体的なコードは冒頭の GitHub リポジトリを参照してください。

現在の react-sat-map は衛星の位置をただ表示するだけですが、以下のような既存の OSS にはもっといろいろな機能があります。

- [Gpredict](https://github.com/csete/gpredict)
- [satvis](https://github.com/Flowm/satvis)
- [KeepTrack](https://github.com/thkruz/keeptrack.space)

ただ、2 次元の Web 地図上にシンプルに表示するものが見当たらなかったのでつくってみました。今後余力があれば、上記 OSS に共通するような主要な機能を追加していきたいです。
