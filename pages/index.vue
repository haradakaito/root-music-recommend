<script setup>
import { request } from '../api/fetcher'
import music_json from './music_list.json'

let map = null;
const routeFigures = [];
const markers = [];
const spotMarkers = [];
const start = ref("博多駅");
const goal = ref("PayPayドーム");
const start_data = ref(null);
const goal_data = ref(null);
const error = ref(null);
const infoWindow = ref(null);
const isloading = ref(false);

// アイコンの設定を事前定義
const iconInfo = new mapscript.value.GLMarkerIconInfo({
  icon: "/images/marker.png",
});

const spotIconInfo = new mapscript.value.GLMarkerIconInfo({
  icon: "/images/spot_marker.png",
});

onMounted(() => {
  map = new mapscript.Map("intern01", {
    target: "#map", // 対象のDOM要素またはセレクタを指定してください
    center: new mapscript.value.LatLng(35.667298, 139.714816), // 地図を表示する際中心となる緯度経度を指定してください
    zoomLevel: 15, // ズームレベルを指定してください
  });
});

const searchRoot = async () => {
  isloading.value = true;
  await searchSpot()
  drawRoute()
  searchRootSpot()
  addMarkers()
}

const searchSpot = async () => {
  if (!goal.value || !start.value) {
    alert("キーワードを入力してください")
    return
  }
  try {
    start_data.value = await request('spot', {
      word: start.value,
      limit: 1,
    })
    goal_data.value = await request('spot', {
      word: goal.value,
      limit: 1,
    })
  } catch (e) {
    // 通信結果がエラーのときは何もしない
    error.value = e
    return
  }
}

const drawRoute = async () => {
  // console.log(goal_data.value)
  try {
    const result = await request('shape_car', {
      start: `${start_data.value.items[0].coord.lat},${start_data.value.items[0].coord.lon}`,
      goal: `${goal_data.value.items[0].coord.lat},${goal_data.value.items[0].coord.lon}`,
      condition: 'free_only',
      format: "geojson",
    })
    // 既存のルート線を除去
    routeFigures.forEach((x) => { map.removeGeoJsonFigure(x) });
    // 地図にルート線を追加するときの書き方です（詳しくは地図仕様書を参照）
    const ROUTE_FIGURE_OPTIONS = {
      polyline: {
        inline: {
          color: mapscript.value.Color.fromColorCodeSixHex('32e6c2', 1.0),
          weight: 7,
        },
        outline: {
          color: mapscript.value.Color.fromColorCodeSixHex('32e6c2', 0.75),
          weight: 14,
        }
      }
    }
    const routeFigure = new mapscript.value.GeoJsonFigureCondition(result, ROUTE_FIGURE_OPTIONS);
    map.addGeoJsonFigure(routeFigure);
    // 変数routeFigures（配列）に追加（削除で使うため）
    routeFigures.push(routeFigure);

    const latlng_list = [];
    result.features.forEach((feature) => {
      const [lng1, lat1, lng2, lat2] = feature.bbox;
      latlng_list.push(new mapscript.value.LatLng(lat1, lng1));
      latlng_list.push(new mapscript.value.LatLng(lat2, lng2));
    });
    const rect = mapscript.util.locationsToLatLngRect(latlng_list);
    map.moveBasedOnLatLngRect(rect, true);
    isloading.value = false;
  } catch (error) {
    console.log(error);
    alert("エラーが発生しました");
    return;
  }
};

const searchRootSpot = async () => {
  try {
    const result = await request('route_car', {
      start: `${start_data.value.items[0].coord.lat},${start_data.value.items[0].coord.lon}`,
      goal: `${goal_data.value.items[0].coord.lat},${goal_data.value.items[0].coord.lon}`,
      condition: 'free_only',
      options: 'turn_by_turn',
    })
    const list = []
    result.items[0].sections.filter(spot => spot.type === "point").map(function (value) {
      list.push(value.coord)
    })
    // console.log(list);
    addSpotMarkers(list);
  } catch (error) {
    console.log(error);
    alert("エラーが発生しました");
    return;
  }
}

// 通信結果に基づいて地図にアイコンを置く
const addMarkers = () => {
  // 既存のマーカーを除去
  map.removeGLMarkers(markers);
  markers.length = 0;

  goal_data.value.items.forEach((spot) => {
    // 検索結果各々に対してマーカーを定義
    const glMarker = new mapscript.object.GLMarker({
      position: new mapscript.value.LatLng(spot.coord.lat, spot.coord.lon), // 表示する緯度経度
      info: iconInfo,
    });
    // 変数markers（配列）に追加
    markers.push(glMarker);
  });
  // 変数markersの内容を地図に反映
  map.addGLMarkers(markers);
};

// 通信結果に基づいて地図にアイコンを置く
const addSpotMarkers = (list) => {
  // 既存のマーカーを除去
  map.removeGLMarkers(spotMarkers);
  spotMarkers.length = 0;
  let cnt = 0;
  list.forEach((spot) => {
    if (cnt !== 0 && cnt !== list.length - 1) {
      // 検索結果各々に対してマーカーを定義
      const glMarker = new mapscript.object.GLMarker({
        position: new mapscript.value.LatLng(spot.lat, spot.lon), // 表示する緯度経度
        info: spotIconInfo,
      });
      const clickFunc = () => {
        console.log(`click${spot.lat}, ${spot.lon}`);
        getCategory(spot.lat, spot.lon)
      }
      const out = () => {
        map.removeInfoWindow(infoWindow.value)
        infoWindow.value = null
      }
      // マーカークリック時に動作
      glMarker.addEventListener('mouseover', clickFunc);
      glMarker.addEventListener('mouseout', out);
      // 変数markers（配列）に追加
      spotMarkers.push(glMarker);
    }
    // listのインデックスを管理
    cnt = cnt + 1;
  });
  // 変数markersの内容を地図に反映
  map.addGLMarkers(spotMarkers);
};

// マーカーの周辺情報からカテゴリを取得
const getCategory = async (lat, lon) => {
  try {
    const result = await request('spot/category_code', {
      category: '01.02.03.04.05.06.07.08',
      coord: `${lat},${lon}`,
      radius: 2000
    })
    const category_id = result.items[0].categories.filter(category => category.level === 'middle')[0].code //カテゴリーのID
    // alert(category_id)
    getMusic(category_id, lat, lon)
  } catch (error) {
    console.log(error);
    alert("エラーが発生しました");
    return;
  }
};

// カテゴリに対応する楽曲を検索
const getMusic = async (category_id, lat, lon) => {
  const music_list = music_json.filter(music => music.id === category_id);
  if (music_list.length < 1) {
    addInfoWindow('楽曲が見つかりませんでした', lat, lon);
    return;
  };
  // 検索した楽曲を表示する関数
  // alert(`${music_list[0].artist_name},${music_list[0].music_name}`);
  addInfoWindow(music_list[0].music_name, music_list[0].artist_name, lat, lon)
}

const addInfoWindow = (music_name, artist_name, lat, lon) => {
  //console.log('addInfoWindow');
  //console.log(`${lat}, ${lon}`);
  // 既存のフキダシを除去
  map.removeInfoWindow(infoWindow.value);
  infoWindow.value = null;
  // 検索結果各々に対してフキダシを定義
  const content_text = `<p class="infowindow_title">曲　名：${music_name}</p><p class="infowindow_title">歌手名：${artist_name}</p>`;
  infoWindow.value = new mapscript.object.InfoWindow({
    content: content_text, // フキダシの中に表示したい内容
    position: new mapscript.value.LatLng(lat, lon), // フキダシを表示する緯度経度
    offset: new mapscript.value.Point(0, -12) // マーカーとフキダシが重ならないようにする
  });
  // フキダシを地図に追加
  map.addInfoWindow(infoWindow.value);
}
</script>

<template>
  <section class="logo-area">
    <h1>No Music<span>No Travel</span></h1>
  </section>
  <div class="loading_layout" v-if="isloading"><svg class="loading" xmlns="http://www.w3.org/2000/svg"
      viewBox="0 0 24 24" width="100" height="100" fill="#1468a3">
      <circle cx="12" cy="2" r="2" opacity=".1">
        <animate attributeName="opacity" from="1" to=".1" dur="1s" repeatCount="indefinite" begin="0" />
      </circle>
      <circle transform="rotate(45 12 12)" cx="12" cy="2" r="2" opacity=".1">
        <animate attributeName="opacity" from="1" to=".1" dur="1s" repeatCount="indefinite" begin=".125s" />
      </circle>
      <circle transform="rotate(90 12 12)" cx="12" cy="2" r="2" opacity=".1">
        <animate attributeName="opacity" from="1" to=".1" dur="1s" repeatCount="indefinite" begin=".25s" />
      </circle>
      <circle transform="rotate(135 12 12)" cx="12" cy="2" r="2" opacity=".1">
        <animate attributeName="opacity" from="1" to=".1" dur="1s" repeatCount="indefinite" begin=".375s" />
      </circle>
      <circle transform="rotate(180 12 12)" cx="12" cy="2" r="2" opacity=".1">
        <animate attributeName="opacity" from="1" to=".1" dur="1s" repeatCount="indefinite" begin=".5s" />
      </circle>
      <circle transform="rotate(225 12 12)" cx="12" cy="2" r="2" opacity=".1">
        <animate attributeName="opacity" from="1" to=".1" dur="1s" repeatCount="indefinite" begin=".625s" />
      </circle>
      <circle transform="rotate(270 12 12)" cx="12" cy="2" r="2" opacity=".1">
        <animate attributeName="opacity" from="1" to=".1" dur="1s" repeatCount="indefinite" begin=".75s" />
      </circle>
      <circle transform="rotate(315 12 12)" cx="12" cy="2" r="2" opacity=".1">
        <animate attributeName="opacity" from="1" to=".1" dur="1s" repeatCount="indefinite" begin=".875s" />
      </circle>
    </svg></div>
  <div id="map" style="white-space: pre-wrap;" />
  <section class="float-area">
    <p>出発地</p>
    <input class="input" type="text" v-model="start" placeholder="例：東京タワー" />
    <p>目的地</p>
    <input class="input" type="text" v-model="goal" placeholder="例：横浜赤レンガ倉庫" />
    <button @click="searchRoot()">決定</button>
  </section>

</template>

<style scoped>
#map {
  position: relative;
  width: 100vw;
  height: 100vh;
  margin: auto;
}

.logo-area {
  position: absolute;
  z-index: 1;
  margin-left: 12px;
  font-size: large;
  font-family: Georgia, 'Times New Roman', Times, serif;
}

h1 {
  position: relative;
  padding: 12px;
  color: #fff;
  border-radius: 10px;
  background-image: -webkit-linear-gradient(315deg, #231557 0%, #44107a 29%, #84ffab 67%, #f4fffe 100%);
  background-image: linear-gradient(-225deg, #231557 0%, #44107a 29%, #84ffab 67%, #f4fffe 100%);
}

h1 span {
  display: inline;
  padding-left: 10px;
  color: rgb(46, 46, 46);
}

.loading {
  position: absolute;
  text-align: center;
  top: 45%;
}

.float-area {
  position: absolute;
  top: 10px;
  left: 10px;
  width: 300px;
  height: 150px;
  border-radius: 8px;
  background-color: hwb(217 87% 5%);
  opacity: 0.9;
  margin-top: 80px;
}

.float-area p {
  padding-left: 20px;
  margin-bottom: auto;
}

.float-area input {
  margin-left: 20px;
}

.float-area button {
  margin-left: 10px;
}

.loading_layout {
  position: absolute;
  display: flex;
  justify-content: center;
  width: 100%;
  height: 100%;
  z-index: 10;
  background-color: #fff;
  opacity: 0.8;
}
</style>

<style>
.infowindow_title {
  font-weight: bold;
  font-family: Georgia, 'Times New Roman', Times, serif;
}
</style>
